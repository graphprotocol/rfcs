# PLAN-0000: Template

<dl>
  <dt>Author</dt>
  <dd>YOUR FULL NAME</dd>

  <dt>Implements</dt>
  <dd><a href="../rfcs/0000-template.md">RFC-0000 Template</a></dd>

  <dt>Engineering Plan pull request</dt>
  <dd><a href="URL">URL</a></dd>

  <dt>Obsoletes (if applicable)</dt>
  <dd><a href="./obsolete/0000-template.md">PLAN-0000 Template</a></dd>

  <dt>Date of submission</dt>
  <dd>YYYY-MM-DD</dd>

  <dt>Date of approval</dt>
  <dd>YYYY-MM-DD</dd>

  <dt>Approved by</dt>
  <dd>First Person, Second Person</dd>
</dl>

## Contents

<!-- toc -->

## Summary

A description of what the Engineering Plan does in 1-2 paragraphs.

## Implementation

How is the change or the feature going to be implemented? Which parts need to be
changed and how?

## Tests

How is the change or the feature going to be tested? Outline what test cases
will be implemented and, roughly, how (e.g. as unit tests, integration tests
etc.). Describe what the corner cases are and how they are accounted for in the
tests.

## Migration

If the change is breaking, even if in parts, are (semi-)automated migrations
possible? If not, how are we going to move users forward (e.g. announce early to
prepare users in time for the change to be activated)? If migrations are
possible, how are they going to work?

## Documentation

How are the changes going to be documented?

## Implementation Plan

An iterative plan for implementing the change or new feature.

Think hard to not skip over potential challenges. Break the implementation down
into a step-by-step plan where the scope of every step is clear and where there
is no, or little, uncertainty about each individual step and the sequence
overall.

**Estimates:** Every step/task must come with an estimate of 1, 2, 3, 4 or 5
days.

**Phases:** For big changes, it can make sense to break down the overall plan
into implementation phases.

**Integration:** Explicit integration points must be included in the plan. What
are we going to submit for review and integration at what point? What can be
integrated early?

In order for a plan to be approved, there must be _extremely high_ confidence
that the plan will work out.

## Open Questions

What are unresolved questions? Which areas may we have forgotten about or
neglected in the plan?
