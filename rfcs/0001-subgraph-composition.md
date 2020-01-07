# RFC-0001: Subgraph Composition

<dl>
  <dt>Author</dt>
  <dd>Jannis Pohlmann</dd>

  <dt>RFC pull request</dt>
  <dd><a href="https://github.com/graphprotocol/rfcs/pull/1">https://github.com/graphprotocol/rfcs/pull/1</a></dd>
  
  <dt>Obsoletes</dt>
  <dd>-</dd>

  <dt>Date of submission</dt>
  <dd>2019-12-08</dd>

  <dt>Date of approval</dt>
  <dd>-</dd>

  <dt>Approved by</dt>
  <dd>-</dd>
</dl>


## Summary

Subgraph composition enables referencing, extending and querying entities across
subgraph boundaries.

## Goals & Motivation

The high-level goal of subgraph composition is to be able to compose subgraph
schemas and data hierarchically. Imagine umbrella subgraphs that combine all the
data from a domain (e.g. DeFi, job markets, music) through one unified, coherent
API. This could allow reuse and governance at different levels and go all the
way to the top, fulfilling the vision of _the_ Graph.

The ability to reference, extend and query entities across subgraph boundaries
enables several use cases:

1. Linking entities across subgraphs.
2. Extending entities defined in other subgraphs by adding new fields.
3. Breaking down data silos by composing subgraphs and defining richer schemas
   without indexing the same data over and over again.
   
Subgraph composition is needed to avoid duplicated work, both in terms of
developing subgraphs as well as indexing them. It is an essential part of the
overall vision behind The Graph, as it allows to combine isolated subgraphs into
a complete, connected graph of the (decentralized) world's data.

Subgraph developers will benefit from the ability to reference data from other
subgraphs, saving them development time and enabling richer data models. dApp
developers will be able to leverage this to build more compelling applications.
Node operators will benefit from subgraph composition by having better insight
into which subgraphs are queried together, allowing them to make more informed
decisions about which subgraphs to index.

## Urgency

Due to the high impact of this feature and its important role in fulfilling the
vision behind The Graph, it would be good to start working on this as early as
possible.

## Terminology

The feature is referred to by _query-time subgraph composition_, short:
_subgraph composition_. 

Terms introduced and used in this RFC:

- _Imported schema_: The schema of another subgraph from which types are
  imported.
- _Imported type_: An entity type imported from another subgraph schema.
- _Extended type_: An entity type imported from another subgraph schema and
  extended in the subgraph that imports it.
- _Local schema_: The schema of the subgraph that imports from another subgraph.
- _Local type_: A type defined in the local schema.

## Detailed Design

The sections below make the assumption that there is a subgraph with the name
`ethereum/mainnet` that includes an `Address` entity type.

### Composing Subgraphs By Importing Types

In order to reference entity types from annother subgraph, a developer would
first import these types from the other subgraph's schema.

Types can be imported either from a subgraph name or from a subgraph ID.
Importing from a subgraph name means that the exact version of the imported
subgraph will be identified at query time and its schema may change in arbitrary
ways over time. Importing from a subgraph ID guarantees that the schema will
never change but also means that the import points to a subgraph version that
may become outdated over time.

Let's say a DAO subgraph contains a `Proposal` type that has a `proposer` field
that should link to an Ethereum address (think: Ethereum accounts or contracts)
and a `transaction` field that should link to an Ethereum transaction. The
developer would then write the DAO subgraph schema as follows:

```graphql
type _Schema_
  @import(
    types: ["Address", { name: "Transaction", as: "EthereumTransaction" }],
    from: { name: "ethereum/mainnet" }
  )

type Proposal @entity {
  id: ID!
  proposer: Address!
  transaction: EthereumTransaction!
}
```

This would then allow queries that follow the references to addresses and
transactions, like

```graphql
{
  proposals { 
    proposer {
      balance
      address
    }
    transaction {
      hash
      block {
        number
      }
    }
  }
}
```

### Extending Types From Imported Schemas

Extending types from another subgraph involves several steps:

1. Importing the entity types from the other subgraph.
2. Extending these types with custom fields.
3. Managing (e.g. creating) extended entities in subgraph mappings.

Let's say the DAO subgraph wants to extend the Ethereum `Address` type to
include the proposals created by each respective account. To achieve this, the
developer would write the following schema:

```graphql
type _Schema_
  @import(
    types: ["Address"],
    from: { name: "ethereum/mainnet" }
  )

type Proposal @entity {
  id: ID!
  proposer: Address!
}

extend type Address {
  proposals: [Proposal!]! @derivedFrom(field: "proposal")
}
```

This makes queries like the following possible, where the query can go "back"
from addresses to proposal entities, despite the Ethereum `Address` type
originally being defined in the `ethereum/mainnet` subgraph.

```graphql
{
  addresses {
    id
    proposals {
      id
      proposer {
        id
    }
  }
}
```

In the above case, the `proposals` field on the extended type is derived, which
means that an implementation wouldn't have to create a local extension type in
the store. However, if `proposals` was defined as

