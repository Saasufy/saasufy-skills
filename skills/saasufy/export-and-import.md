# Exporting and Importing a Service

This guide explains how to export a Saasufy service to a single JSON file and import it back, entirely over the HTTP API.

The control panel's dashboard page has **Import data** and **Export data** buttons which do the same thing over WebSockets. Both directions use the same file format, so a file exported over HTTP can be imported through the control panel and vice versa.

## When to Use This Guide

Use it to:
- Back up a service's schema, its data, or both.
- Copy a service into another account (staging to production, or a clone for testing).
- Keep a schema in version control and re-apply it.
- Bulk-load records into a model.

## Authentication

The API key must be read from the `.saasufy-api-key` file in the repository root. Always read this file first before making API calls.

```bash
SAASUFY_API_KEY=$(cat .saasufy-api-key)
SAASUFY_SERVICE_URL=$(cat .saasufy-service-url)
```

Both APIs take the same key: `Authorization: Bearer $SAASUFY_API_KEY`.

## The Two APIs Involved

An export spans both of Saasufy's HTTP APIs, because a service's schema and its records live behind different endpoints:

| | Base URL | Holds | Guide |
|---|---|---|---|
| **Admin API** | `https://saasufy.com/api/` | `Model`, `ModelField`, `ModelView`, `ModelIndex`, `Aggregation` (and its rules), `CloudFunction`, `Constant`, `ScheduledTask` | [schema-management.md](schema-management.md) |
| **Service API** | `{SERVICE_URL}/api/` | The records of your own models (`Product`, `Message`, ...) | [data-management.md](data-management.md) |

The `APICredential` behind the key needs all four permissions for a full round trip:

- `allowAdminRead` / `allowAdminWrite` — the schema, on the Admin API. On the **Service** API `allowAdminRead`/`allowAdminWrite` additionally make the request an admin of the service, which is what lets an export read every record regardless of access-control rules and lets an import restore record timestamps.
- `allowRead` / `allowWrite` — records, on the Service API.

A key missing a permission gets `403` with a message naming it. See the `APICredential` section of [schema-management.md](schema-management.md).

## How List Reads Behave Over HTTP

Three behaviours of the HTTP read endpoints matter for an export:

1. **A list read returns whole records, not IDs.** Over the WebSocket API a view read answers with a list of IDs which you then fetch one by one; the HTTP endpoints hydrate the page for you. The response is an object:

   ```jsonc
   { "data": [ { /* full record */ }, ... ], "isLastPage": true, "count": 42 }
   ```

   `isLastPage` is only present when the page is the last one. `count` only appears with `getCount=true`.

2. **`accountId` is filled in for you.** The Admin API overwrites `viewParams[accountId]` with the account behind the key, so never pass it. Other params, such as `viewParams[modelId]`, you do pass.

3. **Pages are small by default.** Both APIs default to 10 records and cap `pageSize` at 200; a larger value is silently capped rather than rejected. `pageSize=0` is rejected with `400` — to read just a count, use `pageSize=1&getCount=true`.

```bash
# One page of models, with the total count.
curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/Model?view=accountCustomPositionView&pageSize=200&offset=0&getCount=true'
```

Keep requesting pages at `offset += pageSize` until a response carries `isLastPage`.

## The File Format

A single JSON object. Every top-level key is a **model name**, except three which start with `$` (model names are alphanumeric, so they cannot collide):

```jsonc
{
  "Product": {
    "meta": { /* the Model record, minus id, name, position and accountId */ },
    "fields": { "<fieldName>": { /* ModelField record */ } },
    "views": { "<viewName>": { /* ModelView record */ } },
    "indexes": { "<indexName>": { /* ModelIndex record */ } },
    "aggregations": {
      "<aggregationName>": {
        "meta": { /* Aggregation record, with sourceModel in place of sourceModelId */ },
        "groupRules": [ /* AggregationGroupRule records, in order */ ],
        "aggregateRules": [ /* ... */ ],
        "constantRules": [ /* ... */ ],
        "projectRules": [ /* ... */ ]
      }
    },
    "data": [ { /* one record per entry */ } ]
  },
  "$cloudFunctions": { "<name>": { /* CloudFunction record */ } },
  "$constants": { "<name>": { /* Constant record */ } },
  "$scheduledTasks": { "<name>": { /* ScheduledTask record, with cloudFunction in place of cloudFunctionId */ } }
}
```

A schema-only export writes `"data": []`; a data-only export writes empty objects for `meta`, `fields`, `views`, `indexes` and `aggregations`.

### Names Replace IDs

Saasufy derives the ID of every schema resource from the account and the resource's name — a `Model` ID from `accountId` + `name`, a `ModelField` ID from `accountId` + `modelId` + `name`, and so on. An ID is therefore meaningless in another account, so the format **keys each resource by its name and strips `accountId`**, and replaces every cross-reference with the name it points at:

- `Aggregation.sourceModelId` → `meta.sourceModel`, a model name.
- `ScheduledTask.cloudFunctionId` → `cloudFunction`, a function name (`null` where the task has none, which leaves it disabled).

An import resolves these back to IDs in the target account. Leftover `id`, `modelId` and `aggregationId` values inside `fields`, `views`, `indexes` and the rule arrays are ignored on import; you may strip them for a cleaner file.

### Fields the Export Leaves Out

Dropped from exported **records** (`data` entries), because they name a user or an aggregation shard of the source account:

| Field | Why |
|---|---|
| `createdBy`, `updatedBy` | Account IDs of the users who wrote the record. |
| `shardKey` | Added by Saasufy to a model which feeds an aggregation. |
| `targetGeneration` | Added by Saasufy to an aggregation's target model. |

`createdAt`, `updatedAt` and `updatedByIp` are kept and restored, so a record keeps its history. Restoring them needs an admin credential on the service (`allowAdminWrite`); without one the service stamps the import time instead.

Dropped from exported **schema** resources: `accountId` everywhere, plus the runtime state the service writes back as it runs:

| Resource | Fields |
|---|---|
| `Aggregation` | `lastRunAt`, `lastError`, `lastErrorWorkerIndex`, `rebuildRequestedAt` |
| `ScheduledTask` | `lastRunAt`, `lastError` |

Leaving `rebuildRequestedAt` out matters: carrying it over would ask the target service to rebuild the aggregation from scratch.

### The Account Model

The `Account` table is special (see [account-table.md](account-table.md)). Its **schema** is exported and imported like any other model, except for the fields Saasufy's auth layer owns, which an import skips: `username`, `email`, `authSource`, `walletAddress`, `lastWalletBalance`, `lastIpAddress`, `isDeactivated`.

Its **data** is exported but **never imported** — accounts are created by logging in, not by restoring a file. An importer should skip the `Account` entry's `data` array and say so rather than failing.

Also never declared by a model, in `Account` or anywhere else: `createdBy`, `createdAt`, `updatedBy`, `updatedByIp`, `updatedAt`, `shardKey` and `targetGeneration`. The service maintains them itself, so an import must skip them when creating `ModelField` resources.

## Exporting

### 1. Read the Models

```bash
curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/Model?view=accountCustomPositionView&pageSize=200&offset=0'
```

`accountCustomPositionView` returns them in the order the control panel shows them. Keep an ID-to-name map from this page — the aggregations need it.

For each model, drop `id`, `name`, `position` and `accountId`; the rest is `meta` (the access-control settings, `createdAt` and `updatedAt`). Access control is part of the schema, so an export carries it; read [access-control.md](access-control.md) before changing any of it by hand.

### 2. Read Each Model's Schema

Per model ID, with the same pagination:

```bash
MODEL_ID=... # from step 1

curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET "https://saasufy.com/api/ModelField?view=accountModelCustomPositionView&viewParams[modelId]=$MODEL_ID&pageSize=200&offset=0"

curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET "https://saasufy.com/api/ModelView?view=accountModelAlphabeticalView&viewParams[modelId]=$MODEL_ID&pageSize=200&offset=0"

curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET "https://saasufy.com/api/ModelIndex?view=accountModelAlphabeticalView&viewParams[modelId]=$MODEL_ID&pageSize=200&offset=0"

curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET "https://saasufy.com/api/Aggregation?view=accountModelAlphabeticalView&viewParams[modelId]=$MODEL_ID&pageSize=200&offset=0"
```

Key fields by `name` and aggregations by `aggregationName`. `ModelField` is read in position order so that the imported model lays its fields out the same way.

### 3. Read Each Aggregation's Rules

Per aggregation ID, for all four rule types:

```bash
AGGREGATION_ID=... # from step 2

for RULE_TYPE in AggregationGroupRule AggregationAggregateRule AggregationConstantRule AggregationProjectRule; do
  curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
    -XGET "https://saasufy.com/api/$RULE_TYPE?view=accountAggregationView&viewParams[aggregationId]=$AGGREGATION_ID&pageSize=200&offset=0"
done
```

These go into `groupRules`, `aggregateRules`, `constantRules` and `projectRules` as **arrays, not objects** — a rule has no name, and `accountAggregationView` returns them ordered by `createdAt`, which is the order the pipeline applies them in. Preserve it. See [data-aggregation-pipelines.md](data-aggregation-pipelines.md) for what each rule kind does.

### 4. Read the Account-Level Resources

```bash
for TYPE in CloudFunction Constant ScheduledTask; do
  curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
    -XGET "https://saasufy.com/api/$TYPE?view=accountAlphabeticalView&pageSize=200&offset=0"
done
```

