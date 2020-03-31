# PLAN-0004: Subgraph Grafting

<dl>
  <dt>Author</dt>
  <dd>David Lutterkort</dd>

  <dt>Implements</dt>
  <dd>No RFC</dd>

  <dt>Engineering Plan pull request</dt>
  <dd><a href="https://github.com/graphprotocol/rfcs/pull/17">https://github.com/graphprotocol/rfcs/pull/17</a></dd>

  <dt>Date of submission</dt>
  <dd>2020-03-17</dd>

  <dt>Date of approval</dt>
  <dd>YYYY-MM-DD</dd>

  <dt>Approved by</dt>
  <dd>Jannis Pohlmann</dd>
  <dd>Zac Burns</dd>
</dl>

## Contents

<!-- toc -->

## Summary

This feature makes it possible to resume indexing of a failed subgraph by
creating a new subgraph that uses the failed subgraph's data up to a block
that the subgraph writer deems reliable, and continue indexing using the
new subgraph's mappings. We call this operation *grafting* subgraphs.

The main use case for this feature is to allow subgraph developers to move
quickly past an error in a subgraph, especially during development. The
overarching goal should always be to replace a subgraph that was grafted
with one that was indexed from scratch as soon as that is reasonably
possible.

## Subgraph manifest changes

The user-facing control for this feature will be a new optional
`graft` field in the subgraph manifest. It specifies the base subgraph and
block number in that subgraph that should be used as the basis of the graft:

```yaml
    description: My really very good subgraph
    graft:
      base: Qm...
      block: 123456
```

- It is a deploy-time error if the source subgraph does not exist.
- The block at which the graft happens must be final. Since requiring that
  the graft point is at least `ETHEREUM_REORG_THRESHOLD` would require
  waiting for an unreasonable amount of time before grafting becomes
  possible, grafting will be allowed onto any block the base subgraph has
  already indexed. If an attempt is made to revert that block, the grafted
  subgraph will fail with an error.
- The base subgraph and the grafted subgraph must have GraphQL schemas that
  are compatible with copying data. The grafted subgraph can add and remove
  entity types and attributes from the base subgraph's schema, as well as
  make non-nullable attributes in the base subgraph nullable. In all other
  respects, the two schemas must be identical.
- It is an error to graft onto a base subgraph that uses JSONB storage
  (i.e., the base subgraph must use relational storage)

## Implementation

- All the magic will happen inside `Store.create_subgraph_deployment` which
  will receive an additional argument of type `DeploymentMode` that
  encapsulates the data from the subgraph manifest described above
- Subgraph creation will first set up the subgraph metadata as we do today,
  and then
  1. create the `Layout` and run its DDL in the database
  2. copy the data from the source subgraph into the grafted subgraph using
     `insert into new.table select * from old.table` It would probably be
     faster to copy the data with `create table new.table as table
     old.table`; that requires that we manually create constraints and
     indexes after copying the data. As that requires a deeper change in
     the code that creates the database schema for a subgraph, we'll leave
     that as a possible improvement
  3. copy dynamic data sources from the source subgraph. The copying will
     create new `DynamicEthereumContractDataSource` entities, and all their
     child entities, by creating a new id for the dynamic data source and
     adjusting the id's of child entities accordingly
  4. rewind the new subgraph to the desired block by reverting to that
     block (both data and metadata)
  5. set the `latestEthereumBlockHash` and `latestEthereumBlockNumber` of
     the new subgraph's `SubgraphDeployment` to the desired block
- Since we are changing the manifest format, `graph-cli` will need
  validation for that. `graph-node`'s manifest validation will also need to
  be updated.

## Tests

Details TBD, but the tests need to cover at least subgraph manifests
without a `deployment` and each of the supported `modes`

## Migration

This is purely additive and requires no migration.

## Documentation

The `deployment` field in subgraph manifests will be documented [in the
public
docs](https://thegraph.com/docs/define-a-subgraph#the-subgraph-manifest)
and the [manifest reference](https://github.com/graphprotocol/graph-node/blob/master/docs/subgraph-manifest.md)

## Implementation Plan

- (1) Compare two `Layout`s and report some sort of useful error if they are
  not equal/compatible
- (1) Copy subgraph data after creation of the `Layout`; copy dynamic data
  sources and adjust their id's
- (0.5) Modify `SubgraphDeployment` entity to have optional `graftBase`
  and `graftBlock` attributes and store them (requires database migration)
- (0.5) Rewind an existing subgraph, and set the subgraph's `latestEthereumBlock`
  to that
- (0.5) Extract graft details from the subgraph manifest and pass that into
  `create_subgraph`. Have `create_subgraph` use the above steps to copy the
  source subgraph if the deployment mode asks for that. Check that source
  subgraph has advanced past the graft point/block
- (0.5) Change revert logic to make sure a revert never goes past a graft
  point
- (0.5) Validate `graft` construct in subgraph manifest
- (0.5) Update documentation
- (0.5) Update validation in `graph-cli`

## Open Questions

- Should grafted subgraphs be marked somehow as it will be very hard to
  recreate them in other `graph-node` installations?  Recreating such a
  subgraph would require deploying the source subgraph, letting it index
  past the source block and then deploying the target subgraph. Since the
  source subgraph could itself be the result of grafting, it might be
  necessary to perform these operations recursively
