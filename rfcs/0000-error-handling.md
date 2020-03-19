# RFC-0000: Subgraph error handling and reporting

<dl>
  <dt>Author</dt>
  <dd>Leonardo Y. Schwarzstein</dd>
    <dt>RFC pull request</dt>
  <dd>URL</dd>

  <dt>Date of submission</dt>
  <dd>2020-03-19</dd>

  <dt>Date of approval</dt>
  <dd>YYYY-MM-DD</dd>

  <dt>Approved by</dt>
  <dd>First Person, Second Person</dd>
</dl>

## Contents

<!-- toc -->

## Summary

Synced subgraphs no longer stop syncing on errors. Failed subgraphs return an error along with the data when queried.

The indexing status API is improved and documented.

## Goals & Motivation

Currently, an error in a WASM trigger handler will cause a subgraph to be marked as failed and stop syncing. The proposal is for it still be marked as failed, but discard any changes made by the failed handler and attempt to continue. In the cases where the failure is catastrophic and all handlers start failing, this will not do much, but in the cases where the subgraph can continue operational with a bit of missing data, this will allow applications to continue running on up-to-date data and give developers more time to fix their subgraph.

Less catastrophic failures do have a downside: It's harder to notice that anything is wrong at all. We'll include an error in the graphql response for failed subgraphs. We will improve and document the indexing status API, so that developers can programmatically check details on a subgraph's health and build their own alerts and dashboards.

## Urgency

Sooner is better than later.

## Terminology

No new terminology, but the semantics of the "Failed" status are altered.

## Detailed Design

### Error handling

If a WASM handler terminates with a trap, that is an error. The node discards all entity operations and data source operations done by the handler and continue syncing.

For subgraphs that are not yet synced, any handler error halts syncing, so errors in pending versions will not make their way to the current version. 

When the node starts, we currently reset the failed flag on subgraphs. The flag is no longer be reset on synced subgraphs, since the subgraph will already be past the errors.

Any graphql queries to a failed subgraph will include an error of the form:

```
{
	message: "Subgraph failed"
}
```

So that applications will always be aware that queries may have inconsistent data. Developers should refer to the indexing status API for details on the errors.

### Indexing Status API

The indexing status API will be revamped.

The subgraph indexing status API currently has the top level query:

    indexingStatusesForSubgraphName(subgraphName: String!): [SubgraphIndexingStatus!]!

This is changed to:

    indexingStatusesForSubgraphName(subgraphName: String!): SubgraphIndexingStatus

Which returns null if the name is not found, and returns the status for the current version of the subgraph otherwise.

A field `failedAfter`  is added to record the block previous to the first error.

Errors are made more than just a string with the `Error` type.

The complete indexing status API schema is:

```graphql
type Query {
  indexingStatusesForSubgraphName(subgraphName: String!): SubgraphIndexingStatus
  indexingStatuses(subgraphs: [String!]): [SubgraphIndexingStatus!]!
}

type SubgraphIndexingStatus {
  subgraph: String!
  synced: Boolean!
  errors: [Error!]! # Sorted from first to last
  chains: [ChainIndexingStatus!]!
  node: String!
}

# Fields are moved from `EthereumIndexingStatus` to the interface.
interface ChainIndexingStatus {
  network: String!
  chainHeadBlock: Block
  earliestBlock: Block
  latestBlock: Block
  failedAfter: Block
}

type EthereumIndexingStatus implements ChainIndexingStatus { }

# Renamed from `EthereumBlock` to `Block`
type Block {
  hash: Bytes!
  number: BigInt!
}

type Error {
	message: String!
	
	# Context for the error.
	network: String!
	block: Block!
	transaction: Bytes
	handler: String
}
```

Since the indexing API is a view to the subgraph metadata, an `Error` type is also added to the metadata where subgraph errors are recorded.

## Compatibility

This does incompatible changes to the indexing status API, however that API was so far experimental and undocumented.

The behavior of synced, failed subgraphs is changed to keep indexing. Our consensus is that the current behavior is useful to nobody and this can only be an improvement.

## Drawbacks and Risks

- Changing the behavior of failed subgraphs has risks. There may be unanticipated consequences which may slip through testing. We may have gotten our rationale wrong and users may find the new behavior worse for factors we did not consider.

## Alternatives

- Don't change the default behavior, and put make the "continue on failure" mode opt-in through a flag on the manifest. With the possible intention of making it the default in the future. This exposes us to less risk when rolling this out. The downsides are that adoption and feedback will be slower and developers will need to update their subgraphs in order to benefit.

## Open Questions

- The field name `failedAfter` is up for bikeshedding, aternatives: `lastHealthyBlock`, `blockBeforeFailure`, `lastGoodBlock`...

