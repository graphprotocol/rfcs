# PLAN-0002: Ethereum Tracing Cache

<dl>
  <dt>Author</dt>
  <dd>Zachary Burns</dd>

  <dt>Implements</dt>
  <dd><a href="../rfcs/0002-ethereum-tracing-cache.md">RFC-0002 Ethereum Tracing Cache</a></dd>

  <dt>Engineering Plan pull request</dt>
  <dd><a href="https://github.com/graphprotocol/rfcs/pull/9">https://github.com/graphprotocol/rfcs/pull/9</a></dd>

  <dt>Date of submission</dt>
  <dd>2019-12-20</dd>

  <dt>Date of approval</dt>
  <dd>2020-01-07</dd>

  <dt>Approved by</dt>
  <dd>Jannis Pohlmann, Leo Yvens</dd>
</dl>

## Summary

Implements RFC-0002: Ethereum Tracing Cache

## Implementation

These changes happen within or near `ethereum_adapter.rs`, `store.rs` and `db_schema.rs`.

### Limitations
The problem of reorg turns out to be a particularly tricky one for the cache, mostly due to ranges of blocks being requested rather than individual hashes. To sidestep this problem, only blocks that are older than the reorg threshold will be eligible for caching.

Additionally, there are some subgraphs which may require traces from all or a substantial number of blocks and don't make effective use of filtering. In particular, subgraphs which specify a call handler without a contract address fall into this category. In order to prevent the cache from bloating, any use of Ethereum traces which does not filter on a contract address will bypass the cache.

### EthereumTraceCache

The implementation introduces the following trait, which is implemented primarily by `Store`.

```rust
use std::ops::RangeInclusive;
struct TracesInRange {
    range: RangeInclusive<u64>,
    traces: Vec<Trace>,
}

pub trait EthereumTraceCache: Send + Sync + 'static {
    /// Attempts to retrieve traces from the cache. Returns ranges which were retrieved.
    /// The results may not cover the entire range of blocks. It is up to the caller to decide
    /// what to do with ranges of blocks that are not cached.
    fn traces_for_blocks(contract_address: Option<H160>, blocks: RangeInclusive<u64>
        ) -> Box<dyn Future<Output=Result<Vec<TracesInRange>, Error>>>;
    fn add(contract_address: Option<H160>, traces: Vec<TracesInRange>);
}
```

#### Block schema

Each cached block will exist as its own row in the database in an `eth_traces_cache` table.

```rust
eth_traces_cache(id) {
  id -> Integer,
  network -> Text,
  block_number: Integer,
  contract_address: Bytea,
  traces -> Jsonb,
}
```

A multi-column index will be added on network, block_number, and contract_address.

It can be noted that in the `eth_traces_cache` table, there is a very low cardinality for the value of the network row. It is inefficient for example to store the string `mainnet` millions of times and consider this value when querying. A data oriented approach would be to partition these tables on the value of the network. It is expected that hash partitioning available in Postgres 11 would be useful here, but the necessary dependencies won't be ready in time for this RFC. This may be revisited in the future.

#### Valid Cache Range
Because the absence of trace data for a block is a valid cache result, the database must maintain a data structure indicating which ranges of the cache are valid in an `eth_traces_meta` table. This table also enables eventually implementing cleaning out old data.

This is the schema for that structure:
```rust
id -> Integer,
network -> Text,
start_block -> Integer,
end_block -> Integer,
contract_address -> Nullable<Bytea>,
accessed_at -> Date,
```

When inserting data into the cache, removing data from the cache, or reading the cache, a serialized transaction must be used to preserve atomicity between the valid cache range structure and the cached blocks. Care must be taken to not rely on any data read outside of the serialized transaction, and for the extent of the serialized transaction to not span any async contexts that rely on any `Future` outside of the database itself. The definition of the `EthereumTraceCache` trait is designed to uphold these guarantees.

In order to preserve space in the database, whenever the valid cache range is added it will be added such that adjacent and overlapping ranges are merged into it.

### Cache usage
The primary user of the cache is `EtheriumAdapter<T>` in the `traces` function. 

The correct algorithm for retrieving traces from the cache is surprisingly nuanced. The complication arises from the interaction between multiple subgraphs which may require a subset of overlapping contract addresses. The rate at which indexing proceeds of these subgraphs can cause different ranges of the cache to be valid for a contract address in a single query.

We want to minimize the cost of external requests for trace data. It is likely that it is better to...
 * Make fewer requests
 * Not ask for trace data that is already cached
 * Ask for trace data for multiple contract addresses within the same block when possible.

 There is one flow of data which upholds these invariants. In doing so it makes a tradeoff of increasing latency for the execution of a specific subgraph, but increases throughput of the whole system.

 Within this graph:
 * Edges which are labelled refer to some subset of the output data.
 * Edges which are not labelled refer to the entire set of the output data.
 * Each node executes once for each contiguous range of blocks. That is, it merges all incoming data before executing, and executes the minimum possible times.
 * The example given is just for 2 addresses. The actual code must work on sets of addresses.

 ```mermaid
 graph LR;
    A[Block Range for Contract A & B]
    A --> |Above Reorg Threshold| E
    D[Get Cache A]
    A --> |Below Reorg Threshold A| D
    A --> |Below Reorg Threshold B| H
    E[Ethereum A & B]
    F[Ethereum A]
    G[Ethereum B]
    H[Get Cache B]
    D --> |Found| M
    H --> |Found| M
    M[Result]
    D --> |Missing| N
    H --> |Missing| N
    N[Overlap]
    N --> |A & B| E
    N --> |A| F
    N --> |B| G
    E --> M
    K[Set Cache A]
    L[Set Cache B]
    E --> |B Below Reorg Threshold| L
    E --> |A Below Reorg Threshold| K
    F --> K
    G --> L
    F --> M
    G --> M
 ```

 
This construction is designed to make the fewest number of the most efficient calls possible. It is not as complicated as it looks. The actual construction can be expressed as sequential steps with a set of filters preceding each step.

### Useful dependencies
The feature deals a lot with ranges and sets. Operations like sum, subtract, merge, and find overlapping are used frequently. [nested_intervals](https://crates.io/crates/nested_intervals) is a crate which provides some of these operations.


## Tests

### Benchmark
A temporary benchmark will be added for indexing a simple subgraph which uses call handlers. The benchmark will be run in these scenarios:
* Sync before changes
* Re-sync before changes 
* Sync after changes
* Re-sync after changes

### Ranges
Due to the complexity of the resource minimizing data workflow, it will be useful to have mocks for the cache and database which record their calls, and check that expected calls are made for tricky data sets.

### Database
A real database integration test will be added to test the add/remove from cache implementation to verify that it correctly merges blocks, handles concurrency issues, etc.


## Migration

None

## Documentation

None, aside from code comments

## Implementation Plan: ##

These estimates inflated to account for the author's lack of experience with Postgres, Ethereum, Futures0.1, and The Graph in general.

- (1) Create benchmarks
- Postgres Cache
  - (0.5) Block Cache
  - (0.5) Trace Serialization/Deserialization
  - (1.0) Ranges Cache
  - (0.5) Concurrency/Transactions
  - (0.5) Tests against Postgres
- Data Flow
  - (3) Implementation
  - (1) Unit tests
- (0.5) Run Benchmarks

Total: 8


