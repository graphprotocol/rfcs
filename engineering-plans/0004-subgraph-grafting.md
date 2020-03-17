# PLAN-0004: Subgraph Grafting

<dl>
  <dt>Author</dt>
  <dd>David Lutterkort</dd>

  <dt>Implements</dt>
  <dd>No RFC</dd>

  <dt>Engineering Plan pull request</dt>
  <dd><a href="URL">URL</a></dd>

  <dt>Date of submission</dt>
  <dd>2020-03-17</dd>

  <dt>Date of approval</dt>
  <dd>YYYY-MM-DD</dd>

  <dt>Approved by</dt>
  <dd>First Person, Second Person</dd>
</dl>

## Contents

<!-- toc -->

## Summary

This feature makes it possible to resume indexing of a failed subgraph by
creating a new subgraph that uses the failed subgraph's data up to a block
that the subgraph writer deems reliable, and continue indexing using the
new subgraph's mappings. We call this operation *grafting* subgraphs.

## Subgraph manifest changes

The user-facing control for this feature will be a new optional
`deployment` field in the subgraph's manifest. It can take one of three
forms, and defaults to `create`:

    ---
    description: My really very good subgraph
    # Option 1: (default) The deployment starts with no data in the database
    deployment:
      mode: create
    # Option 2: (copy) The deployment starts with a copy of the data from
    # the given subgraph at the given block
    deployment:
      mode: copy
      subgraph: Qm...
      block: 123456
    # Option 3: (reuse) The deployment reuses the data of an existing subgraph
    # in place. This mode implicitly deletes the source deployment, and any
    # subgraph version using the source deployment will be switched to use the
    # new deployment
    deployment:
      mode: reuse
      subgraph: Qm...
      block: 123456

- For `copy` and `reuse` it is a deploy-time error if the source subgraph
  does not exist or has not indexed the given `block` yet
- To start with, for grafting, the source subgraph and the grafted subgraph
  must have identical GraphQL schemas. We can expand/relax this requirement
  over time.
- It is an error to graft onto a source subgraph that uses JSONB storage
  (i.e., the source subgraph must use relational storage)
- For expediency, we will first implement `copy` and see if `reuse` is
  actually needed - the big advanatge of `reuse` is that it does not
  require copying data and will therefore be faster to initialize a
  subgraph, but it remains to be seen how much of a factor that is in
  practice

## Implementation

- All the magic will happen inside `Store.create_subgraph_deployment` which
  will receive an additional argument of type `DeploymentMode` that
  encapsulates the data from the subgraph manifest described above
- Subgraph creation will first set up the subgraph metadata as we do today,
  and then
    - For `copy`
        1. create the `Layout` and run its DDL in the database
        2. copy the data from the source subgraph into the grafted subgraph
           using `insert into new.table select * from old.table` It would
           probably be faster to copy the data with `create table new.table
           as table old.table`; that requires that we manually create
           constraints and indexes after copying the data. As that requires
           a deeper change in the code that creates the database schema for
           a subgraph, we'll leave that as a possible improvement
        3. copy dynamic data sources from the source subgraph. The copying
           will create new `DynamicEthereumContractDataSource` entities,
           and all their child entities, by creating a new id for the
           dynamic data source and adjusting the id's of child entities
           accordingly
        4. rewind the new subgraph to the desired block by reverting to
           that block (both data and metadata)
        5. set the `latestEthereumBlockHash` and
           `latestEthereumBlockNumber` of the new subgraph's
           `SubgraphDeployment` to the desired block
    - For `reuse`:
        1. change the `deployment` field of the source subgraph's dynamic
           data sources to point at the new subgraph
        2. rename the database schema `sgdNNN` for the source subgraph's
           data to the target subgraph's name using `alter schema sgdOLD
           rename to sgdNEW`
        3. rewind the new subgraph to the desired block
        4. remove the metadata for the old subgraph
        5. delete the entry for `sgdOLD` from `deployment_schemas` Strictly
           speaking, we also need to remove the entry for `sgdOLD` from
           each `graph-node`'s `Store.subgraph_cache` and
           `Store.storage_cache` but do not have a mechanism in place for
           that. We should check, but leaving these entries in the cache
           should be ok; it might lead to less than helpful error messages
           when trying to query the old subgraph but should cause no harm
           beyond that

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

The plan is described above under 'Implementation'

## Open Questions

- Should subgraphs deployed with `copy` or `reuse` be marked somehow as it
  will be very hard to recreate them in other `graph-node` installations?
  Recreating such a subgraph would require deploying the source subgraph,
  letting it index past the source block and then deploying the target
  subgraph. Since the source subgraph could itself be the result of
  grafting, it might be necessary to perform these operations recursively