```graphql
extend type Address {
  proposals: [Proposal!]!
}
```

then it would the subgraph mappings would have to create partial `Address`
entities with `id` and `proposals` fields for all addresses from which proposals
were created. At query time, these entity instances would have to be merged with
the original `Address` entities from the `ethereum/mainnet` subgraph.

### Subgraph Availability

In the decentralized network, queries will be split and routed through the
network based on what indexers are available and which subgraphs they index. At
that point, failure to find an indexer for a subgraph that types were imported
from will result in a query error. The error that a non-nullable field resolved
to null bubbles up to the next nullable parent, in accordance with the [GraphQL
Spec](https://graphql.github.io/graphql-spec/draft/#sec-Errors.Error-result-format).

Until the network is reality, we are dealing with individual Graph Nodes and
querying subgraphs where imported entity types are not also indexed on the same
node should be handled with more tolerance. This RFC proposes that entity
reference fields that refer to imported types are converted to being optional in
the generated API schema. If the subgraph that the type is imported from is not
available on a node, such fields should resolve to `null`.

### Interfaces

Subgraph composition also supports interfaces in the ways outlined below.

#### Interfaces Can Be Imported From Other Subgraphs

The syntax for this is the same as that for importing types:

```graphql
type _Schema_
  @import(types: ["ERC20"], from: { name: "graphprotocol/erc20" })
```

#### Local Types Can Implement Imported Interfaces

This is achieved by importing the interface from another subgraph schema
and implementing it in entity types:

```graphql
type _Schema_
  @import(types: ["ERC20"], from: { name: "graphprotocol/erc20" })

type MyToken implements ERC20 @entity {
  # ...
}
```

#### Imported Types Can Be Extended To Implement Local Interfaces

This is achieved by importing the types from another subgraph schema, defining a
local interface and using `extend` to implement the interface on the imported
types:

```graphql
type _Schema_
  @import(types: [{ name: "Token", as "LPT" }], from: { name: "livepeer/livepeer" })
  @import(types: [{ name: "Token", as "Rep" }], from: { name: "augur/augur" })

interface Token {
  id: ID!
  balance: BigInt!
}

extend LPT implements Token {
  # ...
}
extend Rep implements Token {
  # ...
}
```

#### Imported Types Can Be Extended To Implement Imported Interfaces

This is a combination of importing an interface, importing the types and
extending them to implement the interface:

```graphql
type _Schema_
  @import(types: ["Token"], from: { name: "graphprotocol/token" })
  @import(types: [{ name: "Token", as "LPT" }], from: { name: "livepeer/livepeer" })
  @import(types: [{ name: "Token", as "Rep" }], from: { name: "augur/augur" })

extend LPT implements Token {
  # ...
}
extend Rep implements Token {
  # ...
}
```

#### Implementation Concerns For Interface Support

Querying across types from different subgraphs that implement the same interface
may require a smart algorithm, especially when it comes to pagination. For
instance, if the first 1000 entities for an interface are queried, this range of
1000 entities may be divided up between different local and imported types
arbitrarily.

A naive algorithm could request 1000 entities from each subgraph, applying the
selected filters and order, combine the results and cut off everything after the
first 1000 items. This would generate a minimum of requests but would involve
significant overfetching.

Another algorithm could just fetch the first item from each subgraph, then based
on that information, divide up the range in more optimal ways than the previous
algorith, and satisfy the query with more requests but with less overfetching.

## Compatibility

Subgraph composition is a purely additive, non-breaking change. Existing
subgraphs remain valid without any migrations being necessary.

## Drawbacks And Risks

Reasons that could speak against implementing this feature:

- Schema parsing and validation becomes more complicated. Especially validation
  of imported schemas may not always be possible, depending on whether and when
  the referenced subgraph is available on the Graph Node or not.

- Query execution becomes more complicated. The subgraph a type belongs to must
  be identified and local as well as imported versions of extended entities have
  to be queried separately and be merged before returning data to the client.

## Alternatives

No alternatives have been considered.

There are other ways to compose subgraph schemas using GraphQL technologies such
as [schema
stitching](https://www.apollographql.com/docs/graphql-tools/schema-stitching/)
or [Apollo
Federation](https://www.apollographql.com/docs/apollo-server/federation/introduction/).
However, schema stitching is being deprecated and Apollo Federation requires a
centralized server to serve to extend and merge GraphQL API. Both of these
solutions slow down queries.

Another reason not to use these is that GraphQL will only be _one_ of several
query languages supported in the future. Composition therefore has to be
implemented in a query-language-agnostic way.

## Open Questions

- Right now, interfaces require unique IDs across all the concrete entity types
  that implement them. This is not something we can guarantee any longer if
  these concrete types live in different subgraphs. So we have to handle this at
  query time (or must somehow disallow it, returning a query error).

  It is also unclear how an individual interface entity lookup would look like
  if IDs are no longer guaranteed to be unique:
  ```graphql
  someInterface(id: "?????") {
  }
  ```