Key each by `name` into `$cloudFunctions`, `$constants` and `$scheduledTasks`, and turn each task's `cloudFunctionId` into a `cloudFunction` name using the functions you just read.

A `Constant` marked `isSecret` is exported with its value in plain text, like any other. Treat an export file as a secret.

### 5. Read the Records

Records come from the **Service** API, and every model has a `defaultView` which returns all of its records ordered by `updatedAt` descending:

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET "$SAASUFY_SERVICE_URL/api/Product?view=defaultView&pageSize=200&offset=0&getCount=true"
```

The service must be running for this to answer; deploy it first if it is not (see [deployment-and-testing.md](deployment-and-testing.md)).

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -XPOST 'https://saasufy.com/api/service/start'
```

Two things to watch:

- **Offset paging over `defaultView` is not stable.** The view is ordered by `updatedAt`, so a record written mid-export moves to the front of the ordering and can be read twice or missed. Export from a quiet service, and track the IDs you have already seen so a repeat is dropped rather than written twice.
- **Pace the reads.** The control panel waits 250ms between pages. Reads are metered against the account's usage, and a tight loop over a large model is a lot of load for no gain.

A `multi` field comes back as a JSON array. Keep it as one in the file; the import converts it (see below).

## Importing

Order matters, because each step resolves names against what the previous one created.

### 1. Create the Models, Fields, Views and Indexes

One pass over every model key in the file.

```bash
# The Model. position orders it in the control panel; omit it to append.
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/Model' \
  -d '{"name": "Product", "accessRead": "allow", "position": 0}'
```

A create answers `201` with the new ID as its body. Then, for each entry of `fields`, `views` and `indexes`, POST to `ModelField`, `ModelView` and `ModelIndex` with `name` set to the key and `modelId` set to the ID you just got. Strip `id`, `modelId`, `accountId` and `createdBy` from the file's copy before sending it, and skip any field Saasufy owns (listed under [The Account Model](#the-account-model) above).

Create indexes and views only as [views-and-indexing.md](views-and-indexing.md) describes: a single-field `transformIndex` needs no `ModelIndex` of its own, and an index's `name` is reassigned from its `fields` on deployment. An export taken from a working service already follows this, so send it back as it is.

### 2. Create the Aggregations and Their Rules

A **second pass**, after every model exists — an aggregation's `sourceModel` can name a model which appears later in the file.

```bash
# Resolve meta.sourceModel to an ID, then create the Aggregation.
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/Aggregation' \
  -d '{"aggregationName": "dailyTotals", "modelId": "{TARGET_MODEL_ID}", "sourceModelId": "{SOURCE_MODEL_ID}"}'
```

Then POST each rule array in order, with `aggregationId` set to the new aggregation's ID. Post the rules of one kind sequentially rather than in parallel: their IDs are derived from their contents, and `accountAggregationView` orders them by `createdAt`.

### 3. Create the Cloud Functions, Constants and Scheduled Tasks

In that order — a `ScheduledTask` needs its function's ID.

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/CloudFunction' \
  -d '{"name": "sendDigest", "code": "..."}'

curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/ScheduledTask' \
  -d '{"name": "nightlyDigest", "cloudFunctionId": "{CLOUD_FUNCTION_ID}", "taskInterval": 1, "intervalUnit": "day"}'
```

A task whose `cloudFunction` is `null` is created with `cloudFunctionId: null`, which leaves it disabled — the same state it was exported in. A task naming a function the file does not carry is an error worth reporting rather than guessing at. See [cloud-functions.md](cloud-functions.md), [constants.md](constants.md) and [scheduled-tasks.md](scheduled-tasks.md).

### 4. Deploy

Nothing above is live until the service is (re)started, and the next step writes records into the models you just created:

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -XPOST 'https://saasufy.com/api/service/start'
```

### 5. Write the Records

Against the **Service** API, one POST per record:

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST "$SAASUFY_SERVICE_URL/api/Product" \
  -d '{"id": "9fed41a0-caae-4620-8ed1-931a3b388fa3", "name": "Laptop", "price": 999.99, "createdAt": 1716900000000, "updatedAt": 1716900000000}'
```

Before sending each record:

- **Skip the `Account` model entirely.**
- **Drop the fields an export drops** — `createdBy`, `updatedBy`, `shardKey`, `targetGeneration` — in case the file came from elsewhere. Keep `id`, `createdAt`, `updatedAt` and `updatedByIp`.
- **Flatten non-scalar values.** The HTTP API takes strings, numbers and booleans. A `multi` field must be sent as a **comma-separated string**, not a JSON array (`["a","b"]` → `"a,b"`); posting an array fails with *"Value must be a string"*. Any other object becomes its string form. See [data-management.md](data-management.md).
- **Keep each record's `accountId`** if the model has one. The service only stamps its own account ID onto a record which arrives without one, so a multi-user table restores its ownership intact.

Records belonging to an aggregation's **target** model are in the file, but the aggregation recomputes them. Importing them is harmless and gives the target model its contents immediately; skip them if you would rather let the pipeline fill it.

### 6. Verify

Count the records in each model and compare against the file:

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET "$SAASUFY_SERVICE_URL/api/Product?view=defaultView&pageSize=1&getCount=true"
```

