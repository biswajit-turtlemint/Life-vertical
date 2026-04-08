_# Multitenancy With Baggage and Detached `subscribe()`

This note explains how dynamic broker-based DB routing should work in `minterprise`, what needs to change, and why detached `.subscribe()` calls need special handling.

## Current Routing Model

The Mongo routing path is already baggage-driven:

- `MultiTenantWebFilter` seeds request-scoped routing information
- `BaggageUtil` reads the current baggage value
- `AbstractReactiveMongoDAO` and `MultiTenantMongoFactory` use that baggage value to select the Mongo template / database factory

Relevant files:

- `application/src/main/java/com/application/filter/MultiTenantWebFilter.java`
- `library/src/main/java/com/leads/utils/BaggageUtil.java`
- `library/src/main/java/com/library/dao/mongodb/AbstractReactiveMongoDAO.java`
- `library/src/main/java/com/library/multitenant/mongodb/MultiTenantMongoFactory.java`

Today the routing is still effectively hardcoded because `BaggageUtil.getTenantBaggage(...)` returns `TenantConstant.DEFAULT_TENANT`.

## Target Behavior

We want the routing key to come from `x-broker` on HTTP requests.

End-to-end flow:

1. Request comes in with `x-broker`
2. `MultiTenantWebFilter` extracts `x-broker`
3. Filter stores that value in `TenantConstant.TENANT_DB_BAGGAGE`
4. `BaggageUtil` reads that baggage value
5. DB layer reads the baggage value through `BaggageUtil`
6. Correct `ReactiveMongoTemplate` / `ReactiveMongoDatabaseFactory` is chosen

Even though the baggage key is called `TENANT_DB_BAGGAGE`, it can still carry broker if broker is the routing key for DB selection.

## What To Change

### 1. Fix `MultiTenantWebFilter`

Current issue:

- it does not read `x-broker`
- it hardcodes `DEFAULT_TENANT`

Suggested shape:

```java
package com.application.filter;

import com.application.exceptions.BrokerNotFoundException;
import com.leads.utils.BaggageUtil;
import com.library.constant.TenantConstant;
import io.micrometer.tracing.Tracer;
import lombok.Setter;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.Ordered;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.server.ServerWebExchange;
import org.springframework.web.server.WebFilter;
import org.springframework.web.server.WebFilterChain;
import reactor.core.publisher.Mono;

import java.util.Objects;

import static com.library.constant.FieldNameConstant.X_TRACE_ID;

@Component
@Slf4j
public class MultiTenantWebFilter implements WebFilter, Ordered {

    private final Tracer tracer;
    private final BaggageUtil baggageUtil;

    @Setter
    private int order;

    public MultiTenantWebFilter(Tracer tracer, BaggageUtil baggageUtil) {
        this.tracer = tracer;
        this.baggageUtil = baggageUtil;
    }

    @Override
    public int getOrder() {
        return order;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String broker = extractBroker(exchange);
        String traceId = extractTraceId();

        if (StringUtils.hasText(traceId)) {
            exchange.getResponse().getHeaders().add("X-Server-Trace-Id", traceId);
        }

        return baggageUtil.withBaggage(
                TenantConstant.TENANT_DB_BAGGAGE,
                broker,
                baggageUtil.withBaggage(X_TRACE_ID, traceId, chain.filter(exchange))
        );
    }

    private String extractBroker(ServerWebExchange exchange) {
        String broker = exchange.getRequest().getHeaders().getFirst(TenantConstant.BROKER_HEADER);
        if (!StringUtils.hasText(broker)) {
            throw new BrokerNotFoundException("Missing x-broker header");
        }
        return broker;
    }

    private String extractTraceId() {
        try {
            return Objects.requireNonNull(
                    tracer.currentTraceContext().context(),
                    "No current trace context available"
            ).traceId();
        } catch (NullPointerException e) {
            log.error("Failed to extract trace ID: No active trace context found.", e);
            return null;
        }
    }
}
```

### 2. Fix `BaggageUtil`

Current issue:

- `getTenantBaggage(...)` returns `TenantConstant.DEFAULT_TENANT`
- so DB routing never uses the real request-scoped broker

Suggested shape:

