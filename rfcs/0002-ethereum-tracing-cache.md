# RFC-0002: Ethereum Tracing Cache

<dl>
  <dt>Author</dt>
  <dd>Zac Burns</dd>

  <dt>RFC pull request</dt>
  <dd><a href="https://github.com/graphprotocol/rfcs/pull/4">https://github.com/graphprotocol/rfcs/pull/4</a></dd>

  <dt>Obsoletes (if applicable)</dt>
  <dd>None</dd>

  <dt>Date of submission</dt>
  <dd>2019-12-13</dd>

  <dt>Date of approval</dt>
  <dd>YYYY-MM-DD</dd>

  <dt>Approved by</dt>
  <dd>First Person, Second Person</dd>
</dl>

## Summary

This RFC proposes the creation of a local Ethereum tracing cache to speed up indexing of subgraphs which use block and/or call handlers.

## Motivation

When indexing a subgraph that uses block and/or call handlers, it is necessary to extract calls from the trace of each block that a Graph Node indexes. It is expensive to acquire and process traces from Ethereum nodes in both money and time.

When developing a subgraph it is common to make changes and deploy those changes to a production Graph Node for testing. Each time a change is deployed, the Graph Node must re-sync the subgraph using the same traces that were used for the previous sync of the subgraph. The cost of acquiring the traces each time a change is deployed impacts a subgraph developer's ability to iterate and test quickly.

## Urgency

None

## Terminology

_Ethereum cache_: The new API proposed here.

## Detailed Design

There is an existing `EthereumCallCache` for caching `eth_call` built into Graph Node today. This cache will be extended to support traces, and renamed to `EthereumCache`.

## Compatibility

This change is backwards compatible. Existing code can continue to use the parity tracing API. Because the cache is local, each indexing node may delete the cache should the format or implementation of caching change. In this case of invalidated cache the code will fall back to existing methods for retrieving a trace and repopulating the cache.

## Drawbacks and Risks

Subgraphs which are not being actively developed will incur the overhead for storing traces, but will not ever reap the benefits of ever reading them back from the cache.

If this drawback is significant, it may be necessary to extend `EthereumCache` to provide a custom score for cache invalidation other than the current date. For example, `trace_filter` calls could be invalidated based on the latest update time for a subgraph requiring the trace. It is expected that a subgraph which has been updated recently is more likely to be updated again soon then a subgraph which has not been recently updated.

## Alternatives

None

## Open Questions

None