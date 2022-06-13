# RFC-0009: Block Triggers hash

<dl>
  <dt>Author</dt>
  <dd>Eva Paulino Pace</dd>

  <dt>RFC pull request</dt>
  <dd><a href="https://github.com/graphprotocol/rfcs/pull/26">https://github.com/graphprotocol/rfcs/pull/26</a></dd>

  <dt>Date of submission</dt>
  <dd>2022-05-16</dd>

  <dt>Date of approval</dt>
  <dd>YYYY-MM-DD</dd>
</dl>

## Contents

<!-- toc -->

## Summary

The "Block Triggers hash" will allow indexers to compare their subgraph's main
source of entropy, the chain provider, by storing a hash of the fetched data
that gets into any subgraph's mapping code.

## Goals & Motivation

One of The Graph's coolest features is that queries are deterministic, and
given a `Qm` subgraph hash, indexing it should **always** give the same result.
This is possible because we inherit the blockchain's determinism property,
however there's a big loophole which can break this amazing feature, which is
**the chain provider**.

Currently the main (or only) type of connection we give as option to indexers
(in The Graph Network) is the **JSON-RPC** one. To use it, they can either run a
node themselves or use a third party service like Alchemy. Either way the
provider can be faulty and give incorrect results for a number of different
reasons.

To be a little more specific, let's say there are indexers/nodes `A` and `B`.
Both are indexing subgraph `Z`. Indexer `A` is using Alchemy and `B` is using
Infura.

Given a block `14_722_714` of a determined hash, both providers will very
likely give the same result for these two values (block number and hash),
however other fields such as `gas_used` or `total_difficulty` could be
incorrect. And yes, ideally they would always be correct since they are chain
providers, that's their main job, however what I'm describing is the exact
issue we've faced when testing indexing Ethereum mainnet with the Firehose.

These field/value differences between providers are directly fed into the
subgraph mappings, which are the current input of the POI algorithm and the
base of The Graph's determinism property. Not taking the possible faultyness
of the chain providers into account, can break determinism altogether.

And the biggest problem today is that, to spot these POI differences, we have
to index subgraphs that **use those values in their mappings**. If by any chance
in Firehose shootout we've done in the **integration cluster**, there were no
subgraphs using these values **we wouldn't spot any POI differences**, which
is a **very severe issue**.

POI differences described in the Firehose shootout for reference:
https://gist.github.com/evaporei/660e57d95e6140ca877f338426cea200.

So in summary, the problems being described above are:

- That currently we consider the **chain provider** as a **source of truth**,
 which can only be questioned in behalf of re-orgs;
- We don't have a good way to compare provider input (that could spot POI
 differences) without the indirection of a subgraph mapping.

## Urgency

The urgency per se is low because we can spot POI differences by using
subgraphs themselves, but in the long run it's very likely we'll need some sort
of way to compare provider input between nodes/indexers so that we're more
efficient and accurate when testing new providers or forms of fetching chain
data (eg: Firehose).

## Terminology

`bth` is a candidate table name for "block triggers hash" (up to change).

## Detailed Design

A subgraph can have multiple handlers, each of them can have as their parameter
one of the following types (in Ethereum):

- Event log;
- Call;
- Block.

