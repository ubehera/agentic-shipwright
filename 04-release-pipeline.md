# Release Pipeline

A release is the **output** of a chain of validations, identified by CI run IDs. The publish job takes those run IDs as inputs and refuses to fire without them. Nothing here is auto-triggered; releases are always intentional human dispatches.

## Version scheme

One workable choice — date-based prerelease scheme:

```
vYYYY.M.D-beta.N    prerelease (npm dist-tag: beta)
vYYYY.M.D-alpha.N   alpha (npm dist-tag: alpha)
vYYYY.M.D           stable (npm dist-tag: latest)
```

Why date-based:
- Eyeballs immediately know which release is newer
- No semver argument about whether a fix is patch or minor
- Multiple releases per day are sortable (`.N` increments)
- Stable releases align to a publish date, not a feature

If you need semver (for npm package consumers who depend on semver-range), keep semver for the package itself but tag releases with the date-based scheme in git.

## The validation chain

```
            ┌─────────────────────┐
            │ feature PRs land    │
            │ to main             │
            └──────────┬──────────┘
                       ▼
            ┌─────────────────────┐
            │ preflight workflow  │   triggered on main pushes
            │ (npm pack, install) │   produces: preflight_run_id
            └──────────┬──────────┘
                       ▼
            ┌─────────────────────┐
            │ full-release-       │   manual dispatch with profile=beta|stable|full
            │ validation.yml      │   produces: validation_run_id
            │ (orchestrator)      │
            └──────────┬──────────┘
                       │
            ┌──────────┴──────────┐
            ▼          ▼          ▼
    cross-os      live-e2e    package-acceptance
    (matrix)      (live deps)  (npm pack + install fresh)
            │          │          │
            └──────────┬──────────┘
                       ▼
            ┌─────────────────────┐
            │ release-publish.yml │   manual dispatch with both run IDs
            │ inputs: ref +       │
            │   preflight_run_id  │
            │   validation_run_id │
            └──────────┬──────────┘
                       │
            ┌──────────┴────────────────────┐
            ▼          ▼          ▼         ▼
        plugin       core        Docker    GitHub
        npm pub      npm pub     pub       release
                                  │         │
                                  └────┬────┘
                                       ▼
                                  docs sync
```

## The orchestrator: `full-release-validation.yml`

```yaml
name: Full release validation

on:
  workflow_dispatch:
    inputs:
      ref:
        description: 'Git ref'
        required: true
      profile:
        type: choice
        options: [beta, stable, full]
        default: beta
      rerun_groups:
        description: 'Optional: comma-separated rerun groups (e.g., "cross-os,live-e2e")'

jobs:
  preflight:
    if: ${{ !contains(inputs.rerun_groups, ',') || contains(inputs.rerun_groups, 'preflight') }}
    runs-on: ubuntu-latest
    outputs:
      preflight_run_id: ${{ github.run_id }}
    # ... npm pack + install smoke ...

  cross-os:
    needs: preflight
    if: ${{ inputs.profile != 'beta' || contains(inputs.rerun_groups, 'cross-os') }}
    uses: ./.github/workflows/cross-os-release-checks-reusable.yml
    with:
      ref: ${{ inputs.ref }}
      mode: fresh

  live-e2e:
    needs: preflight
    if: ${{ inputs.profile == 'full' || contains(inputs.rerun_groups, 'live-e2e') }}
    uses: ./.github/workflows/live-and-e2e-checks-reusable.yml
    with:
      ref: ${{ inputs.ref }}

  package-acceptance:
    needs: preflight
    # ... install package from npm pack output, run sanity tests ...

  docker-release-check:
    if: ${{ inputs.profile == 'full' }}
    # ... build Docker image, run health check ...

  release-checks:
    needs: [cross-os, live-e2e, package-acceptance]
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "All required validations passed for ${{ inputs.ref }}"
          echo "Use this run ID in release-publish.yml: ${{ github.run_id }}"
```

Key design points:
- **`profile` input** controls breadth. Beta does the minimum; stable adds cross-OS; full adds Docker + perf.
- **`rerun_groups` input** allows partial reruns. If `live-e2e` flaked, just rerun that subgroup without redoing everything.
- **Final job prints the run ID** to the workflow summary so the publisher can copy-paste it into `release-publish.yml`.

## The publisher: `release-publish.yml`

See `02-ci-workflow-scaffold.md` for the basic shape. Key elaborations:

### Verifying upstream run IDs