Then check each index's `fields` against real field names, as [schema-management.md](schema-management.md) advises after any deployment.

## Re-Importing Over an Existing Service

Because IDs are derived from names, importing a file into an account which already has some of its resources is an **upsert**, not a duplicate. The create fails and you update instead.

A duplicate answers `500` with a body which begins with the existing resource's ID:

```
Duplicate primary key `3f1c...-...`: { ... }
```

Extract that ID and `PUT` the same values to it, which saves looking the resource up by name:

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/ModelField/{EXISTING_ID}' \
  -d '{"required": true, "position": 3}'
```

A `PUT` must leave out the fields which identify the resource, or it fails with *"Some specified fields cannot be modified"*:

| Resource | Cannot be changed by an update |
|---|---|
| `Model` | `name` |
| `ModelField` | `name`, `modelId`, `type` |
| `ModelView`, `ModelIndex` | `name`, `modelId` |
| `Aggregation` | `aggregationName`, `modelId` |
| `CloudFunction`, `Constant`, `ScheduledTask` | `name` |
| Aggregation rules | `aggregationId`, `sourceField`, plus `operation` (aggregate rules) and `targetField` (constant rules) |

A field's `type` being immutable is the one that bites: re-importing a file whose field changed from `string` to `number` leaves the existing field alone. Delete the field and re-import, accepting that its data goes with it.

Aggregation rules have no name to look up, so where a rule already exists, find it among the aggregation's rules by the fields its ID is derived from: `sourceField` for a group or project rule, `sourceField` plus `operation` for an aggregate rule, `targetField` for a constant rule.

Records upsert the same way: a `POST` of a record whose `id` already exists fails as a duplicate, and a `PUT` to `{SERVICE_URL}/api/{ModelName}/{id}` with the remaining fields updates it.

An import never deletes. A resource which exists in the target but not in the file is left as it is, so importing into a used account merges rather than replaces. To replace, delete what you do not want first.

## Importing Raw Data Into One Model

The control panel also accepts a file which is a bare JSON **array** of records, which it loads into a model you name. Over HTTP this is simply the record loop of step 5 against that model, with two conventions the control panel applies and which are worth copying:

- `snake_case` keys are converted to `camelCase` (`first_name` → `firstName`), since a Saasufy field name is alphanumeric.
- A field the model does not declare is created from the first value seen for it: `string`, `number` or `boolean` by its JavaScript type, `string` for anything else. An inferred `id` field is marked `required`.

Creating fields this way means a deployment before the records can be written. Declaring the model up front, as in [schema-management.md](schema-management.md), gives you the constraints and access control that inference cannot guess.

## Important Notes

1. **An export file holds secrets.** Constant values marked `isSecret`, cloud function source, OAuth settings in a model's access configuration, and every record in the service. Keep it out of version control unless you have checked what is in it.
2. **A schema-only export is the one to keep in git.** It is small, diffable, and re-importable; a data export is a backup, not a source file.
3. **`OAuthProvider`, `BlockchainAuthProvider` and `APICredential` are not part of the format.** They carry credentials tied to one deployment, so a copied service needs its providers and keys set up again. See [authentication.md](authentication.md).
4. **Deploy between schema and data.** A record cannot be written to a model the running service has never seen.
5. **Import the models before the aggregations.** Two passes, because `sourceModel` can point forwards.
6. **Order the aggregation rules.** They are applied in `createdAt` order, so post them sequentially.
7. **Page every list read.** Both APIs default to 10 records and cap at 200.
8. **`Account` data is never imported.**
9. **An import is not atomic.** There is no transaction and no rollback across these calls; a failure halfway leaves the service partly imported. Keep the file, fix the cause, and re-run it — the upsert behaviour above makes a re-run safe.

## Related

- [schema-management.md](schema-management.md) — the Admin HTTP API and the fields of every schema resource.
- [data-management.md](data-management.md) — the Service HTTP API and record CRUD.
- [views-and-indexing.md](views-and-indexing.md) — how to declare views and indexes correctly.
- [data-aggregation-pipelines.md](data-aggregation-pipelines.md) — aggregations and their rules.
- [account-table.md](account-table.md) — the `Account` model.
- [deployment-and-testing.md](deployment-and-testing.md) — deploying a service.
