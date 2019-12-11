# RFC-0001: Subgraph composition

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

## Motivation

The ability to reference, extend and query entities across subgraph boundaries
enables several use cases:

1. Linking entities across subgraphs.
2. Extending entities defined in other subgraphs by adding new fields.
3. Breaking down data silos by composing subgraphs and defining richer schemas
   without indexing the same data over and over again.
   
Subgraph composition is needed to avoid duplicated work, both in terms of
developing subgraphs as well as indexing them. It is an essential part of the
overal vision behind The Graph, as it allows to combine isolated subgraphs into
a complete, connected graph of the (decentralized) world's data.

Subgraph developers will benefit from the ability to reference data from other
subgraphs, saving them development time and enabling richer data models. dApp
developers will be able to leverage this to build more compelling applications.
Node operators will benefit from subgraph composition by having better insight
into which subgraphs will be queried together, allowing them to make more
informed decisions about which subgraphs to index.

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

## Detailed Design

The following sections make the following assumptions:

- There is an `ethereum/mainnet` subgraph with an `Address` entity type.

### Composing subgraphs by importing types

In order to reference entity types from a imported schema, a developer would
first import these entity types from the other subgraph.

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
    proposer { balance address }
    transaction { hash block { number } }
  }
}
```

### Extending types from imported schemas

Extending types from another subgraph involves a few steps:

1. Importing entity types from the other subgraph.
2. Extending these entity types with custom fields.
3. Managing (e.g. creating) extended entities in subgraph mappings.

Types can be imported either from a subgraph name or from a subgraph ID.
Importing from a subgraph name means that the exact version of the imported
subgraph will be identified at query time and its schema may change in arbitrary
ways over time. Importing from a subgraph ID guarantees that the schema will
never change but also means that the import points to a subgraph version that
may become outdated over time.

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

This makes queries like the following possible, where you can go "back" from
addresses to proposal entities, despite the Ethereum `Address` type originally
being defined in the `ethereum/mainnet` subgraph.

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
means that an implementation likely wouldn't have to create a local extension
type in the store. However, if `proposals` was defined as

```graphql
extend type Address {
  proposals: [Proposal!]!
}
```

then it would have to be possible for the subgraph mappings to create `Address`
instances and set the `proposals` field. These entity instances would have to be
merged with the corresponding original entities from the `ethereum/mainnet`
subgraph.

### Subgraph availability

In the decentralized network, queries will be split and routed through the
network based on what indexers are available and which subgraphs they index. At
that point, failure to find an indexer for a subgraph that types were imported
from will result in a query error. The error that a non-nullable field resolved
to null bubbles up to the next nullable parent, in accordance with the [GraphQL
Spec](https://graphql.github.io/graphql-spec/draft/#sec-Errors.Error-result-format).

Until the network is reality, we are dealing with individual Graph nodes and
querying subgraphs where imported entity types are not also indexed on the same
node should be handled with more tolerance. This RFC proposes that entity
reference fields that refer to imported types are converted to being optional in
the generated API schema. If the subgraph that the type is imported from is not
available on a node, such fields should resolve to `null`.

## Compatibility

Subgraph composition is a purely additive, non-breaking change. Existing
subgraphs remain valid without any migrations being necessary.

## Drawbacks and Risks

Reasons that could speak against implementing this feature:

- Schema parsing and validation becomes more complicated. Especially validation
  of imported schemas may not always be possible, depending on whether and when
  the referenced subgraph is available on the Graph node or not.

- Query execution becomes more complicated. The subgraph a type belongs to must
  be identified and local and imported versions of extended entities have to be
  queried separately and be merged.

## Alternatives

No alternatives have been considered.

There are other ways to compose subgraph schemas using GraphQL technologies such
as [schema
stitching](https://www.apollographql.com/docs/graphql-tools/schema-stitching/)
or [Apollo
Federation](https://www.apollographql.com/docs/apollo-server/federation/introduction/).
However, schema stitching is being deprecated and Apollo Federation requires a
centralized server to serve a combined GraphQL API.

Another reason not to use these is that GraphQL will only be _one_ of several
query languages supported in the future. Composition therefore has to be
implemented in a query-language-agnostic way.

## Open Questions

- An open question is whether it should be possible to extend imported
  interfaces. For example, `Address` could be an interface in the
  `ethereum/mainnet` subgraph and `Account` and `Contract` could be concrete
  entity types that both implement the `Address` interface. It would be great if
  the proposals field could be added to this interface and the concrete types as
  well by importing all three and extending them.
