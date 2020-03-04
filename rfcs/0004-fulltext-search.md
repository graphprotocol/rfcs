# RFC-0004: Fulltext Search

<dl>
  <dt>Author</dt>
  <dd>Ford Nickels</dd>

  <dt>RFC pull request</dt>
  <dd><a href="https://github.com/graphprotocol/rfcs/pull/11">URL</a></dd>

  <dt>Obsoletes (if applicable)</dt>
  <dd>-</dd>

  <dt>Date of submission</dt>
  <dd>2020-01-05</dd>

  <dt>Date of approval</dt>
  <dd>-</dd>

  <dt>Approved by</dt>
  <dd>-</dd>
</dl>

## Contents

<!-- toc -->

## Summary

The fulltext search filter type is a feature of the GraphQL API that
allows subgraph developers to specify language-specific, lexical,
composite filters that end users can use in their queries. The fulltext
search feature examines all words in a document, breaking it into
individual words and phrases (lexical analysis), and collapsing
variations of words into a single index term (stemming.)

## Goals & Motivation

The current set of string filters available in the GraphQL API is lacking 
fulltext search capabilities that enable efficient searches across entities
and attributes. Wildcard string matching does provide string filtering, but 
users have come to expect the easy to use filtering that comes with fulltext 
search systems.

To facilitate building effective user interfaces human-user friendly query 
filtering is essential. Lexical, composite fulltext search filters can provide 
the tools necessary for front-end developers to implement powerful search 
bars that filter data across multiple fields of an Entity.

The proposed feature aims to provide tools for subgraph developers to define 
composite search APIs that can search across multiple fields and entities.
     
## Urgency

A delay in adding the fulltext search feature will not create issues
with current deployments. However, the feature will represent a
realization of part of the long term vision for the query network. In
addition, several high profile users have communicated that it may be a
conversion blocker. Implementation should be prioritized. 

## Terminology

- _lexeme_: a basic lexical unit of a language, consisting of one word or 
  several words, considered as an abstract unit, and applied to a family 
  of words related by form or meaning.
- _morphology (linguistics)_: the study of words, how they are formed, 
  and their relationship to other words in the same language. 
- _fulltext search index_: the result of lexical and morphological 
  analysis (stemming) of a set of text documents.  It provides frequency 
  and location for the language-specific stems found in the text documents 
  being indexed. 
- _ranking algorithm_: "Ranking attempts to measure how relevant documents 
  are to a particular query, so that when there are many matches the most 
  relevant ones can be shown first." [- Postgres Documentation](https://www.postgresql.org/docs/11/textsearch-controls.html#TEXTSEARCH-RANKING-SEARCH-RESULTS)
  
  **Algorithms**:
  - _standard ranking_: ranking based on the number of matching lexemes.     
  - _cover density ranking_: Cover density is similar to the standard 
    fulltext search ranking except that the proximity of matching lexemes 
    to each other is taken into consideration. This function requires 
    lexeme positional information to perform its calculation, so it ignores 
    any "stripped" lexemes in the index.            

## Detailed Design

### Subgraph Schema

Part of the power of the fulltext search API is the flexibility, so 
it is important to expose a simple interface to facilitate useful applications 
of the index and aim to reduce the need to create new subgraphs for the 
express purpose of updating fulltext search fields. 

For each fulltext search API a subgraph developer must be able to specify:
    1. a language (specified using an `ISO 639-1` code), 
    2. a set of text document fields to include,
    3. relative weighting for each field,
    4. a choice of ranking algorithm for sorting query result items.

The proposed process of adding one or more fulltext search API involves
adding one or more fulltext directive to the `_Schema_` type in the
subgraph's GraphQL schema. Each fulltext definition will have four
required top level parameters: `name`, `language`, `algorithm`, and
`include`. The fulltext search definitions will be used to generate
query fields on the GraphQL schema that will be exposed to the end user.

Enabling fulltext search across entities will be a powerful abstraction 
that allows users to search across all relevant entities in one query. Such 
a search will by definition have polymorphic results. To address this, a 
union type will be generated in the schema for the fulltext search results. 

Validation of the fulltext definition will ensure that all fields referenced 
in the directive are valid String type fields. With subgraph composition 
it will be possible to easily create new subgraphs that add specific fulltext 
search capabilities to an existing subgraph. 

Example fulltext search definition:

