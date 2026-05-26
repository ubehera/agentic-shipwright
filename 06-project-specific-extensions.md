# Project-Specific Extensions — Build Your Own

Five extensions sit outside the core template because they're project-shape-specific: live QA harness, sandbox runner infrastructure, review bot, public-SDK compat contract, and area-owner routing. Each solves a real problem you'll eventually hit if your project scales — this doc covers how to build each one for your context.

The pattern across all five: start with a manual, human-driven version. Automate only what becomes painful. Don't pre-build infrastructure for problems you don't have yet.

## 1. Live QA harness

**Problem it solves:** Some behavior can only be proven in a real environment with real external systems — pushing to a real Slack channel, calling a real provider API, talking to a real database. Unit tests can mock these; only a live run proves end-to-end correctness.

**When you need this:** When you have at least one of:
- Customer-facing integrations (channels, payment processors, third-party APIs)
- Platform-specific bugs that only repro on real hardware (mobile, native UI, OS-specific behavior)
- A "reproduction shape" that's hard to fake (large data sets, specific account permissions, time-of-day behavior)

**Build pattern:**

1. **Define the smallest live-bound test that proves your most-common regression class.** For a messaging platform: "send a message via $CHANNEL, verify it arrived in the right thread." For a payments project: "create a $0.01 charge in test mode, verify webhook fires." For an inference service: "send a known prompt, verify response signature matches the reference."

2. **Wrap it in a `workflow_dispatch` workflow.** No automatic triggers — live tests cost real money / real API quota / real time.
   ```yaml
   on:
     workflow_dispatch:
       inputs:
         scenario: { type: choice, options: [smoke, channel-roundtrip, ...] }
         ref: { type: string, default: main }
   ```

3. **Output structured artifacts.** Screenshots, transcripts, logs uploaded as workflow artifacts. The PR comment that triggers the run can link back to the artifact URL.

4. **Trigger from PR comments.** A bot watches for `/qa <scenario>` comments on PRs with write-access authors, dispatches the workflow with the PR's HEAD ref, and posts the artifact link back as a comment.

5. **Build a panel of scenarios over time.** Each new regression class adds a scenario. Run them all before stable releases (call them from `full-release-validation.yml` per `04-release-pipeline.md`).

**Minimum viable version:** A `Makefile` target your team runs manually pre-release. Output goes in a shared drive. Promote to automation when the manual version is taking too much time.

