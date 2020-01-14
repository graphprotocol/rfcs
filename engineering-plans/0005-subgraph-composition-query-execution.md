# PLAN-0005: Subgraph Composition Query Execution

<dl>
  <dt>Author</dt>
  <dd>Jorge Olivero</dd>

  <dt>Implements</dt>
  <dd><a href="../rfcs/0001-subgraph-composition.md">RFC-0001 Subgraph Composition</a></dd>

  <dt>Engineering Plan pull request</dt>
  <dd><a href="">Engineering Plan PR</a></dd>

  <dt>Obsoletes (if applicable)</dt>
  <dd>-</dd>

  <dt>Date of submission</dt>
  <dd>2020-1-14</dd>

  <dt>Date of approval</dt>
  <dd>TBD</dd>

  <dt>Approved by</dt>
  <dd>TBD</dd>
</dl>

## Summary

Subgraph composition allows a subgraph to import types from another subgraph. This means that the GraphQL queries will need to traverse the subgraph's import graph to pull all of the necessary concrete types.

## Implementation

1. Ensure @subgraphId and @originalName directives on the imported types are used to generate the correct sql queries
2. Ensure @placeholder types are acknowledged during query execution since this means that a type can not be located in a subgraph
3. 

### Schema Merging

Add a `merged_schema(&Schema, HashMap<SchemaReference, Arc<Schema>) -> Schema` function to the `graphql::schema` crate which will add each of the imported types to the provided document with a `@subgraphId` directive denoting which subgraph the type came from. If any of the imported types have non scalar fields, import those types as well.

The `HashMap<SchemaReference, Arc<Schema>` includes all of the schemas in the subgraph's import graph which are available on the Graph Node. For each `@import` directive on the subgraph, find the imported types by tracing their path along the import graph.

- If any schema node along that path is missing or if a type is missing in the schema, add a type definition to the subgraphs merged schema with the proper name, a `@subgraphId(id: "...")` directive (if available), and a `@placeholder` directive denoting that type was not found.
sc- If the type is found, copy it and add a `@subgraphId(id: "...")` directive.
- If the type is imported with the `{ name: "", as: "" }` format, the merged type will include an `@originalName(name: "...")` directive preserving the type name from the original schema.

The `api_schema` function will add all the necessary types and fields for the imported types without requiring any changes.

#### Example #1: Complete merge

Local schema before calling `merged_schema`:

```graphql
type _Schema_
  @import(
    types: ["B"],
    from: { id: "X" }
  )

type A @entity {
  id: ID!
  foo: B!
}
```

Imported Schema X:

```graphql
type B @entity {
  id: ID!
  bar: String
}
```

Schema after calling `merged_schema`:

```graphql
type A @entity {
  id: ID!
  foo: B!
}

type B @entity @subgraphId(id: "X") {
  id: ID!
  bar: String
}
```

#### Example #2: Incomplete merge

Schema before calling `merged_schema`:

```graphql
type _Schema_
  @import(
    types: ["B"],
    from: { id: "X" }
  )

type A @entity {
  id: ID!
  foo: B!
}
```

Imported Schema X:

```graphql
NOT AVAILABLE
```

Schema after calling `merged_schema`

```graphql
type A @entity @subgraphId(id: "...") {
  id: ID!
  foo: B!
}

type B @entity @placeholder {
  id: ID!
}
```

#### Example #3: Complete merge with `{ name: "...", as: "..." }`

Schema before calling `merged_schema`

```graphql
type _Schema_
  @imports(
    types: [{ name: "B", as: "BB" }]
    from: { id: "X" }
  )

type B @entity {
  id: ID!
  foo: BB!
}
```

Imported Schema X:

```graphql
type B @entity {
  id: ID!
  bar: String
}
```

Schema after calling `merged_schema`

```graphql
type B @entity {
  id: ID!
  foo: BB!
}

type BB @entity @subgraphId(id: "X") @originalName(name: "B") {
  id: ID!
  bar: String
}
```

#### Example #4: Complete merge with nested types

Schema before calling `merged_schema`

```graphql
type _Schema_
  @imports(
    types: [{ name: "B", as: "BB" }]
    from: { id: "X" }
  )

type B @entity {
  id: ID!
  foo: BB!
}
```

Imported Schema X:

```graphql
type _Schema_
  @imports(
	types: [{ name: "C", as: "CC" }]
	from: { id: "Y" }
  )

type B @entity {
  id: ID!
  bar: CC!
  baz: DD!
}

type DD @entity {
  id: ID!
}
```

Imported Schema Y:

```graphql
type C @entity {
	id: ID!
}
```

Schema after calling `merged_schema`

```graphql
type B @entity {
  id: ID!
  foo: BB!
}

type BB @entity @subgraphId(id: "X") @originalName(name: "B") {
  id: ID!
  bar: CC!
  baz: DD!
}

type CC @entity @subgraphId(id: "Y") @originalName(name: "C") {
  id: ID!
}

type DD @entity @subgraphId(id: "X") {
  id: ID!
}
```



After the schema document is merged, the `api_schema` function will be called.

### Cache Invalidation

For each schema in the cache, keep a vector of subgraph pointers containing an element for each schema in the subgraph's import graph which was imported by name and the subgraph ID which was used during the schema merge. When a schema is accessed from the schema cache (and possibly only if this check hasn't happened in the last N seconds), check the current version for each of these schemas and run a diff against the versions used for the most recent schema merge. If there are any new versions, re merge the schema.

Currently the `schema_cache` in the `Store` is a `Mutex<LruCache<SubgraphDeploymentId, SchemaPair>>`. A `SchemaPair` consists of two fields: `input_schema` and `api_schema`. To support the refresh flow, `SchemaPair` would be extended to be a `SchemaEntry`, with the fields `input_schema`, `api_schema`, `schemas_imported` (`Vec<(SchemaReference, SubgraphDeploymentId)>`), and a `last_refresh_check` timestamp.

A more performant invalidation solution would be to have the cache maintain a listener notifying it every time a subgraph's current version changes. Upon receiving the notification the listener scans the schemas in the cache for those which should be remerged.


## Tests

1. Schemas are merged correctly when all schemas and imported types are available.

2. Placeholder types are properly inserted into the merged schema when a schema is not available.

3. Placeholder types are properly inserted into the merged schema when the relevant schemas are available but the types are not.

4. The cache is invalidated when the store returns an updated version of a cache entry's dependency.

## Migration

Subgraph composition is an additive feature which doesn't require a special migration plan.

## Documentation

Documentation on https://thegraph.com/docs needs to outline:

1. The reserved _Schema_ type and how to define imports on it.
2. The semantics of importing by subgraph ID vs. subgraph name, i.e. what happens when a subgraph imported by name removes expected types from the schema.
3. How queries are processed when imported subgraphs schemas or types are not available on the graph-node processing the query.

## Implementation Plan

- Implement the `merged_schema` function (2d)
- Write tests for the `merged_schema` function (1d)
- Integrate `merged_schema` into `Store::cached_schema` and update the cache to include the relevant information for imported schemas and types (1d)
- Add cache invalidation logic to `Store::cached_schema` (2d)

## Open Questions

- The execution of queries and subscriptions needs to be updated to leverage the types in a merged schema.
