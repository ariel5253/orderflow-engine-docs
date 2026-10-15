---
id: GOV-GOLDEN-RULE
title: Golden rule
status: accepted
lang: en
pair: none
last-reviewed: 2026-10-14
---

# Golden rule

> **Nothing reaches `main` except through a pull request, and nobody pushes directly to `main` or `dev`. In a code repository, `main` receives only `dev`, when the owner closes a phase or releases the finished product, and `dev` receives only task branches whose verification is green. In `orderflow-engine-docs`, which holds living documentation, task branches go straight to `main`.**

The only exception is the bootstrap commit of a new repository, which is pushed before the branches and the protections exist ([branches-and-environments.md](branches-and-environments.md)).

Every other governance rule serves this one:

| Rule | Where |
|------|-------|
| Which branches exist, and how they are protected | [branches-and-environments.md](branches-and-environments.md) |
| How a pull request is opened, verified, merged and cleaned up | [pull-request-rules.md](pull-request-rules.md) |
| How commit messages are written | [commit-conventions.md](commit-conventions.md) |
| How documents are written, placed and checked | [documentation-rules.md](documentation-rules.md) |
| What every repository must contain, ignore and never contain | [repository-integrity.md](repository-integrity.md) |
