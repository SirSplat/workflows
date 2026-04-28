# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

This repo hosts **reusable GitHub Actions workflows and composite actions** that other repositories call via `workflow_call` (workflows) or `uses:` (composite actions). There is no application code, no test suite, and no local build — changes are validated by triggering CI on a downstream consumer.

## Layout

- `.github/workflows/` — reusable workflows (currently one: `build-and-push-docker-image.yaml`)
- `.github/actions/` — composite actions consumed by the reusable workflow above
- `docs/superpowers/plans/` — implementation plans for non-trivial work

## Workflow: `.github/workflows/build-and-push-docker-image.yaml`

A reusable workflow that builds a multi-arch Docker image (`linux/amd64`, `linux/arm64/v8`) via Buildx + QEMU and pushes it to Docker Hub with provenance and SBOM attestations. Build cache is stored on Docker Hub itself as a separate `:cache` (single-tag mode) or `:cache-<tag_prefix>` (matrix mode) tag on the same image repo.

### Two modes

1. **Single-tag mode** (default — used by `dvdrental`, `postgres`): caller passes `tag: <something>`, the image is pushed as `<owner>/<image_name>:<tag>`. Cache lives at `<owner>/<image_name>:cache`.
2. **Matrix mode** (used by `postgresql/*` — five variants): caller passes `tag_prefix:` plus a JSON `pg_major_matrix:`. The workflow runs one job per `pg_major` and pushes `<owner>/<image_name>:<tag_prefix>-pg<N>` per leg, plus an unsuffixed `<owner>/<image_name>:<tag_prefix>` for the leg matching `default_pg_major`. Cache lives at `<owner>/<image_name>:cache-<tag_prefix>` (one lane per variant — multiple variants of the same `image_name` would otherwise race on the same cache tag).

`PG_MAJOR=<value>` is auto-injected into `build-args` in matrix mode; callers do NOT add it manually.

Optional `status_gist_id` plus `GIST_TOKEN` secret enables shields.io-shaped gist status reporting per leg, with backoff+jitter retry to handle gist-API HTTP 409s under matrix concurrency.

### Inputs

`directory`, `image_name`, `tag`, `runner_debug`, `step_debug`, `pg_major_matrix`, `default_pg_major`, `tag_prefix`, `build_args`, `image_description`, `push_on_pull_request`, `status_gist_id`. All are optional except `directory` and `image_name`.

### Secrets

Required: `DOCKER_USERNAME`, `DOCKER_PASSWORD`. Optional: `GIST_TOKEN` (only when `status_gist_id` is set).

### Push-vs-build behaviour

`push_on_pull_request` defaults to `true` (legacy behaviour). When set to `false`, `pull_request`-event runs build but skip the image push and the `verify-pushed-tags` / gist-report steps. **Login still happens unconditionally** — `cache-to: type=registry` always tries to authenticate to push the build cache regardless of the main image push flag, so gating login on the push condition would surface as 401 errors. Logging in on a PR-only build is cheap and lets the cache get warmed for the eventual merge build.

## Composite actions: `.github/actions/`

The reusable workflow above calls these via cross-repo syntax (`SirSplat/workflows/.github/actions/<name>@main`). They are also independently callable by any other workflow.

- **`validate-dockerhub-creds`** — pre-flight check that `DOCKER_USERNAME` and `DOCKER_PASSWORD` are non-empty, with a `::error::` annotation naming the specific missing secret. The bash check is the real guard — `required: true` on the action input does NOT catch empty-string values from unset secrets, so the runtime check must not be removed.
- **`verify-pushed-tags`** — runs `docker buildx imagetools inspect` per pushed tag and aggregates failures across all tags before failing the step. Uses process substitution rather than a pipe to keep the loop in the main shell so any non-last tag failure correctly fails the action.
- **`report-status-gist`** — opt-in shields.io status badge updater. 8-attempt retry with backoff+jitter for the gist API's HTTP 409 under matrix concurrency. No-op (exit 0 with informative log) when either `gh-token` or `gist-id` is empty.

### Self-pinning at `@main`

The reusable workflow's references to its own composite actions use `@main`. This means:
- Internal changes are atomic via merge to `main` — no separate version bump needed.
- A feature-branch smoke test of the workflow file alone is awkward: the workflow YAML loads from the branch but composite actions resolve from `main`. If you need a pre-merge integration run, either temp-pin the three `uses:` lines to your branch, or split composite actions into a preparatory PR that merges first.

## Validation

No local test framework. Changes are validated by:

1. **`actionlint` static check** (Docker): `docker run --rm -v "$(pwd):/repo" -w /repo rhysd/actionlint:latest -color`. Catches YAML syntax and obvious expression mistakes. Note: `actionlint` only scans `.github/workflows/` by default — composite actions in `.github/actions/` are not exercised unless explicitly pointed at them.
2. **Real downstream run.** Push to a branch and either repoint a known consumer's `uses:` ref temporarily, or wait for the post-merge run on `main` (consumers all pin `@main`).

## Editing notes

- Public surface (`workflow_call` inputs and secrets) must only grow, never shrink. Consumers all pin `@main`, so any breaking change propagates instantly.
- Composite-action input/secret names are also public surface for the workflow's internal call sites and any external direct callers. Treat with the same care.
- When touching `cache-from` / `cache-to`, remember each `tag_prefix` value gets its own cache lane (`:cache-<tag_prefix>`). Legacy callers without `tag_prefix` use the bare `:cache`. Don't unify these without thinking through the concurrent-write race that prompted the split.

## Known consumers

- `SirSplat/dvdrental` — single-tag mode
- `SirSplat/postgres` — single-tag mode (caller-side matrix on postgres major × variant)
- `SirSplat/postgresql/{dbo,latest,pgcrypto,pgtap,pgvector}-docker-image.yaml` — matrix mode, all five share the `image_name: postgresql` and the `:cache-<variant>` lanes
