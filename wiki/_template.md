---
title: Mechanism name
status: planned        # planned | in progress | implemented | deprecated
code: []               # files that implement this, e.g. [src/shared/Fish/Schools.luau]
decisions: []          # entries in decisions.md, e.g. [D6]
related: []            # other wiki pages, e.g. [networking]
last_verified: n/a     # YYYY-MM-DD and commit when someone last checked this page against the code
---

# Mechanism name

## Summary

Two to four sentences: what this does and why it exists.

## Spec

Exactly how it works: data layouts to the byte, formulas, constants (each with a unit and a tag), the order of steps, edge cases, and which side is authoritative. Someone should be able to reimplement it from this page alone. The tags are explained in wiki/README.md, under "How to write a page".

## Invariants

Statements that must always hold. Each one should have a test.

## Tests

Which specs cover this mechanism and how to run them. "None yet" is a valid answer.

## Open questions

Undecided details, unverified assumptions, and prototypes still to run.

## History

- YYYY-MM-DD — what changed — why
