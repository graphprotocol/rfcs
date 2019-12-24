# RFC-0003: Mutations

<dl>
  <dt>Author</dt>
  <dd>dOrg: Jordan Ellis, Nestor Amesty</dd>

  <dt>RFC pull request</dt>
  <dd><a href="TODO">URL</a></dd>

  <dt>Date of submission</dt>
  <dd>2019-12-20</dd>

  <dt>Date of approval</dt>
  <dd>YYYY-MM-DD</dd>

  <dt>Approved by</dt>
  <dd>First Person, Second Person</dd>
</dl>

## Contents

<!-- toc -->

## Summary

GraphQL Mutations allow you to add executable functions to your schema. Callers can invoke these functions using GraphQL queries. An introduction to how Mutations are defined and work can be found [here](https://graphql.org/learn/queries/#mutations). This RFC will assume the reader understands how to use GraphQL Mutations in a traditional Web2 application. Going forward we'll describe how Mutations can be added to The Graph's toolchain, and used to replace web3 write operations the same way The Graph has replaced Web3 read operations.

## Goals & Motivation

Currently, The Graph has created a read semantic layer that describes smart contract protocols, which has made it easier to build applications ontop of complex protocols. Since dApps have two primary interactions with web3 protocols (reading & writing), the next logical addition is write support.

Protocol developers that currently use a subgraph still often publish a Javascript wrapper library for their dApp developers. This is done to help speed up dApp development and promote consistency with protocol usage patterns. With the addition of Mutations to the Graph Protocol's GraphQL tooling, Web3 reading & writing can now both be invoked through GraphQL queries. dApp developers can now simply refer to a single GraphQL schema that defines the entire protocol.

## Urgency

From a developer experience point of view I see this as urgent because it eliminates the need for protocol developers to manaually wrap their graphql query interfaces alongside user-friendly write functions. Additionally from a user experience point of view, mutations provide a solution for optimistic UI updates, which is something dApp developers have been wanting for a long time. Lastly with the whole protocol now defined in GraphQL, existing application layer code generators can now be used to hasten dApp development.

## Terminology

_Mutation_: GraphQL Mutation.  
_Resolver_: Resolver function that's mapped to a mutation.  
_Resolver State_: The resolver function's state (transactions sent, data logged, etc).  
_Optimistic Response_: A response given to the dApp that predicts what the outcome of the mutation's execution will be. If it is incorrect, it will be overwritten with the actual result.  

## Detailed Design

This is the main section of the RFC. What does the proposal include? What are the proposed interfaces/APIs? How are different affected parties, such as users, developers or node operators affected by the change and how are they going to use it?

TODO: design the developer flow, below it have a detailed spec of each aspect.

## Compatibility

Is this proposal backwards-compatible or is it a breaking change? If it is breaking, how could this be mitigated (think: migrations, announcing ahead of time like with hard forks, etc.)?

TODO: no breaking changes will be introduced. This is an optional add-on to existing and new subgraphs.

## Drawbacks and Risks

Why might we _not_ want to do this? What cost would implementing this proposal incur? What risks would be introduced by going down this path?

TODO: compat issues (ES5)

## Alternatives

What other designs have been considered, if any? For what reasons have they not been chosen? Are there workarounds that make this change less necessary?

TODO: talk about existing alternatives, and how they're ineffecient.

## Open Questions

What are unresolved questions?