**Anti-patterns:**
- Running live tests on every PR (cost, quota, flakiness)
- Letting live tests gate routine merges (one external system outage shouldn't block your team)
- Live tests with no observable artifact (then you can't audit "did it actually run?")

## 2. Sandbox runner infrastructure

**Problem it solves:** Broad test gates burden developer machines. Routing them to a reproducible remote runner with explicit env contract makes "I tested it locally" and "CI tested it" comparable.

**When you need this:** When you have at least one of:
- Test suite > 5 minutes on a developer machine
- Memory pressure issues that drive flakes (Node heap OOMs, parallel-worker contention)
- Cross-OS validation (Windows/macOS/Linux) — you can't run all three locally
- Tests that need specific hardware (GPU, more RAM, faster disk)

**Build pattern, in order of increasing cost:**

### Level 1: GitHub Actions runners as your sandbox

```yaml
jobs:
  broad:
    runs-on: ubuntu-latest-large  # or use specific compute via GitHub-hosted runner tiers
    env:
      NODE_OPTIONS: --max-old-space-size=4096
      TEST_PROJECTS_PARALLEL: 6
      VITEST_MAX_WORKERS: 1
    steps:
      - ...
```

For most projects this is enough. GitHub-hosted runners are reproducible (same image, same hardware tier). Set explicit `env:` so the dev can reproduce locally by exporting the same vars.

### Level 2: Self-hosted runners

When GitHub-hosted is too slow or too small, run your own runners on cheap VMs (Hetzner, OVH, Oracle Free, etc.). GitHub publishes the runner agent; you install it on a VM, register with your repo. Same workflow YAML, different `runs-on:` label.

Trade-off: you maintain the VM (security patches, runner upgrades, monitoring). Worth it when you save > 1 hour/day of CI time across the team.

### Level 3: Custom remote-runner service

A separate service that:
- Provisions ephemeral runners on demand (AWS, GCP, fly.io, etc.)
- Exposes a CLI wrapper your devs call: `mybox run --shell -- "pnpm check"`
- Streams output back to the dev's terminal in real time
- Posts artifacts/results to a structured URL the PR can link

Only build this when self-hosted runners can't handle your scale or when you need on-demand resources (e.g., GPU machines that idle expensive when always-on).

**The non-negotiable bit:** **explicit env contract**. Whatever sandbox you pick, lock down the env vars (`NODE_OPTIONS`, parallelism settings, memory limits) so "ran on sandbox" and "ran locally" produce the same shape of result. Without this, the sandbox is just "another environment to debug."

**Anti-patterns:**
- Sandbox with different versions of Node/Python/etc. than developers use
- Sandbox with no time limit (jobs hang forever, burn money)
- Sandbox results not posted back to the PR (then nobody trusts them)

## 3. Review bot

**Problem it solves:** Manual review doesn't scale. A bot that reads the PR + your AGENTS.md + the diff + the touched code paths and emits a structured first-pass verdict gives every PR a baseline review with consistent rubric application.

**When you need this:** When you have at least one of:
- > 10 PRs/week and reviewer attention is a bottleneck
- A complex codebase where reviewers can't hold all the conventions in their head
- Multiple maintainers and you want consistent review quality across them
- A pattern of "I missed this in review" incidents

**Build pattern, three architectural options:**

### Option A: Inline GitHub Action

Simplest. Runs the LLM call inside a GitHub Action workflow on every PR sync.

```yaml
on:
  pull_request_target:
    types: [opened, synchronize, reopened]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - run: |
          node scripts/review-bot.mjs \
            --pr ${{ github.event.pull_request.number }} \
            --rubric .github/review-rubric.md
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

The `review-bot.mjs` script:
1. Fetches PR diff + body + linked issues via gh CLI
2. Loads AGENTS.md (full file) and any scoped AGENTS.md for touched paths
3. Calls the LLM with rubric + context + diff
4. Parses the LLM's structured output (JSON-mode preferred)
5. Posts/updates the durable comment + applies labels via gh CLI

Cheap to start. Limit: LLM call latency adds to every PR sync. Acceptable for projects with < 50 PRs/day.

### Option B: Webhook-driven separate service

For higher volume or more complex review behavior. A service subscribes to GitHub webhooks, queues reviews, runs them with controlled concurrency, posts results back.

Architecture:
- GitHub App with PR events webhook → your service endpoint
- Job queue (Redis, SQS, etc.) for review tasks
- Workers pick up jobs, call LLM, post results
- Durable-comment manager (same logic as Option A, just in the service)

More moving parts but lets you decouple review compute from GitHub Actions billing and add features like cross-PR analysis ("this PR's pattern matches the one that broke prod last month").

### Option C: Hybrid — Action triggers a queue

GitHub Action just queues the job; an external worker does the review. Lower latency than inline (Action returns immediately) but you still need the queue + worker infra.

### The rubric is the engineering work

Don't agonize over which LLM. Anthropic Claude, OpenAI GPT, whatever you can call from your CI works. **The hard part is the rubric:**

1. **Define the label axes** (priority + merge-risk + proof + status — see `03-review-automation.md`)
2. **Write a markdown document** that the bot reads as part of every review. It should include:
   - The list of merge-risk surfaces ("changes to session state, auth boundaries, public APIs, fallback removal earn `merge-risk: 🚨`")
   - The rating ladder definitions
   - Examples of each rating tier
   - Explicit anti-patterns ("don't approve PRs that delete tests without explanation")
3. **Iterate the rubric based on bot failures.** If the bot keeps mis-classifying P1 vs P2, the rubric needs tighter P1 criteria. If it misses risk surfaces, list more of them.
4. **Force structured output.** Have the LLM return JSON: `{ verdict, rating, labelChanges, risks, bestSolution }`. Don't parse freeform text.

### Required ergonomic features

Even a minimal bot needs:

- **Durable comment manager** — find existing bot comment by HTML marker, edit-in-place; don't append.
- **`@bot re-review` parser** — workflow that watches for the comment, fires a fresh review.
- **Supersede behavior** — concurrency block cancels older reviews on PR sync, status updates to "Superseded" in the durable comment so contributors aren't confused.
- **Team gate on mutation commands** — `@bot automerge` / `@bot autofix` only fire from maintainer team members.

**Anti-patterns:**
- Bot posts a new comment per review (graveyard of stale verdicts)
- No re-review mechanism (every change requires re-pushing)
- Bot blocks merge unilaterally on rating alone (rating is a recommendation, humans override)
- Rubric in the bot's code, not in a doc the team can edit (engineers can't iterate on review quality)
- LLM with no structured output schema (you can't parse the verdict reliably)

## 4. Compat contract for public SDK

**Problem it solves:** External consumers of your SDK depend on stability. Breaking changes have a long tail of broken integrations. A compat contract makes breaking changes explicit, time-bounded, and auditable.

**When you need this:** When you have at least one of:
- Public npm/PyPI/crates package with external consumers
- Plugin SDK that third parties extend
- Public API (REST/GraphQL/gRPC) with external clients
- Any contract where breaking changes create real user pain

**Build pattern:**

### Step 1: Pick a versioning policy

Three viable options:

- **Semver (strict):** `MAJOR.MINOR.PATCH`. Major bump = breaking change. Easiest for consumers to reason about. Hardest for you to ship features without breaking changes.
- **Date-based:** `YYYY.M.D` for releases, breaking changes allowed on any release with prior notice. Easier for you, harder for consumers — they have to read release notes.
- **Calver + semver hybrid:** Date for releases, semver-compatible inside ("vYYYY.M.D-beta.N supersedes vYYYY.M.D-beta.N-1"). Used by some active projects (e.g., projects that ship daily or sub-daily).

For an SDK, **strict semver is usually right** because consumers can pin ranges and trust them.

### Step 2: Additive-first rule

New API surface goes first. Old API survives alongside with deprecation metadata. Example for a TS SDK:

```typescript
/**
 * @deprecated Use `searchMemoryV2` instead. Removed in v3.0.0.
 * @see https://docs.example.com/migration/memory-v2
 */
export function searchMemory(query: string): Result[] { ... }

export function searchMemoryV2(opts: SearchOpts): Result[] { ... }
```

Internal callers in your own codebase migrate to `searchMemoryV2` in the same release. Don't let internal code keep using the deprecated path; that creates permanent internal compat.

### Step 3: Deprecation timeline policy

Document an explicit timeline. Example:

> Deprecated APIs are removed after **two minor versions** with the deprecation in place, OR **one major version** boundary. Warnings are emitted at runtime via `console.warn` starting one minor version after the deprecation is added.

This means:
- v2.5: Add deprecation marker + docs migration guide
- v2.6: Add runtime warning when deprecated API is called
- v3.0: Remove deprecated API

Consumers see warnings well before removal; they have months/years to migrate.

### Step 4: Migration helpers

For non-trivial migrations, ship a codemod or a `doctor --fix` command that auto-migrates old usage patterns. Example: a script that finds calls to `searchMemory(string)` and rewrites them to `searchMemoryV2({ query: string })`.

### Step 5: Compat-impact tracking

Each PR that touches the public API surface must document compat impact in a structured way (use the merge-risk label system from `03-review-automation.md`):

- `merge-risk: 🚨 compatibility` label on the PR
- Field in PR template: **"Compat impact:"** with one of:
  - `additive` (new API, no break)
  - `deprecated` (old API marked, still works)
  - `removed` (deprecation cycle complete, API gone)
  - `behavior-change` (signature same, behavior different — usually a bug fix, document the change)
- A `CHANGELOG.md` entry under the right severity section

**Anti-patterns:**
- Removing an API without a deprecation cycle ("we'll just bump major")
- Adding new fields without backward-compatible defaults
- Permanent internal compat shims (then you can never delete the shim)
- No documented migration path ("just figure it out")

## 5. Area-owner routing (the CODEOWNERS pattern)

**Problem it solves:** When a PR touches code area X, the people who maintain X should see it. Without explicit routing, the wrong reviewers spend time on the wrong PRs and the right reviewers miss the right ones.

**When you need this:** When you have at least one of:
- 3+ regular reviewers and a codebase divided into clear ownership domains
- New contributors who don't know who to tag
- Coverage gaps in review ("nobody noticed this auth code changed")

**Build pattern:**

### Step 1: Map ownership domains

Walk your codebase. Identify ~5-15 ownership domains. Examples:
- `src/auth/` → auth team
- `src/channels/discord/` → discord channel owner
- `docs/` → docs team
- `src/sdk/` → SDK maintainers

Don't go finer than this. A CODEOWNERS file with 200 rules is unmaintainable.

### Step 2: Write `CODEOWNERS`

```
# Default reviewer (catches anything not matched below)
*                              @your-org/maintainers

# Auth — high-stakes, two reviewers required
/src/auth/                     @your-org/auth-team
/src/identity/                 @your-org/auth-team

# Channels — one reviewer each
/src/channels/discord/         @discord-owner
/src/channels/slack/           @slack-owner

# SDK — three reviewers (broad consensus)
/src/sdk/                      @your-org/sdk-maintainers

# Docs — anyone in docs team
/docs/                         @your-org/docs-team
*.md                           @your-org/docs-team

# Critical infra
/.github/workflows/            @your-org/ci-maintainers
/scripts/                      @your-org/ci-maintainers
/package.json                  @your-org/maintainers
```

GitHub automatically requests reviews from CODEOWNERS when a PR touches matching paths.

### Step 3: Pair with labeler config

`.github/labeler.yml` (see `02-ci-workflow-scaffold.md`) applies area labels to PRs. Combine with CODEOWNERS so:
- CODEOWNERS routes review request (push notification)
- Labels make PRs filterable in queues (`is:pr is:open label:"channel: discord"`)

### Step 4: Bot auto-mention (optional)

If you have a review bot, it can auto-mention area owners in the verdict comment based on the labels:

```markdown
Likely related people:
- **@discord-owner:** Recent commits to `src/channels/discord/` (file: `src/channels/discord/monitor.ts`, lines: 145-203)
```

Helps when CODEOWNERS auto-request fails (e.g., the owner is on vacation) — manual mentions still surface the PR.

### Step 5: Branch protection (optional)

If you have ownership domains where you want enforcement:
- Settings → Branches → Branch protection rule
- Require review from Code Owners ✓
- Required approving reviews: 1 (or more for high-stakes domains)

This forces a CODEOWNERS review before merge. Use sparingly — it's friction and you don't want it on every domain.

**Anti-patterns:**
- One mega-team listed as owner for everything (defeats the purpose)
- CODEOWNERS that doesn't include yourself (then you can't merge your own PRs)
- Solo maintainer with CODEOWNERS pointing only to them (PR can't merge while they're on vacation; either disable CODEOWNERS protection or add a backup)
- Rules so fine-grained reviewers get notified on every PR (notification fatigue → they stop reading)

## Bringing it together

If you applied all five extensions, your project looks like this:

- **Live QA harness** for behaviors that need real-environment proof
- **Sandbox runner** so dev machines don't carry the test suite weight
- **Review bot** with rubric, structured output, durable comment
- **Compat contract** with deprecation timeline + structured PR-level tracking
- **CODEOWNERS + labels** routing PRs to the right reviewers

That's the full set of project-scaling infrastructure pieces. Each piece is independently valuable; you don't need all five at once. Pick the one that solves your most painful current bottleneck and build it.

**A reasonable maturity ladder:**

| Project stage | Add these in order |
|---|---|
| Solo / 2 people | CODEOWNERS (just lists you) |
| Small team (3-5) | + Labeler + manual review discipline |
| Mid team (5-15) | + Sandbox (GitHub Actions runners are fine) + structured PR template |
| Larger team (15+) | + Review bot + live QA harness |
| Public SDK | + Compat contract |

Don't skip steps. You can't get value from a review bot if your rubric isn't written down. You can't get value from CODEOWNERS if you don't have labels mapping paths to areas.
