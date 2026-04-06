# Ninja vs Minterprise Multitenancy

This document explains:

- how multitenancy is handled in `ninja-service` on `jk-develop`
- how `minterprise` is currently wired
- where `minterprise` is effectively hardcoded today
- how the two approaches are different
- what should ideally be used in `minterprise` right now

This is written from the current code, not from intended architecture comments.

## 1. Short answer

`ninja-service` on `jk-develop` is effectively:

- broker-based DB routing
- session-based request context
- tenant stored as extra request/session metadata

`minterprise` is currently designed as:

- reactive baggage-based DB routing
- tenant-based template lookup

But in practice, `minterprise` is mostly doing:

- always use `axisbank` as the routing key

because the runtime tenant resolution is hardcoded.

If you ask what `minterprise` should ideally use right now, the clean answer is:

- do not copy Ninja session routing
- keep `broker` and `tenant` as separate concepts
- introduce one explicit DB routing key
- for the current setup, derive that DB routing key from `broker`
- keep `tenant` as business context, not as the physical Mongo routing key yet

That is the safest model because the current datasource map in `minterprise` is broker-shaped, not true tenant-shaped.

## 2. Terms used in this document

There are 3 different concepts that are getting mixed today:

### Broker

This is the broker/brand/platform identity.

Examples:

- `axisbank`
- `turtlemint`
- `jkbank`

This usually controls:

- broker config
- insurer config
- host/domain mapping
- templates / callback headers / external integrations
- in Ninja, even DB selection

### Tenant

This is the business tenant or partner tenant that the request belongs to.

Examples could be:

- `axisbank`
- `flipkart`
- `turtlefin`
- `credilio`

This is often used for:

- business rules
- lead handling
- feature behavior
- downstream context

This is not automatically the same thing as the Mongo database key.

### DB routing key

This is the actual key used to choose which Mongo template / DB connection should be used.

This should be a separate concept in code, even if today it ends up equal to broker.

That is the biggest thing missing in current `minterprise`.

## 3. How Ninja handles multitenancy on `jk-develop`

Main files:

- `/Users/biswajitrout/companyProjects/ninja-service/turtlemint/app/Global.java`
- `/Users/biswajitrout/companyProjects/ninja-service/turtlemint/app/shared/RequestAction.java`
- `/Users/biswajitrout/companyProjects/ninja-service/turtlemint/app/common/dao/impl/AbstractBaseDAOImpl.java`
- `/Users/biswajitrout/companyProjects/ninja-service/turtlemint/app/com/agentpro/utils/BrokerConfigUtils.java`

## 3.1 Request bootstrap in Ninja

The main request bootstrapping happens in `Global.onRequest(...)`.

What it does:

1. Reads request `HOST`
2. Infers broker from host
3. If host mapping does not work, falls back to `x-broker`
4. Saves broker into Play session
5. Saves host into Play session
6. Later also saves tenant into session from:
   - request body `tenant`
   - `x-tenant`
   - cookies

### Broker resolution in `Global.onRequest(...)`

Examples from the code:

- if host contains `axisbank` or `.al.mintpro.in` -> broker becomes `AXISBANK`
- if host contains `jkbank` -> broker becomes `JKBANK`
- if host contains `.turtle-feature.com` and nothing else matched -> broker becomes `TURTLEMINT`
- if not found, fallback to `x-broker`
- if still not found, fallback to default broker

So the effective Ninja request path is:

```text
host -> broker
or x-broker -> broker
session[broker] = resolved broker
```

### Tenant handling in Ninja

Tenant is stored too, but differently.

In `Global.setSessionVariables(...)`:

- body JSON field `tenant` goes into session
- `x-tenant` goes into session
- partner cookies may also put tenant into session

So the effective tenant path is:

```text
body tenant / x-tenant / cookies -> session[tenant]
```

Important point:

- broker and tenant are both stored
- but they are not used the same way

## 3.2 DAO routing in Ninja

The actual DB selection happens in:

- `AbstractBaseDAOImpl.determineMongoTemplateByCollectionName(...)`

What it does:

1. If the collection is recognized as a master/common collection, it routes by collection/database name
2. Otherwise it calls:
   - `brokerConfigUtils.getSessionBroker()`
3. Then it does:
   - `mongoTemplateMap.get(broker)`

So the real routing key is:

- session broker

not:

- session tenant

That means Ninja is not truly tenant-routed in the DB layer.

It is:

- broker-routed DB access
- tenant-aware business/session context

## 3.3 What `BrokerConfigUtils` does in Ninja

`BrokerConfigUtils` confirms the same behavior:

- `getSessionBroker()` reads broker from Play session
- `getTenantFromSession()` reads tenant from Play session

Again, both exist, but the DAO code uses broker for DB template resolution.

## 3.4 What Ninja’s model really is

The actual model is:

```text
Host / x-broker -> broker
body / x-tenant / cookies -> tenant
DAO -> mongoTemplateMap[broker]
```

So:

- broker decides DB
- tenant is extra request context

That is why saying "Ninja is multitenant" can be misleading unless we say exactly what layer we mean.

At the DB layer, it is broker-based isolation.

## 4. How Minterprise is currently wired

Main files:

- `/Users/biswajitrout/companyProjects/minterprise/application/src/main/java/com/application/filter/MultiTenantWebFilter.java`
- `/Users/biswajitrout/companyProjects/minterprise/library/src/main/java/com/leads/utils/BaggageUtil.java`
- `/Users/biswajitrout/companyProjects/minterprise/library/src/main/java/com/library/multitenant/mongodb/MultiTenantMongoReactiveDataConfiguration.java`
- `/Users/biswajitrout/companyProjects/minterprise/library/src/main/java/com/library/multitenant/mongodb/MultiTenantMongoFactory.java`
- `/Users/biswajitrout/companyProjects/minterprise/library/src/main/java/com/library/dao/mongodb/AbstractReactiveMongoDAO.java`
- `/Users/biswajitrout/companyProjects/minterprise/library/src/main/java/com/library/configuration/FeatureFlagConfiguration.java`
- `/Users/biswajitrout/companyProjects/minterprise/library/src/main/java/com/leads/configuration/LeadStageInfoConfiguration.java`
- `/Users/biswajitrout/companyProjects/minterprise/library/src/main/java/com/library/constant/TenantConstant.java`
- `/Users/biswajitrout/companyProjects/minterprise/application/src/main/resources/application-local.yml`

## 4.1 Template map creation in Minterprise

`MultiTenantMongoReactiveDataConfiguration` creates:

- `mongoClientMap`
- `reactiveDatabaseFactoryMultiTenant`
- `reactiveMongoTemplateMap`

All of these are keyed by:

- `tenantId`

from config.

In local config, the datasource keys are:

- `turtlemint`
- `axisbank`

From `application-local.yml`:

```yaml
multitenancy:
  turtlemint:
    dataSources:
      - tenantId: turtlemint
      - tenantId: axisbank
```

Important point:

Even though the field is named `tenantId`, the configured values currently look broker-like:

- `turtlemint`
- `axisbank`

This matters a lot for the recommendation later.

## 4.2 DAO routing in Minterprise

Both of these use baggage value `tenant`:

- `AbstractReactiveMongoDAO.getMongoTemplate()`
- `MultiTenantMongoFactory.getTenantFactory()`

The routing logic is:

```text
baggage tenant -> reactiveMongoTemplateMap[tenant]
```

So the intended model in `minterprise` is:

```text
request -> baggage tenant -> Mongo template
```

This is different from Ninja, where:

```text
request -> session broker -> Mongo template
```

## 4.3 Where Minterprise is hardcoded today

This is the key issue.

### `MultiTenantWebFilter`

`MultiTenantWebFilter.extractTenant(...)` currently returns:

- `TenantConstant.DEFAULT_TENANT`

and `DEFAULT_TENANT` is:

- `axisbank`

So every request that passes through the filter currently sets baggage tenant to:

- `axisbank`

### `BaggageUtil`

`BaggageUtil.getTenantBaggage(...)` is also hardcoded to:

- `TenantConstant.DEFAULT_TENANT`

instead of actually reading tracing baggage.

That means even if the filter later sets something dynamic, DAO code still currently resolves:

- `axisbank`

### Feature flags and lead stage config

These two are also directly reading:

- `reactiveMongoTemplateMap.get(TenantConstant.DEFAULT_TENANT)`

in:

- `FeatureFlagConfiguration`
- `LeadStageInfoConfiguration`

So even outside generic DAO routing, some config is pinned to:

- `axisbank`

## 4.4 What Minterprise is effectively doing right now

Although the architecture looks like:

```text
x-tenant -> baggage tenant -> reactiveMongoTemplateMap[tenant]
```

the real behavior today is mostly:

```text
always axisbank -> reactiveMongoTemplateMap["axisbank"]
```

That means:

- request headers are not actually deciding DB routing today
- baggage is not really being read
- tenant-based routing is not truly active

## 5. The exact difference between Ninja and Minterprise

This is the most important comparison.

## 5.1 Request context storage

Ninja:

- uses Play session
- broker and tenant are stored in session

Minterprise:

