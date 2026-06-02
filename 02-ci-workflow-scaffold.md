# CI Workflow Scaffold

A tier model for production-critical projects. Three layers; each layer answers a different question.

## Tier 1: Fast lane (PR-blocking, ≤ 5 min)

Question it answers: *"Does this commit obviously break something?"*

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: ci-${{ github.event.pull_request.number || github.sha }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  preflight:
    runs-on: ubuntu-latest
    outputs:
      lanes: ${{ steps.lanes.outputs.lanes }}
      matrix: ${{ steps.matrix.outputs.matrix }}
    steps:
      - uses: actions/checkout@v6
        with: { fetch-depth: 0 }
      - uses: actions/setup-node@v5
        with: { node-version: '22' }
      - id: lanes
        run: |
          # Emit: lanes={"core":true,"extensions":false,"docs":false,...}
          node scripts/changed-lanes.mjs --json >> lanes.json
          echo "lanes=$(cat lanes.json)" >> "$GITHUB_OUTPUT"
      - id: matrix
        run: |
          # Compute shard matrix from lanes. Each shard gets its own job.
          node scripts/compute-ci-matrix.mjs --lanes lanes.json >> matrix.json
          echo "matrix=$(cat matrix.json)" >> "$GITHUB_OUTPUT"

  check-guards:
    needs: preflight
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v5
        with: { node-version: '22' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm check:no-conflict-markers
      - run: pnpm check:dependency-pins
      - run: pnpm check:import-cycles

  check-types:
    needs: preflight
    if: ${{ fromJSON(needs.preflight.outputs.lanes).core || fromJSON(needs.preflight.outputs.lanes).all }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v5
        with: { node-version: '22' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm check:types

  check-lint:
    needs: preflight
    runs-on: ubuntu-latest
    strategy:
      matrix: ${{ fromJSON(needs.preflight.outputs.matrix).lint }}
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v5
        with: { node-version: '22' }
      - run: pnpm install --frozen-lockfile
      - run: node scripts/run-lint-shard.mjs --shard ${{ matrix.shard }}

  tests-fast:
    needs: preflight
    runs-on: ubuntu-latest
    strategy:
      matrix: ${{ fromJSON(needs.preflight.outputs.matrix).testsFast }}
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v5
        with: { node-version: '22' }
      - run: pnpm install --frozen-lockfile
      - run: node scripts/run-tests.mjs ${{ matrix.testPaths }}
        env:
          # Prefix env vars with your project name (e.g., MYPROJ_VITEST_MAX_WORKERS)
          # so devs reproduce the same parallelism locally
          PROJECT_VITEST_MAX_WORKERS: 2
          NODE_OPTIONS: --max-old-space-size=4096
```

Notes:
- **Preflight job** computes the matrix dynamically from changed lanes. Avoids static matrix limits and skips lanes the PR doesn't touch.
- **`fetch-depth: 0`** on preflight so the lane classifier can diff against the merge base.
- **Concurrency block** keys on PR number (or SHA for pushes). Cancels older runs on edit/synchronize.
- **Permissions are read-only** except where explicitly needed. PR bot workflows separately escalate to `pull-requests: write`.

## Tier 2: Broad lane (PR-blocking, ≤ 20 min, sandboxed)

Question it answers: *"If I land this, does anything across the org break?"*

```yaml
# .github/workflows/ci-broad-sandbox.yml
name: CI broad gate (sandbox)

on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
  workflow_dispatch:

permissions:
  contents: read
  pull-requests: write

concurrency:
  group: ci-broad-${{ github.event.pull_request.number || github.sha }}
  cancel-in-progress: true

jobs:
  broad:
    runs-on: ubuntu-latest
    if: ${{ !github.event.pull_request.draft }}
    steps:
      - uses: actions/checkout@v6
      - name: Acquire sandbox runner
        # Example: acquire an ephemeral runner from your sandbox provider
        # (Blacksmith, AWS, fly.io, a self-hosted runner pool, etc.).
        uses: your-sandbox-provider/acquire-runner@v1
        with:
          provider: <your-runner-provider>
          idle-timeout: 90m
      - run: |
          env CI=1 \
            NODE_OPTIONS=--max-old-space-size=4096 \
            PROJECT_TEST_PROJECTS_PARALLEL=6 \
            PROJECT_VITEST_MAX_WORKERS=1 \
            PROJECT_VITEST_NO_OUTPUT_TIMEOUT_MS=900000 \
            pnpm check
```

Notes:
- **Sandboxed runner** so the broad job doesn't compete for shared GitHub Actions runners.
- **Skip on draft PRs** to avoid spending sandbox time on work-in-progress.
- **Explicit env contract** matches what dev sees when they run via a local sandbox wrapper script.

## Tier 3: Release lane (manual dispatch, run-ID-gated)

Question it answers: *"Is this code safe to ship to users?"*

```yaml
# .github/workflows/release-publish.yml
name: Release publish

on:
  workflow_dispatch:
    inputs:
      ref:
        description: 'Git ref to release'
        required: true
      preflight_run_id:
        description: 'Run ID of the npm-preflight workflow (must have passed)'
        required: true
      full_validation_run_id:
        description: 'Run ID of the full-release-validation workflow (must have passed)'
        required: true
      release_profile:
        type: choice
        options: [beta, stable, full]
        default: beta

permissions:
  contents: write
  packages: write

jobs:
  verify-inputs:
    runs-on: ubuntu-latest
    steps:
      - name: Verify preflight passed
        run: |
          conclusion=$(gh run view ${{ inputs.preflight_run_id }} \
            --repo ${{ github.repository }} \
            --json conclusion --jq '.conclusion')
          [ "$conclusion" = "success" ] || { echo "preflight failed"; exit 1; }
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      - name: Verify full validation passed
        run: |
          conclusion=$(gh run view ${{ inputs.full_validation_run_id }} \
            --repo ${{ github.repository }} \
            --json conclusion --jq '.conclusion')
          [ "$conclusion" = "success" ] || { echo "full validation failed"; exit 1; }
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  publish:
    needs: verify-inputs
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with: { ref: ${{ inputs.ref }} }
      # ... build + publish to npm with explicit dist-tag matching version scheme
```

Notes:
- **Inputs include run IDs** of upstream validations. The job verifies they passed before publishing.
- **No `pull_request` or `push` trigger** — publishing only happens on intentional human action.
- **Release profile choice** lets you do beta/stable/full publishes from the same workflow.

## Reusable workflows for complex multi-stage jobs

If you have a complex matrix (cross-OS, multi-provider, multi-platform), extract a reusable workflow:

```yaml
# .github/workflows/cross-os-release-checks-reusable.yml
name: Cross-OS release checks (reusable)

on:
  workflow_call:
    inputs:
      ref:
        type: string
        required: true
      mode:
        type: string
        description: 'fresh | upgrade'
        default: fresh

jobs:
  matrix:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    # ...
```

Called from `release-publish.yml`:

```yaml
  cross-os:
    uses: ./.github/workflows/cross-os-release-checks-reusable.yml
    with:
      ref: ${{ inputs.ref }}
      mode: fresh
```

## Bot / label automation workflows

Should be their own files, run on `pull_request_target` (so they can write labels even from forked PRs), and have tight permissions:

```yaml
# .github/workflows/labeler.yml
name: Labeler

on:
  pull_request_target:
    types: [opened, synchronize, reopened, edited]

permissions:
  contents: read
  pull-requests: write

concurrency:
  group: labeler-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  label:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/labeler@v6
        with:
          configuration-path: .github/labeler.yml
          sync-labels: true
```

Path-to-label config (`.github/labeler.yml`):

```yaml
# Each label maps to file paths. Adds the label when any matching file changes.
core:
  - changed-files:
    - any-glob-to-any-file: src/core/**

extensions:
  - changed-files:
    - any-glob-to-any-file: extensions/**

docs:
  - changed-files:
    - any-glob-to-any-file:
      - docs/**
      - '*.md'

# Area-specific
'area: plugin-a':
  - changed-files:
    - any-glob-to-any-file: extensions/plugin-a/**

'area: plugin-b':
  - changed-files:
    - any-glob-to-any-file: extensions/plugin-b/**

# Size labels — separate workflow that runs a script
```

## Concurrency cheatsheet

```yaml
# Routine PR work — supersede aggressively
concurrency:
  group: ci-${{ github.event.pull_request.number || github.sha }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

# Metadata events (labels, comments) — supersede + debounce in the workflow itself
concurrency:
  group: labeler-${{ github.event.pull_request.number }}
  cancel-in-progress: true

# Security / compliance — never cancel
concurrency:
  group: security-${{ github.run_id }}
  cancel-in-progress: false

# Release publish — serialize per ref
concurrency:
  group: release-publish-${{ inputs.ref }}
  cancel-in-progress: false
```

## What NOT to do

- **Required check that runs on a schedule, not on PRs.** Scheduled CI catches drift in `main`, not in your PR. Don't gate merge on scheduled jobs.
- **Matrix with `fail-fast: true` for important shards.** First failure cancels the rest; you don't learn what else is broken until the next push.
- **Pinning `actions/checkout@main`.** Pin to a major version tag at minimum (`@v6`), ideally to a SHA, so a bad actions release can't break your CI overnight.
- **`secrets.GITHUB_TOKEN` with default `permissions: write-all`.** Set explicit minimal permissions per workflow.
- **Long-running E2E in the fast lane.** That's what the broad lane is for. Move it.
