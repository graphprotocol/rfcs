# PLAN-0001: Subgraph Schema Merging for Subgraph Composition

<dl>
  <dt>Author</dt>
  <dd>Jorge Olivero</dd>

  <dt>Implements</dt>
  <dd><a href="../rfcs/0001-subgraph-composition.md">RFC-0001 Subgraph Composition</a></dd>

  <dt>Engineering Plan pull request</dt>
  <dd><a href="https://github.com/graphprotocol/rfcs/pull/5">Engineering Plan PR</a></dd>

  <dt>Obsoletes (if applicable)</dt>

  <dt>Date of submission</dt>
  <dd>2019-12-09</dd>

  <dt>Date of approval</dt>
  <dd>TBD</dd>

  <dt>Approved by</dt>
  <dd>TBD</dd>
</dl>

## Summary

Subgraph composition allows a subgraph to import types from a foreign subgraph. Imports are designed to be loosely coupled throughout the deployment and indexing phases, and tightly coupled during query execution for the subgraph.

To generate an API schema for the subgraph, the local schema needs to be merged with each of the foreign schemas that it imports. The merging process needs to take the following into account:

1. An imported schema not being available on the graph-node
2. An imported schema without a definition for the type which the local subgraph imports
3. A merged and cached api schema for a subgraph imported by name is updated
4. Ability to tell which subgraph/schema each type belongs to

## Implementation

There are two important parts to schema merging:

1. Merging the schema

2. Invalidating a cache entry when a subgraph's dependency, referenced by name, is updated and remerging that schema

### Schema Merging

Add a `merged_schema(&Schema, HashMap<SchemaReference, Arc<Schema>) -> Schema` function to the `graphql::schema` crate which will add each of the imported 
types to the proviced document with a @subgraphId diretive denoting which subgraph the type came from. The `api_schema` function will add all the necessary types and fields for the imported types without any changes.

Schema before calling `merged_schema`:

```graphql
type _Schema_
	@import(
		types: ["B"],
		from: { id: "..." }
	)

type A @entity {
	id: ID!
	foo: B!
}
```

Schema after calling `merged_schema`

```graphql
type A @entity @subgraphId("...") {
	id: ID!
	foo: B!
}

type B @entity @subgraphId("...") {
	id: ID!
	bar: String
}
```

After the schema document is merged, the `api_schema` will be called.

### Cache Invalidation

In the initial implementation, the cache will need to check if an entry needs to be updated when it is accessed. To determine whether an entry in the schema cache has been invalidated, each subgraph schema it imports by name needs to be queried in the store for its most current version. The schema cache will maintain a record of the versions used for merging in each entry, and if the version previously used to merge is no longer current that entry needs to be invalidated an remerged before responding to the cache access.

A more performant invalidation solution would be to have the cache maintain a listener notifying it every time a subgraph's current version changes. Upon receiving the notification it would scan the schemas in the cache for those which need to be remerged.


## Tests

1. Schemas are merged correctly when all schemas and imported types are available

2. Placeholder types are properly inserted into the merged schema when a schema is not available

3. Placeholder types are properly inserted into the merged schema when the relevant schemas are available but the types are not

4. The cache is invalidated when the store returns an updated version of a cache entry's dependency

## Migration

Subgraph composition is an additive feature which shouldn't require a special migration plan.

## Documentation

Documenation on https://thegraph.com/docs needs to outline:

1. The reserved _Schema_ type and how to define imports on it
2. The semantics of importing by subgraph id vs. subgraph name, i.e. what happens when a subgraph imported by name removes expected types from the schema
3. How queries are processed when imported subgraphs schemas or types are not available on the graph-node processing the query

## Implementation Plan

- Impelment the `merged_schema` function (2d)
- Write tests for the `merged_schema` function (1d)
- Integrate `merged_schema` into `Store::cached_schema` and update the cache to include the relevant information for imported schemas and types (1d)
- Add cache invalidation logic to `Store::cached_schema` (2d)

## Open Questions

The execution of queries and subscriptions needs to be updated to leverage the types in a merged schema.
