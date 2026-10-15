---
id: HANDOFF
title: Orderflow Engine — Handoff
status: accepted
lang: en
pair: none
last-reviewed: 2026-10-14
---

# Orderflow Engine — Handoff

> **Question this document answers:** where does the work stand right now, and how do I pick it up?

**Opening a new session? Start here.** This is the only place where the verified state lives (§3). Other documents link here instead of copying numbers, because a number copied into two places goes stale in one of them.

## 1. What Orderflow Engine is

It is a pricing engine and an idempotent order confirmation for a multi-country B2B beverage marketplace, built as a technical take-home. The assignment is in [assignment/](../assignment/), and the rules are in [SPEC.md](../spec/SPEC.md).

## 2. Workspace

| Folder | State |
|--------|-------|
| `orderflow-engine-docs` | Specification, plan, contract, data model, architecture, 9 diagrams, governance, ADR in Word and PDF, this handoff |
| `orderflow-engine-api` | **The deliverable.** README (with the four required topics) and ADR-0001 (proposed). Code is scaffolded in T-01 |
| `orderflow-engine-web` | README only. Built in T-17, the optional screen |
| `orderflow-engine-infra` | README only. Full-stack compose and `test-all` in T-01 |
| `orderflow-engine-tools` | Generators and checks: figures, progress chart, ADR in Word and PDF, documentation, integrity and consistency checks. Own Docker image |
| `orderflow-engine-app` | Reserved. Not built |

Each folder is a private GitHub repository of the account `ariel5253`, initialized on 2026-10-14. The **preparation phase (T-00) is closed**: each code repository received it in `dev` through thematic pull requests and then in `main` through `chore: close preparation phase (T-00)`. This repository has only `main` and received every change through a pull request ([branches-and-environments](../governance/branches-and-environments.md)). The pull requests are listed in §8.

Phases close on the owner's order, each with a merge commit on the `main` of the code repositories it changed:

| Phase | Tasks | Closed on | Merge commit |
|-------|-------|-----------|--------------|
| 1. Preparation | T-00 | 2026-10-14 | `chore: close preparation phase (T-00)` in every code repository |
| 2. Scaffold | T-01 | 2026-10-15 | `chore: close scaffold phase (T-01)` in `orderflow-engine-api` and `orderflow-engine-infra` (the other code repositories did not change) |

## 3. State (verified)

| Item | Value | Verified by |
|------|-------|-------------|
| Current task | T-01 done and the scaffold phase closed on 2026-10-15. Next: T-02 | — |
| Specification | **Pre-release.** SPEC and TASKS are still being improved before the first version. Formal change control starts only after the first version is accepted | Measured review: 21 of 21 assignment bullets specified, the readable history now included; 5 ACs beyond the assignment, each justified in SPEC §14.3 |
| Tasks done | T-00 (21 task and 5 phase-closing pull requests, §8) and T-01 (9 task and 2 phase-closing pull requests, §9) | `gh pr list --state merged` on each repository on 2026-10-15 |
| Branch protections | Rulesets are rejected by GitHub on private repositories of a free plan (HTTP 403). The rules are followed by hand and the rulesets are applied when the repositories become public | `gh api repos/ariel5253/<repository>/rulesets` on 2026-10-14 |
| Documentation check | OK, including the rule that no relative link leaves its repository | `node orderflow-engine-tools/docs-check/check-docs.mjs` on 2026-10-14 |
| Repository integrity | OK in all six repositories: ignore files, `.gitattributes`, Docker files, `.env.example`, no secrets | `node orderflow-engine-tools/integrity/check-integrity.mjs` on 2026-10-14 |
| Tools image | Builds. Inside the container, `check-docs` and `check-integrity` pass, and the figures and progress chart are byte-identical to the host's | `docker build` and `docker run` of `orderflow-engine-tools` on 2026-10-14 |
| Standalone repositories | Each folder copied alone has 0 broken links | Copy of each folder outside the workspace, links resolved, on 2026-10-14 |
| ADR-0001 length | One A4 page at Arial 12 (2 cm margins), in English and in Spanish, as Word and PDF | `python orderflow-engine-tools/render/build_deliverables.py` fails above one page; page count = 1 on 2026-10-14 |
| Deliverable metadata | Author = license holder, title = document title, no generator or application name | docx core properties and PDF info dictionary read on 2026-10-14 |
| API tests | 3 green: `DomainPurityArchTest` (2 rules, each proven to fail with a probe class), `HealthIT` (Testcontainers PostgreSQL). JaCoCo gate on `pricing` wired and proven with an uncovered probe class | `bash orderflow-engine-infra/scripts/test-all.sh` → `test-all: OK` on 2026-10-15 |
| Web tests | not started | — |
| PIT mutation score | not run (runs in T-16) | — |
| Time spent | recorded at T-16 | — |

Update this table at the end of every task, using the output of the command in the last column.

## 4. Task progress

Project start: 2026-10-13. Cut line after: T-16. Update this table at the end of every task (GC-8), then regenerate the chart with `node orderflow-engine-tools/progress/progress.mjs`. `check-docs` fails if the chart does not match the table.

![Task progress](progress.svg)

