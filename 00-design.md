# CI/CD Design for Multi-Team Production-Critical Projects

A paradigm for projects with 5+ regular contributors, real users on the receiving end of regressions, multiple owners spanning multiple subsystems, and a release that ships software you can't easily un-ship.

If you're solo or pre-product, this is overkill — pick three principles and skip the rest. The point is to lift ideas that scale, not to copy the whole stack.

## 0. Why these gates work — and what they can't

The seven principles below are *mechanisms*. This section is the *model* behind them: why a CI/PR process catches defects the author was sure weren't there, when to trust a green check, and — critically — the *classes* of defect no gate catches. It's a lens, not a closed taxonomy. Deploy the mechanisms without the model and you cargo-cult: an agent (or human) wiring CI from a checklist builds theater. Read this first.

### 0.1 The defect model — error and self-review share a frame

An author's mistake is rarely random; it's the systematic product of the frame they reasoned in. The fix looked best because the same model that *generated* it also *graded* it. A blind spot isn't "something they forgot to check" — it's something *invisible from where they stood*. That's why a careful, well-intentioned author ships a confident, wrong change and their own re-read sails past it: **the grader inherits the generator's priors.** Self-review is weak not because authors are lazy but because error and self-assessment are *correlated* — same origin.

The corollary that drives everything else: a second check only helps if its verdict is *decoupled* from whatever made the author wrong. A reviewer who shares the author's training, reads only the author's narrative, and grades by the same taste is *different* but not *independent* — they miss the same things. **Diversity of failure modes is the design target; diversity of mechanism is only the means.**

### 0.2 The external-oracle principle

> **A gate earns its place by relocating the burden of proof from the author's *intention* to an *external oracle* — something that judges the artifact, not the author's report of it. Its strength is the product of three factors: how *deterministic* it is (immune to confidence), how much *more than the author chose to look at* it covers, and whether it *consults reality* rather than the author's claim about reality.**

For any gate, ask: *what oracle does this consult, and how far is that oracle from the author's intention?* That one question is the sharpest diagnostic available. It surfaces theater — a check whose oracle is patch-shape is judging a *proxy*, not reality. And it surfaces fake defense-in-depth — two gates sharing one oracle are one gate wearing two hats. (A proxy dressed as reality can still pass it — see Goodhart, §0.3.)

A gate decouples from the author on one or more axes — useful when *designing* one, because it names what to vary:

