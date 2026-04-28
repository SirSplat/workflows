# workflows

Shared GitHub Actions reusable workflows and composite actions used across `SirSplat/*` repositories.

## What's here

- **`build-and-push-docker-image.yaml`** — multi-arch Docker image build + push to Docker Hub, with optional matrix mode for variant builds (used by `SirSplat/postgresql`'s five PostgreSQL variants).
- Three composite actions in `.github/actions/`:
  - `validate-dockerhub-creds` — pre-flight check on Docker Hub credentials
  - `verify-pushed-tags` — confirms each pushed tag resolves via `docker buildx imagetools inspect`
  - `report-status-gist` — opt-in shields.io status badge updater with retry+jitter

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

This pushes `<user>/my-image:<commit-sha>` and uses `<user>/my-image:cache` for build cache.

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

This pushes `<user>/postgresql:dbo-pg13` … `dbo-pg18` plus the unsuffixed `<user>/postgresql:dbo` (from the `default_pg_major: 18` leg). Cache lane is `<user>/postgresql:cache-dbo`. `PG_MAJOR=<value>` is auto-injected into `build-args` per leg — do not add it manually.

`push_on_pull_request: false` is the postgresql convention: PR-event runs build the image to verify but don't push. The build cache is still warmed.

### With status badge reporting

Add to either mode:

```yaml
    with:
      ...
      status_gist_id: ${{ vars.STATUS_GIST_ID }}
    secrets:
      ...
      GIST_TOKEN: ${{ secrets.GIST_TOKEN }}
```

The action PATCHes a shields.io-shaped JSON file inside the gist named `<tag_prefix>-pg<N>.json` (matrix mode) or `<tag>.json` (single-tag mode). Set up the gist once, populate `STATUS_GIST_ID` repo variable and `GIST_TOKEN` repo secret, and badges update automatically.

## Inputs reference

| Input | Required | Default | Purpose |
|---|---|---|---|
| `directory` | yes | — | Path to the Dockerfile context (build context root). |
| `image_name` | yes | — | Image name without `<owner>/` prefix. |
| `tag` | no | `''` | Single-tag mode push tag. Ignored if `tag_prefix` is set. |
| `runner_debug` | no | `false` | Sets `ACTIONS_RUNNER_DEBUG=true`. |
| `step_debug` | no | `false` | Sets `ACTIONS_STEP_DEBUG=true`. |
| `pg_major_matrix` | no | `'[""]'` | JSON array driving an internal `pg_major` matrix axis. Default `'[""]'` = single non-pg leg = legacy behaviour. |
| `default_pg_major` | no | `''` | In matrix mode, which leg also gets the unsuffixed `tag_prefix` tag (e.g. `dbo` alongside `dbo-pg18`). |
| `tag_prefix` | no | `''` | Switches to multi-tag matrix mode driven by `metadata-action`. |
| `build_args` | no | `''` | Multiline build-args string. `PG_MAJOR=<value>` is auto-injected per leg in matrix mode. |
| `image_description` | no | `''` | Sets `org.opencontainers.image.description` label. |
| `push_on_pull_request` | no | `true` | When `false`, PR-event runs build but skip the image push. Login + cache push still happen. |
| `status_gist_id` | no | `''` | Empty disables shields.io status reporting. |

## Secrets reference

| Secret | Required | Purpose |
|---|---|---|
| `DOCKER_USERNAME` | yes | Docker Hub username. |
| `DOCKER_PASSWORD` | yes | Docker Hub password or PAT with `read,write` scope on the target image repository. |
| `GIST_TOKEN` | no | Required only when `status_gist_id` is set. PAT with `gist` scope. |

## Validation

No local test framework — multi-arch buildx + QEMU + registry push is awkward to reproduce locally. Changes are validated either with `actionlint` static analysis or by triggering a real CI run on a downstream consumer.

## License

MIT — see `LICENSE`.
