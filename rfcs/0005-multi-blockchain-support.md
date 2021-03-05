# RFC-0005: Multi-Blockchain Support

<dl>
  <dt>Author</dt>
  <dd>Jannis Pohlmann and Leonardo Yvens</dd>

  <dt>RFC pull request</dt>
  <dd><a href="https://github.com/graphprotocol/rfcs/pull/21">https://github.com/graphprotocol/rfcs/pull/21</a></dd>

  <dt>Date of submission</dt>
  <dd>2019-12-20</dd>

  <dt>Date of approval</dt>
  <dd>2021-03-05</dd>
</dl>

## Summary

Multi-blockchain support allows subgraphs to index data from blockchains other
than Ethereum.

## Goals & Motivation

The main objective of multi-blockchain support is to add support for subgraphs
to index data from a growing number of blockchains. At the time of writing this
RFC, only Ethereum and blockchains compatible with the Ethereum JSON-RPC API are
supported.

This feature unlocks the same use cases on other blockchains that The Graph has
made possible on Ethereum, including indexing historical data, interacting with
smart contracts (if supported by the respective blockchain), handling reorgs (if
applicable) and triggering off of blockchain-specific features like events or
transactions.

## Urgency

Since this is a big feature and one that requires a substantial refactoring of
the existing codebase, and _may_ introduce breaking changes, it would be good to
start working on this as soon as possible.

## Terminology

The feature proposed in this RFC is referred to as **multi-blockchain support**.
Other terms that are introduced in this RFC or are relevant when talking about
it are listed below.

- **Network** - a decentralized technology or similar, such as Ethereum,
  Substrate, or IPFS.
- **Blockchain** - a blockchain technology, such as Ethereum or Substrate.
- **Chain** - a specific instance of a _blockchain_, such as Ethereum mainnet.
- **Chain subgraph** - a built-in subgraph that indexes data intrinsic to a
  _chain_, such as its blocks, transactions, accounts and balances.
- **Trigger** - a _chain_ event or similar that can be observed on a _chain_ and
  that can be used to trigger indexing work via a subgraph mapping. Examples:
  blocks, transactions, contract method calls, logs/events.

## Detailed Design

Multi-blockchain support touches almost every aspect of The Graph, including:

- Subgraph manifests and data sources
- Code generation
- Mapping APIs and types
- Subgraph indexing
- Storage

This section specifies how all of these components are updated to support other
chains and what the integration of additional chains looks like. With this
feature implemented, subgraph developers will be able to define new types of
data sources for supported chains. Indexers will be able to configure which of
these chains they want to enable.

### Subgraph Manifest

The structure of the subgraph manifest is already desigend for being extended
with new types of data sources. An Ethereum data source is currently defined with

```yaml
kind: ethereum/contract
name: Gravity
source:
  chain: mainnet
  address: '2E645469f354BB4F5c8a05B3b30A929361cf77eC'
  startBlock: 5000000
  abi: Gravity
mapping:
  kind: wasm/assemblyscript
  apiVersion: 0.0.3
  eventHandlers:
  - event: NewGravatar(address,address,uint256)
    handler: handleNewGravatar
  callHandlers:
  - function: createGravatar(string,string)
    handler: handleCreateGravatar
  blockHandlers:
  - handler: handleBlock
```

Multi-blockchain opens this up to other blockchains by allowing data sources to
have `kind` values other than `ethereum/contract`. Support for individual
blockchains will require separate RFCs, each of which will introduce one or more
data source types with their own `kind` values such as e.g.
`substrate/contract`.

One change that simplifies working with ABIs and makes the code generated for
them available across data sources is to move ABIs to the top level of the
manifest and mark them with a `kind` as well:

```yaml
kind: subgraph:1.0.0
schema:
  file: ./schema.graphql
bindings:
- kind: ethereum/contract
  name: Gravity
  file: ./abis/Gravity.json
- kind: ethereum/contract
  name: Gravatar
  file: ./abis/Gravatar.json
dataSources:
  ...
templates:
  ...
```

Manifests can be migrated from the old structure to this automatically. It will
however require code changes and a database migration to support top-level ABIs.

### Code Generation

