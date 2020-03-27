# RFC-0003: Multi-Blockchain Support

<dl>
  <dt>Author</dt>
  <dd>Jannis Pohlmann</dd>

  <dt>RFC pull request</dt>
  <dd><a href="https://github.com/graphprotocol/rfcs/pull/8">https://github.com/graphprotocol/rfcs/pull/8</a></dd>

  <dt>Obsoletes (if applicable)</dt>
  <dd>-</dd>

  <dt>Date of submission</dt>
  <dd>2019-12-20</dd>

  <dt>Date of approval</dt>
  <dd>-</dd>

  <dt>Approved by</dt>
  <dd>-</dd>
</dl>

## Contents

<!-- toc -->

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

> **Note:** A general overhaul of the subgraph manifest spec was
> [discussed](https://github.com/graphprotocol/research/issues/89). This would
> be a fairly disruptive change, even if most subgraphs could be migrated
> automatically. The current notion of _data sources_ is easy for everyone to
> understand, however, and it is not clear yet how data source templates fit
> into the manifest structure proposed in this discussion.

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
> - Graph Node: Plugin frameworks in Rust are somewhat of a question mark right
>   now. [This
>   article](http://adventures.michaelfbryan.com/posts/plugins-in-rust/)
>   provides a good overview over how it could be done. There is also an
>   [unofficial guide to dynamic loading &
>   plugins](https://michael-f-bryan.github.io/rust-ffi-guide/dynamic_loading.html).
>   An alternative could be optional dependencies behind feature flags, to pull
>   in support for specific chains at build time.
>
> - Graph CLI: Gluegun, used for Graph CLI, includes an extension framework that
>   would allow to inject code generation and type conversion support from
>   separate NPM packages.
>
> - Graph TS: Separate NPM packages for chain-specific mapping APIs and types as
>   suggested in the previous note would make extending Graph TS easy.

### Subgraph Indexing

This is primarily about how subgraphs are indexed inside Graph Node. At the time
of writing, this is strongly tied to Ethereum. Each subgraph is connected to an
Ethereum chain-specific store and an Ethereum block stream. The indexing logic
hard-codes processing of Ethereum blocks, events and calls.

The reason stores are tied to specific chains is because they not only contain
the subgraph data but also store information about which block the chain is on.
This in turn is used in block ingestion and in the block stream to decide how to
move forward and how to handle chain reorgs.

All that subgraphs really care about during indexing, however, is:

1. Which triggers are relevant to the subgraph and its mappings?

2. How can these triggers be updated when new data sources are created from
   templates during indexing?
  
3. When do changes generated by processing triggers need to be reverted to
   account for a reorg?
  
None of these require a store that is tied to a specific chain.

To support multiple blockchains, subgraph indexing is changed as follows:

1. Instead of multiple chain-specific stores, a single store is passed to the
   subgraph instance manager.
   
2. Also passed to the subgraph instance manager is a set of supported networks,
   specifically blockchains (e.g. Ethereum and Substrate). Each blockchain
   provides a set of supported chains (e.g. Ethereum mainnet or Polkadot's relay
   chain). All of these chains provide blockchain-specific host exports for WASM
   and a chain-specific block stream builder. Each chain's block stream outputs
   blocks with chain-specific triggers.
   
3. When a subgraph starts up, it obtains one or more block streams that match
   the chains the data sources of the subgraph and processes the blocks and
   triggers emitted by these streams. Triggers are essentially opaque data types
   that implement a set of traits allowing to match them against data sources /
   trigger handlers and pass on to WASM mappings.
   
4. The host exports from all matching blockchains are passed to the WASM runtime
   for exposing to the mappings.

### Storage

A number of features in the current store interface is based on so-called
Ethereum block pointers, a pair of block hash and block number. This is changed
to a generic block pointer type, where the hash is a byte array rather than an
Ethereum 256-bit hash.

Another part of the current store that is specific to Ethereum is the chain
store interface. Assuming the ongoing work on chain indexing is developed far
enough to replace the current block ingestion and block streaming, this
interface can be removed. The only part that will have to be kept is the
so-called chain head listener. This is moved into either the chain or the chain
indexer component.

### Adding Support for a New Blockchain

This section covers what it takes to add support for a new blockchain. Please
note that this RFC specifies the details only to an extent that allow us to
assess whether the direction is feasible. Names and data structure details may
vary in the implementation.

#### Graph Node

Each blockchain integration in Graph Node lives in the `graph-node` repository
under `network/<name>/`, like the already existing `network/ethereum/` support.

Graph Node includes a number of new component traits and data types to enable
multi-blockchain support:

```rust
/// A compact representation of a block on a chain.
struct BlockPointer {
    pub number: BigInt,
    pub hash: Bytes,
}

/// Represents a block in a particular chain. Each chain has its own
/// implementation of the `Block` trait.
trait Block: ToAsc {
    fn number(&self) -> BigInt;
    fn hash(&self) -> Bytes;
    fn pointer(&self) -> BlockPointer;
    fn parent_pointer(&self) -> Option<BlockPointer>;
}

/// Represents a decentralized network like Ethereum, Polkadot or similar.
enum Network {
    // Variant for blockchain networks.
    Blockchain(Box<dyn Blockchain>),
}

/// Represents a blockchain supported by the node.
trait Blockchain {
    type Chain: Chain;
    type Options;

    fn new(options: Self::Options) -> Self;
    
    // Looks up a chain by name. Fails if the node doesn't support the chain.
    fn chain(&self, name: String) -> Result<Self::Chain, Error>;
}

/// Represents a network (essentially an instance of a `Chain`).
trait Chain {
    type Block: Block;
    type ChainIndexer: ChainIndexer<Self::Block>;
    type BlockStream: BlockStream<Self::Block>;

    // Methods common to all chains. These are needed by the chain indexer
    // and block stream.
    fn latest_block(&self, options: ???) -> LatestBlockFuture;
    fn block_by_number(&self, options: ???) -> BlockByNumberFuture;
    fn block_by_hash(&self, options: ???) -> BlockByHashFuture;

    /// Indexes chain data hand handles reorgs.
    fn indexer(&self, options: ???) -> Result<Self::ChainIndexer, Error>;
    
    /// Provides a block/trigger stream for given options (including
    /// data sources, to generate filters).
    fn block_stream(&self, options: ???) -> Result<Self::BlockStream, Error>;
}

/// Events emitted by a chain indexer.
pub enum ChainIndexerEvent<T: Block> {
    Revert {
        from: BlockPointer,
        to: BlockPointer,
    },
    AddBlock(Box<T>),
}

/// Common trait for all chain indexers.
trait ChainIndexer<T: Block>: Stream<ChainIndexerEvent<T>> {}

/// Triggers emitted by a block stream. Triggers can be passed to 
/// AssemblyScript and implement a canonical ordering to ensure
/// they are not processed out of order.
trait ChainTrigger: ToAsc + Ord {
    fn matches_data_source(&self, data_source: &DataSource) -> bool;
    fn matches_handler(&self, handler: &DataSourceHandler) -> bool;
}

/// Combination of a block and subgraph-specific triggers for it.
struct BlockWithTriggers<T: Block> {
    block: Box<T>,
    triggers: Vec<dyn NetworkTrigger>,
}

/// Events emitted by a block stream.
enum BlockStreamEvent<T: Block> {
    Revert(BlockPointer),
    Block(BlockWithTriggers<T>)
}

/// Common trait for all block streams.
trait BlockStream<T: Block>: Stream<BlockStreamEvent<T>> {}
```

Any blockchain integration has to implement the above traits, except for
`ChainIndexer` which there are is a default implementation in `graph-core` that
work with any `Chain` implementation.

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
get the Graph Node traits for integrating arbitrary chains right. Overall, the
work resulting from this RFC may easily cover one or two months; note that
estimating this work is the responsibility of the engineering plan, however.

## Alternatives

There are nuances where the design could look different, specifically around how
the manifest spec could be updated and whether new blockchain integrations are
developed as plugins or not. For the moment, an approach _without_ plugins is
considered the most efficient path forward.

## Open Questions

- How does processing historic data for dynamic data sources into this design?
  If they are to be indexed in parallel to the other data sources, then it's
  almost like they need their own temporary block stream until they have caught
  up. Should they be blocking or index in parallel to the rest of the subgraph?
  Projects like DAOstack have requested parallel indexing.
  
- IPFS data sources / data source dependencies. How do they fit into this
  design?