- **Frame** — judges against a model the author didn't author (an external rubric; a reviewer who rebuilds the decision surface from the code, not from the PR narrative).
- **Input** — consumes evidence the author didn't (the full suite runs files the author never opened; a parser reads the literal bytes, not the intent).
- **Criterion** — "correct" is fixed in advance and external (a schema, a contract, a pass/fail oracle the author can't move to where their work already stands).
- **Modality** — demands a *different kind* of evidence (the author made an *argument*; the gate requires an *observation*).

These axes aren't a second model — they're the levers that move the three oracle factors: Criterion and Modality buy *determinism* and *reality*; Input buys *coverage beyond what the author chose*. Vary an axis only to strengthen the oracle.

**Determinism is the strongest form, because it's independence from confidence itself.** The dangerous defects are *confidently* wrong, wrapped in fluent justification. A type-checker or a parser is the one judge that can't be talked out of a verdict — it never reads the justification. Weight a pipeline toward verdicts that can't be argued with.

Four real misses, the gate that caught each, and the oracle it consulted:

| The author missed… | because (blind spot) | caught by | oracle consulted |
|---|---|---|---|
| a fix at the **wrong layer** (suppressed the symptom where they were editing, not where the invariant lived) | local reasoning + satisficing — first coherent fix graded "good enough" | a reviewer mandated to rebuild the whole decision surface and demand the *best* fix, not a *plausible* one | a second judgment over the *whole* surface (frame + criterion) |
| a **sibling** test broke | mental blast-radius was "files I touched," not "behavior I changed" | the full suite — it has *no model of relevance*, which is the feature | the whole artifact's behavior (input) |
| a **machine-parsed** field, silently broken by "improving" the prose | unknown-unknown — didn't know a consumer parsed it | a deterministic validator, failing instantly + specifically | a fixed schema, immune to confidence (criterion + determinism) |
| "**tests pass, therefore correct**" | proof-vs-claim conflation | a real-behavior gate demanding a live demonstration | observed reality (modality) |

### 0.3 The Named-Defense Rule (design heuristic)

For every gate you add, finish this sentence: *"This gate defends against ⟨named blind spot⟩; its oracle is ⟨what it consults⟩, which is ⟨deterministic / covers what the author didn't / consults reality⟩ — and therefore decoupled from the author's ⟨frame / input / criterion / modality⟩."* Lead with the oracle; independence is what you derive from it, not the starting point. If you can't name the blind spot, the gate is decorative. Then run four tests — each phrased so the *failing* answer names the fix:

1. **Redundancy** — *Could the author, in good faith with more time, have run this exact gate against themselves and gotten the green they want?* If yes → it mostly duplicates self-review; it's a real gate only on the axis the author can't move (they can re-read the diff; they *can't* will a full suite green or argue a parser out of a verdict).
2. **Confidence** — *Does the verdict change if the author is more eloquent or more certain?* If yes → it's coupled to confidence; backstop it with a deterministic or observational check.
3. **Portfolio** — *Across all gates, do any two share the same oracle and the same inputs?* If yes → they fail together: apparent defense-in-depth, one effective layer. Stack gates by oracle, not by count — three checks that read the diff against personal taste are one check.
4. **Goodhart** — *What stops an author meeting this check's letter and missing its spirit?* If the answer isn't "a gate whose bar is *not* fully exposed — an open-ended judgment review," → it will be gamed, the more precisely the more exactly you specified it.

### 0.4 The verification ceiling — what no gate catches

The four examples above are *survivors*: defects a gate happened to catch. The defects that reach production are the ones with **no gate-shaped signature**, and a pipeline that hides this trains people to mistake green for correct:

- **Spec-is-wrong / wrong problem.** Every gate checks the artifact against the *stated* intent. If the intent is wrong, all gates go green.
- **Sins of omission.** Gates check what's *in* the diff — not the validation never written, the error path never handled, the case no test covers.
- **Design / taste / wrong-abstraction.** Make the wrong-layer fix one notch subtler — right layer, wrong abstraction, future-coupling — and it passes everything. Taste isn't gate-able.
- **Emergent / integration / temporal.** Races, ordering, retry storms, "wrong only after N cycles." A live demo shows *one* execution — evidence of presence, never of absence.
- **Performance / resource** regressions that are correct-but-slow, or only degrade at scale.
- **Security logic that "works."** Returns the right answer for every test input and the wrong one for the adversarial case nobody wrote.

Say the ceiling out loud in your pipeline: **green means "no defect of a shape a gate can see," not "correct." The absence of a red check is the absence of evidence, not evidence of safety** — and the residual classes above are exactly what survives to production, so green should *raise* scrutiny on design, spec, and omission, not end it. This is why the judgment-based reviewer is the *least* disposable gate, not the soft one: it's the only thing that looks at the classes no machine can.

### 0.5 Anti-lessons — what NOT to conclude

The model has *operational corollaries* — failure modes a pipeline trains into a team, faster into an agent that optimizes whatever proxy you hand it, literally and tirelessly:

- **More gates ≠ more quality.** Gates cost latency, maintenance, and trust — a finite *budget*. Every new gate must name a defect class no existing gate catches; otherwise it's cost, not safety. Redundant correlated checks buy the *feeling* of safety with none of the coverage.
- **Proxy ≠ target; shaping the artifact to the gate is the smell of gaming.** When honest compliance costs more than plausible compliance, authors satisfy the proxy: an author who breaks a parsed field by *rewriting it to taste* will, under a proof contract, produce a *claim* of "tested" rather than a test — unless the gate demands an observation it can't fabricate.
- **Self-review isn't worthless — it's worthless *in the same frame*.** It turns high-value the moment you **change the oracle**: re-derive the spec from scratch, run the *full* suite (not the changed-files slice), demand a live demo of *your own* claim, and pick the input that fails *if you're wrong*. Most of what a careful author can do before any reviewer sees the work is exactly this.
- **A flaky required check is a bug, not noise.** It teaches "retry until green," which inverts the gate into a laundering machine. Fix it or make it advisory; never re-roll a red.
- **"I ran a thing" ≠ a demonstration.** A live proof is only proof to the extent it exercises the case *at risk*; happy-path-on-one-record is theater that manufactures confidence.

### 0.6 Why agents need this more than humans

An LLM agent's generation and self-assessment are the *same forward pass over the same context* — its self-review is maximally coupled to its error, more than any tired human's. Seven amplifiers, each of which defeats self-review and weak gates:

- **Confidently fluent when wrong** — weaponizing the one channel self-review trusts.
- **Finite, volatile context** — routinely omits the sibling and the contract.
- **Satisfices** on the first coherent path.
- **No persistent memory** of the project's contracts.
- **"Improves" by rewriting** — breaking contracts it didn't know existed.
- **Eager to declare done.**
- **Goodharts literally** — satisfies a visible check efficiently, misses the intent.

So for agent-authored work the bar is higher — but the cure isn't "trust the author less," it's **change the oracle, not the effort**. A frame-coupled blind spot can't be fixed by trying harder in the same frame; it *can* be fixed by the author re-deriving the spec, running the full suite, and demoing the at-risk case (§0.5). What *can't* be delegated is the judgment that the author's frame, however careful, is the thing under suspicion — so the pipeline supplies the rest: the contract memory the agent lacks (deterministic validators), the whole-surface context its window omits (broad lanes + sibling-surface review), the higher-than-functioning bar its satisficing skips (best-not-plausible rubric), the evidence-not-assertion definition of done its eagerness overruns (proof contract), and one judgment gate its optimizer can't teach-to-the-test (irreducible open-ended review). Each is chosen *to decouple* its verdict from the way the agent is confidently wrong — the design target of §0.1, not a guarantee any single gate achieves.

The seven principles that follow each instantiate one or more of these defenses. Read them with the question: *which blind spot does this defend, and what oracle does it consult?*

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
