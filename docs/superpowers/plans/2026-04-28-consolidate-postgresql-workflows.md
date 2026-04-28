# Consolidate postgresql/* workflows onto the shared workflow — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend `SirSplat/workflows/.github/workflows/build-and-push-docker-image.yaml` to absorb the five near-identical multi-variant Docker build workflows in `SirSplat/postgresql`, eliminating ~700 lines of duplicated YAML and bringing all action pins under central control. Existing consumers (`dvdrental`, `postgres`) keep working with zero edits.

**Architecture:** Three changes work together. (1) Three new composite actions in this repo encapsulate the duplicated sub-routines (`validate-dockerhub-creds`, `verify-pushed-tags`, `report-status-gist`). (2) The shared reusable workflow gains seven optional inputs whose defaults preserve current behaviour, plus a `pg_major` matrix dimension that activates only when callers provide a non-trivial `pg_major_matrix`. (3) Each of the five `postgresql/*-docker-image.yaml` files is rewritten as a ~25-line caller that delegates to the extended shared workflow.

**Tech Stack:** GitHub Actions reusable workflows (`workflow_call`), composite actions, `docker/metadata-action@v6`, `docker/build-push-action@v7`, `actionlint` for static validation.

**Verification model:** No unit-test framework exists for workflows. Verification at each step uses (a) `actionlint` static analysis (Docker), (b) GitHub's own workflow-file validation visible after `git push` via `gh workflow view`, and (c) end-to-end downstream `workflow_dispatch` runs for any phase that touches behaviour. TDD is not feasible; the closest analogue is "static-lint-then-trigger".

