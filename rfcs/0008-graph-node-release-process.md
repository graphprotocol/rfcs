# RFC-0008: Graph Node Release Process

<dl>
  <dt>Author</dt>
  <dd>Eva Paulino Pace</dd>

  <dt>RFC pull request</dt>
  <dd><a href="https://github.com/graphprotocol/rfcs/pull/25">RFC</a></dd>

  <dt>Date of submission</dt>
  <dd>2022-05-04</dd>

  <dt>Date of approval</dt>
  <dd>TBD</dd>

  <dt>Approved by</dt>
  <dd>TBD0, TBD1</dd>
</dl>

## Contents

<!-- toc -->

## Summary

Currently the Graph Node release process isn't well defined, and it has a few
flaws that consume engineering time and make the commit history confusing and
non-intuitive. This RFC aims to propose a solution to the problem.

## Goals & Motivation

### Current Challenges

- The process for cutting the release isn’t well-defined;
- We don’t track all the changes, and it’s time-consuming to discover/document
  all these for the release;
- There’s no upper bound on waiting for feedback from Indexers on a release;
- It’s hard to know if patches on point releases are on master or not;
- Git branching/handling is not consistent.

### How it works nowadays

Once we decide on a `master` commit for a new release candidate, we create a
`release` PR from it pointing to the old `release` branch. Then after
this branch is validated on the **integration cluster**, the PR gets closed.

After the release is out, we have to re-do the `Cargo.toml` and `NEWS.md`
in a new PR against `master`.

## Urgency

"Medium". We can keep the process the way it is, but it will not scale over
time. We should implement a different process in the near to medium term.

## Terminology

Branches:

- `master`: Same as today;
- `release/0.27`: Long living branches created from `master`. They receive more
  PRs if patches are needed for this `minor`;
- `feature`: Same as today (eg: `person-name/feature-name`).

## Detailed Design

This RFC proposes that we keep using `master` and `feature` branches as they
are today.

The difference arises when we try to create a new release. We should define a
point in `master`'s history that will be the new release candidate just like
nowadays. However, the new branch will be a long-living one referring to a
`minor` Graph Node version.

We'll first test the chosen commit on the **integration cluster**, and after
that's validated, the branch can be created, and before the tag is too, the
`NEWS.md` and `Cargo.toml` files should be updated.

There's [this script](https://github.com/graphprotocol/graph-node/blob/42c017e8f02f1859ffdadcae4e3a6fe773811881/scripts/update-cargo-version.sh) to help changing the latter, and these commits should
be included in `master` too.

After that, the git can be created and shared with indexers in the
[#indexer-announcements](https://discord.com/channels/438038660412342282/791422773348007936) Discord channel.

## Compatibility

There are no breaking change and compatibility concerns.

## Drawbacks and Risks

This will "introduce complexity", but since currently, the process isn't clear,
in my opinion introducing a clear process will actually simplify things.

## Alternatives

We could use `git flow`, but we don't need all of the complexity and branches
it adds.

## Open Questions

What are unresolved questions?
