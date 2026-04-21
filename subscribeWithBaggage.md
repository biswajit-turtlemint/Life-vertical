# `subscribeWithBaggage` Design Note

## Purpose

This note explains the only parts we really need for detached multitenant
reactive work:

- why raw detached `subscribe()` is risky in this codebase
- why interceptor-style or `subscribe()` override ideas are not a safe solution
- why `subscribeWithBaggage(...)` is the chosen approach
- which `BaggageUtil` methods matter for that flow

This document is intentionally focused. It is not a full Reactor guide.

## The Actual Problem

Our multitenant DB routing depends on broker baggage being present at the moment
Mongo routing happens.

The normal request flow is:

`x-broker -> MultiTenantWebFilter -> baggage -> DAO/factory routing -> Mongo selection`

This works fine as long as execution stays inside the same returned reactive
chain.

The problem starts when code manually detaches work like this:

```java
somePublisher.subscribe(...);
```

At that point:

- a new subscription boundary is created
- the original request-scoped baggage may not be present where the detached work runs
- Mongo routing may fall back to the default tenant

So the problem is not `Mono` or `Flux` themselves. The problem is detached
subscription that needs tenant-aware DB work without explicitly reseeding the
tenant baggage.

## What We Need the Solution To Do

For detached async work, the solution must do these steps every time:

1. capture the current broker from baggage before detaching
2. open a baggage scope for the detached publisher
3. run the publisher inside that scope
4. close the scope properly
5. log enough information to debug which detached operation ran with which broker

That is exactly what `subscribeWithBaggage(...)` does.

## Why Overriding `subscribe()` Will Not Work

This idea sounds attractive at first: "why not make normal `subscribe()`
automatically behave like `subscribeWithBaggage()`?"

In practice, it does not fit Java + Reactor safely.

### 1. `Mono` and `Flux` are Reactor library types

We do not own `Mono` or `Flux`. They come from Reactor.

That means:

- we cannot modify their source
- we cannot add new instance methods to them
- we cannot safely redefine their existing `subscribe()` behavior

So there is no clean way to make this compile:

```java
publisher.subscribeWithBaggage(...)
```

unless we create our own wrapper utility around the publisher.

### 2. Java does not have extension methods

In some languages, you can add methods to existing types externally. Java does
not support that model.

So we cannot "attach" a custom method to `Mono` or `Flux` the way Kotlin
extension functions would allow.

That is why the correct Java shape is:

```java
baggageUtil.subscribeWithBaggage(...);
```

instead of:

```java
mono.subscribeWithBaggage(...);
```

### 3. Subclassing Reactor types is not a good solution

In theory, someone might think about creating custom types like:

```java
class TenantAwareMono<T> extends Mono<T> { ... }
```

This is not a good design here.

Why:

- Reactor operators return Reactor-managed publishers, not our custom subclass
- we would lose the custom type as soon as normal Reactor chaining continues
- it becomes fragile and difficult to maintain
- it fights the framework instead of composing with it

So subclassing does not give us a practical codebase-wide solution.

## Why Spring Interceptors or AOP Will Not Work Reliably

This is the other common idea: "can we intercept `subscribe()` somehow?"

Not reliably.

### 1. `subscribe()` is usually called inside method bodies, not on Spring beans

Spring AOP works best when a Spring bean method is invoked through the Spring
proxy boundary.

Example of what AOP is good at:

```java
someService.doWork();
```

Example of what our problem usually looks like:

```java
repository.save(entity).subscribe();
```

That `subscribe()` call is just a local method invocation on an object inside a
method body. Spring AOP does not reliably wrap arbitrary internal local calls
like that.

So even if we add an aspect, it will not give us dependable coverage over every
detached `subscribe()` call site.

### 2. We need the broker captured at the exact detach point

Even if an interceptor could see a `subscribe()` call, it still has a deeper
problem:

- the right broker value must be captured before the chain detaches
- that capture must happen at the exact place where detached execution is being initiated

This is not just "run some generic advice before subscribe."
It is:

- read current baggage now
- preserve it
- reopen it around the detached publisher

That is business-critical contextual behavior, not just generic interception.

So even a clever interceptor would still need logic equivalent to
`subscribeWithBaggage(...)`, and it would be harder to reason about and debug.

### 3. AOP would be too implicit for a high-risk flow

Tenant routing is not something we want hidden behind surprising magic.

If detached async code touches multitenant DB routing, the code should make that
decision visible:

```java
baggageUtil.subscribeWithBaggage(...)
```