Currently this is the data structure that we use in code for representing what
gets into a mapping handler ([source](https://github.com/graphprotocol/graph-node/blob/b8faa45fc96f53226648ca65ddacff164b75e018/chain/ethereum/src/trigger.rs#L44)):

```rust
pub enum MappingTrigger {
    Log {
        block: Arc<LightEthereumBlock>,
        transaction: Arc<Transaction>,
        log: Arc<Log>,
        params: Vec<LogParam>,
        receipt: Option<Arc<TransactionReceipt>>,
    },
    Call {
        block: Arc<LightEthereumBlock>,
        transaction: Arc<Transaction>,
        call: Arc<EthereumCall>,
        inputs: Vec<LogParam>,
        outputs: Vec<LogParam>,
    },
    Block {
        block: Arc<LightEthereumBlock>,
    },
}
```

> Note: This actually gets converted to either an `AscEthereumLog`,
> `AscEthereumCall` or `AscEthereumBlock` to be called correctly by
> the subgraph's AssemblyScript mapping/handler code.

This data that gets filled by whatever provider the node/indexer is using, which
can be from **JSON-RPC** or the **Firehose**.

> Note: Each chain has their `MappingTrigger` definition, we're using Ethereum's
> for simplicity.

The code that actually executes the mapping code for each trigger candidate
looks like this (with a ton of simplification):

```rust
fn process_triggers(
    &self,
    triggers: Vec<C::TriggerData>,
) -> Result<(), MappingError> {
    for trigger in triggers {
        self.process_trigger(&trigger)?;
    }

    Ok(())
}
```

This is executed per relevant block, given the subgraph's data sources
(contracts that it's "listening"). To give a bit of a perspective in numbers,
the `uniswap-v2` subgraph for example usually has around ~30 triggers per block.

The proposal here would be to hash this vector per block and save in a database
record (under the subgraph namespace `sgdX`), resulting in a value that could
look like this:

```
id           | 61c7e6beb2cda15ca38fb09506abecd6429f53ce985b9509a0319ee9bb3b013d
subgraph_id  | QmZwNVFxXjUcSD42n5Lztg2D4DorZLUPGv8gTAXQpf5Wy9
block_number | 22062424
block_hash   | \x92cb334274a3f78c4a2bab96853992deca00fd57ee8c44c69c93dfd9f012f707
digest       | \x2b6ecfca88033617cf5bfbfd332ba593591bab1a955e049271d2fd7696a55a9e5d4024282b96a639f2077d0cdaa99000734768ae106498a8514f0819b64fc8f65c9637dd788e472e3287eb15e3945338c27e38eaec8a27268aca8cbb45f59d7cbb9893611b9e56f4850dc5696673f7a8c7b695d2d9127811856e1ccc76a942a29dca9e1a92287c807470c3dfd7d3f0fffeb66d29a22278a7746ddf4fb9a6f76a6ac11dd98cc65f5bfacda4ebdbca51e161fadaffd3434594886ded4301929e9a6098995277c9b36513a0592e06d0a6a8035506c281bd7e0e14e01149a9122cfd6e980f65519a8eb28b8612c245ef66f671761361d4317d53c9e4af03ed4999a0
provider     | mainnet-0
created_at   | 2022-05-13 22:27:08.304456+00
```

Technical considerations:

- It will probably make sense to first convert this `Vec<Trigger>` into another
  data structure, where there's only one `LightEthereumBlock` and all repeated
  data gets converted into a `Set` (eg: `BTreeSet`);
- On the hash itself
  - The `digest` field can be a lot smaller, the value here is from a POI,
    which we don't need to keep the same properties;
  - It should be calculated in a separate thread/future to the mapping code
    execution, so it doesn't degrade performance;
- On the database side
  - As mentioned before, this data should go under the subgraph's namespaced 
    database, aka `sgdX`, just as where the current POI table stands `poi2$`;
  - The table name could be something along the lines of `bth` (block triggers
    hash).

This value could then be exposed via the `indexing status API`, just like the
POI, but it could be used by indexers (or us) to compare if the data
`graph-node` fetched, given a `provider`, `subgraph` and `block` has the same
result.

We could also add a utility `graphman` command that fetches that specific data
for the `provider`, `subgraph` and `block`. This data could be either be
compared with other providers in `graph-node` itself, or the task could perhaps
be delegated to the new `graph-ixi` cross checking tool.

This would allow us (and indexers!) to be:

- Confident that different providers give the same results;
- Give another tool to catch determinism bugs (very important given the
  properties we want to maintain);
- And perhaps for the future, add value to the network itself outside the
  side utility.

## Compatibility

There are no backwards compatibility concerns.

## Drawbacks and Risks

The only point that should be well thought of in the implementation side is
how to calculate the `Vec<Trigger>` hash efficiently, so that it doesn't go
over the same data more than once (eg: same block).

Also, the new database table should keep the same properties that the other
subgraph's entity (and POI) tables have, which have their data "re-made" on
re-orgs, rewinds, etc.

## Alternatives

We could store each `MappingTrigger` hash in a separate database recored,
however that would create many more database results than necessary in my
opinion. This would make more sense if we were actually storing the data
itself, not a hash of it.

## Open Questions

- What hash algorithm to use?
- What table name should we use?
- Do we want to store any more provider data?

> Note: Ethereum calls aren't being included in this RFC, however since we
> already store those in a "call cache" table, we can just hash the stored
> data, perhaps in a different table.
