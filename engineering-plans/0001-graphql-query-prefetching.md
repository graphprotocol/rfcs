# PLAN-0001: GraphQL Query Prefetching

<dl>
  <dt>Author</dt>
  <dd>David Lutterkort</dd>

<dt>Implements</dt>
<dd>No RFC - no user visible changes</dd>

<dt>Engineering Plan pull request</dt>
<dd><a href="<https://github.com/graphprotocol/rfcs/pull/2>">https://github.com/graphprotocol/rfcs/pull/2</a></dd>

<dt>Date of submission</dt>
<dd>2019-11-27</dd>

<dt>Date of approval</dt>
<dd>2019-12-10</dd>

  <dt>Approved by</dt>
  <dd>Jannis Pohlmann, Leo Yvens</dd>
</dl>

This is not really a plan as it was written and discussed before we adopted
the RFC process, but contains important implementation detail of how we
process GraphQL queries.

## Contents

<!-- toc -->


## Implementation Details for prefetch queries


### Goal

For a GraphQL query of the form

```graphql
query {
  parents(filter) {
    id
    children(filter) {
      id
    }
  }
}
```

we want to generate only two SQL queries: one to get the parents, and one
to get the children for all those parents. The fact that `children` is
nested under `parents` requires that we add a filter to the `children`
query that restricts children to those that are related to the parents we
fetched in the first query to get the parents. How exactly we filter the
`children` query depends on how the relationship between parents and
children is modeled in the GraphQL schema, and on whether one (or both) of
the types involved are interfaces.

The rest of this writeup is concerned with how to generate the query for
`children`, assuming we already retrieved the list of all parents.

The bulk of the implementation of this feature can be found in
`graphql/src/store/prefetch.rs`, `store/postgres/src/jsonb_queries.rs`, and
`store/postgres/src/relational_queries.rs`


### Handling first/skip

We never get all the `children` for a parent; instead we always have a
`first` and `skip` argument in the children filter. Those arguments need to
be applied to each parent individually by ranking the children for each
parent according to the order defined by the `children` query. If the same
child matches multiple parents, we need to make sure that it is considered
separately for each parent as it might appear at different ranks for
different parents. In SQL, we use the `rank()` window function for this:

```sql
select *
  from (
    select c.*,
           rank() over (partition by parent_id order by ...) as pos
      from (query to get children) c)
 where pos >= skip and pos < skip + first
```

### Handling interfaces

If `parents` or `children` (or both) are interfaces, we resolve the
interfaces into the concrete types implementing them, produce a query for
each combination of parent/child concrete type and combine those queries
via `union all`.

Since implementations of the same interface will generally differ in the schema
they use, we can not form a `union all` of all the data in the tables for
these concrete types, but have to first query only attributes that we know
will be common to all entities implementing the interface, most notably the
`vid` (a unique identifier that identifies the precise version of an
entity), and then later fill in the details of each entity by converting it
directly to JSON.

That means that when we deal with children that are an interface, we will
first select only the following columns (where exactly they come from
depends on how the parent/child relationship is modeled)

```sql
select '{__typename}' as entity, c.vid, c.id, parent_id
```

and form the `union all` of these queries. We then use that union to rank
children as described above.


### Handling parent/child relationships

How we get the children for a set of parents depends on how the
relationship between the two is modeled. The interesting parameters there
are whether parents store a list or a single child, and whether that field
is derived, together with the same for children.

There are a total of 16 combinations of these four boolean variables; four
of them, when both parent and child derive their fields, are not
permissible. It also doesn't matter whether the child derives its parent
field: when the parent field is not derived, we need to use that since that
is the only place that contains the parent -> child relationship. When the
parent field is derived, the child field can not be a derived field.

That leaves us with the following combinations of whether the parent and
child store a list or a scalar value, and whether the parent is derived:


