---
id: GOV-COMMITS
title: Commit conventions
status: accepted
lang: en
pair: none
last-reviewed: 2026-10-14
---

# Commit conventions

> **Question this document answers:** how is a commit message written?

A commit message is **one short, explicit line** that follows [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) and ends with its task:

```text
<type>(<scope>): <imperative summary> (T-NN)
```

Task pull requests are squash-merged, so **the pull request title is the commit** on `dev`, or on `main` in `orderflow-engine-docs`. The reason for the change lives in the pull request, not in the commit.

| Rule | Detail |
|------|--------|
| One line | No body and no footer |
| Length | At most 60 characters, including the task tag |
| Summary | Imperative mood ("add", not "added"), lowercase, no final period |
| Task tag | ` (T-NN)` at the end: the task in [TASKS](../spec/TASKS.md) that the commit closes |
| Breaking change | `!` after the type or scope, e.g. `feat(api)!: rename quote fields (T-10)` |
| Language | English (GC-7) |

| Type | Use |
|------|-----|
| `feat` | New behavior |
| `fix` | A defect corrected through the normal flow |
| `hotfix` | An urgent defect correction. It still goes through `dev` before `main` ([golden rule](golden-rule.md)). It is a project extension of the standard types, so a commit linter must be configured to accept it |
| `docs` | Documentation only |
| `test` | Tests only |
| `refactor` | Code change with no behavior change |
| `perf` | Performance improvement |
| `build` | Build system or dependencies |
| `chore` | Maintenance with no production effect |
| `release` | The release merge of the finished product into `main` |

Scopes are the module or area: `pricing`, `quoting`, `ordering`, `outbox`, `api`, `db`, `web`, `infra`, `spec`, `governance`, `tools`.

Two lines carry no task tag:

| Commit | When |
|--------|------|
| `chore: initialize repository` | The bootstrap commit of a new repository ([branches-and-environments](branches-and-environments.md)) |
| `release: vX.Y.Z` | The merge commit of a release pull request, only for the finished product |

A phase-closing merge into `main` keeps its task tag, for example `chore: close preparation phase (T-00)` ([pull-request-rules](pull-request-rules.md)).

Examples, taken from [TASKS](../spec/TASKS.md):

| Task | Commit |
|------|--------|
| T-02 | `feat(pricing): add money value object (T-02)` |
| T-07 | `test(pricing): cover rounding and countries (T-07)` |
| T-11 | `feat(ordering): confirm orders atomically (T-11)` |

The documentation check fails when a commit line in TASKS breaks any of these rules.