- should use reactive tracing baggage / request context
- today the baggage read/write path is incomplete and hardcoded

## 5.2 DB routing key

Ninja:

- DB key = broker

Minterprise intended:

- DB key = tenant

Minterprise actual today:

- DB key = hardcoded default tenant `axisbank`

## 5.3 Tenant meaning

Ninja:

- tenant is mostly business metadata
- DB routing ignores it in the general DAO path

Minterprise intended:

- tenant is overloaded to mean both:
  - request business tenant
  - DB routing key

This overloading is the main design problem.

## 5.4 Framework model

Ninja:

- synchronous Play app
- session is a natural place for broker / tenant storage

Minterprise:

- reactive Spring WebFlux app
- session-style routing would be the wrong fit
- baggage or Reactor context is the correct place for per-request routing context

## 5.5 Current datasource shape

Ninja:

- its template map is already broker-oriented

Minterprise:

- even though the map key is named `tenantId`, the currently configured values are broker-shaped:
  - `axisbank`
  - `turtlemint`

This means the code wants tenant-based routing, but the runtime config still resembles broker-based routing.

## 6. Why copying Ninja directly into Minterprise is not ideal

Even if Ninja works, copying it as-is into `minterprise` would not be the right move.

## 6.1 Session is the wrong primitive for Minterprise

Ninja uses session because Play + browser-driven requests make that natural.

`minterprise` is mostly service-to-service / API-driven / reactive.

For this setup:

- request header resolution
- request host resolution
- Reactor context / tracing baggage

is a better fit than session.

## 6.2 Request body should not be your primary routing source

Ninja can store tenant from request body into session.

In `minterprise`, DB routing should not primarily depend on body content because:

- routing is infrastructure-level behavior
- body parsing happens later
- headers/host are more stable for request routing

## 6.3 Broker and tenant should not stay overloaded

If `minterprise` keeps using one field called `tenant` to mean:

- actual business tenant
- Mongo routing key
- default config namespace

then this will keep causing confusion.

The code will stay hard to reason about when `tenant != broker`.

## 7. What should ideally be used in Minterprise right now

This is the recommendation based on the current code and config, not on a future perfect design.

## 7.1 Recommended model

Use 3 separate request-context values:

### 1. `broker`

Used for:

- broker config
- insurer config
- host/domain behavior
- template selection
- partner/integration behavior

### 2. `businessTenant`

Used for:

- request/business identity
- partner/tenant-specific business rules
- future true tenant-aware behavior

### 3. `mongoRoutingKey`

Used only for:

- `reactiveMongoTemplateMap[...]`

This should be the only thing the DAO layer uses for DB selection.

## 7.2 What should `mongoRoutingKey` be today

Given the current `minterprise` setup, the best current choice is:

- derive `mongoRoutingKey` from `broker`

Why:

1. Current datasource keys are `axisbank` and `turtlemint`
2. Those look like broker DBs, not true tenant DBs
3. Feature/config code is already effectively behaving as if there is one default broker DB
4. This is the closest behavior to Ninja, but implemented properly for reactive code

So right now the ideal runtime model should be:

```text
host / x-broker -> broker
x-tenant -> businessTenant
mongoRoutingKey = brokerToMongoKey(broker)
DAO -> reactiveMongoTemplateMap[mongoRoutingKey]
```

That gives you:

- stable DB routing
- no hardcoded `axisbank`
- no confusion between DB key and business tenant

## 7.3 Why I would not use raw `x-tenant` as DB key right now

Because the current system is not ready for true tenant-routed DB access yet.

Reasons:

1. datasource map currently has only:
   - `axisbank`
   - `turtlemint`
2. `TenantConstant` contains many business tenants, but there are not matching Mongo templates for all of them
3. `FeatureFlagConfiguration` and `LeadStageInfoConfiguration` are still default-tenant based
4. the rest of the codebase still often treats broker and tenant as very close concepts

If you directly switch DB routing to raw `x-tenant` today, requests like:

- `x-broker = turtlefin`
- `x-tenant = flipkart`

would likely fail unless:

- `reactiveMongoTemplateMap["flipkart"]`

exists and all supporting config/caches are tenant-ready too.

## 8. The ideal practical design for Minterprise

The clean design would be:

## 8.1 Resolve request context in the web filter

`MultiTenantWebFilter` should resolve:

- `broker`
- `businessTenant`
- `mongoRoutingKey`

Suggested order:

### Broker resolution

1. derive from host
2. fallback to `x-broker`
3. fallback to default broker

### Business tenant resolution

1. `x-tenant`
2. fallback to broker
3. fallback to default tenant

### Mongo routing key resolution

Current phase:

- `mongoRoutingKey = broker`

Future phase if true tenant DB isolation exists:

- `mongoRoutingKey = tenant`
- or `mongoRoutingKey = brokerTenantMapping(tenant, broker)`

## 8.2 Put all 3 into baggage

For example:

- `broker`
- `tenant`
- `mongoRoutingKey`

Then:

- business code can read broker and tenant
- DAO can read only mongo routing key

## 8.3 Make DAO routing use `mongoRoutingKey`

Instead of:

- `getTenantBaggage(TenantConstant.TENANT_DB_BAGGAGE)`

you should read:

- `mongoRoutingKey`

This avoids overloading business tenant as DB selector.

## 8.4 Fix config loaders

`FeatureFlagConfiguration` and `LeadStageInfoConfiguration` should not directly use:

- `reactiveMongoTemplateMap.get(DEFAULT_TENANT)`

They should either:

- use `mongoRoutingKey`

or:

- use `masterReactiveMongoTemplate` if they are truly global/common collections

Right now they are part of the reason multitenancy stays effectively hardcoded.

## 9. Recommended implementation phases

This is the safest rollout path.

## Phase 1: Remove hardcoding but keep current DB shape

Goal:

- make routing dynamic without changing the DB model

Do:

1. `MultiTenantWebFilter`
   - resolve broker dynamically
   - resolve tenant dynamically
   - compute `mongoRoutingKey = broker`
2. `BaggageUtil`
   - stop returning hardcoded default
   - read actual baggage
3. `AbstractReactiveMongoDAO`
   - read `mongoRoutingKey`
4. `MultiTenantMongoFactory`
   - read `mongoRoutingKey`
5. `FeatureFlagConfiguration`
   - use routing key or master template
6. `LeadStageInfoConfiguration`
   - use routing key or master template

Result:

- requests no longer always hit `axisbank`
- current DB layout still works

## Phase 2: Separate business tenant from DB key cleanly

Goal:

- remove conceptual confusion

Do:

1. add explicit constants / baggage names for:
   - broker
   - business tenant
   - mongo routing key
2. stop using `tenant` variable name for physical DB routing

Result:

- code becomes easier to reason about
- future tenant-specific changes become safer

## Phase 3: True tenant-routed DBs, if really needed

Goal:

- each business tenant can have its own DB

Do only if:

- you actually provision datasource entries for those tenants
- caches/config loaders are also tenant-aware

At that point:

- `mongoRoutingKey` can be switched from broker-based to tenant-based

without breaking the rest of the code, because routing is already separated from business tenant.

## 10. Example scenarios

## Scenario 1: Axis Bank request

Headers:

- `x-broker = axisbank`
- `x-tenant = axisbank`

Recommended resolved context:

```text
broker = axisbank
businessTenant = axisbank
mongoRoutingKey = axisbank
```

DB used:

- `reactiveMongoTemplateMap["axisbank"]`

## Scenario 2: Turtlemint broker with partner tenant

Headers:

- `x-broker = turtlemint`
- `x-tenant = flipkart`

Recommended resolved context today:

```text
broker = turtlemint
businessTenant = flipkart
mongoRoutingKey = turtlemint
```

DB used today:

- `reactiveMongoTemplateMap["turtlemint"]`

Business code still knows:

- tenant is `flipkart`

This is the safest current behavior.

## Scenario 3: Future true tenant DBs

If one day you add:

- `reactiveMongoTemplateMap["flipkart"]`

and all config/caches also support it, then you can move to:

```text
broker = turtlemint
businessTenant = flipkart
mongoRoutingKey = flipkart
```

without changing the rest of the routing architecture.

## 11. Final recommendation

If you ask what `minterprise` should ideally use right now, the recommendation is:

- do not keep the current hardcoded `axisbank` behavior
- do not directly copy Ninja session routing
- do not immediately switch to raw `x-tenant` DB routing

Instead:

1. keep `broker` and `tenant` as separate request-context values
2. add one explicit `mongoRoutingKey`
3. for the current system, set `mongoRoutingKey = broker`
4. make all DAO and config lookups use `mongoRoutingKey`
5. keep `tenant` for business logic only

That gives you:

- the same practical DB routing outcome as Ninja
- a better fit for reactive `minterprise`
- no more hardcoded `axisbank`
- a clean migration path to true tenant routing later if needed

## 12. One-line comparison

Ninja today:

```text
broker resolves DB, tenant is extra context
```

Minterprise should become:

```text
broker and tenant are both request context, but an explicit mongoRoutingKey decides DB
```

And right now that `mongoRoutingKey` should be broker-derived, not raw tenant-derived.