```java
package com.leads.utils;

import com.library.constant.TenantConstant;
import io.micrometer.tracing.Baggage;
import io.micrometer.tracing.BaggageInScope;
import io.micrometer.tracing.Span;
import io.micrometer.tracing.Tracer;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@Component
@Slf4j
public class BaggageUtil {

    private final Tracer tracer;

    public BaggageUtil(Tracer tracer) {
        this.tracer = tracer;
    }

    public BaggageInScope createBaggageInScope(String baggageName, String value) {
        return tracer.createBaggageInScope(baggageName, value);
    }

    public String getTenantBaggage(String baggageName) {
        return getBaggage(baggageName);
    }

    public String getBaggage(String baggageName) {
        if (tracer == null) {
            return TenantConstant.DEFAULT_TENANT;
        }

        Baggage baggage = tracer.getBaggage(baggageName);
        String value = baggage != null ? baggage.get() : null;

        return StringUtils.hasText(value) ? value : TenantConstant.DEFAULT_TENANT;
    }

    public <T> Mono<T> withBaggage(String baggageName, String value, Mono<T> mono) {
        if (!StringUtils.hasText(value)) {
            return mono;
        }

        return Mono.using(
                () -> tracer.createBaggageInScope(baggageName, value),
                ignored -> Mono.defer(() -> mono),
                BaggageInScope::close
        );
    }

    public <T> Flux<T> withBaggage(String baggageName, String value, Flux<T> flux) {
        if (!StringUtils.hasText(value)) {
            return flux;
        }

        return Flux.using(
                () -> tracer.createBaggageInScope(baggageName, value),
                ignored -> Flux.defer(() -> flux),
                BaggageInScope::close
        );
    }

    public Span createSpan(String spanName) {
        return tracer.nextSpan().name(spanName).start();
    }

    public Tracer.SpanInScope setSpan(Span span) {
        return tracer.withSpan(span);
    }
}
```

### 3. Tighten the DB Lookup

The DB layer is already reading from baggage. Only a stronger null / missing-template check is useful.

Suggested shape:

```java
protected ReactiveMongoTemplate getMongoTemplate() {
    String tenant = baggageUtil.getTenantBaggage(TenantConstant.TENANT_DB_BAGGAGE);
    Assert.hasText(tenant, "Unable to get tenant to determine mongoTemplate!");

    ReactiveMongoTemplate template = reactiveMongoTemplateMap.get(tenant);
    Assert.notNull(template, "No mongoTemplate configured for tenant: " + tenant);

    return template;
}
```

## What Happens To `TenantConstant.DEFAULT_TENANT`

Not every `DEFAULT_TENANT` usage should be replaced.

There are 2 categories.

### Request-scoped flow

These should become baggage-driven:

- `MultiTenantWebFilter`
- `BaggageUtil.getTenantBaggage(...)`
- DB routing classes like `AbstractReactiveMongoDAO` and `MultiTenantMongoFactory`

In this path, the broker is:

1. extracted from request header
2. written into baggage
3. read from baggage later

### Non-request entry points

These should not try to "read from request baggage" because there is no HTTP request there:

- schedulers
- Kafka consumers
- report jobs
- standalone async jobs

Examples:

- `reports/src/main/java/com/reports/utils/ReportSchedulerUtil.java`
- `library/src/main/java/com/library/services/kafka/BaseKafkaConsumer.java`

For these flows, you must seed baggage explicitly from:

- message metadata
- scheduler config
- explicit tenant argument
- fallback default, if that is the intended design

So the rule is:

- HTTP request path -> extract broker and put it in baggage
- DB path -> read broker from baggage
- background path -> explicitly create baggage before DB work

## Why Detached `.subscribe()` Is a Problem

The problem is not only "another thread".

The bigger issue is that:

```java
repo.save(entity).subscribe();
```

creates a detached subscription.

That detached subscription is no longer guaranteed to run inside the same request-scoped baggage that started in `MultiTenantWebFilter`.

That means when DB routing happens later:

- baggage may be missing, so routing falls back to `DEFAULT_TENANT`
- or if baggage scope is not managed correctly, a reused thread can carry stale baggage from another flow

So yes, with dynamic broker support, it is possible to route to the wrong broker if detached work is not re-seeded correctly.

More precisely:

- the most common failure mode is fallback to default tenant / broker
- stale broker bleed is also possible if baggage scopes are opened and never closed

