---
id: GOV-PULL-REQUESTS
title: Pull request rules
status: accepted
lang: en
pair: none
last-reviewed: 2026-10-14
---

# Pull request rules

> **Question this document answers:** how is a pull request opened, verified, merged and cleaned up?

## Task pull requests (task branch → `dev`, or → `main` in docs)

| Rule | Detail |
|------|--------|
| One task, one or more pull requests | The branch is `<type>/T-NN-slug`, created from the latest base. A task whose diff is too large to review in one sitting is split into thematic pull requests. Each one stands on its own and carries the task tag |
| Base | `dev` in a code repository, `main` in `orderflow-engine-docs` ([branches-and-environments.md](branches-and-environments.md)). GitHub defaults to `main`, so in a code repository the base is set explicitly: `gh pr create --base dev` |
| Title | A Conventional Commit of at most 60 characters ending with the task tag ([commit-conventions.md](commit-conventions.md)) |
| Body | The template below, complete. A pull request without verification output is not merged |
| Merge | **Squash**. The commit is exactly the pull request title, with an empty body, so the verification output never becomes part of the history |
| After merge | The task branch is deleted |
| Documentation | Updated in the same pull request as the behavior it describes (GC-6) |

The squash merge sets the one-line commit explicitly and deletes the task branch:

```bash
gh pr merge <number> --squash --delete-branch --subject "<the pull request title>" --body ""
```

Body template:

```markdown
## Task
T-NN — <title> (link to the task in TASKS)

## Covers
AC-xx, AC-yy

## Verification
<commands run and their output: ./mvnw verify, check-docs, check-integrity, test-all>

## Notes
<deviations from the task, if any, and why>
```

## Phase-closing pull requests (`dev` → `main`, code repositories)

| Rule | Detail |
|------|--------|
| When | The owner closes a phase, for example the preparation phase (T-00) |
| Title | `chore: close <phase> phase (T-NN)`. No version and no tag |
| Body | The pull requests included and the verification output. It stays in the pull request, not in the commit |
| Merge | **Merge commit**, never squash, so every per-task commit stays reachable from `main` |
| After merge | `dev` stays, so the merge never deletes its head branch |

```bash
gh pr merge <number> --merge --subject "chore: close <phase> phase (T-NN)" --body ""
```

## Release pull requests (`dev` → `main`, code repositories)

| Rule | Detail |
|------|--------|
| When | Only when the finished product is accepted, and always before the delivery link is sent |
| Title | `release: vX.Y.Z` |
| Body | The tasks included and the final verification output. It stays in the pull request, not in the commit |
| Merge | **Merge commit**, never squash |
| After merge | `dev` stays |
| When `dev` and `main` already match | No pull request, since it would be empty: an annotated tag `vX.Y.Z` on `main` and a GitHub Release with the notes |

The merge commit takes the release title instead of GitHub's default "Merge pull request #N":

```bash
gh pr merge <number> --merge --subject "release: vX.Y.Z" --body ""
```