For details on the GraphQL schema for each row in this table, see the
section at the end. The `Join cond` indicates how we can find the children
for a given parent. There are four different join conditions in this table.

When we query children, we need to have the id of the parent that child is
related to (and list the child multiple times if it is related to multiple
parents) since that is the field by which we window and rank children.

For join conditions of type C and D, the id of the parent is not stored in
the child, which means we need to join with the `parents` table.

Let's work out the details of these queries; the implementation uses
`struct EntityLink` in `graph/src/components/store.rs` to distinguish
between the different types of joins and queries.

| Case | Parent list? | Parent derived? | Child list? | Join cond                  | Type |
|------|--------------|-----------------|-------------|----------------------------|------|
|    1 | TRUE         | TRUE            | TRUE        | child.parents ∋ parent.id  | A    |
|    2 | FALSE        | TRUE            | TRUE        | child.parents ∋ parent.id  | A    |
|    3 | TRUE         | TRUE            | FALSE       | child.parent = parent.id   | B    |
|    4 | FALSE        | TRUE            | FALSE       | child.parent = parent.id   | B    |
|    5 | TRUE         | FALSE           | TRUE        | child.id ∈ parent.children | C    |
|    6 | TRUE         | FALSE           | FALSE       | child.id ∈ parent.children | C    |
|    7 | FALSE        | FALSE           | TRUE        | child.id = parent.child    | D    |
|    8 | FALSE        | FALSE           | FALSE       | child.id = parent.child    | D    |

#### Type A

Use when parent is derived and child is a list

```sql
select c.*, parent_id
 from {children} c join lateral unnest(c.{parent_field}) parent_id
where parent_id = any($parent_ids)
```

Data needed to generate:

-   children: name of child table
-   parent_ids: list of parent ids
-   parent_field: name of parents field (array) in child table

The implementation uses a `EntityLink::Direct` for joins of this type.


#### Type B

Use when parent is derived and child is not a list

```sql
select c.*, c.{parent_field} as parent_id
 from {children} c
where c.{parent_field} = any($parent_ids)
```

Data needed to generate:

-   children: name of child table
-   parent_ids: list of parent ids
-   parent_field: name of parent field (scalar) in child table

The implementation uses a `EntityLink::Direct` for joins of this type.


#### Type C

Use when parent is a list and not derived

```sql
select c.*, p.id as parent_id
 from {children} c, {parents} p
where p.id = any($parent_ids)
  and c.id = any(p.{child_field})
```

Data needed to generate:

-   children: name of child table
-   parent_ids: list of parent ids
-   parents: name of parent table
-   child_field: name of child field (array) in parent table

The implementation uses a `EntityLink::Parent` for joins of this type.


#### Type D

Use when parent is not a list and not derived

```sql
select c.*, p.id as parent_id
 from {children} c, {parents} p
where p.id = any($parent_ids)
  and c.id = p.child_field
```

Data needed to generate:

-   children: name of child table
-   parent_ids: list of parent ids
-   parents: name of parent table
-   child_field: name of child field (scalar) in parent table

The implementation uses a `EntityLink::Parent` for joins of this type.


### Putting it all together

Note that in all of these queries, we ultimately return the typename of
each entity, together with a JSONB representation of that entity. We do
this for two reasons: first, because different child tables might have
different schemas, which precludes us from taking the union of these child
tables, and second, because Diesel does not let us execute queries where
the type and number of columns in the result is determined dynamically.

We need to to be careful though to not convert to JSONB too early, as that
is slow when done for large numbers of rows. Deferring conversion is
responsible for some of the complexity in these queries.

In the following, we only go through the queries for relational storage;
for JSONB storage, there are similar considerations, though they are
somewhat simpler as the `union all` in the below queries turns into
an `entity = any(..)` clause with JSONB storage, and because we do not need
to convert to JSONB data.

