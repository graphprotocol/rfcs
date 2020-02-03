# RFC-0003: Mutations

<dl>
  <dt>Author</dt>
  <dd>dOrg: Jordan Ellis, Nestor Amesty</dd>

  <dt>RFC pull request</dt>
  <dd><a href="https://github.com/graphprotocol/rfcs/pull/10">URL</a></dd>

  <dt>Date of submission</dt>
  <dd>2019-12-20</dd>

  <dt>Date of approval</dt>
  <dd>2020-2-03</dd>

  <dt>Approved by</dt>
  <dd>Jannis Pohlmann</dd>
</dl>

## Contents

<!-- toc -->

## Summary

GraphQL mutations allow developers to add executable functions to their schema. Callers can invoke these functions using GraphQL queries. An introduction to how mutations are defined and work can be found [here](https://graphql.org/learn/queries/#mutations). This RFC will assume the reader understands how to use GraphQL mutations in a traditional Web2 application. This proposal describes how mutations are added to The Graph's toolchain, and used to replace Web3 write operations the same way The Graph has replaced Web3 read operations.

## Goals & Motivation

The Graph has created a read semantic layer that describes smart contract protocols, which has made it easier to build applications on top of complex protocols. Since dApps have two primary interactions with Web3 protocols (reading & writing), the next logical addition is write support.

Protocol developers that use a subgraph still often publish a Javascript wrapper library for their dApp developers (examples: [DAOstack](https://github.com/daostack/client), [ENS](https://github.com/ensdomains/ensjs), [LivePeer](https://github.com/livepeer/livepeerjs/tree/master/packages/sdk), [DAI](https://github.com/makerdao/dai.js/tree/dev/packages/dai), [Uniswap](https://github.com/Uniswap/uniswap-sdk)). This is done to help speed up dApp development and promote consistency with protocol usage patterns. With the addition of mutations to the Graph Protocol's GraphQL tooling, Web3 reading & writing can now both be invoked through GraphQL queries. dApp developers can now simply refer to a single GraphQL schema that defines the entire protocol.

## Urgency

This is urgent from a developer experience point of view. With this addition, it eliminates the need for protocol developers to manually wrap GraphQL query interfaces alongside developer-friendly write functions. Additionally, mutations provide a solution for optimistic UI updates, which is something dApp developers have been seeking for a long time (see [here](https://github.com/aragon/nest/issues/21)). Lastly with the whole protocol now defined in GraphQL, existing application layer code generators can now be used to hasten dApp development ([some examples](https://dev.to/graphqleditor/top-3-graphql-code-generators-1gnj)).

## Terminology

* _Mutations_: Collection of mutations.  
* _Mutation_: A GraphQL mutation.  
* _Mutations Schema_: A GraphQL schema that defines a `type Mutation`, which contains all mutations. Additionally this schema can define other types to be used by the mutations, such as `input` and `interface` types.  
* _Mutations Manifest_: A YAML manifest file that is used to add mutations to an existing subgraph manifest. This manifest can be stored in an external YAML file, or within the subgraph manifest's YAML file under the `mutations` property.  
* _Mutation Resolvers_: Code module that contains all resolvers.  
* _Resolver_: Function that is used to execute a mutation's logic.  
* _Mutation Context_: A context object that's created for every mutation that's executed. It's passed as the 3rd argument to the resolver function.  
* _Mutation States_: A collection of mutation states. One is created for each mutation being executed in a given query.  
* _Mutation State_: The state of a mutation being executed. Also referred to in this document as "_State_". It is an aggregate of the core & extended states (see below). dApp developers can subscribe to the mutation's state upon execution of the mutation query. See the `useMutation` examples below.  
* _Core State_: Default properties present within every mutation state. Some examples: `events: Event[]`, `uuid: string`, and `progress: number`.  
* _Extended State_: Properties the mutation developer defines. These are added alongside the core state properties in the mutation state. There are no bounds to what a developer can define here. See examples below.  
* _State Events_: Events emitted by mutation resolvers. Also referred to in this document as "_Events_". Events are defined by a `name: string` and a `payload: any`. These events, once emitted, are given to reducer functions which then update the state accordingly.  
* _Core Events_: Default events available to all mutations. Some examples: `PROGRESS_UPDATE`, `TRANSACTION_CREATED`, `TRANSACTION_COMPLETED`.  
* _Extended Events_: Events the mutation developer defines. See examples below.  
* _State Reducers_: A collection of state reducer functions.  
* _State Reducer_: Reducers are responsible for translating events into state updates. They take the form of a function that has the inputs [event, current state], and returns the new state post-event. Also referred to in this document as "_Reducer(s)_".  
* _Core Reducers_: Default reducers that handle the processing of the core events.  
* _Extended Reducers_: Reducers the mutation developer defines. These reducers can be defined for any event, core or extended. The core & extended reducers are run one after another if both are defined for a given core event. See examples below.  
* _State Updater_: The state updater object is used by the resolvers to dispatch events. It's passed to the resolvers through the mutation context like so: `context.graph.state`.
* _State Builder_: An object responsible for (1) initializing the state with initial values and (2) defining reducers for events.  
* _Core State Builder_: A state builder that's defined by default. It's responsible for initializing the core state properties, and processing the core events with its reducers.  
* _Extended State Builder_: A state builder defined by the mutation developer. It's responsible for initializing the extended state properties, and processing the extended events with its reducers.  
* _Mutations Config_: Collection of config properties required by the mutation resolvers. Also referred to in this document as "_Config_". All resolvers share the same config. It's passed to the resolver through the mutation context like so: `context.graph.config`.  
* _Config Property_: A single property within the config (ex: ipfs, ethereum, etc).  
* _Config Generator_: A function that takes a config argument, and returns a config property. For example, "localhost:5001" as a config argument gets turned into a new IPFS client by the config generator.
* _Config Argument_: An initialization argument that's passed into the config generator function. This config argument is provided by the dApp developer.
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
    types: ./mutations/index.d.ts
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
specVersion: ...
repository: https://npmjs.com/package/...
schema:
  file: ./schema.graphql
resolvers:
  apiVersion: 0.0.1
  kind: javascript/es5
  file: ./index.js
  types: ./index.d.ts
```

NOTE: `resolvers.types` is required. More on this below.

### Mutations Schema

The mutations schema defines all of the mutations in the subgraph. The mutations schema builds on the subgraph schema, allowing the use of types from the subgraph schema, as well as defining new types that are used only in the context of mutations. For example, starting from a base subgraph schema:  
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

  setEntityName(
    entity: MyEntity!
    name: String!
  ): NewNameSet!
}
```

`graph-cli` handles the parsing and validating of these two schemas. It verifies that the mutations schema defines a `type Mutation` and that all of the mutations within it are defined in the resolvers module (see next section).  

### Mutation Resolvers

Each mutation within the schema must have a corresponding resolver function defined. Resolvers will be invoked by whatever engine executes the mutation queries (ex: Apollo Client). They are executed locally within the client application.

Mutation resolvers of kind `javascript/es5` take the form of an ES5 javascript module. This module is expected to have a default export that contains the following properties:
  * `resolvers: MutationResolvers` - The mutation resolver functions. The shape of this object must match the shape of the `type Mutation` defined above. See the example below for demonstration of this. Resolvers have the following prototype, [as defined in graphql-js](https://github.com/graphql/graphql-js/blob/9dba58eeb6e28031bec7594b6df34c4fd74459b0/src/type/definition.js#L906):  
    ```typescript
    import { GraphQLFieldResolver } from 'graphql'

    interface MutationContext<
      TConfig extends ConfigGenerators,
      TState,
      TEventMap extends EventTypeMap
    > {
      [prop: string]: any,
      graph: {
        config: ConfigProperties<TConfig>,
        dataSources: DataSources,
        state: StateUpdater<TState, TEventMap>
      }
    }

    interface MutationResolvers<
      TConfig extends ConfigGenerators,
      TState,
      TEventMap extends EventTypeMap
    > {
      Mutation: {
          [field: string]: GraphQLFieldResolver<
            any,
            MutationContext<TConfig, TState, TEventMap>
          >
      }
    }
    ```
  * `config: ConfigGenerators` - A collection of config generators. The config object is made up of properties, that can be nested, but all terminate in the form of a function with the prototype:
    ```typescript
    type ConfigGenerator<TArg, TRet> = (arg: TArg) => TRet

    interface ConfigGenerators {
      [prop: string]: ConfigGenerator<any, any> | ConfigGenerators
    }
    ```
    See the example below for a demonstration of this.

  * `stateBuilder: StateBuilder` (optional) - A state builder interface responsible for (1) initializing extended state properties and (2) reducing extended state events. State builders implement the following interface:  
    ```typescript
    type MutationState<TState> = CoreState & TState
    type MutationEvents<TEventMap> = CoreEvents & TEventMap

    interface StateBuilder<TState, TEventMap extends EventTypeMap> {
      getInitialState(uuid: string): TState,
      // Event Specific Reducers
      reducers?: {
        [TEvent in keyof MutationEvents<TEventMap>]?: (
          state: MutationState<TState>,
          payload: InferEventPayload<TEvent, TEventMap>
        ) => OptionalAsync<Partial<MutationState<TState>>>
      },
      // Catch-All Reducer
      reducer?: (
        state: MutationState<TState>,
        event: Event
      ) => OptionalAsync<Partial<MutationState<TState>>>
    }

    interface EventPayload { }

    interface Event {
      name: string
      payload: EventPayload
    }

    interface EventTypeMap {
      [name: string]: EventPayload
    }

    // Optionally support async functions
    type OptionalAsync<T> = Promise<T> | T

    // Infer the payload type from the event name, given an EventTypeMap
    type InferEventPayload<
      TEvent extends keyof TEvents,
      TEvents extends EventTypeMap
    > = TEvent extends keyof TEvents ? TEvents[TEvent] : any
    ```
    See the example below for a demonstration of this.

For example:  
`mutations/index.js`
```typescript
import {
  Event,
  EventPayload,
  MutationContext,
  MutationResolvers,
  MutationState,
  StateBuilder,
  ProgressUpdateEvent
} from "@graphprotocol/mutations"

import gql from "graphql-tag"
import { ethers } from "ethers"
import {
  AsyncSendable,
  Web3Provider
} from "ethers/providers"
import IPFS from "ipfs"

// Typesafe Context
type Context = MutationContext<Config, State, EventMap>

/// Mutation Resolvers
const resolvers: MutationResolvers<Config, State, EventMap> = {
  Mutation: {
    async createEntity (source: any, args: any, context: Context) {
      // Extract mutation arguments
      const { name, value } = args.options

      // Use config properties created by the
      // config generator functions
      const { ethereum, ipfs } = context.graph.config

      // Create ethereum transactions...
      // Fetch & upload to ipfs...

      // Dispatch a state event through the state updater
      const { state } = context.graph
      await state.dispatch("PROGRESS_UPDATE", { progress: 0.5 })

      // Dispatch a custom extended event
      await state.dispatch("MY_EVENT", { myValue: "..." })

      // Get a copy of the current state
      const currentState = state.current

      // Send another query using the same client.
      // This query would result in the graph-node's
      // entity store being fetched from. You could also
      // execute another mutation here if desired.
      const { client } = context
      await client.query({
        query: gql`
          myEntity (id: "${id}") {
            id
            name
            value
          }
        }`
      })

      ...
    },
    async setEntityName (source: any, args: any, context: Context) {
      ...
    }
  }
}

/// Config Generators
type Config = typeof config

const config = {
  // These function arguments are passed in by the dApp
  ethereum: (arg: AsyncSendable): Web3Provider => {
    return new ethers.providers.Web3Provider(arg)
  },
  ipfs: (arg: string): IPFS => {
    return new IPFS(arg)
  },
  // Example of a custom config property
  property: {
    // Generators can be nested
    a: (arg: string) => { },
    b: (arg: string) => { }
  }
}

/// (optional) Extended State, Events, and State Builder

// Extended State
interface State {
  myValue: string
}

// Extended Events
interface MyEvent extends EventPayload {
  myValue: string
}

type EventMap = {
  "MY_EVENT": MyEvent
}

// Extended State Builder
const stateBuilder: StateBuilder<State, EventMap> = {
  getInitialState(): State {
    return {
      myValue: ""
    }
  },
  reducers: {
    "MY_EVENT": async (state: MutationState<State>, payload: MyEvent) => {
      return {
        myValue: payload.myValue
      }
    },
    "PROGRESS_UPDATE": (state: MutationState<State>, payload: ProgressUpdateEvent) => {
      // Do something custom...
    }
  },
  // Catch-all reducer...
  reducer: (state: MutationState<State>, event: Event) => {
    switch (event.name) {
      case "TRANSACTION_CREATED":
        // Do something custom...
        break
    }
  }
}

export default {
  resolvers,
  config,
  stateBuilder
}

// Required Types
export {
  Config,
  State,
  EventMap,
  MyEvent
}
```

NOTE: It's expected that the mutations manifest has a `resolvers.types` file defined. The following types must be defined in the .d.ts type definition file:
  - `Config`
  - `State`
  - `EventMap`
  - Any `EventPayload` interfaces defined within the `EventMap`

### dApp Integration

In addition to the resolvers module defined above, the dApp has access to a run-time API to help with the instantiation and execution of mutations. This package is called `@graphprotocol/mutations` and is defined like so:
  - `createMutations` - Create a mutations interface which enables the user to `execute` a mutation query and `configure` the mutation module.  
    ```typescript
    interface CreateMutationsOptions<
      TConfig extends ConfigGenerators,
      TState,
      TEventMap extends EventTypeMap
    > {
      mutations: MutationsModule<TConfig, TState, TEventMap>,
      subgraph: string,
      node: string,
      config: ConfigArguments<TConfig>
      mutationExecutor?: MutationExecutor<TConfig, TState, TEventMap>
    }

    interface Mutations<
      TConfig extends ConfigGenerators,
      TState,
      TEventMap extends EventTypeMap
    > {
      execute: (query: MutationQuery<TConfig, TState, TEventMap>) => Promise<MutationResult>
      configure: (config: ConfigArguments<TConfig>) => void
    }

    const createMutations = <
      TConfig extends ConfigGenerators,
      TState = CoreState,
      TEventMap extends EventTypeMap = { },
    >(
      options: CreateMutationsOptions<TConfig, TState, TEventMap>
    ): Mutations<TConfig, TState, TEventMap> => { ... }
    ```

  - `createMutationsLink` - wrap the mutations created above in an ApolloLink.  
    ```typescript
    const createMutationsLink = <
      TConfig extends ConfigGenerators,
      TState,
      TEventMap extends EventTypeMap,
    > (
      { mutations }: { mutations: Mutations<TConfig, TState, TEventMap> }
    ): ApolloLink => { ... }
    ```

For applications using Apollo and React, a run-time API is available which mimics commonly used hooks and components for executing mutations, with the addition of having the mutation state available to the caller. This package is called `@graphprotocol/mutations-apollo-react` and is defined like so:
  - `useMutation` - see https://www.apollographql.com/docs/react/data/mutations/#executing-a-mutation  
    ```typescript
    import { DocumentNode } from "graphql"
    import {
      ExecutionResult,
      MutationFunctionOptions,
      MutationResult,
      OperationVariables
    } from "@apollo/react-common"
    import { MutationHookOptions } from "@apollo/react-hooks"
    import { CoreState } from "@graphprotocol/mutations"

    type MutationStates<TState> = {
      [mutation: string]: MutationState<TState>
    }

    interface MutationResultWithState<TState, TData = any> extends MutationResult<TData> {
      state: MutationStates<TState>
    }

    type MutationTupleWithState<TState, TData, TVariables> = [
      (
        options?: MutationFunctionOptions<TData, TVariables>
      ) => Promise<ExecutionResult<TData>>,
      MutationResultWithState<TState, TData>
    ]

    const useMutation = <
      TState = CoreState,
      TData = any,
      TVariables = OperationVariables
    >(
      mutation: DocumentNode,
      mutationOptions: MutationHookOptions<TData, TVariables>
    ): MutationTupleWithState<TState, TData, TVariables> => { ... }
    ```

  - `Mutation` - see https://www.howtographql.com/react-apollo/3-mutations-creating-links/  
    ```typescript
    interface MutationComponentOptionsWithState<
      TState,
      TData,
      TVariables
    > extends BaseMutationOptions<TData, TVariables> {
      mutation: DocumentNode
      children: (
        mutateFunction: MutationFunction<TData, TVariables>,
        result: MutationResultWithState<TState, TData>
      ) => JSX.Element | null
    }

    const Mutation = <
      TState = CoreState,
      TData = any,
      TVariables = OperationVariables
    >(
      props: MutationComponentOptionsWithState<TState, TData, TVariables>
    ): JSX.Element | null => { ... }
    ```

For example:  
`dApp/src/App.tsx`  
```typescript
import {
  createMutations,
  createMutationsLink
} from "@graphprotocol/mutations"
import {
  Mutation,
  useMutation
} from "@graphprotocol/mutations-apollo-react"
import myMutations, { State } from "mutations-js-module"
import { createHttpLink } from "apollo-link-http"

const mutations = createMutations({
  mutations: myMutations,
  // Config args, which will be passed to the generators
  config: {
    // Config args can take the form of functions to allow
    // for dynamic fetching behavior
    ethereum: async (): AsyncSendable => {
      const { ethereum } = (window as any)
      await ethereum.enable()
      return ethereum
    },
    ipfs: "http://localhost:5001",
    property: {
      a: "...",
      b: "..."
    }
  },
  subgraph: "my-subgraph",
  node: "http://localhost:8080"
})

// Create Apollo links to handle queries and mutation queries
const mutationLink = createMutationLink({ mutations })
const queryLink = createHttpLink({
  uri: "http://localhost:8080/subgraphs/name/my-subgraph"
})

// Create a root ApolloLink which splits queries between
// the two different operation links (query & mutation)
const link = split(
  ({ query }) => {
    const node = getMainDefinition(query)
    return node.kind === "OperationDefinition" &&
           node.operation === "mutation"
  },
  mutationLink,
  queryLink
)

// Create an Apollo Client
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

// exec: execution function for the mutation query
// loading: https://www.apollographql.com/docs/react/data/mutations/#tracking-mutation-status
// state: mutation state instance
const [exec, { loading, state }] = useMutation<State>(
  CREATE_ENTITY,
  {
    client,
    variables: {
      options: { name: "...", value: 5 }
    }
  }
)

// Access the mutation's state like so:
state.createEntity.myValue

// Optimistic responses can be used to update
// the UI before the execution has finished.
// More information can be found here:
// https://www.apollographql.com/docs/react/performance/optimistic-ui/
const [exec, { loading, state }] = useMutation(
  CREATE_ENTITY,
  {
    optimisticResponse: {
      __typename: "Mutation",
      createEntity: {
        __typename: "MyEntity",
        name: "...",
        value: 5,
        // NOTE: ID must be known so the
        // final response can be correlated.
        // Please refer to Apollo's docs.
        id: "id"
      }
    },
    variables: {
      options: { name: "...", value: 5 }
    }
  }
)
```
```html
// Use the Mutation JSX Component
<Mutation
  mutation={CREATE_ENTITY}
  variables={{options: { name: "...", value: 5 }}}
>
  {(exec, { loading, state }) => (
    <button onClick={exec} />
  )}
</Mutation>
```


## Compatibility

No breaking changes will be introduced, as mutations are an optional add-on to a subgraph.

## Drawbacks and Risks

Nothing apparent at the moment.

## Alternatives

The existing alternative that protocol developers are creating for dApp developers has been described above.

## Open Questions

- **How can mutations pickup where they left off in the event of an abrupt application shutdown?**
  Since mutations can contain many different steps internally, it would be ideal to be able to support continuing resolver execution in the event the dApp abruptly shuts down.

- **How can dApps understand what steps a given mutation will take during the course of its execution?**
  dApps may want to present to the user friendly progress updates, letting them know a given mutation is 3/4ths of the way through its execution (for example) and a high level description of each step. I view this as closely tied to the previous open question above, as we could support continuing resolver executions if we know what step it's currently undergoing. A potential implementation could include adding a `steps: Step[]` property to the core state, where `Step` looks similar to:
  ```typescript
  interface Step {
    id: string
    title: string
    description: string
    status: 'pending' | 'processing' | 'error' | 'finished'
    current: boolean
    error?: Error
    data: any
  }
  ```

  This, plus a few core events & reducers, would be all we need to render UIs like the ones seen here: https://ant.design/components/steps/

- **Should dApps be able to define event handlers for mutation events?**
  dApps may want to implement their own handlers for specific events emitted from mutations. These handlers would be different from the reducers, as we wouldn't want them to be able to modify the state. Instead they could store their own state elsewhere within the dApp based on the events.

- **Should the Graph Node's schema introspection endpoint respond with the "full" schema, including the mutations' schema?**
  Developers could fetch the "full" schema by looking up the subgraph's manifest, read the `mutations.schema.file` hash value, and fetching the full schema from IPFS. Should the graph-node support querying this full schema directly from the graph-node itself through the introspection endpoint?  

- **Will server side execution ever be a reality?**
  I have not thought of a trustless solution to this, am curious if anyone has any ideas of how we could make this possible.  

- **Will The Graph Explorer support mutations?**
  We could have the explorer client-side application dynamically fetch and include mutation resolver modules. Configuring the resolvers module dynamically is problematic though. Maybe there are a few known config properties that the explorer client supports, and for all others it allows the user to input config arguments (if they're base types).  
