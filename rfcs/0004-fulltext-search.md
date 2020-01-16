# RFC-0000: Template

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

This RFC proposes adding a fulltext search filter type to the GraphQL API 
to allow subgraph developers to specify language-specific, lexical, 
composite filters that end users can use in their queries. The fulltext 
index will be created by examining all words in a document, breaking it 
into individul words and phrases (lexical analysis), and collapsing 
variations of words into a single index term (stemming.)

## Goals & Motivation

The current set of string filters available in our GraphQL API is lacking 
the tools needed to build effective, modern interfaces. Wildcard string 
matching does provide string filtering, but users have come to expect the 
easy to use filtering that comes with fulltext search systems.

To facilitate building effective user interfaces we must provide human-user 
friendly query filtering. Lexical, composite fulltext search filters can 
provide the tools necessary for front-end developers to implement powerful 
search bars that filter data across multiple fields of an Entity.

With the proposed feature I aim to:
  - provide tools for subgraph developers to define composite search 
    indexes that can search across multiple fields and entities, and
  - integrate powerful, user friendly text search that is flexible and 
    can be easily defined by a subgraph developer ultimately providing 
    the tools necesssary for front end developers to create effective 
    user interfaces without the need to middleware.
     
## Urgency

A delay in adding the fulltext search feature  will not create issues with 
current deployments. However, it will represent a realization of part of 
the long term vision for the query network. Also several high profile users 
have communicated that it may be a conversion blocker, so implementation 
should be prioritized. 

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

### Subgraph Definition

Part of the power of fulltext search indexes are their flexibility, so 
it is important to expose a simple interface to facilitate useful 
applications of the index and ulimately aim to reduce the need to create 
new subgraphs for the express purpose of updating fulltext search fields 
for end-use cases. 

For each fulltext search index a subgraph developer must be able to specify:
    1. a language, 
    2. a set of text document fields to include,
    3. relative weighting for each field in the index.
    4. a choice of ranking algorithm for sorting query result items

The proposed process of adding one or more fulltext search index to an Entity 
involves adding one or more fulltext directive to the `_Schema_` type in the 
subgraph's GraphQL schema.  Each fulltext definition will have four required
top level parameters: `name`, `language`, `algorithm`, and `include`. 
The fulltext search definitions will be used to generate queries on the GraphQL
schema that will be exposed to the end user.  

Applying fulltext search indexes across entities will be a powerful 
abstraction allowing users to search across all relevant entities in one
query. An index search across entities will by definition have polymorphic
results, so in that case a union type will also be generated in the schema 
for the query result items. 

Verification of the index definition will ensure that all fields referenced 
are valid String type fields. To be clear, the proposed interface will add 
the fulltext search index definitions to the subgraph deployment hash, so 
adding or updating a fulltext search index will require an update to the subgraph. 
With subgraph composition it will be possible to easily create new subgraphs 
that add specific fulltext search indexes to an existing subgraph. 

Example fulltext search definition:

```graphql
type _Schema_ 
  @fulltext(
    name: "media"
    ...
  )
  @fulltext(
    name: "search",
    language: "english",
    algorithm: "ranked",
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
union FulltextMediaResultItem = ...
union FulltextSearchResultItem = Band | Album | Musician
type Query {
  media...
  search(text: String!, first: Int, skip: Int, block: Int): [FulltextSearchResultItem!]!
}
```

### GraphQL Query interface

End users of the subgraph will have access to the queries generated for the
fulltext search indexes alongside the other queries generated for each 
entity in the subgraph.  In the case of a fulltext search index that spans
multiple entities, [inline fragments](https://graphql.org/learn/queries/#inline-fragments) 
may be used in the query to deal with the polymorphic result items. In the 
front-end the `__typename` in each result item can be used to render the 
components accordingly.  

In the "text" parameter supplied to the query there will be one special 
operator supported, the proximity operator. Several operators are available: 
and, or, and proximity (`&`, `|`, `<->`.) The proximity operator allows 
one to specify max distance between search terms: 3 words apart → `<3>`. 

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
system complexity and feature implementation time I propose starting with the 
fulltext search features built in to PostgreSQL.

A FullText search field will get its own column in the Entity table just 
like any other field; however, the data stored will be the result of the 
lexical, morphological analysis of text documents performed on the fields 
included in the index. The fulltext search field will be created using the 
Postgres ts_vector function and will be indexed using a GIN index. The subgraph 
developer will define a ranking algorithm to be used to sort query results,
so the end-user facing API remains easy to use without any requirement to 
understand the ranking algorithms.   

Limitations of Postgres fulltext search indexes and search algorithms: 
  - only certain languages available (expandable with plugins like [PGroonga](https://pgroonga.github.io/)), 
    - **languages supported out of the box**: 
        Danish, Dutch, English, Finnish, French, German, Hungarian, Italian, 
        Norwegian, Portuguese, Romanian, Russian, Spanish, Swedish, and Turkish.
  - select algorithms are available, and 
  - limited to PostgreSQL storage. 

Alternative technologies (not in any particular order): 
  - Written in Rust:
    - [Tantivy](https://github.com/tantivy-search/tantivy)
    - [Toshi](https://github.com/toshi-search/Toshi)
    - [Sonic](https://github.com/valeriansaliou/sonic)
    - [MeiliSearch](https://github.com/meilisearch/MeiliSearch)
    - [Bayard](https://github.com/bayard-search/bayard)

## Compatibility

This proposal does not change any existing interfaces, so no migrations 
will be necessary for existing subgraph deployments. 

## Drawbacks and Risks

There is little risk in implementing this feature.  The proposed solution 
uses native Postgres fulltext features and there is a nonzero probability 
this choice results in slower than optimal write and read times; 
however the tradeoff in implementation time/complexity and the existence 
of production use case testimonials tempers my apprehension here. 

Are there any desired "fulltext search like" capabilities that I've missed in this proposal?   

## Alternatives

An alternative design for the feature would allow more flexibility for 
Graph Node operators in their index implementation and create a marketplace 
for indexes. This design moved the definition of custom indexes out of the 
subgraph definition; the subgraph would be deployed without them and they 
could be added later using a new Graph Explorer interface (in Hosted-Service context) 
or a JSON-RPC request directly to a Graph Node. One of the benefits of this 
design is that the definition of uniqueness for a subgraph does not include 
the custom indexes, so a new subgraph deployment and corresponding syncing 
work does not have to be added in order to create or update an index. 
However, it also introduces significant added complexity: will a separate 
query marketplace and discovery registry then be required for finding nodes 
with your needed subgraph-index combination.  

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
the addition or update of an @index field does not require a full blockchain 
resync, rather the index itself just needs to be rebuilt. 
How can we allow fulltext search index (and eventually other index types) 
updates without requiring a full subgraph resync? 