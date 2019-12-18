# PLAN-0001: Remove JSONB Storage

<dl>
  <dt>Author</dt>
  <dd>David Lutterkort</dd>

  <dt>Implements</dt>
  <dd>No RFC - no user visible changes</dd>

  <dt>Engineering Plan pull request</dt>
  <dd><a href="https://github.com/graphprotocol/rfcs/pull/7">https://github.com/graphprotocol/rfcs/pull/7</a></dd>

  <dt>Date of submission</dt>
  <dd>YYYY-MM-DD</dd>

  <dt>Date of approval</dt>
  <dd>YYYY-MM-DD</dd>

  <dt>Approval needed</dt>
  <dd>Yaniv/Jess/Eva on general approach and help with notifying users</dd>
  <dd>Core devs on migration strategy</dd>

  <dt>Approved by</dt>
  <dd>First Person, Second Person</dd>
</dl>

## Summary

Remove JSONB storage from `graph-node`. That means that we want to remove
the old storage scheme, and only use relational storage going
forward. At a high level, removal has to touch the following areas:

* user subgraphs in the hosted service
* user subgraphs in self-hosted `graph-node` instances
* subgraph metadata in `subgraphs.entities` (see [this issue](https://github.com/graphprotocol/graph-node/issues/1394))
* the `graph-node` code base

Because it touches so many areas and different things, JSONB storage
removal will need to happen in several steps, the last being actual removal
of JSONB code. The first three steps above are independent of each other
and can be done in parallel.

## Implementation

### User Subgraphs in the Hosted Service

We will need to communicate to users that they need to update their
subgraphs if they still use JSONB storage. Currently, there are ~ 580
subgraphs
([list](https://gist.github.com/lutter/2e7a7716b70b4144fe0b6a5f1c9066bc))
belonging to 220 different organizations using JSONB storage. It is quite
likely that the vast majority of them is not needed anymore and simply left
over from somebody trying something out.

We should contact users and tell them that we will delete their subgraph
after a certain date (say 2020-02-01) _unless_ they deploy a new version of
the subgraph (with an explanation why etc. of course) Redeploying their
subgraph is all that is needed for those updates.

### Self-hosted User Subgraphs

We will need to tell users that the 'old' JSONB storage is deprecated and
support for it will be removed as of some target date, and that they need
to redeploy their subgraph.

Users will need some documentation/tooling to help them understand
* which of their deployed subgraphs still use JSONB storage
* how to remove old subgraphs
* how to remove old deployments

### Subgraph Metadata in `subgraphs.entities`

We can treat the `subgraphs` schema like a normal subgraph, with the
exception that some entities must not be versioned. For that, we will need
to adopt code that makes it possible to write entities to the store without
recording their version (or, more generally, so that there will only be one
version of the entity, tagged with a block range `[0,)`)

We will manually create the DDL for the `subgraphs.graphql` schema and run
that as part of a database migration. In that migration, we will also copy
the existing metadata from `subgraphs.entities` and
`subgraphs.entity_history` into their new tables.

### The Code Base

Delete all code handling JSONB storage. This will mostly affect
`entities.rs` and `jsonb_queries.rs` in `graph-store-postgres`, but there
are also smaller things like that we do not need the annotations on
`Entity` to serialize them to the JSON format that JSONB uses.

## Tests

Most of the code-level changes are covered by the existing test suite. The
major exception is that the migration of subgraph metadata needs to be
tested and checked manually, using a recent dump of the production
database.

## Migration

See above on migrating data in the `subgraphs` schema.

## Documentation

No user-facing documentation is needed.

## Implementation Plan

_No estimates yet as we should first agree on this general course of
action_

* Notify hosted users to update their subgraph or have it deleted by date X
* Mark JSONB storage as deprecated and announce when it will be removed
* Provide tool to ship with `graph-node` to delete unused deployments and
  unneeded subgraphs
* Add affordance to not version entities to relational storage code
* Write SQL migrations to create new subgraph metadata schema and copy
  existing data
* Delete old JSONB code
* On start of `graph-node`, add check for any deployments that still use
  JSONB storage and log warning messages telling users to redeploy (once
  the JSONB code has been deleted, this data can not be accessed any more)

## Open Questions

None
