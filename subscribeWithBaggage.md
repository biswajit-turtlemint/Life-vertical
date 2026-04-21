# `subscribeWithBaggage` and `BaggageUtil` Reference

## Purpose

This document explains the utility methods currently available in
[`BaggageUtil.java`](/Users/biswajitrout/companyProjects/minterprise/library/src/main/java/com/leads/utils/BaggageUtil.java),
especially the `subscribeWithBaggage(...)` overloads that were added to reduce
repeated `withBaggage(...).subscribe(...)` boilerplate.

The goal is simple:

- capture the current broker from baggage
- re-seed baggage for detached async execution
- keep Mongo multitenant routing correct
- reduce repeated code at `subscribe()` call sites

## Important Constraint

We cannot override Reactor's built-in `Mono.subscribe()` or `Flux.subscribe()`.

Why:

- `Mono` and `Flux` are library classes from Reactor
- Java does not support extension methods
- Reactor global hooks are too broad for this use case
- Spring AOP cannot safely intercept every local `publisher.subscribe(...)` call

Because of that, the safe pattern is:

- normal returned chains continue to use Reactor normally
- detached fire-and-forget work uses `subscribeWithBaggage(...)`

## What `BaggageUtil` Contains Today

`BaggageUtil` currently has four groups of methods:

1. baggage creation and lookup helpers
2. `withBaggage(...)` wrappers for `Mono` and `Flux`
3. `subscribeWithBaggage(...)` overloads for compact detached subscriptions
4. span helpers

---

## 1. Baggage Creation and Lookup Helpers

### `createBaggageInScope(String baggageName, String tenant)`

What it does:

- opens a Micrometer baggage scope for the given baggage key and value

When to use:

- only when code explicitly needs a raw `BaggageInScope`
- most business code should prefer `withBaggage(...)`

### `getTenantBaggage(String baggageName)`

What it does:

- convenience wrapper over `getBaggage(...)`

Why it exists:

- older call sites may use a tenant-specific method name
- behavior is the same as `getBaggage(...)`

### `addBaggage(String baggageName, String value)`

What it does:

- directly writes a baggage value into scope using the tracer
- logs the added baggage entry

When to use:

- only if code must imperatively seed baggage right now
- for reactive publishers, `withBaggage(...)` is usually cleaner

### `getBaggage(String baggageName)`

What it does:

- reads the current baggage value from the tracer
- falls back to `TenantConstant.DEFAULT_TENANT` if baggage is missing or blank
- logs the resolved value

Why it matters:

- this is the source of truth used by the multitenant DB routing code
- `subscribeWithBaggage(...)` uses this method to capture the broker before detaching

Example log:

```text
[TenantRouting][Baggage] baggageName=tenant resolvedValue=axisbank
```

---

## 2. `withBaggage(...)` Wrappers

These methods do not subscribe by themselves. They only return a wrapped
publisher that opens baggage before execution and closes it afterward.

### `withBaggage(String baggageName, String value, Mono<T> mono)`

What it does:

- opens baggage scope before the `Mono` runs
- executes the original `Mono`
- closes baggage scope at the end

Use this when:

- you already control the full reactive chain
- you want to return the wrapped `Mono`
- you do not want to subscribe immediately

### `withBaggage(String baggageName, String value, Flux<T> flux)`

What it does:

- same behavior as the `Mono` version
- applies to a `Flux`

Use this when:

- you want a baggage-aware `Flux`
- the subscription will happen elsewhere

Example logs:

```text
[TenantRouting][Baggage] opening scope name=tenant value=axisbank
[TenantRouting][Baggage] closing scope name=tenant value=axisbank
```

---

## 3. `subscribeWithBaggage(...)` Overloads

These are the compact methods added to reduce repeated code at detached
subscription sites.

All overloads follow the same base flow:

1. log scheduling
2. capture current broker from baggage
3. wrap the publisher with `withBaggage(TENANT_DB_BAGGAGE, broker, ...)`
4. subscribe
5. log success, empty completion, next item, failure, and/or completion depending on publisher type

### Internal Helpers Used by All Overloads

#### `captureCurrentBroker()`

What it does:

- reads broker from `TenantConstant.TENANT_DB_BAGGAGE`
- falls back through `getBaggage(...)`
- logs the captured broker

Example log:

```text
[subscribeWithBaggage] captured broker=axisbank
```

#### `buildOperationDetails(String operationName, String context)`

What it does:

- combines operation name and optional context into one log string

Examples:

- `LifeQuotes.RequestHistory`
- `LifeQuotes.RequestHistory | referenceId=AHWSIYH5E69`

### A. Kept `Mono` Overloads