Code generation for schemas remains untouched by multi-blockchain support. Code
generation for ABIs is extended to support more ABIs than just Ethereum's. Based
on the type of ABI in top-level `bindings` section in the manifest, different
code generation logic is applied.

What code is generated exactly will vary from chain to chain and is therefore
not specified in this RFC.

### Mapping APIs and Types

Every chain comes with its own APIs and type system. Ethereum, for example, has
contracts, events, calls, blocks, an API to make contract calls and a variety of
low-level types for tuples/structs, byte arrays and signed/unsigned integers.

The APIs that come with each chain are bundled in a host-exported module named
after the blockchain. This requires the current Ethereum-specific types such as
`EthereumEvent`, `EthereumCall`, `EthereumValue` etc. to be renamed. From a
developer's perspective, each chain module can be imported from
`@graphprotocol/graph-ts` as follows:

```js
import { ethereum, substrate } from '@graphprotocol/graph-ts'

// Types available now:
//
// - ethereum.Block
// - ethereum.Event
// - ethereum.Contract
// - ethereum.Call
// - ethereum.Value
//
// - substrate.Block
// - ...
```

Most, but not all of these, are internal to code generation. The same goes for
low-level types such as Ethereum's `uint256` or `bytes32` type. These are
already sufficiently abstracted behind `Bytes`, `BigInt`, `BigDecimal` and the
code generated for Ethereum tuples.

Additional types may be necessary for other chains, but it expected that the
types implemented today are sufficient to start with.

> **Note:** An alternative to the above could be to split chain modules up into
> their own files or event packages, allowing them to be imported with either
> ```js
> import { ... } from '@graphprotocol/graph-ts/ethereum'
> import { ... } from '@graphprotocol/graph-ts/substrate'
> ```
> or
> ```js
> import { ... } from '@graphprotocol/ethereum'
> import { ... } from '@graphprotocol/substrate'
> ```

Each blockchain integration must include such a module as well as a WASM host
exports implementation to back this module and type conversions from/to
AssemblyScript for low-level types.

> **Note:** Integrating a blockchain initially requires modifications across
> Graph Node, Graph CLI and Graph TS. A more convenient extension concept can
> still be developed later.
>
> - Graph Node: An alternative could be optional dependencies behind feature flags, to pull in
>   support for specific chains at build time.
>
> - Graph CLI: Gluegun, used for Graph CLI, includes an extension framework that
>   would allow to inject code generation and type conversion support from
>   separate NPM packages.
>
> - Graph TS: Separate NPM packages for chain-specific mapping APIs and types as
>   suggested in the previous note would make extending Graph TS easy.

### Subgraph Indexing

