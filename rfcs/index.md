# RFCs

## What is an RFC?

An RFC describes a change to Graph Protocol, for example a new feature. Any
substantial change goes through the RFC process, where the change is described
in an RFC, is proposed a pull request to the `rfcs` repository, is reviewed,
currently by the core team, and ultimately is either either approved or
rejected.

## RFC process

### 1. Create a new RFC

RFCs are numbered, starting at `0001`. To create a new RFC, create a new branch
of the `rfcs` repository. Check the existing RFCs to identify the next number to
use. Then, copy the [RFC template](./rfcs/0000-template.md) to a new file in the
`rfcs/` directory. For example:

```sh
cp templates/rfc.md src/rfcs/015-fulltext-search.md
```

Write the RFC, commit it to the branch and open a [pull
request](https://github.com/graphprotocol/rfcs/pulls) in the `rfcs` repository.

### 2. RFC review

After an RFC has been submitted through a pull request, it is being reviewed. At
the time of writing, every RFC needs to be approved by

- at least one Graph Protocol founder, and
- at least one member of the core development team.

### 3. RFC approval

Once an RFC is approved, the RFC meta data (see the
[template](./rfcs/0000-template.md)) is updated and the pull request is merged
by the original author or a Graph Protocol team member.

## Approved RFCs

- No RFCs have been approved yet.

## Obsolete RFCs

Obsolete RFCs are moved to the `rfcs/obsolete` directory in the `rfcs`
repository. They are listed below for reference.

- No RFCs have been obsoleted yet.

## Rejected RFCs

Rejected RFCs can be found by filtering open and closed pull requests by those
that are labeled with `rejected`. This list can be [found
here](https://github.com/graphprotocol/rfcs/issues?q=label:rfc+label:rejected).
