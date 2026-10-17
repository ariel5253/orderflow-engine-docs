---
id: GOV-BRANCHES
title: Branches and environments
status: accepted
lang: en
pair: none
last-reviewed: 2026-10-14
---

# Branches and environments

> **Question this document answers:** which branches exist, what each one means, and how they are protected?

This is a technical test, so there is no QA or staging environment. `main` must never be contaminated with unfinished work.

## Branches by repository kind

| Repository kind | Repositories | Long-lived branches |
|-----------------|--------------|---------------------|
| Code | `orderflow-engine-api`, `orderflow-engine-web`, `orderflow-engine-infra`, `orderflow-engine-tools`, `orderflow-engine-app` | `main` and `dev` |
| Living documentation | `orderflow-engine-docs` | `main` only. The specification changes as the work advances, so each change lands on `main` as soon as its pull request is verified |

| Branch | Meaning | Receives | Lifetime |
|--------|---------|----------|----------|
| `main` | Code: the last closed phase, then the delivered product. Docs: the current documentation. It is the default branch the evaluator sees | Code: only phase-closing and release pull requests from `dev`. Docs: task pull requests | Permanent |
| `dev` (code only) | Integration of finished tasks | Only task pull requests | Permanent |
| `<type>/T-NN-slug` | One task of [TASKS](../spec/TASKS.md), or one part of it, e.g. `feat/T-04-tier-discounts` | Commits of that task | Deleted when its pull request is merged |

## Bootstrap (the only exception to the golden rule)

A new repository has no branches and no protections, so its first commit must be pushed directly to `main`. That commit is kept minimal, so `main` never holds unfinished work:

1. Push `chore: initialize repository` to `main`. It holds only `LICENSE`, `.gitignore` and `.gitattributes`, so the ignore and line-ending rules ([repository-integrity.md](repository-integrity.md)) apply from the first commit.
2. In a code repository, create `dev` from that commit.
3. Apply the protections below.
4. All real work, starting with T-00, enters through pull requests: into `dev` in a code repository, into `main` in `orderflow-engine-docs`.

## Protections

Applied as **repository rulesets** right after the bootstrap. Rulesets support a different merge method per branch, which classic branch protection does not. GitHub enforces rulesets on private repositories only on paid plans. While a repository is private on a free plan, the same rules are followed by hand and the rulesets are applied when it becomes public. A content repository with only `main` (`orderflow-engine-docs`, `orderflow-engine-page`) has the docs `main` column.

| Rule | Code `main` | Code `dev` | Docs `main` |
|------|-------------|------------|-------------|
| Pull request required (no direct push) | Yes | Yes | Yes |
| Required approvals | 0. The owner is the only contributor, and GitHub does not let an author approve their own pull request | 0 | 0 |
| Allowed merge method | Merge commit | Squash | Squash |
| Block force pushes | Yes | Yes | Yes |
| Restrict deletions | Yes | Yes | Yes |

The repository setting **"Automatically delete head branches"** is enabled once the deletion rulesets are active, because they keep `dev` from being deleted when a phase-closing pull request merges. Until then, each task branch is deleted by its own merge command ([pull-request-rules.md](pull-request-rules.md)).

## Phase closing

The owner closes a phase when its tasks are done, for example the preparation phase (T-00). In each code repository, a pull request from `dev` into `main` is merged with a merge commit titled `chore: close <phase> phase (T-NN)`. It carries no version and no tag. `dev` stays.

## Verification without CI

The assignment does not ask for CI, so no pipeline is built. Verification runs locally, and its output is pasted into the pull request ([pull-request-rules.md](pull-request-rules.md)). Adding GitHub Actions later is an adoption path ([plan.md](../spec/plan.md) §3).

## Delivery

Versions exist only for the finished product, and a release is a pull request `release: vX.Y.Z` from `dev` into `main` in each code repository, merged **before** the repository link is sent. This delivery was sent without one, on the owner's decision: every phase was already closed into `main` with its merge commit, so the default branch shows the full work and every per-task commit.

Repositories were private while the work was in progress. They became public at delivery, on 2026-10-17, and the rulesets were applied then.
