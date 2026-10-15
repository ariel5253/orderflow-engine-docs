---
id: GOV-INTEGRITY
title: Repository integrity
status: accepted
lang: en
pair: none
last-reviewed: 2026-10-14
---

# Repository integrity

> **Question this document answers:** what must every repository contain, ignore and never contain, so that it runs on its own and never leaks a secret?

`orderflow-engine-tools/integrity/check-integrity.mjs` enforces every rule below. Rules that depend on a file (a `Dockerfile`, a `compose.yaml`) apply as soon as that file exists.

## 1. Docker and environment per repository

| Repository | Own Docker | `.env.example` | Runs with |
|------------|-----------|----------------|-----------|
| `orderflow-engine-api` | `Dockerfile`, `compose.yaml` (service + PostgreSQL), from T-01 | Yes | `docker compose up --build` |
| `orderflow-engine-web` | `Dockerfile` (nginx), `compose.yaml`, from T-17 | Yes | `docker compose up --build` |
| `orderflow-engine-tools` | `Dockerfile` (Python and Node) | No | `docker run` with the workspace mounted (see its README) |
| `orderflow-engine-infra` | `compose.yaml` for the full stack, from T-01 | Yes | `docker compose up --build` |
| `orderflow-engine-docs`, `orderflow-engine-app` | None: content only, and reserved | No | — |

- **Standalone:** the build context and every path in a `Dockerfile` or `compose.yaml` stay inside their own repository. Only `orderflow-engine-infra` may reference sibling repositories, because it orchestrates them. Its README states the side-by-side clone layout this requires.
- **Single test command unchanged:** `./mvnw verify` stays the API's test command and uses Testcontainers. Docker is the way to run the service, not the way to test it.

## 2. Environment files

| Rule | Detail |
|------|--------|
| `.env` is never versioned | Every `.gitignore` lists `.env`. Once a repository is under git, the check also fails if git tracks a `.env`. The content of a local `.env` is never scanned: it is the one place where real values live |
| `.env` never enters an image | Every `.dockerignore` lists `.env` |
| `.env.example` is versioned where a repository runs with configuration | It lists every key, each preceded by a comment line. A secret key is empty or `change-me`. Other keys may carry their non-secret local default. Never a real secret |
| One source for the API configuration keys | Every key in the API's `.env.example` comes from the configuration table in [plan.md](../spec/plan.md) §2, and enters it in the task that wires it |

## 3. Required ignore entries

| Repository | `.gitignore` | `.dockerignore` (when there is a `Dockerfile`) |
|------------|--------------|-----------------------------------------------|
| All | `.env`, operating-system and editor files | `.env`, `.git` |
| `orderflow-engine-api` | `target/` | `target/` |
| `orderflow-engine-web` | `node_modules/`, `dist/`, `.angular/` | `node_modules/`, `dist/` |
| `orderflow-engine-tools` | `__pycache__/`, `.venv/` | `__pycache__/`, `.venv/` |

## 4. Line endings and binaries

Every repository has the same `.gitattributes`:

| Pattern | Rule | Why |
|---------|------|-----|
| `*` | `text=auto eol=lf` | One line ending everywhere. `mvnw` and shell scripts stay runnable on Linux |
| `*.cmd`, `*.bat`, `*.ps1` | `text eol=crlf` | Windows scripts keep Windows line endings |
| `*.pdf`, `*.docx`, `*.png`, `*.jpg` | `binary` | Never altered by line-ending conversion |

## 5. Secrets

| Rule | Scope |
|------|-------|
| No private keys, cloud access keys or tokens (`-----BEGIN … PRIVATE KEY-----`, `AKIA…`, `ghp_…`, `xox?-…`) | Every text file of every repository |
| No non-empty value for a key whose name contains `password`, `secret`, `token` or `api key` | `.env.example` and any other `.env.*`, `*.yml`, `*.yaml`, `*.properties`. Allowed values: empty, `change-me`, or a `${VARIABLE}` reference |
