# CI/CD Design for Multi-Team Production-Critical Projects

A paradigm for projects with 5+ regular contributors, real users on the receiving end of regressions, multiple owners spanning multiple subsystems, and a release that ships software you can't easily un-ship.

If you're solo or pre-product, this is overkill — pick three principles and skip the rest. The point is to lift ideas that scale, not to copy the whole stack.

## 1. The seven principles

Each principle below trades short-term ergonomics for long-term defect cost. Pick them up in this order; later ones depend on earlier ones being honest.

### 1.1 Scoped before broad

**Principle:** A check that fires on every commit must be fast enough that nobody routes around it. A check that catches everything must exist somewhere, but doesn't have to be on the dev's keyboard.

**Example implementation:** Two-tier local commands.
- `pnpm check:changed` / `pnpm test:changed` — fires only against lanes derived from the git diff. Touches `extensions/plugin-a/*`? Only `extensions/plugin-a` lint + tests + boundary checks run.
- `pnpm check` / `pnpm test` — full suite. Slow. Rarely run by humans; reserved for CI and pre-release.

A small script (`scripts/changed-lanes.mjs`) reads `git diff --name-only` against the merge-base, runs a regex classifier (e.g., `core`, `extensions`, `apps`, `docs`, `tooling`), and emits the set of lanes the diff touches. Any change to `package.json`, lockfiles, or `tsconfig.json` triggers the `all` lane. Every guard script and test runner reads that lane set.

**Why:** A 90-second `check:changed` gets run before every commit. A 12-minute `check` does not. If you only have the latter, developers commit broken code and chase failures on CI. The fast lane has to feel like part of typing.

**Trade-off:** The lane classifier becomes load-bearing. Get the regexes wrong and broken changes slip through. Mitigate by treating the classifier itself as covered by tests and by erring toward "wider lane" when ambiguous.

### 1.2 Sandbox the blast radius

**Principle:** Anything that would burden a developer's machine — full test suite, Docker E2E, cross-OS build, live provider calls — runs in a sandbox, not locally. The sandbox is reproducible and parameterized.

**Example implementation:** A remote-runner service (AWS-backed or Blacksmith-backed) wraps any broad work. The dev invokes:

```sh
node scripts/sandbox-wrapper.mjs run --shell -- "pnpm check:changed"
```

The wrapper enforces an explicit env contract — e.g., `<PROJ>_TEST_PROJECTS_PARALLEL=6`, `<PROJ>_VITEST_MAX_WORKERS=1`, `NODE_OPTIONS=--max-old-space-size=4096`, `<PROJ>_VITEST_NO_OUTPUT_TIMEOUT_MS=900000`. Same params every time. Results post back to the PR.

**Why:** Without this, "I ran tests locally and they passed" becomes a meaningless statement — was it on the laptop with 64GB RAM and no parallelism, or in CI on a 4GB worker? Sandboxing makes local-vs-CI equivalence explicit.

Also: developers don't sacrifice their machine to run a full test suite. The fast lane runs locally, the broad lane runs in the cloud, and the dev never has to choose between "wait 10 minutes" or "skip the check."

**Trade-off:** Sandbox infra needs to exist. For smaller projects, this can be a GitHub Actions workflow that you trigger via `gh workflow run` from your terminal instead of a custom sandbox service.

### 1.3 Structured proof as a contract

**Principle:** Every PR that changes behavior must include evidence — not just "tests pass" but "I ran X command in Y environment and Z happened." The evidence has a parseable schema so a robot can verify it's there.

**Example implementation:** A `Real behavior proof` workflow (`.github/workflows/real-behavior-proof.yml`) parses PR body for these field labels:

```
Behavior addressed: ...
Real environment tested: ...
Exact steps or command run after this patch: ...
Evidence after fix: ...
Observed result after fix: ...
What was not tested: ...
```

A node script (`scripts/github/real-behavior-proof-policy.mjs`) reads the PR body with multi-name aliases, applies a battery of regex patterns to detect mock-only evidence vs live commands/screenshots/logs, and labels the PR `proof: supplied` / `proof: sufficient` / `triage: needs-real-behavior-proof`. Sufficient labels are required for merge.

**Why:** "Did you test this?" is unanswerable in code review without structured evidence. With this contract, the PR description IS the test report. The bot rejects PRs without it. Reviewers don't have to chase.

