# RFC-0004: Fulltext Search

<dl>
  <dt>Author</dt>
  <dd>Jorge Olivero</dd>

  <dt>RFC pull request</dt>
  
  <dt>Obsoletes (if applicable)</dt>
  <dd>-</dd>

  <dt>Date of submission</dt>
  <dd>2020-03-06</dd>

  <dt>Date of approval</dt>
  <dd>2020-03-06</dd>

  <dt>Approved by</dt>
  <dd></dd>
</dl>

## Contents

<!-- toc -->

## Summary

Initial designs of subgraph composition allowed imports to be linked dynamically
at query time when schemas are imported by subgraph name. This setup makes it
difficult to provide a reliable experience to end users who are querying because
it provides no guarantees that the dynamically linked subgraphs are deployed and
fully indexed. To address this, schema imports should be static.

## Goals & Motivation

Finish and deploy subgraph composition.

## Urgency

-

## Detailed Design

The `graph-cli` and `graph-node` components need to be updated.

### graph-cli

1. Resolve all imports by name to the latest subgraph version hash leveraging a
   new rpc method on the graph-node when the `graph dep update` (?) command is run
2. Incorporate the subgraph version hashes from the previous task into one of
   these options:
   - A new yaml document mapping `SubgraphName` -> `SubgraphId`
   - A new yaml field on the existing manifest document mapping `SubgraphName` ->
   `SubgraphId`
   - Rewrite the existing graphql schema and replace all `@import( from: { name:
   "ethereum/mainnet" } )` with `@import( from: { id: "Qm...", name:
   "ethereum/mainnet" } )`

### graph-node

1. Add json rpc method to resolve a set of subgraph names to their latest hash
   version
2. Depending on how locked dependencies are incorporated into the
   subgraph_deploy command
   - If a separate hash is provided in the deploy command for the locked
   dependencies: 
     1. Resolve the hash
     2. Save the locked dependencies into a new/existing relation
     3. Incorporate the locked dependencies into SubgraphManifest validation
   - If the locked dependencies are a new mapping on the manifest then:
     1. Ensure the new properties on the manifest are saved correctly
     2. Incorporate the locked dependencies into the SubgraphManifest validation
   - If the locked dependencies are just a part of the GraphQL input schema
   definition:
     1. Incorporate locked dependencies into the SubgraphManifest validation
3. Remove the use of `SchemaReference::ByName` in the subgraph manifest
validation code and the store schema cache
4. Remove the concept of unstable imports in the schema cache 
 
## Compatibility

This proposal does not change any existing interfaces, so no migrations 
will be necessary for existing subgraph deployments. 

## Drawbacks and Risks

## Alternatives

## Open Questions