#### `subscribeWithBaggage(String operationName, String context, Mono<T> mono)`

What it does:

- same as above, but also includes a context string in logs
- uses default no-op success and error callbacks

Use this when:

- you want compact call sites
- helper-owned logs are enough

Current real usage:

```java
baggageUtil.subscribeWithBaggage(
        "LifeQuotes.RequestHistory",
        "referenceId=" + r.getReferenceId(),
        savePremiumRequestHistory(r));
```

### B. `Mono` Overload With Custom Callbacks

#### `subscribeWithBaggage(String operationName, String context, Mono<T> mono, Consumer<T> onSuccess, Consumer<Throwable> onError)`

What it does:

- full `Mono` overload
- caller supplies success and error handlers
- helper still handles baggage capture and standard logs

Use this when:

- you want compact code plus custom side effects

Mono log behavior:

- when the `Mono` emits a value:
  - logs `completed ... with broker=...`
- when the `Mono` completes without emitting:
  - logs `completed empty ... with broker=...`
- when it fails:
  - logs `failed ... with broker=... cause=...`

### C. Kept `Flux` Overloads

#### `subscribeWithBaggage(String operationName, String context, Flux<T> flux)`

What it does:

- same as above, with operation name and context
- uses default no-op `onNext`, `onError`, and `onComplete`

Current real usage:

```java
baggageUtil.subscribeWithBaggage(
        "LifeQuotes.RiderPrices",
        "referenceId=" + serviceRequest.getReferenceId() + ", planCode=" + serviceRequest.getPlanCode(),
        lifeRiderPricesAsyncService.generateAndSaveRiderPrices(
                JsonUtils.fromJsonObject(serviceRequest, QuotationRequest.class))
                .subscribeOn(Schedulers.parallel()));
```

### D. `Flux` Overload With Custom Callbacks

#### `subscribeWithBaggage(String operationName, String context, Flux<T> flux, Consumer<T> onNext, Consumer<Throwable> onError, Runnable onComplete)`

What it does:

- full `Flux` overload
- operation name and context control logs
- caller provides all custom callbacks

Flux log behavior:

- for every emitted item:
  - logs `next ... with broker=...`
- on failure:
  - logs `failed ... with broker=... cause=...`
- on completion:
  - logs `completed ... with broker=...`

---

## 4. Span Helpers

### `createSpan(String spanName)`

What it does:

- creates and starts a tracing span

### `setSpan(Span span)`

What it does:

- puts the given span into scope

These are unrelated to the detached `subscribe()` multitenancy fix, but they
are still part of `BaggageUtil`.

---

## How to Choose the Right Overload

### Use `withBaggage(...)` when:

- you are returning a `Mono` or `Flux`
- the chain is still part of normal reactive flow
- you do not want to subscribe immediately

### Use `subscribeWithBaggage(...)` when:

- you are manually detaching work with `subscribe()`
- the detached work needs broker-aware DB routing
- you want the helper to capture broker and re-seed baggage

### Use the compact overloads when:

- helper logs are enough
- you only need operation name and maybe context

Best compact pattern:

```java
baggageUtil.subscribeWithBaggage(
        "SomeOperation",
        "referenceId=" + referenceId,
        monoOrFlux);
```

### Use the callback overloads when:

- you need custom `onSuccess`, `onNext`, `onError`, or `onComplete` behavior

---

## What Logs to Search

### Common helper logs

```text
[subscribeWithBaggage] scheduling
[subscribeWithBaggage] captured broker=
[subscribeWithBaggage] completed
[subscribeWithBaggage] completed empty
[subscribeWithBaggage] next
[subscribeWithBaggage] failed
```

### Baggage scope logs

```text
[TenantRouting][Baggage] opening scope name=tenant
[TenantRouting][Baggage] closing scope name=tenant
```

### Baggage resolution log

```text
[TenantRouting][Baggage] baggageName=tenant resolvedValue=
```

---

## Recommended Team Rule

- If you are returning a reactive chain, do not manually call `subscribe()`.
- If you must manually detach work and that work can touch multitenant DB routing, use `subscribeWithBaggage(...)`.
- Prefer the compact `operationName + context + publisher` overload unless custom callbacks are genuinely needed.

## Summary

The added `BaggageUtil` methods are meant to solve two separate problems:

- `withBaggage(...)` solves publisher wrapping
- `subscribeWithBaggage(...)` solves detached subscription boilerplate

The overloads now exist in the minimal useful set:

- named + context overloads for compact real-world usage
- named + context + callback overloads for advanced cases

That gives us a small API surface that can be reused across the codebase without
repeating broker capture and baggage re-seeding logic at every detached
`subscribe()` call site.