**Trade-off:** PR descriptions get longer. Some legitimate PRs (docs-only, internal refactors) need an override mechanism. Build that override in early — `proof: override` label applied by maintainers, with a written reason. Don't let the contract become busywork for trivial changes.

### 1.4 Bot-mediated review with a rubric

**Principle:** A review bot reads the PR body, the diff, and the codebase guidance docs (AGENTS.md or equivalent). It emits a structured verdict — rating + risk labels + findings — that the human reviewer either accepts or overrides. The bot is fast, consistent, and never tired.

**Example implementation:** A review bot lives in a separate repo and listens to GitHub events via `repository_dispatch`. For each PR it:

1. Reads root `AGENTS.md` / `CONTRIBUTING.md` (full file, not snippets) and any scoped guides in the touched paths.
2. Reads the PR diff, related issues, and previous reviews.
3. Emits a structured verdict comment with:
   - **Rating** on a multi-tier merge-readiness ladder (e.g., "ready" / "minor review" / "needs work" / "blocked"). Pick a scheme that's clear to your team.
   - **Risk labels:** `merge-risk: 🚨 <surface-name>` for surfaces that can corrupt state, block users, or break upgrades.
   - **Status labels:** `📣 needs proof`, `⏳ waiting on author`, `👀 ready for maintainer look`, `🚀 automerge armed`.
   - **Proof labels:** `proof: supplied`, `proof: sufficient`.
4. Re-reviews on `@bot re-review` comment or new commits.

The verdict comment is durable — same comment edited in place, not a new one per review.

**Why:** Manual review doesn't scale, but skipping review loses quality. A bot with a rubric gives every PR a structured first pass, and the rating is a clear signal for humans on what needs scrutiny.

**Trade-off:** You need an LLM with judgment + a fast feedback loop + clear rubric definitions. The rubric is the engineering work, not the LLM. A bad rubric makes the bot useless; a good one makes it indispensable.

### 1.5 Risk-labeled merge process

**Principle:** Priority (`P1`) is about blast radius if the bug exists. Merge-risk is about correctness confidence in the fix. They're orthogonal. Both should be visible on every PR, and merge-risk gates lifecycle.

**Example implementation:** Labels split into independent axes.
- **Priority:** `P1` / `P2` / `P3` — how much do users feel this if it ships broken
- **Merge risk:** `🚨 <surface-name>` — e.g., `🚨 session-state`, `🚨 availability`, `🚨 compatibility` — how easy is it for this fix to introduce a new regression

A PR can be `P1 / 🚨 session-state` (high-impact bug, fix touches risky surface — needs careful review even though urgency is high), or `P3 / no merge risk` (low-impact, safe fix — automerge candidate).

Your AGENTS.md / CONTRIBUTING.md should enumerate the merge-risk surfaces explicitly. Common ones: public APIs, persistent state boundaries, auth/identity, config/default schemas, migrations, fallback behavior, startup checks. Any change to those earns a merge-risk label *even with green CI*.

**Why:** Without this separation, P1 becomes the "merge faster" label and the most dangerous bugs get the least scrutiny. With it, P1 means "important to land" and 🚨 means "land carefully."

**Trade-off:** Adds a label decision per PR. Bot can suggest, human confirms.

### 1.6 Release pipeline gated by run IDs

**Principle:** A release is the output of a chain of validations, each one identified by a CI run ID. The publish job takes those run IDs as inputs and refuses to fire without them.

**Example implementation:** The release flow is:

```
preflight_run_id           ← passes npm-preflight (pack + install smoke)
full_release_validation    ← passes plugin-prerelease + release-checks + cross-OS + live-e2e
↓
release-publish.yml inputs: preflight_run_id + full_release_validation_run_id
↓
plugin npm publish → core npm publish → GitHub release → docs sync
```

Every step is `workflow_dispatch` only — no auto-publish. The `full-release-validation.yml` is a meta-workflow that dispatches and polls children (CI, plugin prerelease, release checks, install smoke, cross-OS, live E2E, Docker, QA, perf), supports rerun groups for partial reruns, and has release profiles (`beta` / `stable` / `full`) to control breadth.

A workable version scheme: `vYYYY.M.D-beta.N` for prerelease; npm dist-tag must match. See `04-release-pipeline.md` for alternatives.

**Why:** Releases are the highest-stakes operation. Manual triggering + run-ID gating means the publisher (human or bot) is explicitly attesting "I checked these specific validations passed for this specific code." No "the last run looked green" — only "this run ID 12345 passed."

