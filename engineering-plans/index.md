# Engineering Plans

## What is an Engineering Plan?

Engineering Plans are plans to turn an [RFC](./rfcs.md) into an implementation
in the core Graph Protocol tools like Graph Node, Graph CLI and Graph TS. Every
substantial development effort that follows an RFC is planned in the form of an
Engineering Plan.

## Engineering Plan process

### 1. Create a new Engineering Plan

Like RFCs, Engineering Plans are numbered, starting at `0001`. To create a new
plan, create a new branch of the `rfcs` repository. Check the existing plans to
identify the next number to use. Then, copy the [Engineering Plan
template](./engineering-plans/0000-template.md) to a new file in the
`engineering-plans/` directory. For example:

```sh
cp templates/engineering-plan.md src/engineering-plans/015-fulltext-search.md
```

Write the Engineering Plan, commit it to the branch and open a [pull
request](https://github.com/graphprotocol/rfcs/pulls) in the `rfcs` repository.

### 2. Engineering Plan review

After an Engineering Plan has been submitted through a pull request, it is being
reviewed. At the time of writing, every Engineering Plan needs to be approved by

- the Tech Lead, and
- at least one member of the core development team.

## Approved Engineering Plans

- No engineering plans have been approved yet.

## Obsolete Engineering Plans

Obsolete Engineering Plans are moved to the `engineering-plans/obsolete`
directory in the `rfcs` repository. They are listed below for reference.

- No Engineering Plans have been obsoleted yet.

## Rejected Engineering Plans

Rejected Engineering Plans can be found by filtering open and closed pull
requests by those that are labeled with `rejected`. This list can be [found
here](https://github.com/graphprotocol/rfcs/issues?q=label:engineering-plan+label:rejected).
