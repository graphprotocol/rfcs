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

Synced subgraphs no longer halt on mapping errors, but are marked as unhealthy. Unhealthy subgraphs return an error along with the data when queried.

The indexing status API is improved and documented.

## Goals & Motivation

Currently, an error in a WASM trigger handler will cause a subgraph to be marked as failed and stop syncing. The proposal is for it to discard any changes made by the failed handler, mark the subgraph with the new 'unhealthy' status and continue syncing. In the cases where the error is catastrophic and all handlers start erroing, this will not do much, but in the cases where the subgraph can continue operational with a bit of missing data, this will allow applications to continue running on up-to-date data and give developers more time to fix their subgraph.

Less catastrophic errors do have a downside: It's harder to notice that anything is wrong at all. We'll include an error in the graphql response for unhealthy and failed subgraphs. We will improve and document the indexing status API, so that developers can programmatically check details on a subgraph's health and build their own alerts and dashboards.

## Urgency

Sooner is better than later.

## Terminology

A new subgraph health status called "unhealthy" is added, so instead of a subgraph being failed or not, it can be healthy, unhealthy or failed.

## Detailed Design

### Error handling

Subgraphs now have three possible health status: 'Healthy' is normal operation, 'Unhealthy' is syncing with errors, and 'Failed' is halted due to an error.

If a WASM handler terminates with a trap, the subgraph becomes unhealthy. The node discards all entity operations and data source operations done by the handler and continues syncing.

For subgraphs that are not yet synced, any handler error is still a failure and halts syncing, so errors in pending versions will not make their way to the current version. 

When the node starts, we currently reset the failed flag on subgraphs. Failed subgraphs will still be reset to healthy to attempt to recover, but unhealthy subgraphs are kept unhealthy, since the subgraph will already be past the errors.

Any graphql queries to an unhealthy subgraph will include an error of the form:

```
{
	message: "unhealthy"
}
```

Or `message: "failed"` if failed. So that applications will always be aware that queries may have inconsistent data. Developers should refer to the indexing status API for details on the errors.

### Indexing Status API

The indexing status API will be revamped.

The following top level queries are added:

    indexingStatusForCurrentVersion(subgraphName: String!): SubgraphIndexingStatus
    indexingStatusForPendingVersion(subgraphName: String!): SubgraphIndexingStatus

Which returns null if the name is not found, and returns the status for the current version of the subgraph otherwise.

A field `lastHealthyBlock`  is added to record the block previous to the first error.

Errors are made more than just a string with the `Error` type.

The complete indexing status API schema is:

```graphql
type Query {
  indexingStatusForCurrentVersion(subgraphName: String!): SubgraphIndexingStatus
  indexingStatusForPendingVersion(subgraphName: String!): SubgraphIndexingStatus
  indexingStatusesForSubgraphName(subgraphName: String!): [SubgraphIndexingStatus!]!
  indexingStatuses(subgraphs: [String!]): [SubgraphIndexingStatus!]!
}

type SubgraphIndexingStatus {
  subgraph: String!
  synced: Boolean!
  health: Health!
  
  # Sorted from first to last, limited to 1000
  errors: [Error!]!
  chains: [ChainIndexingStatus!]!
  node: String!
}

# Fields are moved from `EthereumIndexingStatus` to the interface.
interface ChainIndexingStatus {
  network: String!
  chainHeadBlock: Block
  earliestBlock: Block
  latestBlock: Block
  lastHealthyBlock: Block
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
	handler: String
}

enum Health {
  "Subgraph syncing normally"
  healthy,
  "Subgraph syncing but with errors"
  unhealthy,
  "Subgraph halted due to errors"
  failed,
}
```

Since the indexing API is a view to the subgraph metadata, an `Error` type is also added to the metadata where subgraph errors are recorded. The `health` field is also added to the metadata, replacing the  `failed` flag which is deprecated.

## Compatibility

This does incompatible changes to the indexing status API, however that API was so far experimental and undocumented.

The behavior of synced subgraphs when encountering an error is changed to keep indexing. Our consensus is that the current behavior is useful to nobody and this can only be an improvement.

## Drawbacks and Risks

- Changing the behavior of errors on synced subgraphs has risks. There may be unanticipated consequences which may slip through testing. We may have gotten our rationale wrong and users may find the new behavior worse for factors we did not consider.

## Alternatives & Extensions

- Don't change the default behavior, and put make the "continue on failure" mode opt-in through a flag on the manifest. With the possible intention of making it the default in the future. This exposes us to less risk when rolling this out. The downsides are that adoption and feedback will be slower and developers will need to update their subgraphs in order to benefit.
- Future extension: Allow mappings to define a trap handler that catches traps on a specific trigger handler. The mapping can then decide whether the subgraph should fail, be set as unhealthy, or just continue normally.

## Open Questions

No unresolved questions.