```yaml
verify-inputs:
  runs-on: ubuntu-latest
  steps:
    - name: Verify preflight passed and matches ref
      run: |
        run_data=$(gh run view ${{ inputs.preflight_run_id }} \
          --repo ${{ github.repository }} \
          --json conclusion,headBranch,headSha)
        conclusion=$(echo "$run_data" | jq -r '.conclusion')
        head_sha=$(echo "$run_data" | jq -r '.headSha')

        [ "$conclusion" = "success" ] || { echo "preflight failed"; exit 1; }

        ref_sha=$(git rev-parse "${{ inputs.ref }}")
        [ "$head_sha" = "$ref_sha" ] || { echo "preflight ran against $head_sha, but publishing $ref_sha"; exit 1; }
      env:
        GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

The SHA match check is the important one — a maintainer could otherwise paste a run ID from an older commit that passed, even though the current `ref` has new commits the validation didn't see.

### Publishing in dependency order

For a monorepo with plugins + core:

```yaml
publish-plugins:
  needs: verify-inputs
  strategy:
    matrix: { plugin: [plugin-a, plugin-b, plugin-c, ...] }
  runs-on: ubuntu-latest
  steps:
    - name: Publish ${{ matrix.plugin }} to npm
      run: cd extensions/${{ matrix.plugin }} && npm publish --tag ${{ inputs.profile == 'stable' && 'latest' || inputs.profile }}

publish-core:
  needs: publish-plugins   # plugins first, core depends on them at install time
  runs-on: ubuntu-latest
  steps:
    - run: npm publish --tag ${{ inputs.profile == 'stable' && 'latest' || inputs.profile }}

create-release:
  needs: publish-core
  runs-on: ubuntu-latest
  steps:
    - name: Generate release notes
      run: node scripts/release/notes.mjs --since-tag <last-tag> > notes.md
    - name: Create GitHub release
      run: |
        gh release create v${{ inputs.version }} \
          --title "v${{ inputs.version }}" \
          --notes-file notes.md \
          --prerelease=${{ inputs.profile != 'stable' }}

docs-sync:
  needs: create-release
  if: ${{ inputs.profile == 'stable' }}
  runs-on: ubuntu-latest
  steps:
    - name: Trigger docs sync workflow in docs repo
      run: gh workflow run docs-sync --repo your-org/docs --ref main
```

## What gates a release

A release is allowed to ship if:

1. `preflight_run_id` exists, conclusion=success, head SHA matches `ref`
2. `full_validation_run_id` exists, conclusion=success, head SHA matches `ref`
3. The publisher has explicit team membership (gated by `if: github.actor == 'release-publisher-team'` or branch protection)
4. The ref is in an allowed pattern (`main`, `release/YYYY.M.D`, or `prerelease/*`)
5. The CHANGELOG entry exists for the active version

If any of these fail, the publish doesn't fire.

## Prerelease vs stable

| Profile | Preflight | Cross-OS | Live E2E | Docker | Perf | npm tag | GitHub release |
|---|---|---|---|---|---|---|---|
| `beta` | ✓ | — | — | — | — | `beta` | prerelease |
| `stable` | ✓ | ✓ | — | — | — | `latest` | release |
| `full` | ✓ | ✓ | ✓ | ✓ | ✓ (advisory) | `latest` | release |

Beta releases run weekly off `main` for early-adopter users; stable releases run when a feature is ready to land for everyone; full release is the curated quarterly checkpoint that exercises everything.

## CHANGELOG discipline

- Active version section sits at the top of `CHANGELOG.md` under `## Unreleased` or `## vNEXT`.
- Single-line bullets only under `### Fixes` / `### Changes` / `### Features`.
- Contributor PRs don't edit CHANGELOG; the maintainer adds the entry during landing. This avoids merge conflicts when many PRs land same day.
- After release, the active section becomes the previous version section; a new `## Unreleased` is created.

## Common release-day failure modes

| Failure | Mitigation |
|---|---|
| Old run ID used after force-push | Verify SHA matches in `verify-inputs` |
| Plugin published, core fails — half-released | Use `needs: publish-plugins` and a final dry-run before publish; consider reverting via `npm deprecate` if half-published |
| Docs sync runs before release notes — docs stale | `docs-sync` requires `needs: create-release` |
| Wrong npm dist-tag — beta tag overwrites latest | Always set explicit `--tag`; never default to `latest` for non-stable profiles |
| CHANGELOG entry missing | Workflow grep checks `CHANGELOG.md` head for `### ` + active-version entry |
| Release notes generated from wrong base | Pass `--since-tag` explicitly; don't auto-detect |