Note that for the windowed queries below, the entity we return will have
`parent_id` and `pos` attributes. The `parent_id` is necessary to attach
the query result to the right parents we already have in memory. The JSONB
queries need to somehow insert the `parent_id` field into the JSONB data
they return.

In the most general case, we have an `EntityCollection::Window` with
multiple windows. The query for that case is

```sql
with matches as (
  -- Limit the matches for each parent
  select c.*
    from (
      -- Rank matching children for each parent
      select c.*,
             rank() over (partition by c.parent_id order by {query.order}) as pos
        from (
          {window.children_uniform(sort_key, block)}
          union all
            ... range ober all windows) c) c
   where c.pos > {skip} and c.pos <= {skip} + {first})
-- Get the full entity for each match
select m.entity, to_jsonb(c.*) as data, m.parent_id, m.pos
  from matches m, {window.child_table()} c
 where c.vid = m.vid and m.entity = '{window.child_type}'
 union all
       ... range over all windows
 -- Make sure we return the children for each parent in the correct order
 order by parent_id, pos
```

When there is only one window, we can simplify the above query. The
simplification basically inlines the `matches` CTE. That is important as
CTE's in Postgres before Postgres 12 are optimization fences, even when
they are only used once. We therefore reduce the two queries that Postgres
executes above to one for the fairly common case that the children are not
an interface.

```sql
select '{window.child_type}' as entity, to_jsonb(c.*) as data
  from (
    -- Rank matching children
    select c.*,
          rank() over (partition by c.parent_id order by {query.order}) as pos
     from ({window.children_detailed()}) c) c
 where c.pos >= {window.skip} and c.pos <= {window.skip} + {window.first}
 order by c.parent_id,c.pos
```
When we do not have to window, but only deal with an
`EntityCollection::All` with multiple entity types, we can simplify the
query by avoiding ranking and just using an ordinary `order by` clause:

```sql
with matches as (
  -- Get uniform info for all matching children
  select '{entity_type}' as entity, id, vid, {sort_key}
    from {entity_table} c
   where {query_filter}
   union all
     ... range over all entity types
   order by {sort_key} offset {query.skip} limit {query.first})
-- Get the full entity for each match
select m.entity, to_jsonb(c.*) as data, c.id, c.{sort_key}
  from matches m, {entity_table} c
 where c.vid = m.vid and m.entity = '{entity_type}'
 union all
       ... range over all entity types
 -- Make sure we return the children for each parent in the correct order
     order by c.{sort_key}, c.id
```

And finally, for the very common case of a GraphQL query without nested
children that uses a concrete type, not an interface, we can further
simplify this, again by essentially inlining the `matches` CTE to:

```sql
select '{entity_type}' as entity, to_jsonb(c.*) as data
  from {entity_table} c
 where query.filter()
 order by {query.order} offset {query.skip} limit {query.first}
```

## Boring list of possible GraphQL models

These are the eight ways in which a parent/child relationship can be
modeled. For brevity, I left the `id` attribute on each parent and child
type out.

This list assumes that parent and child types are concrete types, i.e.,
that any interfaces involved in this query have already been reolved into
their implementations and we are dealing with one pair of concrete
parent/child types.

```graphql
# Case 1
type Parent {
  children: [Child] @derived
}

type Child {
  parents: [Parent]
}

# Case 2
type Parent {
  child: Child @derived
}

type Child {
  parents: [Parent]
}

# Case 3
type Parent {
  children: [Child] @derived
}

type Child {
  parent: Parent
}

# Case 4
type Parent {
  child: Child @derived
}

type Child {
  parent: Parent
}

# Case 5
type Parent {
  children: [Child]
}

type Child {
  # doesn't matter
}

# Case 6
type Parent {
  children: [Child]
}

type Child {
  # doesn't matter
}

# Case 7
type Parent {
  child: Child
}

type Child {
  # doesn't matter
}

# Case 8
type Parent {
  child: Child
}

type Child {
  # doesn't matter
}
```
