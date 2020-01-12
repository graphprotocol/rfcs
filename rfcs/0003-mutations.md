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

* _Mutations_: Collection of mutations.  
* _Mutation_: A GraphQL mutation.  
* _Mutations Schema_: A GraphQL schema that defines a `type Mutation` that contains all mutations. Additionally, this schema can define other types to be used by the mutations, such as `input` and `interface` types.  
* _Mutations Manifest_: A YAML manifest file that is used to add mutations to an existing subgraph manifest.  
* _Mutation Resolvers_: Code module that contains all resolvers.  
* _Resolver_: Function that is used to execute a mutation.  
* _Mutation State_: The state of a mutation being executed. It's passed to the resolver through the mutation context.  
* _Mutation Context_: A context object that's created for every mutation that's executed. It's passed as an argument to the resolver.  
* _Config_: Collection of config properties required by the mutation resolvers.  
* _Config Property_: A single property within the config (ex: ipfs, ethereum, etc).  
* _Config Generator_: A function that takes a config value, and returns a config property. For example, "localhost:5001" as a config value gets turned into a new IPFS client by the config generator.
* _Config Value_: An initialization value that's passed into the config generator. This config value is provided by the dApp developer.
* _Optimistic Response_: A response given to the dApp that predicts what the outcome of the mutation's execution will be. If it is incorrect, it will be overwritten with the actual result.  

## Detailed Design

The sections below illustrate how a developer would add mutations to an existing subgraph, and then add those mutations to a dApp.

### Mutations Manifest

The subgraph manifest (`subgraph.yaml`) now has an extra property named `mutations` which is the mutations manifest.

`subgraph.yaml`
```yaml
specVersion: ...
...
mutations:
  repository: https://npmjs.com/package/...
  schema:
    file: ./mutations/schema.graphql
  resolvers:
    apiVersion: 0.0.1
    kind: javascript/es5
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
repository: https://npmjs.com/package/...
schema:
  file: ./schema.graphql
resolvers:
  apiVersion: 0.0.1
  kind: javascript/es5
  file: ./index.js
```

### Mutations Schema

The mutations schema defines all of the mutations in our subgraph. The mutations schema is a super-set of the subgraph's schema. For example, starting from a base subgraph schema:  
`schema.graphql`
```graphql
type MyEntity @entity {
  id: ID!
  name: String!
  value: BigInt!
}
```

Developers can define mutations that reference these subgraph schema types. Additionally new `input` and `interface` types can be defined for the mutations to use:  
`mutations/schema.graphql`
```graphql
input MyEntityOptions {
  name: String!
  value: BigInt!
}

interface NewNameSet {
  oldName: String!
  newName: String!
}

type Mutation {
  createEntity(
    options: MyEntityOptions!
  ): MyEntity!

  setEnityName(
    entity: MyEntity!
    name: String!
  ): NewNameSet!
}
```

`graph-cli` handles the combining, parsing, and validating of these two schemas. The `graph-cli` verifies that the mutations schema defines a `type Mutation`, that all of the mutations within it are defined in the resolvers module (see next section).  

### Mutation Resolvers

Each mutation within the schema must have a corresponding resolver function defined. Resolvers will be invoked by whatever engine executes the query. They are executed locally within the client application.

Mutation resolvers of kind `javascript` take the form of a javascript module. This module is expected to have a default export that contains the following properties:
  * resolvers - The mutation resolver functions.
  * config - A collection of config generators.

`mutations/index.js`
```javascript
const resolvers = {
  Mutation: {
    async createEntity (_, args, context) {
      // Extract mutation arguments
      const { name, value } = args.options

      // Use config properties created by the
      // config generator functions
      const { ethereum, ipfs } = context.graph.config

      // Fetch datasource addresses & abis
      const { MyContract } = context.graph.datasources
      await MyContract.abi
      await MyContract.address

      // Modify a state object, which relays updates back
      // to the subscribed dApp
      const { state } = context.graph
      state.addTransaction("tx_hash")

      ...
    },
    async setEntityName (_, args, context) {
      ...
    }
  }
}

// Config generators
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
  },
  rootProperty: {
    nestedProperty (value) {
      ...
    }
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
  // Config values, which will be passed to the generators
  config: {
    ethereum: async () => {
      const { ethereum } = (window as any)
      await ethereum.enable()
      return ethereum
    },
    ipfs: "http://localhost:5001",
    customProperty: 5,
    rootProperty: {
      nestedProperty: "foo"
    }
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

// state === mutation state
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

- **What paradigm should the mutation state follow?**  
  One option is to have the resolver's call into a single interface that modifies the backing data. Whenever this data is modified, the entirety of it is passed to the dApp. The downside here is that the dApp doesn't know what has changed within the data, and is forced to represent it in its entirety in order to not miss anything.  

  Another option is to implement something similar to Redux, where the resolvers fire off events with corresponding payloads of data. These events map to reducers, which take in this payload of data and decide what to do with it. The dApp could implement these reducers, and choose how it would want to react to the various events.