```graphql
type _Schema_ 
  @fulltext(
    name: "media"
    ...
  )
  @fulltext(
    name: "search",
    language: EN, # variant of `_FullTextLanguage` enum
    algorithm: RANKED, # variant of `_FullTextAlgorithm` enum
    include: [
      {
        entity: "Band",
        fields: [
          { name: "name", weight: 5 },
        ]
      },
      {
        entity: "Album",
        fields: [
          { name: "title", weight: 5 },
        ]
      },
      {
        entity: "Musician",
        fields: [
          { name: "name", weight: 10 },
          { name: "bio", weight: 5 },
        ]
      }
    ]
  )
```

The schema generated from the above definition: 
```graphql
union _FulltextMediaEntity = ...
union _FulltextSearchEntity = Band | Album | Musician
type Query {
  media...
  search(text: String!, first: Int, skip: Int, block: Block_height): [FulltextSearchResultItem!]!
}
```

### GraphQL Query interface

End users of the subgraph will have access to the fulltext search
queries alongside the other queries available for each entity in the
subgraph. In the case of a fulltext search defined across multiple
entities,
[inline fragments](https://graphql.org/learn/queries/#inline-fragments)
may be used in the query to deal with the polymorphic result items. In
the front-end the `__typename` field can be used to distinguish the
concrete entity types of the returned results.

In the `text` parameter supplied to the query there will be several operators
available to the end user.  Included are the and, or, and proximity operators
(`&`, `|`, `<->`.) The special, proximity operator allows clients to specify 
the maximum distance between search terms: `foo<3>bar` is equivalent to 
requesting that `foo` and `bar` are at most three words apart. 



Example query using inline fragments and the proximity operator: 
```graphql
query {
  search(text: "Bob<3>run") {
    __typename
    ... on Band { name label { id } }
    ... on Album { title numberOfTracks }
    ... on Musician { name bio }
  }
}
```

### Tools and Design

Fulltext search query system implementations often involve specific systems 
for storing and querying the text documents; however, in an effort to reduce 
system complexity and feature implementation time I propose starting with 
extending the current store interface and storage implemenation with fulltext 
search features rather than use a fulltext specific interface and storage
system.

A FullText search field will get its own column in a table dedicated to fulltext
data. The data stored will be the result of the lexical, morphological analysis 
of text documents performed on the fields included in the index. The fulltext 
search field will be created using the Postgres ts_vector function and will 
be indexed using a GIN index. The subgraph developer will define a ranking 
algorithm to be used to sort query results,so the end-user facing API remains 
easy to use without any requirement to understand the ranking algorithms.   

## Compatibility

This proposal does not change any existing interfaces, so no migrations 
will be necessary for existing subgraph deployments. 

## Drawbacks and Risks

The proposed solution uses native Postgres fulltext features and there is 
a nonzero probability this choice results in slower than optimal write and 
read times; however the tradeoff in implementation time/complexity and the 
existence of production use case testimonials tempers my apprehension here. 

In future phases of the network the storage layer may get a redesign with 
indexes being overhauled to facilitate query result verification. Postgres
based fulltext search implementation would not be translatable to another 
storage system, so at the least a reevaluation of the tools used for analysis, 
indexing, and querying would be required.  

## Alternatives

An alternative design for the feature would allow more flexibility for
Graph Node operators in their index implementation and create a
marketplace for indexes. In the alternate, the definition of fulltext
search indexes could be moved out of the subgraph schema. The subgraph
would be deployed without them and they could be added later using a new
Graph Explorer interface (in Hosted-Service context) or a JSON-RPC
request directly to a Graph Node. Moving the creation of fulltext search
indexes/queries out of the schema would mean that that the definition of
uniqueness for a subgraph does not include the custom indexes, so a new
subgraph deployment and subgraph re-syncing work does not have to be
added in order to create or update an index. However, it also introduces
significant added complexity. A separate query marketplace and discovery
registry would be required for finding nodes with the needed
subgraph-index combination.

## Open Questions

Full-text search queries introduce new issues with maintaining query 
result determinism which will become a more potent issue with the 
decentralized network. A fulltext search query and a dataset are not enough 
to determine the output of the query, the index is vital to establish a 
deterministic causal relationship to the output data.  Query verification 
will need to take into account the query, the index, the underlying dataset, 
and the query result.  Can we find a healthy compromise between being 
prescriptive about the indexes and algorithms in order to allow formal 
verification and allowing indexer node operators to experiment with 
algorithms and indexes in order to continue to improve query speed and results? 

Since a fulltext search field is purely derivative of other Entity data
the addition or update of an @fulltext directive does not require a full
blockchain resync, rather the index itself just needs to be rebuilt.
There is room for optimization in the future by allowing fulltext search
definition updates without requiring a full subgraph resync.