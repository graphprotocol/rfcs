# Engineering Plans

## What is an Engineering Plan?

Engineering Plans are plans to turn an [RFC](../rfcs/index.md) into an
implementation in the core Graph Protocol tools like Graph Node, Graph CLI and
Graph TS. Every substantial development effort that follows an RFC is planned in
the form of an Engineering Plan.

## Engineering Plan process

### 1. Create a new Engineering Plan

Like RFCs, Engineering Plans are numbered, starting at `0001`. To create a new
plan, create a new branch of the `rfcs` repository. Check the existing plans to
identify the next number to use. Then, copy the [Engineering Plan
template](https://github.com/graphprotocol/rfcs/blob/master/engineering-plans/0000-template.md)
to a new file in the `engineering-plans/` directory. For example:

```sh
cp engineering-plans/0000-template.md engineering-plans/0015-fulltext-search.md
```

Write the Engineering Plan, commit it to the branch and open a [pull
request](https://github.com/graphprotocol/rfcs/pulls) in the `rfcs` repository.

In addition to the Engineering Plan itself, the pull request must include the
following changes:

- a link to the Engineering Plan on the [Approved Engineering Plans](./approved.md) page, and
- a link to the Engineering Plan under `Approved Engineering Plans` in `SUMMARY.md`.

### 2. Engineering Plan review

After an Engineering Plan has been submitted through a pull request, it is being
reviewed. At the time of writing, every Engineering Plan needs to be approved by

- the Tech Lead, and
- at least one member of the core development team.

### 3. Engineering Plan approval

Once an Engineering Plan is approved, the Engineering Plan meta data (see the
[template](https://github.com/graphprotocol/rfcs/blob/master/engineering-plans/0000-template.md))
is updated and the pull request is merged by the original author or a Graph
Protocol team member.