This section discusses how the subgraph indexing components and data structures will change, with
some being reused and others becoming chain-specific and therefore abstracted by Rust traits. [This
diagram](https://whimsical.com/multi-blockchain-13ZinTmYAUn5YknGjcZc4e) gives an overview of the
desired architecture for a running subgraph. The arrows describe the flow of data.

Data enters the system by being requested from the chain
itself, for example through JSON-RPC calls. The __adapters__ are the components responsible for
interacting directly with the chain. They are broken up into specialized traits for each component
that requires an adapter, such as the `IngestorAdapter` and `TriggersAdapter`. For performance, the
adapter may cache and index into the local database the data pulled from the chain. Each chain will
have its own DB schema (aka namespace) under which adapters may create any necessary custom DB
tables.

The __block ingestor__ and the __chain store__ will be reused across chains. There will be one
instance of each per chain, an `IngestorAdapter` will provide access chain specific data. Each
`blocks` table will be under the DB schema reserved for the corresponding chain. The
`ChainHeadListener`, which handles the PG channel that notifies of new blocks, will also be reused
across chains.

The __block stream__ is initially chain-specific, though as we implement the first new chains, we
will seek to build a more reusable block stream implementation. A reusable block stream would depend
on a chain-specific `TriggersAdapter`. For performance, the block stream will be made asynchronous
from the instance manager. So it will no longer have direct access to the subgraph store.

The __instance manager__, __mapping runtime__ and related core components will be reused, though it
will take considerable refactoring to move out the Ethereum-specific code.

The __subgraph store__ will be reused since graphql entities behave the same for any chain. Even for
the manifest and dynamic data source metadata we should be able to reuse the existing tables.

#### Data structures

A chain will define associated types such as `Block` and `Trigger` which will be produced by the
adapters and over which the reusable components will need to be made generic. Since the `Chain`
trait aggregates all chain-specific associated types, most of the codebase will be made generic over
a `C: Chain` type parameter. The manifest data structure will also need to be made generic over the
chain data source type.

The `BlockPointer` type is shared across all chains, the existing `EthereumBlockPointer` will be
renamed and its `hash` field will have the type changed from `H256` to `Bytes`.

#### Chain Indexer

Chains may include a built-in subgraph for block data, this is to be detailed in a future RFC. But
this is an optional feature and does not replace the block ingestor.

### Adding Support for a New Blockchain

This section covers what it takes to add support for a new blockchain. Please
note that this RFC specifies the details only to an extent that allow us to
assess whether the direction is feasible. Names and data structure details may
vary in the implementation.

#### Graph Node

Each blockchain integration in Graph Node lives in the `graph-node` repository
under `network/<name>/`, like the already existing `network/ethereum/` support.

Graph Node includes a number of new component traits and data types to enable
multi-blockchain support, sketched below:

```rust
/// A compact representation of a block on a chain.
struct BlockPtr {
    pub number: u64,
    pub hash: Bytes,
}

/// Represents a block in a particular chain. Each chain has its own
/// implementation of the `Block` trait.
trait Block: ToAsc {
    fn number(&self) -> u64 { self.ptr().number }
    fn hash(&self) -> { self.ptr().hash }
    fn ptr(&self) -> BlockPointer;
    fn parent_ptr(&self) -> Option<BlockPointer>;
}

trait DataSource<C: Blockchain> { /* ... */ }
trait IngestorAdapter<C: Blockchain> { /* ... */ }
trait TriggersAdapter<C: Blockchain> { /* ... */ }
trait HostFns<C: Blockchain> { /* ... */ }

trait BlockStream<C: Blockchain> {
      fn new(&self, start_block: BlockPointer, data_sources: Vec<C::DataSource>, trigger_adapter: Arc<C::TriggerAdapter>) -> Result<Self, Error>;
}

/// Represents a blockchain supported by the node.
trait Chain {
    /// The DB schema for the `blocks` table and any custom tables.
    const DB_SCHEMA: &'static str;

    type Block: Block;
    type DataSource: DataSource<Self>;
    type IngestorAdapter: IngestorAdapter<Self>;
    type TriggersAdapter: IngestorAdapter<Self>;
    type BlockStream: BlockStream<Self>;
    type HostFns: HostFns<Self>;
    type ChainTrigger: ChainTrigger<Self>;
    // ...other adapters
}

/// Triggers emitted by a block stream. Triggers can be passed to
/// AssemblyScript and implement a canonical ordering to ensure
/// they are not processed out of order.
trait ChainTrigger<C: Blockchain>: ToAsc + Ord {
    fn matches_data_source(&self, data_source: &C::DataSource) -> bool;
    fn matches_handler(&self, handler: &C::DataSourceHandler) -> bool;
}

/// Combination of a block and subgraph-specific triggers for it.
struct BlockWithTriggers<C: Blockchain> {
    block: Box<C::Block>,
    triggers: Vec<C::Trigger>,
}

/// Events emitted by a block stream.
enum BlockStreamEvent<C: Blockchain> {
    Revert(BlockPointer),
    Block(BlockWithTriggers<C>)
}

/// Common trait for all block streams.
trait BlockStream<C: Blockchain>: Stream<BlockStreamEvent<C>> {}
```

In addition, each blockchain will have to be configured in the main `graph-node`
executable. This RFC does not include details on how this takes place, because
the configuration options may vary from blockchain to blockchain. Ultimately,
what `graph-node` builds before starting up is a set of supported networks /
blockchains, each with a set of supported chains. These are then passed on to,
for instance, the subgraph instance manager.

#### Graph CLI

Each blockchain integration comes in the form of a new JS module in the
`graph-cli` repo under `src/networks/`. The existing Ethereum support is moved
to `src/networks/ethereum/`.

Each support module is required to export the following structure from its
`index.js`:

```js
module.exports = {
  // Extensions to the GraphQL schema used to validate the manifest.
  // Could come in the form of new binding and data source types:
  //
  //   extend union Binding SubstrateContractBinding
  //   extend union DataSource SubstrateDataSource
  //   extend union DataSourceTemplate SubstrateDataSourceTemplate
  //   ...
  //
  manifestExtensions: graphql.Document,

  // Type conversions to and from AssemblyScript.
  //
  // Conversions are defined as an array of tuples, each of which
  // defines the source type (as a pattern or string), the target
  // type (as a pattern or string), and a code snippet that performs
  // the conversion.
  //
  // An example for Ethereum could look as follows:
  typeConversions: {
    fromAssemblyScript: [
      ['Address', 'address', code => `ethereum.Value.fromAddress(${code})`],
      ['boolean', 'bool', code => `ethereum.Value.fromBoolean(${code})`],
      ['Bytes', 'byte', code => `ethereum.Value.fromFixedBytes(${code})`],
      ['Bytes', 'bytes', code => `ethereum.Value.fromBytes(${code})`],
      [
        'Bytes',
        /^bytes(1|2|3|4|5|6|7|8|9|10|11|12|13|14|15|16|17|18|19|20|21|22|23|24|25|26|27|28|29|30|31|32)$/,
        code => `ethereum.Value.fromFixedBytes(${code})`,
      ],
      ...
    ],
    toAssemblyScript: [
      ['address', 'Address', code => `${code}.toAddress()`],
      ['bool', 'boolean', code => `${code}.toBoolean()`],
      ['byte', 'Bytes', code => `${code}.toBytes()`],
      [
        /^bytes(1|2|3|4|5|6|7|8|9|10|11|12|13|14|15|16|17|18|19|20|21|22|23|24|25|26|27|28|29|30|31|32)?$/,
        'Bytes',
        code => `${code}.toBytes()`,
      ],
      ...
    ]
  },

  // Code generation for bindings,
  codegen: (binding) => {
    // Return a string to be put into the output file for the binding
  }
}
```

Mapping files ending with `.ts` are automatically detected wherever they appear
in the manifest and included in builds and deployments.

#### Graph TS

Each blockchain supported in mappings comes with its own `network/<name>.ts`
file in the `graph-ts` repository. Each of these has at least two sections: one
for declaring host exports that are required for mappings to interact with the
chain, another to define blockchain-specific types and helpers written in
AssemblyScript. The file `networks/ethereum.ts` file could look as follows:

```js
import { Address } from '../index'

export declare namespace ethereum {
  function call(contract: Address, fn: string, args: Array<Value>): void
}

export namespace ethereum {
    class Value { ... }
    class Block { ... }
    class Transaction { ... }
    class Event { ... }
    class Contract { ... }
}
```

The main `index.ts` in the `graph-ts` repository re-exports each of these chain
integrations via

```js
export * from './networks/ethereum'
export * from './networks/substrate'
```

So that they can be imported into subgraph mappings with

```js
import { ethereum, substrate } from '@graphprotocol/graph-ts'
```

### Cross-Chain Indexing

For the time being, the plan is to limit subgraphs to a single chain each. So
subgraphs are not able to index across different blockchains or even different
chains (e.g. Ethereum mainnet _and_ Ropsten) at the same time.

## Compatibility

The proposed design is mostly backwards-compatible, except for a few areas:

- The manifest changes require a subgraph manifest migration in Graph CLI, as
  well as database migration for the subgraph meta data to match the new
  structure.

- Moving AssemblyScript types from e.g. `EthereumValue` to `ethereum.Value`
  _may_ break subgraphs that directly access these types. This is ok, because
  the majority (if not all) subgraphs only use them indirectly through generated
  code. This does not affect subgraphs that are already deployed.

## Drawbacks and Risks

Generally, supporting more blockchains is an opportunity much more than it is a
risk. It widens the reach of The Graph to communities beyond Ethereum.

One risk in the design itself is that it may require a bit of experimentation to
get the Graph Node traits for integrating arbitrary chains right.

## Alternatives

There are nuances where the design could look different, specifically around how
the manifest spec could be updated and whether new blockchain integrations are
developed as plugins or not. For the moment, an approach _without_ plugins is
considered the most efficient path forward.

## Open Questions

- None.
