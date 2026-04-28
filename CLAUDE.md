# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

This repo hosts **reusable GitHub Actions workflows and composite actions** that other repositories call via `workflow_call` (workflows) or `uses:` (composite actions). There is no application code, no test suite, and no local build — changes are validated by triggering CI on a downstream consumer.

## Interaction style

This repo's owner prefers brisk, actionable communication. Mirror that.

- **ALWAYS default to concise.** State the result first, reasoning second. Don't lead with three paragraphs of context.
- **ALWAYS use tables for cross-cutting or comparative information** — inputs, callers, before/after states, options. Don't bury comparisons in prose.
- **ALWAYS offer numbered options at decision gates.** Single-digit replies (`1`, `2`, `yes`) are how this user picks. If you ask a question, enumerate the choices; never bury actionable choices inside a paragraph.
- **NEVER prepend unsolicited preamble.** Given a task, do it and report results.
- **Mirror brevity.** A one-line message gets at most a one-paragraph reply.

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

## Verification discipline

Because verification depends on real CI on consumer repos, it's easy to claim "verified" based on partial signals. Don't.

**NEVER claim a CI run passed unless you've checked all of:**

1. The PR's full check status: `gh pr checks <number>`. A single push or PR-open can trigger multiple event types (`workflow_dispatch`, `push`, `pull_request`, `schedule`) — each is a SEPARATE run with independent pass/fail. Watching one is not watching all.
2. Per-step outcomes per matrix leg:
   ```bash
   gh api repos/<owner>/<repo>/actions/runs/<id>/jobs \
     --jq '.jobs[].steps[] | "\(.name): \(.conclusion // .status)"'
   ```
   A run-level `success` with `fail-fast: false` doesn't preclude an individual step having been skipped (changing the meaning of the "success") or having succeeded only in a previous run.

**ANTI-PATTERNS — these are NOT verification:**

- `gh run watch --exit-status` returning `0` for one run, when the same push triggered other runs you didn't watch.
- `docker buildx imagetools inspect <user>/<image>:<tag>` resolving the expected tag — that just proves the tag exists on the registry. It does NOT prove THIS run pushed it. Tags from prior nightly/cron runs persist.
- `gh run list` showing a recent run as `success` without inspecting its per-step outcomes.

**When a run fails**, immediately fetch the failed step's log without waiting for the user to surface the error:

```bash
gh run view <run-id> --log-failed
# or for a specific failed job:
gh api repos/<owner>/<repo>/actions/jobs/<job-id>/logs
```

Report the actual error (root cause, not just the run-level conclusion) before asking the user for direction.

## Editing notes

- Public surface (`workflow_call` inputs and secrets) must only grow, never shrink. Consumers all pin `@main`, so any breaking change propagates instantly.
- Composite-action input/secret names are also public surface for the workflow's internal call sites and any external direct callers. Treat with the same care.
- When touching `cache-from` / `cache-to`, remember each `tag_prefix` value gets its own cache lane (`:cache-<tag_prefix>`). Legacy callers without `tag_prefix` use the bare `:cache`. Don't unify these without thinking through the concurrent-write race that prompted the split.
- **ALWAYS prefer in-repo fixes over consumer-repo edits.** When work could be done in this repo OR in a consumer repo (e.g. `dvdrental`, `postgres`, `postgresql`), default to this repo's surface. Touching consumers expands blast radius, multiplies edits, and risks introducing bugs in unrelated code. Only touch a consumer repo when the user explicitly authorises it — and that includes verification patterns like temp-pinning a consumer at a feature branch.

### Plan-rigour checklist for prescriptive YAML

Plans that prescribe exact YAML for this repo are fragile: in-context bugs caught by reviewers in past work include subshell exit-code traps, heredoc-delimiter collisions, conditional-expression edge cases for non-default callers, and cross-step auth dependencies hidden behind `if:` gates. Before accepting a YAML block as the spec, manually trace:

- **Bash subshell traps.** Anything piped into `while` (e.g. `printf | while`) runs the loop in a subshell — `failed=1` flags don't survive past the loop and the exit code is dominated by the last iteration. Use process substitution (`while ...; do ...; done < <(...)`) when you need exit-code accumulation across iterations.
- **Heredoc delimiter collisions.** A `<<EOF ... EOF` heredoc fed user-supplied content can be silently truncated if the content contains a bare `EOF` line. Use a delimiter that cannot appear in legal input (e.g. `BUILDARGS_EOF` for build-args, since a bare `BUILDARGS_EOF` is not a valid `KEY=VALUE`).
- **Caller-archetype expression walks.** For any conditional `${{ ... }}` expression, walk it through every caller archetype in "Known consumers" and confirm the rendered string. Default values, empty strings, and matrix-axis empty cases all interact.
- **Step-skip auth interactions.** When gating cred-consuming steps on `if:`, ask whether ANY downstream step (e.g. `cache-to: type=registry`) requires those creds independently of the gate condition. The auth flow is a hidden dependency — `cache-to` always tries to authenticate even when the main `push:` is false.

## Known consumers

- `SirSplat/dvdrental` — single-tag mode
- `SirSplat/postgres` — single-tag mode (caller-side matrix on postgres major × variant)
- `SirSplat/postgresql/{dbo,latest,pgcrypto,pgtap,pgvector}-docker-image.yaml` — matrix mode, all five share the `image_name: postgresql` and the `:cache-<variant>` lanes