That line is explicit. A new developer can see:

- this is detached work
- this is multitenancy-sensitive
- baggage is being preserved intentionally

That clarity is valuable in reviews and debugging.

## Why Reactor Global Hooks Are Also Not the Right Answer

Another possible idea is to use Reactor hooks like `Hooks.onEachOperator(...)`
or similar global instrumentation.

This is also not a good fit.

Why:

- hooks are global
- they affect every reactive chain in the application
- they are much broader than the actual problem we are solving
- they make behavior harder to reason about
- they can create subtle side effects in unrelated flows

Most importantly, our need is narrow:

- only detached subscriptions that require tenant-aware DB routing

A global Reactor hook is too large a hammer for that.

## Chosen Approach

The chosen approach is explicit helper-based composition:

- request entry-point seeding remains in the filter
- DB routing remains baggage-based
- detached work uses `subscribeWithBaggage(...)`

This gives us:

- safe broker capture
- explicit behavior at the risky boundary
- reusable code
- consistent logs
- no hidden framework magic

## The `BaggageUtil` Methods That Matter

For this detached subscribe solution, these methods are the ones that matter.

### `getBaggage(String baggageName)`

Purpose:

- reads the current baggage value
- falls back to the default tenant if missing

Why it matters:

- `subscribeWithBaggage(...)` uses this to capture the broker before detaching

### `withBaggage(String baggageName, String value, Mono<T> mono)`

Purpose:

- wraps a `Mono` so it runs inside a baggage scope

Why it matters:

- this is the building block used internally by `subscribeWithBaggage(...)`

### `withBaggage(String baggageName, String value, Flux<T> flux)`

Purpose:

- same as above, for `Flux`

### `subscribeWithBaggage(String operationName, String context, Mono<T> mono)`

Purpose:

- compact detached `Mono` subscription with helper-managed logs and baggage scope

Use this when:

- the detached work does not need custom success/error behavior

### `subscribeWithBaggage(String operationName, String context, Mono<T> mono, Consumer<T> onSuccess, Consumer<Throwable> onError)`

Purpose:

- detached `Mono` subscription with custom callbacks

Use this when:

- the caller needs success or error side effects

### `subscribeWithBaggage(String operationName, String context, Flux<T> flux)`

Purpose:

- compact detached `Flux` subscription with helper-managed logs and baggage scope

Use this when:

- the detached `Flux` does not need custom `onNext` or completion logic

### `subscribeWithBaggage(String operationName, String context, Flux<T> flux, Consumer<T> onNext, Consumer<Throwable> onError, Runnable onComplete)`

Purpose:

- detached `Flux` subscription with custom callbacks

Use this when:

- the caller needs item handling or custom completion logic

## Why the Public API Was Kept Small

We intentionally trimmed the overloads down to the 4 methods above.

Why:

- the API should guide developers toward one clear pattern
- too many convenience overloads make the utility harder to understand
- the `operationName + context` form gives the best logs for debugging
- one compact overload and one callback overload per publisher type is enough

This keeps the standard usage predictable.

## Example of the Preferred Pattern

Compact `Mono`:

```java
baggageUtil.subscribeWithBaggage(
        "LifeQuotes.RequestHistory",
        "referenceId=" + r.getReferenceId(),
        savePremiumRequestHistory(r));
```

Compact `Flux`:

```java
baggageUtil.subscribeWithBaggage(
        "LifeQuotes.RiderPrices",
        "referenceId=" + serviceRequest.getReferenceId() + ", planCode=" + serviceRequest.getPlanCode(),
        lifeRiderPricesAsyncService.generateAndSaveRiderPrices(
                JsonUtils.fromJsonObject(serviceRequest, QuotationRequest.class))
                .subscribeOn(Schedulers.parallel()));
```

## Team Rule

The rule we want developers to follow is:

- if you are returning a reactive chain, do not manually call `subscribe()`
- if you are detaching tenant-aware async work, use `subscribeWithBaggage(...)`

That rule is much easier to enforce in reviews, linting, or CI than any attempt
to transparently change how Reactor itself behaves.

## Summary

Interceptor-style or `subscribe()` override ideas do not work well here because:

- we do not own Reactor types
- Java cannot extend those APIs the way we need
- AOP does not reliably cover local `subscribe()` calls
- global Reactor hooks are too broad and too implicit
- tenant capture must happen explicitly at the detach point

So the right solution is not to hide detached subscription behavior.
It is to make it explicit, small, reusable, and easy to review through
`subscribeWithBaggage(...)`.
