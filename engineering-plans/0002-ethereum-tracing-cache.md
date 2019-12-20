# PLAN-0002: Ethereum Tracing Cache

<dl>
  <dt>Author</dt>
  <dd>Zachary Burns</dd>

  <dt>Implements</dt>
  <dd><a href="../rfcs/0002-ethereum-tracing-cache.md">RFC-0002 Ethereum Tracing Cache</a></dd>

  <dt>Engineering Plan pull request</dt>
  <dd><a href="https://github.com/graphprotocol/rfcs/pull/9">https://github.com/graphprotocol/rfcs/pull/9</a></dd>

  <dt>Obsoletes (if applicable)</dt>
  <dd>None</dd>

  <dt>Date of submission</dt>
  <dd>2019-12-20</dd>

  <dt>Date of approval</dt>
  <dd>YYYY-MM-DD</dd>

  <dt>Approved by</dt>
  <dd>First Person, Second Person</dd>
</dl>

## Summary

Implements RFC-0002: Ethereum Tracing Cache

## Implementation

These changes happen within `ethereum_adapter.rs`, `store.rs` and `db_schema.rs`.

### ServiceCache trait
EthereumCallCache will be renamed to ServiceCache and have roughly this signature:
```rust
trait ServiceCache {
    // Provided function
    fn get<T: Deserialize>(
        // Arguments to fetch
        args: impl Hash,
        // Perform the work if not found in the cache
        f: impl Future<Output=Result<Vec<u8>, Error>>,
        // Uniquely identify the cache and it's version number. This is essentially a namespace.
        cache_id: &str
    ) -> Box<dyn Future<Output=Result<T, Error=Error>>>;

    // Required functions
    fn try_store(key: [u8; 16], data: &[u8]) -> Box<dyn Future<Output=Result<(), Error>>>;
    fn try_get(key: [u8; 16]) -> Box<dyn Future<Output=Result<Option<Vec<u8>, Error>>>>;
}
```

This signature aims to fix 2 problems:
  * Today, EthereumCallCache get_call is blocking, and used by code which returns a Future. This is [not recommended for async code](https://doc.rust-lang.org/stable/std/future/trait.Future.html#runtime-characteristics) and can cause problems for both runtimes and future combinators. The new signature allows for the database work to be migrated to use non-blocking apis, or to be spawned in a threadpool.
  * This signature moves the `get or call then set` logic into the trait where it only need be written once, rather than everywhere it is used. This also allows for certain optimizations, like only calculating the hash of the arguments once.

The Hash trait would be from [digest-hash](https://crates.io/crates/digest-hash/0.3.0) allowing tiny_keccak use for data structures, taking over the roll that `contract_call_id` fills today.

`contract_call` and `traces` would be modified to use this new trait.

### Database Schema
`eth_call_cache` would be changed to:
```
service_cache (id) {
        id -> Bytea,
        return_value -> Bytea,
    }
```
The `contract_address` and `block_number` are removed, as they are not read today and are specific to eth_call.

`eth_call_meta` would be changed to:
```
service_cache_meta (id) {
        id -> Bytea,
        accessed_at -> Date,
    }
```

## Tests

A temporary benchmark will be added for indexing a simple subgraph which uses call handlers. The benchmark will be run in these scenarios:
* Sync before changes
* Re-sync before changes 
* Sync after changes
* Re-sync after changes


## Migration

The old `eth_call_cache` and `eth_call_meta` tables will be deleted.

## Documentation

None, aside from code comments

## Implementation Plan
- [ ] Create benchmarks
- [ ] Refactor existing `EthereumCallCache` into `ServiceCache` trait
- [ ] Use `ServiceCache` trait for traces
- [ ] Run benchmarks
- [ ] Migrate/delete old database schema


**Estimates:**
- (1) Create benchmarks
- (2) Refactor existing `EthereumCallCache` - into `ServiceCache` trait
- (1) Use `ServiceCache` trait for traces
- (0.5) Run benchmarks
- (0.5) Migrate/delete old database schema

Total: 5

**Phases:**
None

**Integration:**
Potential integration after refactoring existing eth_cache to use new trait.

## Open Questions

None