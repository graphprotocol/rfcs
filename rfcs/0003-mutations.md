# RFC-0003: Mutations

<dl>
  <dt>Author</dt>
  <dd>dOrg: Jordan Ellis, Nestor Amesty</dd>

  <dt>RFC pull request</dt>
  <dd><a href="https://github.com/graphprotocol/rfcs/pull/10">URL</a></dd>

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

GraphQL mutations allow developers to add executable functions to their schema. Callers can invoke these functions using GraphQL queries. An introduction to how mutations are defined and work can be found [here](https://graphql.org/learn/queries/#mutations). This RFC will assume the reader understands how to use GraphQL mutations in a traditional Web2 application. This proposal describes how mutations are added to The Graph's toolchain, and used to replace web3 write operations the same way The Graph has replaced Web3 read operations.

## Goals & Motivation

The Graph has created a read semantic layer that describes smart contract protocols, which has made it easier to build applications ontop of complex protocols. Since dApps have two primary interactions with web3 protocols (reading & writing), the next logical addition is write support.

Protocol developers that use a subgraph still often publish a Javascript wrapper library for their dApp developers (examples: [DAOstack](https://github.com/daostack/client), [ENS](https://github.com/ensdomains/ensjs), [LivePeer](https://github.com/livepeer/livepeerjs/tree/master/packages/sdk), [DAI](https://github.com/makerdao/dai.js/tree/dev/packages/dai), [Uniswap](https://github.com/Uniswap/uniswap-sdk)). This is done to help speed up dApp development and promote consistency with protocol usage patterns. With the addition of mutations to the Graph Protocol's GraphQL tooling, Web3 reading & writing can now both be invoked through GraphQL queries. dApp developers can now simply refer to a single GraphQL schema that defines the entire protocol.

## Urgency

This is urgent from a developer experience point of view. With this addition, it eliminates the need for protocol developers to manually wrap GraphQL query interfaces alongside developer-friendly write functions. Additionally, mutations provide a solution for optimistic UI updates, which is something dApp developers have been seeking for a long time (see [here](https://github.com/aragon/nest/issues/21)). Lastly with the whole protocol now defined in GraphQL, existing application layer code generators can now be used to hasten dApp development ([some examples](https://dev.to/graphqleditor/top-3-graphql-code-generators-1gnj)).

## Terminology

* _Mutation_: A GraphQL mutation.  
* _Resolver_: Function that is used to execute a mutation.  
* _Mutation State_: The state of a mutation being executed.  
* _Optimistic Response_: A response given to the dApp that predicts what the outcome of the mutation's execution will be. If it is incorrect, it will be overwritten with the actual result.  

## Detailed Design

### Mutation Manifest

`subgraph.yaml`
```yaml
specVersion: ...
...
mutations:
  specVersion: 0.0.1
  repository: https://npmjs.com/package/...
  schema:
    file: ./mutations/schema.graphql
  resolvers:
    kind: javascript
    file: ./mutations/index.js
dataSources: ...
...
```

Alternatively, the mutation manifest can be external like so:  
`subgraph.yaml`
```yaml
specVersion: ...
...
mutations:
  file: ./mutations/mutations.yaml
dataSources: ...
...
```
`mutations/mutations.yaml`
```yaml
specVersion: 0.0.1
repository: https://npmjs.com/package/...
schema:
  file: ./schema.graphql
resolvers:
  kind: javascript
  file: ./index.js
```

### Mutation Schema
`schema.graphql`
```graphql
type MyEntity @entity {
  id: ID!
  name: String!
  value: BigInt!
}
```

`mutations/schema.graphql`
```graphql
input MyEntityOptions {
  name: String!
  value: BigInt!
}

type Mutation {
  createEntity(
    options: MyEntityOptions!
  ): MyEntity!

  setEnityName(
    entity: MyEntity!
    name: String!
  ): MyEntity!
}
```
**NOTE:** GraphQL types from the subgraph's schema.graphql are automatically included in this file.

### Mutation Resolvers
`mutations/index.js`
```javascript
const resolvers = {
  Mutation: {
    async createEntity (_, args, context) {
      // Extract mutation arguments
      const { name, value } = args.options

      // Use configuration properties created by the
      // config generator functions below
      const { ethereum, ipfs } = context.graph.config

      // Fetch datasource addresses & abis
      const { MyContract } = context.graph.datasources
      await MyContract.abi
      await MyContract.address

      // Modify a state object, which relays updates back
      // to the subscribed dApp
      const { mutationState } = context.graph
      mutationState.addTransaction("tx_hash")

      ...
    },
    async setEntityName (_, args, context) {
      ...
    }
  }
}

// Configuration Setters
const config = {
  // These function arguments are passed in by the dApp
  ethereum (provider) {
    return new ethers.providers.Web3Provider(provider)
  },
  ipfs (provider) {
    return new IPFS(provider)
  },
  customProperty (value) {
    return value + 2
  }
}

export default {
  resolvers,
  config
}
```

### dApp Integration
```javascript
const {
  createMutations,
  createMutationsLink,
  useMutation
} = require("@graphprotocol/mutations-ts")
const myMutations = require("mutations-js-module")

const mutations = createMutations({
  mutations: myMutations,
  subgraph: "my-subgraph",
  node: "http://localhost:8080",
  // Configuration Getters
  config: {
    ethereum: async () => {
      const { ethereum } = (window as any)
      await ethereum.enable()
      return ethereum
    },
    ipfs: "http://localhost:5001",
    customProperty: 5
  }
})

// Create an Apollo Links
const mutationLink = createMutationLink({ mutations })
const queryLink = createHttpLink({
  uri: "http://localhost:5001/subgraphs/name/my-subgraph"
})

const link = split(
  ({ query }) => {
    const node = getMainDefinition(query);
    return node.kind === "OperationDefinition" &&
           node.operation === "mutation"
  },
  mutationLink,
  queryLink
);

// Create Apollo Client
const client = new ApolloClient({
  link,
  cache: new InMemoryCache()
})

const CREATE_ENTITY = gql`
  mutation createEntity($options: MyEntityOptions) {
    createEntity(options: $options) {
      id
      name
      value
    }
  }
`

// state === resolver's state
const [exec, { loading, state }] = useMutation(
  CREATE_ENTITY,
  {
    client,
    variables: {
      options: { name: "...", value: 5 }
    }
  }
)

// Optimistic responses can be used to update
// the UI before the execution has finished
const [exec, { loading, state }] = useMutation(
  CREATE_ENTITY,
  {
    optimisticResponse: {
      myEntity: {
        id: "...",
        name: "...",
        value: 5,
      }
    },
    update(proxy, { data }) {
      // result = data.myEntity
    },
    onError(error) {
      ...
    },
    variables: {
      options: { name: "...", value: 5 }
    }
  }
)
```

## Compatibility

No breaking changes will be introduced, as mutations are an optional add-on to a subgraph.

## Drawbacks and Risks

I have some thoughts but they are rather verbose and tangential. Would love some feedback on this from others first.

## Alternatives

The existing alternative that protocol developers are creating for dApp developers has been described above.

## Open Questions

- **Should the resolvers module be ES5 compliant?**  
  The prototype was originally developed under these conditions. ES5 compliance has since been abandoned as it has proven nearly impossible to successfully transpile all dependencies into a single monolithic module.
- **How should the dApp configure the resolvers module?**  
  The dApp knows best how to: connect to the various web3 networks, handle key signature requests, and all other user / dApp specific things. Therefore a way for the dApp to configure the resolvers in a specific way is required. The resolvers are still responsible for defining what configuration options are required by the dApp (IPFS provider, Web3 provider, etc).
- **What paradigm should the mutation state follow?**  
  One option is to have the resolver's call into a single interface that modifies the backing data. Whenever this data is modified, the entirety of it is passed to the dApp. The downside here is that the dApp doesn't know what has changed within the data, and is forced to represent it in its entirety in order to not miss anything.  

  Another option is to implement something similar to Redux, where the resolvers fire off events with corresponding payloads of data. These events map to reducers, which take in this payload of data and decide what to do with it. The dApp could implement these reducers, and choose how it would want to react to the various events.