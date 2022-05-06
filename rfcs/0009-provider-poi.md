# RFC-0009: Provider POI

<dl>
  <dt>Author</dt>
  <dd>Eva Paulino Pace</dd>

  <dt>RFC pull request</dt>
  <dd><a href="https://github.com/graphprotocol/rfcs/pull/26">https://github.com/graphprotocol/rfcs/pull/26</a></dd>

  <dt>Date of submission</dt>
  <dd>2022-05-05</dd>

  <dt>Date of approval</dt>
  <dd>YYYY-MM-DD</dd>
</dl>

## Contents

<!-- toc -->

## Summary

A brief summary of the proposal in 1-3 paragraphs.

A "Provider POI" will allow indexers to compare their subgraph's main source of
entropy, the chain provider.

## Goals & Motivation

What are the reasons for proposing the change? Why is it needed? What will the
benefits be and for whom?

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
providers, that's their main job however what I'm describing is the exact
issue we've faced when testing indexing Ethereum mainnet with the Firehose.

And the biggest problem today is that, to spot these POI differences, we have
to run subgraphs that **use those values in their mappings**. If by any chance
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

How urgent is this proposal? How soon or late should or must an implementation
of this be available?

The urgency per se is low because we can spot POI differences by using
subgraphs themselves, but in the long run it's very likely we'll need some sort
of way to compare provider input between nodes/indexers so that we're more
efficient and accurate when testing new providers or forms of fetching chain
data (eg: Firehose).

## Terminology

What terminology are we introducing? If this RFC is for a new feature, what is
this feature going to be referred to? The goal is that we all speak the same
language and use the same terms when talking about the change.

TBD.

## Detailed Design

This is the main section of the RFC. What does the proposal include? What are
the proposed interfaces/APIs? How are different affected parties, such as users,
developers or node operators affected by the change and how are they going to
use it?

## Compatibility

Is this proposal backwards-compatible or is it a breaking change? If it is
breaking, how could this be mitigated (think: migrations, announcing ahead of
time like with hard forks, etc.)?

There are no backwards compatibility concerns.

## Drawbacks and Risks

Why might we _not_ want to do this? What cost would implementing this proposal
incur? What risks would be introduced by going down this path?

## Alternatives

What other designs have been considered, if any? For what reasons have they not
been chosen? Are there workarounds that make this change less necessary?

## Open Questions

What are unresolved questions?