## Why Approach 1 Is the Right Solution

We want to keep `.subscribe()` for detached async work.

The 3 approaches were:

1. `doFirst(() -> createBaggageInScope(...)).subscribe()`
2. `contextWrite(ctx -> ctx.put(...)).subscribe()`
3. `subscribeOn(...).doFirst(() -> createBaggageInScope(...)).subscribe()`

### Why approach 2 is not enough

This codebase routes DB from tracer baggage, not Reactor `Context`.

So:

```java
repo.save(entity)
    .contextWrite(ctx -> ctx.put(TenantConstant.TENANT_DB_BAGGAGE, tenant))
    .subscribe();
```

does not fix DB routing unless DAO code is redesigned to read Reactor `Context`.

Today DAO reads:

- `baggageUtil.getTenantBaggage(...)`

not:

- `ContextView`

### Why approach 3 is not the real fix

`subscribeOn(...)` only changes scheduler.

It does not solve tenant propagation by itself.

So approach 3 is basically approach 1 plus an extra scheduler hop.

### Why approach 1 is correct

Approach 1 matches the actual routing model of this repo:

- DB lookup reads baggage
- so detached work must recreate baggage before the detached subscription starts

That is why approach 1 is the correct idea.

However, raw approach 1 has one problem:

```java
repo.save(entity)
    .doFirst(() -> baggageUtil.createBaggageInScope(TenantConstant.TENANT_DB_BAGGAGE, broker))
    .subscribe();
```

The baggage scope is never closed here.

So the safe version is to use a helper that opens and closes the baggage scope.

## Recommended Pattern For Detached `.subscribe()`

Capture the current broker before detaching:

```java
String broker = baggageUtil.getBaggage(TenantConstant.TENANT_DB_BAGGAGE);
```

Then wrap the detached work:

```java
baggageUtil.withBaggage(
        TenantConstant.TENANT_DB_BAGGAGE,
        broker,
        repo.save(entity)
).subscribe();
```

If needed:

```java
baggageUtil.withBaggage(
        TenantConstant.TENANT_DB_BAGGAGE,
        broker,
        someService.doSomething()
).subscribe();
```

If a scheduler switch is separately needed:

```java
baggageUtil.withBaggage(
        TenantConstant.TENANT_DB_BAGGAGE,
        broker,
        repo.save(entity).subscribeOn(Schedulers.boundedElastic())
).subscribe();
```

In that case, multitenancy is still solved by baggage seeding, not by `subscribeOn(...)`.

## Example Replacements

### Before

```java
issuanceResultRepository.save(issuanceResult)
        .log()
        .subscribeOn(Schedulers.boundedElastic())
        .subscribe();
```

### After

```java
String broker = baggageUtil.getBaggage(TenantConstant.TENANT_DB_BAGGAGE);

baggageUtil.withBaggage(
        TenantConstant.TENANT_DB_BAGGAGE,
        broker,
        issuanceResultRepository.save(issuanceResult)
                .log()
                .subscribeOn(Schedulers.boundedElastic())
).subscribe();
```

### Before

```java
defaultAggregatorService.saveAggregatorPremiumResult(quotationResponse).subscribe();
```

### After

```java
String broker = baggageUtil.getBaggage(TenantConstant.TENANT_DB_BAGGAGE);

baggageUtil.withBaggage(
        TenantConstant.TENANT_DB_BAGGAGE,
        broker,
        defaultAggregatorService.saveAggregatorPremiumResult(quotationResponse)
).subscribe();
```

## Practical Rule

Use this rule everywhere:

- same reactive chain as the request -> baggage is already available
- detached `.subscribe()` -> explicitly recreate baggage before subscribing
- scheduler / Kafka / cron entry point -> explicitly seed baggage because there is no request context

## Summary

For dynamic broker routing:

- `MultiTenantWebFilter` should extract `x-broker`
- baggage should carry that broker under `TENANT_DB_BAGGAGE`
- `BaggageUtil` should read baggage instead of hardcoding `DEFAULT_TENANT`
- DB routing code can keep using `BaggageUtil`
- detached `.subscribe()` blocks must re-seed baggage before DB work starts

So yes, the subscribe problem is real, and the correct fix for this codebase is approach 1 conceptually, implemented safely via a helper like `withBaggage(...)`._