**Trade-off:** Adds friction to release day. That's the point. Releases that are easy to do are easy to do wrong.

### 1.7 Concurrency that respects event types

**Principle:** Different GitHub event types deserve different concurrency policies. Aggressively cancel routine PR sync events. Don't cancel scheduled or security scans.

**Example implementation:** Every workflow has an explicit `concurrency` block keyed by purpose.

```yaml
# Routine PR work: cancel older runs aggressively
concurrency:
  group: ci-${{ github.event.pull_request.number || github.sha }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

# Bot dispatch: debounce labeled/unlabeled events
concurrency:
  group: review-bot-${{ github.event.pull_request.number }}
  cancel-in-progress: true

# Security scan: never cancel
concurrency:
  group: codeql-${{ github.run_id }}
  cancel-in-progress: false
```

Plus: the review bot's dispatch workflow debounces labeled/unlabeled with a short sleep at the start (e.g., 20 seconds), so a burst of label flips doesn't trigger 5 reviews.

**Why:** Without this, every PR sync queues a fresh CI run while the old one is still running. Wastes minutes. Hides results. With it, only the most recent state runs. With the debounce, label storms don't multiply work.

**Trade-off:** Slightly slower feedback on edited PRs (the cancelled run has to restart). Worth it.

## 2. Anti-patterns to avoid

1. **One giant CI job.** Sharding into a fast lane + broad lane + release lane is non-negotiable. The fast lane has to fit in the developer's attention span; the broad lane has to be honest.
2. **Required checks that are flaky.** A flaky required check teaches developers to retry until green. Either fix the flake or make the check advisory.
3. **PR descriptions as free text.** Without a parsed schema, you can't enforce evidence. Build the schema before the volume of PRs becomes unmanageable.
4. **Release as `git push`.** Releases are an output of validation, not a step in development. Gate them on run IDs, not on someone clicking a button on Friday.
5. **Merging fail-open changes silently.** Anything that weakens a safety net (removes a fallback, relaxes validation, adds a config compat shim) needs an explicit risk label, even if CI is green.
6. **Reviewing the diff in isolation.** The bot rubric should require reviewing sibling surfaces — if your fix touches one code path, what does the equivalent sibling path look like elsewhere? One-sided fixes need sibling-surface proof.
7. **Force-pushing other people's branches without `Allow edits by maintainers`.** Both parties need to consent to the takeover model upfront, in writing.

## 3. Adaptation guide — when to skip each principle

| Principle | Skip if |
|---|---|
| Scoped before broad | Total test suite runs under 90 seconds |
| Sandbox blast radius | You don't have a sandbox infra to point at — start with GitHub Actions runners, add sandbox layer when local-vs-CI drift becomes a recurring issue |
| Structured proof | < 3 active contributors and you trust each other's "tested locally" claims |
| Bot-mediated review | No LLM budget or no clear rubric — but at least have a written CONTRIBUTING.md or AGENTS.md so future-you (or future-bot) can read the same context |
| Risk-labeled merge | < 1 incident per quarter from bad merges — until then, P1/P2/P3 alone is fine |
| Run-ID-gated release | Releases happen < monthly and you can manually verify each one |
| Event-type concurrency | CI runs cost < $50/month — but adding `concurrency:` blocks is one line each, just do it |

## 4. Implementation order

If starting fresh, build in this order:

1. `scripts/changed-lanes.mjs` and the `check:changed` / `test:changed` commands. Get the fast lane right first.
2. Two-tier CI: `ci.yml` with a preflight job + sharded matrix; opt out per-shard via the changed-paths classifier.
3. PR template with the Real behavior proof schema. Parse it in a workflow.
4. Concurrency blocks on every workflow.
5. Labels: priority + risk + status + proof axes.
6. Review automation (bot or human-driven rubric).
7. Release pipeline with run-ID gates.
8. Sandbox layer for broad gates if/when local-vs-CI drift bites.

Each layer assumes the previous ones exist. Don't add structured proof before you have lanes; don't add bot review before you have labels.

## See also

- `01-pr-template.md` — copy-pasteable PR description with Real behavior proof schema
- `02-ci-workflow-scaffold.md` — concrete YAML tier examples + sharding pattern
- `03-review-automation.md` — label rubric + bot integration contract
- `04-release-pipeline.md` — version scheme + gated publish flow
- `05-maintainer-ops.md` — force-push rebase pattern + pre-merge validation report + automerge triggers
