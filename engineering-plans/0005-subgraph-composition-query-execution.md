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

Since subgraph composition allows a subgraph to import types from another subgraph, GraphQL query execution will need to traverse a subgraph's import graph to resolve concrete types.
Because imported schemas/types are loosely coupled, query execution also needs to address missing types in the api schema during resolution.
In addition, imported types which have been renamed need to be mapped back to their original name so that queries can resolve correctly.
Lastly, to simplify query execution graph-node needs to deny subgraphs with JSONB storage from using subgraph composition.

## Implementation

1. Support Custom Type Names: Ensure @originalName directives on the imported types are used to query the correct sql tables.
2. Codify New Query Runtime Errors: Ensure @placeholder types are acknowledged and proper errors are returned during query execution when a type's subgraph is not deployed to the graph-node resolving the query.
3. Query Correct Subgraph: Ensure that @subgraphId directives on imported types are used correctly in all cases.
4. Add Subgraph Composition Feature Flag: Ensure subgraphs JSONB storage can not use subgraph composition.

### Support Custom Type Names

When a type is imported with the name/as syntax:

```graphql
type _Schema_ @import(types: [{ name: "A", as: "B" }], from: { id: "..." })
```

an `@originalName` directive is added to it when it is merged into the target schema:

```graphql
type B @entity @subgraphId(id: "...") @originalName(name: "A") {
	id: ID!
	...
}
```

GraphQL query resolution needs to make use of the `@originalName` directive to ensure SQL queries are made against the correct tables.

The follow code locations need to be updated:

1. https://github.com/graphprotocol/graph-node/blob/master/graphql/src/store/resolver.rs#L362-L379f

2. https://github.com/graphprotocol/graph-node/blob/master/graphql/src/store/query.rs#L21-L27

### Codify New Query Runtime Errors

When a type is imported from a schema that is unavailable a placeholder is created to ensure the schema remains complete:

```graphql
type _Schema_ @import(types: [{ name: "A", as: "B" }], from: { id: "..." })
```

when the subgraph with `id: "..."` is unavailable during the merge a placeholder will be created:

```graphql
type B @entity @placeholder @originalName(name: "A") {
	id: ID!
}
```

Since the `@subgraphId` won't be available for this type during query execution, we have to handle this as query runtime error in the follow locations:

1. https://github.com/graphprotocol/graph-node/blob/master/graphql/src/store/resolver.rs#L359

### Query Correct Subgraph

Imported types in a merged schema which are available on the Graph Node executing the query will have `@subgraphId` directives actings are pointers to the relevant database table.
In the prefetch portion of query resolution we run JOINs across tables for a subgraph's types to aggregate the children for a single parent or a set of parent objects. Instead of modifying sql for the JOIN
to ensure that a join across tables for two different subgraphs works, we can remove the JOIN and make the prefetch code only execute queries on a single table at a time.

Adapt the work described in this issue: https://github.com/graphprotocol/graph-node/issues/1450


### Add Subgraph Composition Feature Flag

Do not allow subgraph composition for subgraphs which use the JSONB storage engine.

## Tests

### Unit Tests:

-

### Integration Tests:

1. Querying an A and [A] with an imported field B and [B] with an imported field C

## Migration

Subgraph composition is an additive feature and does not require a migration plan.

## Documentation

Document the following aspects of query execution along side more general subgraph composition documentation:

1. New runtime errors which could occur during query execution

## Implementation Plan

- Support custom type names (1d)
- Codify new query execution runtime errors (1d)
- Add subgraph composition feature flag (1d)
- Ensure correct subgraph is queried during single object resolution and prefetch paths (3d)

## Open Questions