**Constraints:**
- Backwards compatibility is mandatory. `dvdrental/docker-image.yaml` and `postgres/ci.yaml` must continue to work without any edits in their repos.
- Public surface (`workflow_call` inputs and secrets) must only grow, never shrink. No renames. The previously-required `tag` input becomes optional with default `""`, which is a non-breaking change from a caller's perspective.
- `vars.STATUS_GIST_ID` cannot be auto-forwarded across `workflow_call` (variables don't propagate). Solution: callers pass it as the `status_gist_id` input.
- Internal references from the shared workflow to the new composite actions must use the cross-repo `uses: SirSplat/workflows/.github/actions/<name>@main` syntax, NOT `uses: ./.github/actions/<name>`. Reusable workflows do not auto-checkout sibling files.
- One workflow PR is merged before any postgresql migration PR. The migration PR depends on the shared change being live on `main`.

---

## File Structure

### `SirSplat/workflows` (this repo) — extension

| Path | Action | Responsibility |
|---|---|---|
| `.github/actions/validate-dockerhub-creds/action.yml` | Create | Pre-flight check that `DOCKER_USERNAME` / `DOCKER_PASSWORD` are non-empty; fails early with a clear message if not. |
| `.github/actions/verify-pushed-tags/action.yml` | Create | Post-build verification that every tag emitted by `metadata-action` resolves through `docker buildx imagetools inspect`. |
| `.github/actions/report-status-gist/action.yml` | Create | Best-effort PATCH of a shields.io-shaped gist payload with 8-attempt backoff+jitter; non-fatal. |
| `.github/workflows/build-and-push-docker-image.yaml` | Modify | Add 7 optional inputs + 1 optional secret; introduce `pg_major` matrix; drive tags through `metadata-action` correctly; call the three new composite actions. |
| `docs/superpowers/plans/2026-04-28-consolidate-postgresql-workflows.md` | (already created) | This plan. |
| `CLAUDE.md` | Modify (last task) | Document the new inputs / composite actions for future Claude sessions. |
| `README.md` | Modify (last task) | One-line caller examples for the matrix mode. |

### `SirSplat/postgresql` (separate repo) — migration

| Path | Action | Responsibility |
|---|---|---|
| `.github/workflows/dbo-docker-image.yaml` | Rewrite | Phase-2 proof-of-concept caller. ~25 lines. |
| `.github/workflows/latest-docker-image.yaml` | Rewrite | Phase-3 batch migration. |
| `.github/workflows/pgcrypto-docker-image.yaml` | Rewrite | Phase-3 batch migration. |
| `.github/workflows/pgtap-docker-image.yaml` | Rewrite | Phase-3 batch migration. |
| `.github/workflows/pgvector-docker-image.yaml` | Rewrite | Phase-3 batch migration. |

### Locked-in names (referenced across tasks)

- New inputs: `pg_major_matrix`, `default_pg_major`, `tag_prefix`, `build_args`, `image_description`, `push_on_pull_request`, `status_gist_id`
- Modified input: `tag` (optional, default `""`)
- New optional secret: `GIST_TOKEN`
- Composite action input names:
  - `validate-dockerhub-creds`: `docker-username`, `docker-password`
  - `verify-pushed-tags`: `tags`
  - `report-status-gist`: `gist-id`, `filename`, `label`, `message`, `color`, `gh-token`
- Branch names: `extend-shared-workflow` (this repo), `migrate-dbo-onto-shared-workflow` (postgresql phase 2), `migrate-remaining-onto-shared-workflow` (postgresql phase 3)

---

## Phase 1: Build the extension in this repo

### Task 1: Create the `validate-dockerhub-creds` composite action

**Files:**
- Create: `.github/actions/validate-dockerhub-creds/action.yml`

- [ ] **Step 1: Create the branch**

```bash
git checkout main && git pull
git checkout -b extend-shared-workflow
```

- [ ] **Step 2: Create the action file**

```yaml
# .github/actions/validate-dockerhub-creds/action.yml
name: Validate Docker Hub credentials
description: Fail early if DOCKER_USERNAME or DOCKER_PASSWORD is empty.

inputs:
  docker-username:
    description: 'Docker Hub username (pass secrets.DOCKER_USERNAME)'
    required: true
  docker-password:
    description: 'Docker Hub password or PAT (pass secrets.DOCKER_PASSWORD)'
    required: true

runs:
  using: composite
  steps:
    - name: Check credentials are present
      shell: bash
      env:
        DOCKER_USERNAME: ${{ inputs.docker-username }}
        DOCKER_PASSWORD: ${{ inputs.docker-password }}
      run: |
        if [ -z "$DOCKER_USERNAME" ] || [ -z "$DOCKER_PASSWORD" ]; then
          echo "::error::Docker Hub credentials are not configured. Set DOCKER_USERNAME and DOCKER_PASSWORD secrets in the calling repository."
          exit 1
        fi
        echo "Docker Hub credentials present for user '${DOCKER_USERNAME}'."
```

- [ ] **Step 3: Lint with actionlint**

```bash
docker run --rm -v "$(pwd):/repo" -w /repo rhysd/actionlint:latest -color
```

Expected: no errors, no output beyond the header line.

- [ ] **Step 4: Commit**

```bash
git add .github/actions/validate-dockerhub-creds/action.yml
git commit -m "Add validate-dockerhub-creds composite action

Pre-flight check used by the shared Docker build workflow. Fails early
with a clear annotation if DOCKER_USERNAME or DOCKER_PASSWORD is empty,
rather than letting docker/login-action emit a less helpful error
mid-pipeline."
```

---

### Task 2: Create the `verify-pushed-tags` composite action

**Files:**
- Create: `.github/actions/verify-pushed-tags/action.yml`

- [ ] **Step 1: Create the action file**

```yaml
# .github/actions/verify-pushed-tags/action.yml
name: Verify pushed tags
description: Confirms each tag emitted by docker/metadata-action resolves via `docker buildx imagetools inspect`.

inputs:
  tags:
    description: 'Multiline string of fully-qualified image tags (typically steps.meta.outputs.tags).'
    required: true

runs:
  using: composite
  steps:
    - name: Inspect each tag
      shell: bash
      env:
        TAGS: ${{ inputs.tags }}
      run: |
        if [ -z "$TAGS" ]; then
          echo "::error::No tags provided to verify-pushed-tags. Did metadata-action emit anything?"
          exit 1
        fi
        printf '%s\n' "$TAGS" | while IFS= read -r image; do
          [ -z "$image" ] && continue
          echo "Verifying ${image}"
          docker buildx imagetools inspect "$image" > /dev/null
        done
```

- [ ] **Step 2: Lint**

```bash
docker run --rm -v "$(pwd):/repo" -w /repo rhysd/actionlint:latest -color
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add .github/actions/verify-pushed-tags/action.yml
git commit -m "Add verify-pushed-tags composite action

Post-build sanity check: every tag emitted by metadata-action must
resolve through docker buildx imagetools inspect. Catches the case
where build-push-action reports success but the registry didn't
actually accept all tags (rare, but happens with provenance/SBOM
quirks)."
```

---

### Task 3: Create the `report-status-gist` composite action

**Files:**
- Create: `.github/actions/report-status-gist/action.yml`

- [ ] **Step 1: Create the action file**

```yaml
# .github/actions/report-status-gist/action.yml
name: Report status to gist
description: Best-effort PATCH of a shields.io-shaped status payload to a gist file. Non-fatal; logs a warning if it cannot complete after 8 attempts.

inputs:
  gist-id:
    description: 'Gist ID to PATCH. If empty, the action is a no-op.'
    required: false
    default: ''
  filename:
    description: 'Filename within the gist to write (e.g. "dbo-pg18.json").'
    required: true
  label:
    description: 'Shields.io label (left side of the badge).'
    required: true
  message:
    description: 'Shields.io message (right side of the badge).'
    required: true
  color:
    description: 'Shields.io color (e.g. brightgreen, red, lightgrey).'
    required: true
  gh-token:
    description: 'Token with gist write scope. If empty, the action is a no-op.'
    required: false
    default: ''

runs:
  using: composite
  steps:
    - name: PATCH gist with retry+jitter
      shell: bash
      env:
        GH_TOKEN: ${{ inputs.gh-token }}
        GIST_ID: ${{ inputs.gist-id }}
        FILENAME: ${{ inputs.filename }}
        LABEL: ${{ inputs.label }}
        MESSAGE: ${{ inputs.message }}
        COLOR: ${{ inputs.color }}
      run: |
        set -e
        if [ -z "$GH_TOKEN" ] || [ -z "$GIST_ID" ]; then
          echo "report-status-gist: gh-token or gist-id empty; skipping (configured no-op)."
          exit 0
        fi
        STATUS_JSON=$(jq -n \
          --arg label "$LABEL" \
          --arg msg "$MESSAGE" \
          --arg color "$COLOR" \
          '{schemaVersion:1, label:$label, message:$msg, color:$color}')
        PAYLOAD=$(jq -n \
          --arg fname "$FILENAME" \
          --arg content "$STATUS_JSON" \
          '{files: {($fname): {content: $content}}}')
        # The gist API serialises writes; concurrent matrix legs all
        # PATCHing the same gist trigger HTTP 409. Retry with backoff
        # plus jitter. A persistent failure here is non-fatal.
        for attempt in 1 2 3 4 5 6 7 8; do
          if echo "$PAYLOAD" | gh api -X PATCH "gists/$GIST_ID" --input -; then
            exit 0
          fi
          SLEEP_S=$((attempt * 3 + RANDOM % 5))
          echo "Gist update attempt ${attempt}/8 failed (likely 409 from a concurrent matrix leg); retrying in ${SLEEP_S}s..."
          sleep "$SLEEP_S"
        done
        echo "::warning::Gist status update still failing after 8 attempts; badge may be stale."
        exit 0
```

- [ ] **Step 2: Lint**

```bash
docker run --rm -v "$(pwd):/repo" -w /repo rhysd/actionlint:latest -color
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add .github/actions/report-status-gist/action.yml
git commit -m "Add report-status-gist composite action

Extracted from postgresql/*.yaml's identical 'Report build status to
status gist' steps. Encapsulates the 8-attempt retry+jitter handling
for HTTP 409s when concurrent matrix legs PATCH the same gist. No-op
when either gist-id or gh-token is empty, so the shared workflow can
unconditionally call it."
```

---

### Task 4: Extend the shared workflow's input/secret surface

**Files:**
- Modify: `.github/workflows/build-and-push-docker-image.yaml` (lines 1–35, the `on: workflow_call:` block)

- [ ] **Step 1: Replace the workflow_call block**

Open `.github/workflows/build-and-push-docker-image.yaml` and replace the existing `on:` block (lines 5–35 in the current file) with:

```yaml
on:
  workflow_call:
    inputs:
      directory:
        description: 'The directory containing the Dockerfile'
        required: true
        type: string
      image_name:
        description: 'The Docker image name (without registry/owner prefix)'
        required: true
        type: string
      tag:
        description: 'Single-tag mode: the tag to push. Ignored if tag_prefix is set. Optional in matrix mode.'
        required: false
        type: string
        default: ''
      runner_debug:
        description: 'Enable runner debug logging'
        required: false
        type: boolean
        default: false
      step_debug:
        description: 'Enable step debug logging'
        required: false
        type: boolean
        default: false
      pg_major_matrix:
        description: 'JSON array of PostgreSQL major versions to build in parallel. Default `[""]` runs a single non-pg-aware leg.'
        required: false
        type: string
        default: '[""]'
      default_pg_major:
        description: 'Which value of pg_major (when matrix mode is active) also receives the unsuffixed `tag_prefix` tag.'
        required: false
        type: string
        default: ''
      tag_prefix:
        description: 'Prefix for variant tags. When set, drives multi-tag mode (`<prefix>-pg<major>` plus `<prefix>` for the default). Empty string falls back to single-tag `tag` mode.'
        required: false
        type: string
        default: ''
      build_args:
        description: 'Multiline build-args string passed through to docker/build-push-action.'
        required: false
        type: string
        default: ''
      image_description:
        description: 'Sets org.opencontainers.image.description label. Empty disables.'
        required: false
        type: string
        default: ''
      push_on_pull_request:
        description: 'Whether to push the image on pull_request events. Default true preserves legacy behaviour.'
        required: false
        type: boolean
        default: true
      status_gist_id:
        description: 'Gist ID for shields.io status reporting. Empty disables.'
        required: false
        type: string
        default: ''
    secrets:
      DOCKER_USERNAME:
        required: true
      DOCKER_PASSWORD:
        required: true
      GIST_TOKEN:
        required: false
```

- [ ] **Step 2: Lint**

```bash
docker run --rm -v "$(pwd):/repo" -w /repo rhysd/actionlint:latest -color
```

Expected: no errors. (The `jobs:` block downstream still references the old behaviour — actionlint won't object until the inputs are actually used incorrectly.)

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/build-and-push-docker-image.yaml
git commit -m "Extend shared workflow input/secret surface

Adds pg_major_matrix, default_pg_major, tag_prefix, build_args,
image_description, push_on_pull_request, status_gist_id inputs and
GIST_TOKEN optional secret. Defaults preserve current behaviour for
existing callers (dvdrental, postgres). The 'tag' input becomes
optional (was required:true) which is a non-breaking change for
existing callers that still pass it."
```

---

### Task 5: Refactor the shared workflow body to use the matrix and composite actions

**Files:**
- Modify: `.github/workflows/build-and-push-docker-image.yaml` (the entire `jobs:` block)

- [ ] **Step 1: Replace the `jobs:` block**

Replace everything from `jobs:` to end-of-file with:

```yaml
jobs:
  build-and-push:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        pg_major: ${{ fromJson(inputs.pg_major_matrix) }}

    steps:
    - name: Checkout repository
      uses: actions/checkout@v6

    - name: Enable Debug Logging
      if: ${{ inputs.runner_debug }}
      run: echo "ACTIONS_RUNNER_DEBUG=true" >> $GITHUB_ENV

    - name: Enable Step Debug Logging
      if: ${{ inputs.step_debug }}
      run: echo "ACTIONS_STEP_DEBUG=true" >> $GITHUB_ENV

    - name: Validate Docker Hub credentials
      if: ${{ inputs.push_on_pull_request || github.event_name != 'pull_request' }}
      uses: SirSplat/workflows/.github/actions/validate-dockerhub-creds@main
      with:
        docker-username: ${{ secrets.DOCKER_USERNAME }}
        docker-password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Set up QEMU
      uses: docker/setup-qemu-action@v4
      with:
        platforms: all

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v4

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v6
      with:
        images: ${{ secrets.DOCKER_USERNAME }}/${{ inputs.image_name }}
        tags: |
          type=raw,value=${{ inputs.tag }},enable=${{ inputs.tag != '' && inputs.tag_prefix == '' }}
          type=raw,value=${{ inputs.tag_prefix }}-pg${{ matrix.pg_major }},enable=${{ inputs.tag_prefix != '' && matrix.pg_major != '' }}
          type=raw,value=${{ inputs.tag_prefix }},enable=${{ inputs.tag_prefix != '' && inputs.default_pg_major != '' && matrix.pg_major == inputs.default_pg_major }}
        labels: |
          org.opencontainers.image.description=${{ inputs.image_description }}

    - name: Create Buildx builder
      run: |
        docker buildx create --name mybuilder --use
        docker buildx inspect --bootstrap

    - name: Log in to Docker Hub
      if: ${{ inputs.push_on_pull_request || github.event_name != 'pull_request' }}
      uses: docker/login-action@v4
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and push Docker image
      uses: docker/build-push-action@v7
      with:
        context: ${{ inputs.directory }}
        file: ${{ inputs.directory }}/Dockerfile
        push: ${{ inputs.push_on_pull_request || github.event_name != 'pull_request' }}
        provenance: mode=max
        sbom: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        build-args: ${{ inputs.build_args }}
        platforms: linux/amd64,linux/arm64/v8
        cache-from: type=registry,ref=${{ secrets.DOCKER_USERNAME }}/${{ inputs.image_name }}:cache
        cache-to: type=registry,ref=${{ secrets.DOCKER_USERNAME }}/${{ inputs.image_name }}:cache,mode=max

    - name: Verify pushed tags
      if: ${{ inputs.push_on_pull_request || github.event_name != 'pull_request' }}
      uses: SirSplat/workflows/.github/actions/verify-pushed-tags@main
      with:
        tags: ${{ steps.meta.outputs.tags }}

    - name: Report build status to status gist
      if: ${{ always() && inputs.status_gist_id != '' && (inputs.push_on_pull_request || github.event_name != 'pull_request') }}
      uses: SirSplat/workflows/.github/actions/report-status-gist@main
      with:
        gist-id: ${{ inputs.status_gist_id }}
        filename: ${{ inputs.tag_prefix }}-pg${{ matrix.pg_major }}.json
        label: pg${{ matrix.pg_major }}
        message: ${{ job.status == 'success' && 'passing' || (job.status == 'cancelled' && 'cancelled' || 'failing') }}
        color: ${{ job.status == 'success' && 'brightgreen' || (job.status == 'cancelled' && 'lightgrey' || 'red') }}
        gh-token: ${{ secrets.GIST_TOKEN }}
```

- [ ] **Step 2: Lint**

```bash
docker run --rm -v "$(pwd):/repo" -w /repo rhysd/actionlint:latest -color
```

Expected: no errors. (If actionlint complains about `fromJson` on a string-typed input — which it sometimes does — that's fine; the runtime evaluation is correct.)

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/build-and-push-docker-image.yaml
git commit -m "Wire matrix, metadata-driven tags, and composite actions

- Adds pg_major matrix dimension. Default '[\"\"]' (single empty leg)
  preserves the non-pg behaviour for dvdrental/postgres. Postgresql
  callers will pass e.g. '[\"13\",\"14\",\"15\",\"16\",\"17\",\"18\"]'.
- Drives push tags through metadata-action correctly. Three conditional
  type=raw entries: legacy single-tag, variant-pg<major>, and variant
  default. Existing callers (tag set, tag_prefix empty) emit exactly
  one tag matching today's behaviour.
- push: now respects push_on_pull_request input (default true =
  unchanged for existing callers).
- Cred validation, tag verification, and gist reporting are now wired
  via the composite actions. validate/verify run unconditionally for
  push paths; gist reporting is a no-op unless status_gist_id is set."
```

---

### Task 6: Static-validate and smoke-test against existing callers

**Files:** none modified.

- [ ] **Step 1: Push the branch**

```bash
git push -u origin extend-shared-workflow
```

- [ ] **Step 2: Confirm GitHub accepts the workflow file**

```bash
gh workflow view build-and-push-docker-image.yaml --ref extend-shared-workflow
```

Expected: shows the workflow's input list including the new fields. If GitHub rejected the YAML, this command errors with the parse problem.

- [ ] **Step 3: Smoke-test backwards compat against `dvdrental`**

This step requires temporarily pointing `dvdrental` at the branch. Manual action — DO NOT make this edit silently.

> Tell the user: "Phase-1 smoke test needs `SirSplat/dvdrental/.github/workflows/docker-image.yaml` line 20 changed from `@main` to `@extend-shared-workflow` for one run, then reverted. Want me to open a throwaway PR there, or do you want to do it yourself in your IDE?"

If the user authorises:
```bash
# In a checkout of SirSplat/dvdrental
sed -i 's|build-and-push-docker-image.yaml@main|build-and-push-docker-image.yaml@extend-shared-workflow|' .github/workflows/docker-image.yaml
git checkout -b smoke-test-extend-shared-workflow
git add .github/workflows/docker-image.yaml
git commit -m "TEMPORARY: point at workflows@extend-shared-workflow for smoke test"
git push -u origin smoke-test-extend-shared-workflow
gh workflow run docker-image.yaml --ref smoke-test-extend-shared-workflow
```

- [ ] **Step 4: Wait for the dvdrental run, confirm results**

```bash
gh run list --workflow=docker-image.yaml --branch smoke-test-extend-shared-workflow --limit 1
gh run view <run-id> --log
```

Expected:
- Status: success
- No `Node.js 20 actions are deprecated` annotations.
- `Verify pushed tags` step appears in the log and passes (proves the new composite action works).
- `Validate Docker Hub credentials` step passes.
- `Report build status to status gist` step is skipped (status_gist_id empty).
- Image `<user>/dvdrental:latest` is present on Docker Hub (`docker buildx imagetools inspect <user>/dvdrental:latest`).

- [ ] **Step 5: Revert the dvdrental smoke-test commit**

```bash
# In dvdrental
git checkout main
git push origin --delete smoke-test-extend-shared-workflow
gh pr close <pr-number> 2>/dev/null || true  # if it was a PR
```

- [ ] **Step 6: Open the PR in this repo**

```bash
# Back in SirSplat/workflows
gh pr create --title "Extend shared Docker workflow + extract composite actions" --body "$(cat <<'EOF'
## Summary

Extends the shared workflow to absorb the multi-variant matrix-build pattern used by all five `SirSplat/postgresql/*-docker-image.yaml` workflows, eliminating ~700 lines of duplicated YAML once the postgresql migration follows. Adds three composite actions for the duplicated sub-routines.

## Backwards compatibility

Verified: a smoke-test run of `SirSplat/dvdrental` against this branch succeeds with the existing inputs. All new inputs default to current behaviour. The previously-required `tag` input becomes optional (default `""`); existing callers continue to pass it.

## New inputs (all optional)

- `pg_major_matrix` — JSON array. Default `'[""]'` = single non-pg leg (= today).
- `default_pg_major` — which matrix leg also receives unsuffixed `tag_prefix`.
- `tag_prefix` — when set, switches to multi-tag mode driven by metadata-action.
- `build_args` — passed to build-push-action.
- `image_description` — sets `org.opencontainers.image.description` label.
- `push_on_pull_request` — default `true` (unchanged behaviour).
- `status_gist_id` — empty disables shields.io status gist reporting.

## New optional secret

- `GIST_TOKEN` — gist-write-scoped token. Required only when `status_gist_id` is set.

## New composite actions in this repo

- `.github/actions/validate-dockerhub-creds`
- `.github/actions/verify-pushed-tags`
- `.github/actions/report-status-gist`

The reusable workflow calls them via the cross-repo syntax `SirSplat/workflows/.github/actions/<name>@main`. Self-pinning at `@main` matches the consumer pinning convention.

## Migration plan (separate PR in postgresql repo)

This PR is the prerequisite. Once merged, a follow-up PR in `SirSplat/postgresql` will rewrite the five `*-docker-image.yaml` files as ~25-line callers of the extended workflow. Phase 2 of plan `docs/superpowers/plans/2026-04-28-consolidate-postgresql-workflows.md`.

## Test plan

- [x] Smoke-tested against `SirSplat/dvdrental` on this branch — green, no deprecations, correct tag pushed.
- [ ] Reviewer sanity-check the matrix metadata-action conditionals.
- [ ] Merge.
EOF
)"
```

- [ ] **Step 7: Do not merge yet**

The workflow PR stays open through Phase 2. Phase 2's success is the actual proof, and merging this prematurely would force the postgresql migration PR to update its `@extend-shared-workflow` pin in two places.

---

## Phase 2: Prove the extension on `postgresql/dbo`

### Task 7: Migrate `postgresql/dbo-docker-image.yaml` onto the extended shared workflow

**Files (in `SirSplat/postgresql`):**
- Rewrite: `.github/workflows/dbo-docker-image.yaml`

- [ ] **Step 1: Branch in postgresql**

```bash
# In a checkout of SirSplat/postgresql
git checkout main && git pull
git checkout -b migrate-dbo-onto-shared-workflow
```

- [ ] **Step 2: Replace the file with the caller version**

Replace the entire contents of `.github/workflows/dbo-docker-image.yaml` with:

```yaml
name: Build and Push dbo Docker Image

on:
  push:
    branches: [main]
    paths:
      - 'dbo/Dockerfile'
      - 'dbo/initdb.sh'
      - '.github/workflows/dbo-docker-image.yaml'
  pull_request:
    branches: [main]
    paths:
      - 'dbo/Dockerfile'
      - 'dbo/initdb.sh'
      - '.github/workflows/dbo-docker-image.yaml'
  workflow_dispatch:
  schedule:
    - cron: '0 0 * * 0'

jobs:
  build-and-push:
    uses: SirSplat/workflows/.github/workflows/build-and-push-docker-image.yaml@extend-shared-workflow
    with:
      directory: ./dbo
      image_name: postgresql
      tag_prefix: dbo
      pg_major_matrix: '["13","14","15","16","17","18"]'
      default_pg_major: '18'
      build_args: |
        PG_VARIANT=alpine
      image_description: 'PostgreSQL on Alpine with the dbo application role pre-created as a non-superuser'
      push_on_pull_request: false
      status_gist_id: ${{ vars.STATUS_GIST_ID }}
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
      GIST_TOKEN: ${{ secrets.GIST_TOKEN }}
```

Note the `@extend-shared-workflow` pin — this will be updated to `@main` once the workflow PR merges.

> ⚠️ **PG_MAJOR build-arg gap.** The original passes `PG_MAJOR=${{ matrix.pg_major }}` per leg; the `build_args` input only takes a static string. The shared workflow can't see the matrix axis from the caller. Two resolutions exist:
> 1. **Recommended (simple):** Have each Dockerfile read `PG_MAJOR` from a build-arg defaulting to the matrix value the shared workflow will inject. To enable that, **augment Task 5** of Phase 1 to inject `PG_MAJOR=${{ matrix.pg_major }}` automatically when `matrix.pg_major != ''`, *prepended* to `inputs.build_args`. (See follow-up Task 7a below.)
> 2. **Alternative (caller-driven):** Add a `pg_major_build_arg_name` input. More flexible, more surface.
>
> Choose option 1. It's the convention every postgresql variant already uses (`PG_MAJOR=...`), so encoding it once in the shared workflow is correct.

**Stop here** — do not commit Task 7's file yet. Go back and execute Task 7a in the workflows repo first.

---

### Task 7a (gap-fix): Inject `PG_MAJOR` build-arg automatically when in matrix mode

**Files (back in `SirSplat/workflows`, branch `extend-shared-workflow`):**
- Modify: `.github/workflows/build-and-push-docker-image.yaml` — the `build-args:` line in the `Build and push Docker image` step

- [ ] **Step 1: Add a synthesise-build-args step**

Insert this step *immediately before* the `Build and push Docker image` step:

```yaml
    - name: Synthesise build args
      id: build_args
      shell: bash
      env:
        USER_BUILD_ARGS: ${{ inputs.build_args }}
        PG_MAJOR: ${{ matrix.pg_major }}
      run: |
        {
          echo 'value<<EOF'
          if [ -n "$PG_MAJOR" ]; then
            echo "PG_MAJOR=${PG_MAJOR}"
          fi
          if [ -n "$USER_BUILD_ARGS" ]; then
            printf '%s\n' "$USER_BUILD_ARGS"
          fi
          echo 'EOF'
        } >> "$GITHUB_OUTPUT"
```

- [ ] **Step 2: Update the `build-args:` line**

Change:
```yaml
        build-args: ${{ inputs.build_args }}
```
to:
```yaml
        build-args: ${{ steps.build_args.outputs.value }}
```

- [ ] **Step 3: Lint**

```bash
docker run --rm -v "$(pwd):/repo" -w /repo rhysd/actionlint:latest -color
```

Expected: no errors.

- [ ] **Step 4: Commit and push**

```bash
git add .github/workflows/build-and-push-docker-image.yaml
git commit -m "Auto-inject PG_MAJOR build-arg when matrix.pg_major is set

Postgresql callers always need PG_MAJOR=<value> as a per-leg build-arg.
Rather than make every caller wire it manually (which would require
exposing the matrix axis in caller-passed build_args, which isn't
possible across workflow_call), the shared workflow synthesises it
from matrix.pg_major and prepends it to user-supplied build_args.

For non-matrix callers (matrix.pg_major == '') no PG_MAJOR is added,
preserving current behaviour."
git push origin extend-shared-workflow
```

- [ ] **Step 5: Re-run the dvdrental smoke test (sanity)**

The synthesise step changes the build-args path even for non-matrix callers (it just becomes a passthrough). Confirm dvdrental still works:

```bash
# In dvdrental smoke-test branch
gh workflow run docker-image.yaml --ref smoke-test-extend-shared-workflow
gh run list --workflow=docker-image.yaml --branch smoke-test-extend-shared-workflow --limit 1
```

Expected: success, identical output to before.

---

### Task 7 (resumed): Commit the dbo migration

- [ ] **Step 3: Lint the postgresql workflow file**

```bash
# In SirSplat/postgresql
docker run --rm -v "$(pwd):/repo" -w /repo rhysd/actionlint:latest -color
```

Expected: actionlint may warn that the `uses:` ref is a non-`@main` branch. That's expected for now.

- [ ] **Step 4: Commit and push**

```bash
git add .github/workflows/dbo-docker-image.yaml
git commit -m "Migrate dbo workflow onto extended shared workflow

Replaces 159 lines of bespoke YAML with a 25-line caller. All
behaviour preserved: same matrix, same tags, same gist reporting,
same PR-vs-push semantics. Pinned at the workflows repo's
extend-shared-workflow branch for verification; will be repinned
to @main once that PR merges."
git push -u origin migrate-dbo-onto-shared-workflow
```

- [ ] **Step 5: Trigger the migrated workflow**

```bash
gh workflow run dbo-docker-image.yaml --ref migrate-dbo-onto-shared-workflow
```

- [ ] **Step 6: Wait and verify**

```bash
gh run list --workflow=dbo-docker-image.yaml --branch migrate-dbo-onto-shared-workflow --limit 1
gh run view <run-id>
```

Expected:
- All 6 matrix legs (pg13..pg18) green.
- Each leg pushes tags `dbo-pg<N>`.
- `pg18` additionally pushes the unsuffixed `dbo` tag.
- No `Node.js 20 actions are deprecated` annotation.
- `Verify pushed tags` succeeds in every leg.
- `Report build status to status gist` either succeeds (if `STATUS_GIST_ID` var is set) or skips cleanly.
- Inspect a couple of pushed tags to confirm Docker Hub has them:

```bash
docker buildx imagetools inspect <user>/postgresql:dbo-pg18
docker buildx imagetools inspect <user>/postgresql:dbo
```

Both should resolve, with manifest indices for `linux/amd64` and `linux/arm64/v8`.

---

### Task 8: Merge the workflow extension PR, repin dbo, open the postgresql PR

- [ ] **Step 1: Merge the workflows PR**

```bash
# In SirSplat/workflows
gh pr merge --squash --delete-branch
```

- [ ] **Step 2: Repin dbo from `@extend-shared-workflow` to `@main`**

```bash
# In SirSplat/postgresql, on migrate-dbo-onto-shared-workflow branch
sed -i 's|build-and-push-docker-image.yaml@extend-shared-workflow|build-and-push-docker-image.yaml@main|' .github/workflows/dbo-docker-image.yaml
git add .github/workflows/dbo-docker-image.yaml
git commit -m "Repin dbo onto workflows@main now that the extension is merged"
git push
```

- [ ] **Step 3: Trigger one more dbo run on the branch to confirm `@main` resolves**

```bash
gh workflow run dbo-docker-image.yaml --ref migrate-dbo-onto-shared-workflow
gh run list --workflow=dbo-docker-image.yaml --branch migrate-dbo-onto-shared-workflow --limit 1
```

Expected: green across all 6 legs.

- [ ] **Step 4: Open the postgresql migration PR (dbo-only at this point)**

```bash
gh pr create --title "Migrate dbo workflow onto shared workflow" --body "$(cat <<'EOF'
## Summary

Replaces the bespoke 159-line `dbo-docker-image.yaml` with a 25-line caller of the extended shared workflow at `SirSplat/workflows`. Behaviour preserved: same 6-leg `pg_major` matrix, same multi-tag scheme (`dbo-pgN` plus `dbo` for the default leg), same gist reporting, same PR-vs-push behaviour.

This is the proof-of-concept migration. The remaining four variants (`latest`, `pgcrypto`, `pgtap`, `pgvector`) follow in a separate PR once this one merges and bakes for a cycle.

## Verification

- [x] All 6 matrix legs green on the migration branch.
- [x] Tags `<user>/postgresql:dbo-pg{13,14,15,16,17,18}` and `<user>/postgresql:dbo` present on Docker Hub.
- [x] No Node 20 deprecation warnings.
- [x] Gist status report unchanged (still updates the same filenames).

## Risk

Low. The shared workflow has been smoke-tested against `dvdrental` and now end-to-end against `dbo` itself. Rollback = revert this PR; the bespoke version is preserved in git history.
EOF
)"
```

---

## Phase 3: Migrate the remaining four variants

### Task 9: Migrate `postgresql/latest-docker-image.yaml`

**Files:** Rewrite `.github/workflows/latest-docker-image.yaml` in `SirSplat/postgresql`.

- [ ] **Step 1: Open a fresh branch off `main` (after Phase 2's PR has merged)**

```bash
git checkout main && git pull
git checkout -b migrate-remaining-onto-shared-workflow
```

- [ ] **Step 2: Replace the file**

```yaml
name: Build and Push latest Docker Image

on:
  push:
    branches: [main]
    paths:
      - 'latest/Dockerfile'
      - 'latest/initdb.sh'
      - '.github/workflows/latest-docker-image.yaml'
  pull_request:
    branches: [main]
    paths:
      - 'latest/Dockerfile'
      - 'latest/initdb.sh'
      - '.github/workflows/latest-docker-image.yaml'
  workflow_dispatch:
  schedule:
    - cron: '0 0 * * 0'

jobs:
  build-and-push:
    uses: SirSplat/workflows/.github/workflows/build-and-push-docker-image.yaml@main
    with:
      directory: ./latest
      image_name: postgresql
      tag_prefix: latest
      pg_major_matrix: '["13","14","15","16","17","18"]'
      default_pg_major: '18'
      build_args: |
        PG_VARIANT=alpine
      image_description: 'PostgreSQL on Alpine with pgTAP, pgcrypto, and pgvector installed plus the dbo application role'
      push_on_pull_request: false
      status_gist_id: ${{ vars.STATUS_GIST_ID }}
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
      GIST_TOKEN: ${{ secrets.GIST_TOKEN }}
```

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/latest-docker-image.yaml
git commit -m "Migrate latest workflow onto shared workflow"
```

---

### Task 10: Migrate `postgresql/pgcrypto-docker-image.yaml`

- [ ] **Step 1: Replace the file**

```yaml
name: Build and Push pgcrypto Docker Image

on:
  push:
    branches: [main]
    paths:
      - 'pgcrypto/Dockerfile'
      - 'pgcrypto/initdb.sh'
      - '.github/workflows/pgcrypto-docker-image.yaml'
  pull_request:
    branches: [main]
    paths:
      - 'pgcrypto/Dockerfile'
      - 'pgcrypto/initdb.sh'
      - '.github/workflows/pgcrypto-docker-image.yaml'
  workflow_dispatch:
  schedule:
    - cron: '0 0 * * 0'

jobs:
  build-and-push:
    uses: SirSplat/workflows/.github/workflows/build-and-push-docker-image.yaml@main
    with:
      directory: ./pgcrypto
      image_name: postgresql
      tag_prefix: pgcrypto
      pg_major_matrix: '["13","14","15","16","17","18"]'
      default_pg_major: '18'
      build_args: |
        PG_VARIANT=alpine
      image_description: 'PostgreSQL on Alpine with pgcrypto installed and the dbo application role'
      push_on_pull_request: false
      status_gist_id: ${{ vars.STATUS_GIST_ID }}
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
      GIST_TOKEN: ${{ secrets.GIST_TOKEN }}
```

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/pgcrypto-docker-image.yaml
git commit -m "Migrate pgcrypto workflow onto shared workflow"
```

---

### Task 11: Migrate `postgresql/pgtap-docker-image.yaml`

- [ ] **Step 1: Replace the file**

```yaml
name: Build and Push pgtap Docker Image

on:
  push:
    branches: [main]
    paths:
      - 'pgtap/Dockerfile'
      - 'pgtap/initdb.sh'
      - '.github/workflows/pgtap-docker-image.yaml'
  pull_request:
    branches: [main]
    paths:
      - 'pgtap/Dockerfile'
      - 'pgtap/initdb.sh'
      - '.github/workflows/pgtap-docker-image.yaml'
  workflow_dispatch:
  schedule:
    - cron: '0 0 * * 0'

jobs:
  build-and-push:
    uses: SirSplat/workflows/.github/workflows/build-and-push-docker-image.yaml@main
    with:
      directory: ./pgtap
      image_name: postgresql
      tag_prefix: pgtap
      pg_major_matrix: '["13","14","15","16","17","18"]'
      default_pg_major: '18'
      build_args: |
        PG_VARIANT=alpine
      image_description: 'PostgreSQL on Alpine with pgTAP installed and the dbo application role'
      push_on_pull_request: false
      status_gist_id: ${{ vars.STATUS_GIST_ID }}
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
      GIST_TOKEN: ${{ secrets.GIST_TOKEN }}
```

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/pgtap-docker-image.yaml
git commit -m "Migrate pgtap workflow onto shared workflow"
```

---

### Task 12: Migrate `postgresql/pgvector-docker-image.yaml`

- [ ] **Step 1: Replace the file**

```yaml
name: Build and Push pgvector Docker Image

on:
  push:
    branches: [main]
    paths:
      - 'pgvector/Dockerfile'
      - 'pgvector/initdb.sh'
      - '.github/workflows/pgvector-docker-image.yaml'
  pull_request:
    branches: [main]
    paths:
      - 'pgvector/Dockerfile'
      - 'pgvector/initdb.sh'
      - '.github/workflows/pgvector-docker-image.yaml'
  workflow_dispatch:
  schedule:
    - cron: '0 0 * * 0'

jobs:
  build-and-push:
    uses: SirSplat/workflows/.github/workflows/build-and-push-docker-image.yaml@main
    with:
      directory: ./pgvector
      image_name: postgresql
      tag_prefix: pgvector
      pg_major_matrix: '["13","14","15","16","17","18"]'
      default_pg_major: '18'
      build_args: |
        PG_VARIANT=alpine
      image_description: 'PostgreSQL on Alpine with pgvector installed and the dbo application role'
      push_on_pull_request: false
      status_gist_id: ${{ vars.STATUS_GIST_ID }}
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
      GIST_TOKEN: ${{ secrets.GIST_TOKEN }}
```

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/pgvector-docker-image.yaml
git commit -m "Migrate pgvector workflow onto shared workflow"
```

---

### Task 13: Verify all four migrated workflows on the branch

- [ ] **Step 1: Push the branch**

```bash
git push -u origin migrate-remaining-onto-shared-workflow
```

- [ ] **Step 2: Trigger each workflow**

```bash
for f in latest pgcrypto pgtap pgvector; do
  gh workflow run "${f}-docker-image.yaml" --ref migrate-remaining-onto-shared-workflow
done
```

- [ ] **Step 3: Wait and verify**

```bash
gh run list --branch migrate-remaining-onto-shared-workflow --limit 24
```

Expected: 24 matrix legs (4 workflows × 6 pg majors), all green. Spot-check pushed tags:

```bash
for variant in latest pgcrypto pgtap pgvector; do
  for pg in 13 14 15 16 17 18; do
    docker buildx imagetools inspect "<user>/postgresql:${variant}-pg${pg}" > /dev/null 2>&1 && echo "OK: ${variant}-pg${pg}" || echo "MISSING: ${variant}-pg${pg}"
  done
  docker buildx imagetools inspect "<user>/postgresql:${variant}" > /dev/null 2>&1 && echo "OK: ${variant} (default)" || echo "MISSING: ${variant} (default)"
done
```

Expected: all "OK".

- [ ] **Step 4: Open the migration PR**

```bash
gh pr create --title "Migrate latest, pgcrypto, pgtap, pgvector onto shared workflow" --body "$(cat <<'EOF'
## Summary

Completes the migration started in the dbo PR. Replaces 4 × ~150-line bespoke workflows with 4 × ~25-line callers of the shared workflow at `SirSplat/workflows`. ~600 net lines deleted from this repo.

## Verification

- [x] All 24 matrix legs (4 variants × 6 pg majors) green on the branch.
- [x] All 24 variant-pgN tags present on Docker Hub.
- [x] All 4 unsuffixed default tags (`latest`, `pgcrypto`, `pgtap`, `pgvector`) present.
- [x] Gist badges still updating per leg.

## Risk

Low. Same shared workflow that's been live powering the `dbo` workflow for one cycle.
EOF
)"
```

- [ ] **Step 5: Merge after review**

```bash
gh pr merge --squash --delete-branch
```

---

## Phase 4: Documentation

### Task 14: Update `CLAUDE.md` and `README.md` in the workflows repo

**Files (in `SirSplat/workflows`):**
- Modify: `CLAUDE.md`
- Modify: `README.md`

- [ ] **Step 1: Branch**

```bash
git checkout main && git pull
git checkout -b document-shared-workflow-extension
```

- [ ] **Step 2: Replace the `Workflows` section in `CLAUDE.md`**

Find the heading `### .github/workflows/build-and-push-docker-image.yaml` and the paragraph beneath it. Replace through the end of the consumer-call example with:

```markdown
### `.github/workflows/build-and-push-docker-image.yaml`

A reusable workflow that builds a multi-arch Docker image (linux/amd64, linux/arm64/v8) via Buildx + QEMU and pushes it to Docker Hub with provenance and SBOM attestations. Cache is stored on Docker Hub itself as a separate `:cache` tag on the same image repo.

**Two modes:**

1. **Single-tag mode** (default, used by `dvdrental`, `postgres`): caller passes `tag: <something>`, image is pushed as `<owner>/<image_name>:<tag>`.
2. **Matrix mode** (used by `postgresql/*`): caller passes `tag_prefix:` plus a JSON `pg_major_matrix:`. The workflow runs one job per `pg_major` and pushes `<owner>/<image_name>:<tag_prefix>-pg<N>` per leg, plus an unsuffixed `<owner>/<image_name>:<tag_prefix>` for the leg matching `default_pg_major`. `PG_MAJOR=<N>` is auto-injected into `build-args`.

Optional `status_gist_id` plus `GIST_TOKEN` secret enables shields.io-shaped gist status reporting per leg, with backoff+jitter retry to handle gist-API 409s under matrix concurrency.

**Inputs:** `directory`, `image_name`, `tag` (single-mode), `runner_debug`, `step_debug`, `pg_major_matrix`, `default_pg_major`, `tag_prefix`, `build_args`, `image_description`, `push_on_pull_request`, `status_gist_id`.

**Required secrets:** `DOCKER_USERNAME`, `DOCKER_PASSWORD`. **Optional:** `GIST_TOKEN`.

**Internal composite actions** (in `.github/actions/`, callable independently):
- `validate-dockerhub-creds` — pre-flight non-empty check on the Docker Hub creds.
- `verify-pushed-tags` — `docker buildx imagetools inspect` per pushed tag.
- `report-status-gist` — opt-in shields.io status badge updater with retry+jitter.

The reusable workflow references these via the cross-repo syntax `SirSplat/workflows/.github/actions/<name>@main`. Self-pinning at `@main` matches the consumer pinning convention; any internal change is atomic via merge to `main`.
```

- [ ] **Step 3: Add a matrix-mode example to `README.md`**

Append:

```markdown

## Caller examples

### Single-tag mode

```yaml
jobs:
  publish:
    uses: SirSplat/workflows/.github/workflows/build-and-push-docker-image.yaml@main
    with:
      directory: ./app
      image_name: my-image
      tag: ${{ github.sha }}
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

### Matrix mode (multi-version variant builds)

```yaml
jobs:
  publish:
    uses: SirSplat/workflows/.github/workflows/build-and-push-docker-image.yaml@main
    with:
      directory: ./dbo
      image_name: postgresql
      tag_prefix: dbo
      pg_major_matrix: '["13","14","15","16","17","18"]'
      default_pg_major: '18'
      build_args: |
        PG_VARIANT=alpine
      push_on_pull_request: false
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

This pushes `<user>/postgresql:dbo-pg13` … `dbo-pg18` plus `<user>/postgresql:dbo`.
```

- [ ] **Step 4: Commit and push**

```bash
git add CLAUDE.md README.md
git commit -m "Document the extended shared workflow's two modes and composite actions"
git push -u origin document-shared-workflow-extension
```

- [ ] **Step 5: Open and merge the docs PR**

```bash
gh pr create --title "Document extended shared workflow" --body "Docs follow-up to the workflow extension and postgresql migration. No code changes."
gh pr merge --squash --delete-branch
```

---

## Self-Review Notes

(Plan author's checklist, applied during writing.)

- **Spec coverage:** every requirement from the brief (extend workflow, three composite actions, postgresql migration, backwards compat for `dvdrental`/`postgres`, no public surface shrinkage) maps to a task. Build-arg matrix injection (Task 7a) was identified during plan writing — not in the original brief, but flagged with rationale.
- **Placeholder scan:** no "TBD", no "similar to Task N", no implicit edits. Each YAML block is the full intended file content (or the full intended replacement region).
- **Type/name consistency:** input names locked at top of plan and used identically throughout. Composite-action input names (`docker-username`, `docker-password`, `tags`, `gist-id`, etc.) match between Tasks 1–3 (definitions) and Task 5 (consumer). Branch names locked.
- **Known soft spots:**
  - Task 5's `image_description` always emits the OCI label, even with empty value. Acceptable noise. A conditional alternative would require an extra step.
  - The `verify-pushed-tags` step runs per matrix leg. With 6 legs × N tags, that's a few `imagetools inspect` calls per leg, but each is fast and they're parallel across legs. No optimisation needed.
  - The `actions/checkout@v6` step in the shared workflow is unused if no caller's repo content is touched (the build context comes from the *caller's* repo, which is checked out by default during a `workflow_call`). Leaving the step in place to match the pre-extension shape.

---

Plan complete and saved to `docs/superpowers/plans/2026-04-28-consolidate-postgresql-workflows.md`. Two execution options:

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration. Best when you want to review each composite action / each postgresql migration as it lands.

**2. Inline Execution** — I work the tasks in this session, batching where natural, checkpointing for review. Faster end-to-end, lighter on context-switching, but you see less granular progress.

Which approach?