| Task | Title | Planned | Status | Done on |
|------|-------|---------|--------|---------|
| T-00 | Specification, governance, README, ADR draft | D1 | done | 2026-10-14 |
| T-01 | Scaffold API and workspace test command | D1 | done | 2026-10-15 |
| T-02 | Money and currency policy | D1 | todo | |
| T-03 | Pricing pipeline and golden harness | D1 | todo | |
| T-04 | Tier discounts | D2 | todo | |
| T-05 | Combos and allocation | D2 | todo | |
| T-06 | Gifts and gift tax policy | D2 | todo | |
| T-07 | Rounding and multi-country | D2 | todo | |
| T-08 | Schema and seed data | D3 | todo | |
| T-09 | Error handling and API i18n | D3 | todo | |
| T-10 | Quote use case and endpoints | D3 | todo | |
| T-11 | Order confirmation | D4 | todo | |
| T-12 | Invariants under concurrency and failure | D4 | todo | |
| T-13 | Order history | D4 | todo | |
| T-14 | Outbox relay | D4 | todo | |
| T-15 | Retention housekeeping | D5 | todo | |
| T-16 | Final documentation and verification | D5 | todo | |
| T-17 | Order-summary screen (optional) | D5 | todo | |

## 5. Commands to resume

```bash
# Clone the workspace: every repository side by side
for r in api web infra docs tools app; do gh repo clone ariel5253/orderflow-engine-$r; done

# From the workspace root (the folder that contains orderflow-engine-*)
node orderflow-engine-tools/docs-check/check-docs.mjs      # documentation governance
node orderflow-engine-tools/integrity/check-integrity.mjs  # ignore files, line endings, secrets
python orderflow-engine-tools/diagrams/generate_diagrams.py # regenerate and check the figures
python orderflow-engine-tools/render/build_deliverables.py  # ADR in Word and PDF (needs LibreOffice)
# From T-01 on:
orderflow-engine-infra/scripts/test-all.sh                 # API verify + web tests + docs and integrity checks
(cd orderflow-engine-api && ./mvnw verify)                 # API only
(cd orderflow-engine-api && docker compose up --build)     # run the service and its database
```

Toolchain found on the build machine:
- Java 21
- Node 24
- Docker Desktop (required by Testcontainers and by every `compose.yaml`)
- Python 3.11 with `python-docx` and `pypdf` (tools; see its `requirements.txt`)
- LibreOffice (tools: PDF conversion)

Maven is not installed globally, so the Maven Wrapper is used.

## 6. Open decisions

| Code | Decision pending | Owner |
|------|------------------|-------|
| OD-01 | Resolved on 2026-10-14: the repositories are initialized and private. They become public at delivery, after the release pull requests, and the rulesets are applied then | Owner |
| OD-02 | Scope cuts, if any, at the end of day 4 (see the cut line in TASKS) | Owner |
| OD-03 | Resolved on 2026-10-14: the owner is `ariel5253`. Repositories keep citing each other by name, without links, while they are private | Owner |

## 7. Traps already found

| Code | Trap | How to avoid it |
|------|------|-----------------|
| TR-01 | `java.util.Currency` reports COP with 2 decimals | Scale always comes from the `countries` table (SPEC §4) |
| TR-02 | Code prefixes that look alike get confused (e.g. rounding points and requirements) | Every code family has a distinct prefix. See the codes table in the [document map](../README.md) |
| TR-03 | A rule a check cannot enforce decays | Prefer adding a rule to `check-docs` or an ArchUnit test over writing a guideline |
| TR-04 | Scope creep: good ideas the assignment does not ask for | SPEC §1 is the boundary. Anything new updates it first (GC-9) |
| TR-05 | `check-docs` runs in the workspace, where all repositories sit side by side, but the evaluator clones one repository at a time | A relative link may never leave its repository. `check-docs` enforces this |
| TR-06 | GitHub rejects rulesets on private repositories of a free plan | Follow the protection rules by hand while private, and never enable automatic deletion of head branches before the deletion rulesets exist, or a phase-closing merge deletes `dev` |
| TR-07 | Windows clones have `core.filemode=false`, so a plain `git add` records `mvnw` and `scripts/test-all.sh` without the executable bit and they fail on Linux and macOS | Stage them with `git add --chmod=+x`, then check `git ls-files -s` shows `100755` |
| TR-08 | Until T-02 adds the first pricing domain class, `./mvnw -Pmutation verify` stops with "No mutations found" on purpose, and the API README says so | Remove that qualifying sentence from both READMEs in T-02 (GC-6) |

## 8. Preparation phase pull requests

| Repository | Task pull requests | Phase closing |
|------------|--------------------|---------------|
| `orderflow-engine-tools` | #1 documentation check · #2 repository integrity check · #3 progress chart generator · #4 diagram generator with checks · #5 ADR rendering to Word and PDF · #6 consistency checks · #7 Docker image and README | #8 |
| `orderflow-engine-docs` | #1 constitution and specification · #2 plan, task plan and contract · #3 data model and architecture · #4 golden rule and git workflow · #5 documentation and integrity rules · #6 architecture diagrams · #7 assignment and document map · #8 ADR in Word and PDF · #9 this handoff | Not applicable: `main` only |
| `orderflow-engine-api` | #1 README with the required topics · #2 ADR-0001 draft | #3 |
| `orderflow-engine-web` | #1 README of the summary screen | #2 |
| `orderflow-engine-infra` | #1 README with the clone layout | #2 |
| `orderflow-engine-app` | #1 README of the reserved app | #2 |

## 9. Scaffold phase pull requests

Each API pull request was verified on a clean export of its own commit (`git archive` of the branch head, then `./mvnw verify`), so every commit on `dev` builds on its own.

| Repository | Task pull requests | Phase closing |
|------------|--------------------|---------------|
| `orderflow-engine-api` | #4 service and test command · #5 domain purity and health tests · #6 container image and compose · #7 run notes and not included | #8 |
| `orderflow-engine-infra` | #3 full-stack compose · #4 workspace test command · #5 stack and test command documentation | #6 |
| `orderflow-engine-docs` | #10 migration decision and environment scope · #11 this handoff | Not applicable: `main` only |
