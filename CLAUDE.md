# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

This repo hosts **reusable GitHub Actions workflows** that other repositories call via `workflow_call`. There is no application code, no test suite, and no local build — changes are validated by consumer repos that reference these workflows.

## Workflows

### `.github/workflows/build-and-push-docker-image.yaml`

A reusable workflow that builds a multi-arch Docker image (linux/amd64, linux/arm64/v8) via Buildx + QEMU and pushes it to Docker Hub with provenance and SBOM attestations. It uses Docker Hub itself as the build cache backend (`type=registry,ref=<user>/<image>:cache`) — the cache is stored as a separate `:cache` tag on the same repo.

Inputs: `directory` (Dockerfile context), `image_name`, `tag`, plus optional `runner_debug` / `step_debug` toggles that flip `ACTIONS_RUNNER_DEBUG` / `ACTIONS_STEP_DEBUG`.
Required secrets: `DOCKER_USERNAME`, `DOCKER_PASSWORD`.

Consumers call it like:

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

## Editing notes

- Consumers pin by ref (branch, tag, or SHA). Treat `main` as published — breaking changes to inputs/secrets will break every caller silently until they update. Prefer additive changes; if a breaking change is needed, cut a new tag and let consumers migrate.
- Recent history shows the cache configuration was iterated on several times (`fc72471`, `1266524`, `4ad2b36`, `960f4f2`). When touching `cache-from` / `cache-to`, remember the cache lives in the same Docker Hub repo as the image — this is intentional, not a stray reference.
- The image tag passed into `docker/metadata-action` is `${{ inputs.image_name }}:${{ inputs.tag }}` (no namespace), but the actual push tag is `${{ secrets.DOCKER_USERNAME }}/${{ inputs.image_name }}:${{ inputs.tag }}`. Keep these consistent if you refactor.

## Validation

There is no way to run this workflow locally. Validate changes by:
1. Pushing to a branch and having a consumer repo reference `SirSplat/workflows/.github/workflows/<file>@<branch>`.
2. Or using `act` / similar tooling against a sample caller — but multi-arch buildx + QEMU + registry push is awkward to reproduce locally, so a real run on a fork is usually faster.
