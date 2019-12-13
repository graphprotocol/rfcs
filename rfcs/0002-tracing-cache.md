# RFC-0000: Template

<dl>
  <dt>Author</dt>
  <dd>Zac Burns</dd>

  <dt>RFC pull request</dt>
  <dd><a href="URL">URL</a></dd>

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

This RFC proposes the creation of a local etherium tracing cache to speed up indexing of subgraphs which use callHandler.

## Motivation

If a subgraph has callHandlers specified, it is necessary to extract calls from a trace of each block that a graph node indexes. It is expensive to acquire and process traces from etherium nodes, both in money and time.

A production graph node may index multiple subgraphs and repeatedly access the same traces many times - especially for recent blocks. Having that data on hand in a pre-processed and easily filtered format may substantially reduce the cost of acquiring and converting trace data into triggers relevant to a subgraph.

## Urgency

None

## Terminology

None.

## Detailed Design

The design comes in 2 parts. The caching hierarchy, and the easily filtered data format.

### Cache Hierarchy
In `etherium_adapter.rs` is a function `traces`, which unconditionally makes a call to eth.web3. A hierarchical cache would be inserted here with 3 layers - In-Memory, On-Disk, and Fetch.

#### In-Memory Layer
At the top level of the cache is the in-memory cache. For a graph node which is indexing multiple subgraphs in parallel, it may be likely that they require the latest blocks simultaneously. By storing the buffer containing immutable trace data for a block in a rc::Weak, a low-risk, low-cost cache is introduced. The most recent trace, and all currently in-use traces would be maintained.

#### On-Disk Layer
The In-Memory layer is not expected to remain in cache for very long in an effort to reduce memory consumption. An on-disk layer helps to prevent more calls to the fetch layer than is absolutely necessary.

This on-disk layer would be implemented as a binary file store with a cache eviction policy over postgres. It is expected that the most recent blocks would be indexed the most frequently, so the scoring mechanism for the cache would probably be some mix between LRU, block recency, and block finality.

#### Fetch layer
The existing code in the `traces` function would be renamed to `fetch_traces` and would be the bottom layer of the cache. At first, this would exist like it does today. In the future this may be augmented to either delegate to fetching from a parity API or to calculate traces on demand using a local etherium node, depending on which resources are currently the bottleneck.

#### Futures re-use
In order to prevent concurrent loads of data from one cache layer to another (Eg: multiple concurrent calls to the underlying eth.web3 API with the same parameters) the hierarchical cache would have a mechanism to delegate multiple tasks to the same future with a NonReady state.

### Binary Format

Trace data is extremely large. Compressing the data directly impacts how many traces for blocks we can keep at any layer of the cache.

Ideally our binary format would have the following properties:
* Ability to skip records which are filtered out without parsing
* Zero copy to allow for a subset of a record's fields to be read when deciding whether to filter it. (Alternatively, easily separated hot/cold storage)
* Little to no overhead for the schema, as there are no optional fields and backwards compatibility can be amortized into the collection
* Processable as a linear stream with minimal cache misses, ideally not requiring the entire file to be loaded at once.
* Move redundant data outside of the Trace (eg: Block Number)

JSON is the format the data is delivered in today. JSON is neither compressed nor fast: https://eng.uber.com/trip-data-squeeze/. JSON does not have any of the properties of our ideal format. The entirety of a JSON file needs to be parsed whether or not all of it is actually used. JSON's self-description and flexibility come at a large cost for data which has a schema. As much of the data in a trace is binary data or data which does not have an established representation in JSON, additional overheads are incurred in encoding representations of traces in the primitives that JSON supports.

Flatbuffers seems like a good initial candidate here which makes reasonable tradeoffs between skip/zero-copy and compression ratios. Significant gains over Flatbuffers are possible with a custom format, but this is not likely a good place to spend time in the short term. 

This is subject to change if it is decided that filtering/indexing should happen within postgres.

## Compatibility

This change is backwards compatible. Existing code can continue to use the parity tracing API. Because the cache is local, each indexing node may delete the cache should the format or implementation of caching change. In this case of invalidated cache the code will fall back to existing methods for retrieving a trace and repopulating the cache.

## Drawbacks and Risks

In some cases, a graph node may not frequently get a cache hit when requesting a trace. If cache hits are relatively rare, the additional cost of pre-processing the data and caching it would only add to the overhead that exists today. It is currently up to a combination of the implementation details of block_stream and the set of subgraphs being indexed as to whether this strategy would be a win for a particular graph node. It may be necessary to monitor statistics about the behavior of the caching layer to keep tabs on whether this change continues to be a win in a production environment.

To mitigate this risk, it may be worth frontloading work on the cache hierarchy to gather statistics about block trace access patterns and how frequently cache hits are expected for existing graph nodes.

## Alternatives

A general purpose etherium cache has been suggested. It is likely that much of the work here would be able to be repurposed for such a cache.

## Open Questions

What, if any, ordering constraints exist on triggers across multiple contract addresses?
