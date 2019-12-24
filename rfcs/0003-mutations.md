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

* _Mutation_: GraphQL Mutation.  
* _Resolver_: Resolver function that's mapped to a mutation.  
* _Resolver State_: The resolver function's state (transactions sent, data logged, etc).  
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

Alternatively, you can store the mutations manifest externally like so:  
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
  createMutationsLink
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

const [exec] = useMutation(
  CREATE_ENTITY,
  {
    client,
    variables: {
      options: { name: "...", value: 5 }
    }
  }
)

// Or alternatively, we can subscribe to resolver
// state updates, and also utilize an optimistic response
const [exec, { loading, state }] = useMutationAndSubscribe(
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

// loading === boolean
// state === resolver's state
```

## Compatibility

No breaking changes will be introduced, as mutations are an optional add-on to a subgraph.

## Drawbacks and Risks

I have some thoughts but they are rather verbose and tangential. Would love some feedback on this from others first.

## Alternatives

The existing alternative that protocol developers are creating for dApp developers has been described above.

## Open Questions

- **Should the resolvers module be ES5 compliant?**  
  We've been operating under this assumption while developing the prototype. We have since scrapped this requirement as it has proven nearly impossible to successfully transpile our own source, along with all our dependencies, into a single monolithic module. If anyone has experience doing this I would love chat!
- **How should the dApp configure the resolvers module?**  
  The dApp knows best how to: connect to the various web3 networks, handle key signature requests, and all other user / dApp specific things. We need a way for the dApp to configure the resolvers in a specific way given the resolver's requirements (IPFS provider, Web3 provider, etc).
- **What paradigm should the resolver state follow?**  
  One option is to have the resolver's call into a single interface that modifys the backing data. Whenever this data is modified, the entirety of it is passed to the dApp. The downside here is that the dApp doesn't know what has changed within the data, and is forced to represent it in its entirety in order to not miss anything.  

  Another option is to implement something similar to Redux, where the resolvers fire off events with corresponding payloads of data. These events map to reducers, which take in this payload of data and decide what to do with it. The dApp could implement these reducers, and choose how it would want to react to the various events.