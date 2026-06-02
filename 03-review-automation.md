# Review Automation: Labels, Rubric, Bot Contract

A label model + review-bot contract for PR triage at scale. The labels are the engineering work; the bot is just an executor.

## The four label axes

Every active PR carries one label from each axis (or none, if not yet triaged). They're orthogonal.

### Axis 1: Priority

How much does the user feel this if it ships broken / never lands?

| Label | Meaning |
|---|---|
| `P1` | High-priority user-facing bug or regression; broken workflow; data loss; security concern |
| `P2` | Normal backlog priority; limited blast radius |
| `P3` | Cosmetic / nice-to-have / non-urgent |

### Axis 2: Merge risk

How easy is it for this fix to introduce a new regression on a sensitive surface?

Define your own merge-risk surface names. Examples:

| Label | Trigger |
|---|---|
| `merge-risk: 🚨 state` | Touches persistent state, session/context boundaries, auth/identity, or correctness fences |
| `merge-risk: 🚨 availability` | New process-global resources, queues, locks that can wedge if cleanup is skipped |
| `merge-risk: 🚨 compatibility` | Public API changes, config schema changes, fallback removal, stricter validation |
| `merge-risk: 🚨 data` | Schema migrations, data-loss-possible paths, irreversible operations |
| (none) | Routine code change without correctness/lifecycle risk |

These get applied even on green-CI PRs when the *changed surface* matches the trigger list. AGENTS.md or equivalent should enumerate the trigger surfaces.

### Axis 3: Proof

Did the PR include the evidence the contract requires? (See `01-pr-template.md`.)

| Label | Meaning |
|---|---|
| `proof: supplied` | Structured fields present, content non-empty |
| `proof: sufficient` | Evidence is live (commands, screenshots, sandbox artifacts) not mock-only |
| `proof: override` | Maintainer applied explicitly; PR description has a written exemption reason |
| `triage: needs-real-behavior-proof` | Fields missing or empty |

### Axis 4: Status

What state is the PR in for the contributor / maintainer workflow?

| Label | Meaning |
|---|---|
| `status: ⏳ waiting on author` | Bot or maintainer left findings to address |
| `status: 📣 needs proof` | Real behavior proof inadequate; contributor action required |
| `status: 👀 ready for maintainer look` | No contributor-facing blockers; awaiting human reviewer |
| `status: 🚀 automerge armed` | All gates green; bot can land when ready |
| `status: 🔒 blocked by upstream` | Waiting on a sibling PR or external dependency |

## The rating ladder

A merge-readiness rating applied by the review bot. Pick a scheme that's memorable for your team (themed emojis, plain text labels, role names, etc.). Below is one example with thematic emojis; any well-defined multi-tier system works as long as the criteria for each tier are written down:

| Rating | Meaning | Typical merge action |
|---|---|---|
| 🚢 launch-ready | Rare. Exceptional readiness, strong proof, clean implementation, convincing validation. | Automerge candidate. |
| ⚓ seaworthy | Very strong. Minor maintainer review expected. | Light maintainer pass, merge. |
| 🧭 charted | Good normal PR. Likely mergeable with ordinary review. | Standard maintainer review. |
| 🛠️ in-refit | Useful signal, but proof or patch confidence limited. | Needs more evidence or another iteration. |
| 🪵 taking-water | Thin signal. Proof, validation, or implementation needs work. | Substantial revision before merge. |
| ⛔ dry-docked | Not merge-ready. Missing/unusable proof, or serious correctness/safety concerns. | Block merge. Request fixes. |

**Critical:** "overall" rating follows the **weaker** of "proof" and "patch quality." A perfect patch with weak proof gets bottlenecked on proof. This forces the proof discipline to actually matter.

## The bot review contract

The review bot reads the PR + AGENTS.md/CLAUDE.md (full files, not snippets) + scoped guides for touched paths. Then emits:

```json
{
  "verdict": "needs-changes | needs-proof | ready-for-maintainer | landable",
  "rating": "<your-tier-name>",
  "ratings": {
    "proof": "<tier-name>",
    "patch_quality": "<tier-name>"
  },
  "labelChanges": {
    "add": ["merge-risk: 🚨 state"],
    "remove": ["triage: needs-real-behavior-proof"]
  },
  "labelJustifications": {
    "merge-risk: 🚨 state": "<one-line reason why the touched surface earns this label>"
  },
  "risks": [
    {
      "severity": "P1",
      "summary": "<concise description of the specific risk>",
      "location": "<file>:<line>"
    }
  ],
  "bestSolution": "<one-line description of the recommended fix shape>",
  "reviewMetrics": [
    {
      "name": "<metric name, e.g. 'Public API surfaces added'>",
      "count": 1,
      "rationale": "<why this metric matters for merge readiness>"
    }
  ]
}
```

Important: outputs are **structured**, not freeform. The bot's job is to translate a code review into a machine-readable verdict so a human can quickly decide what to do.

## Durable review comment

The bot maintains **one comment per PR**, edited in place. New reviews update the same comment. This is non-negotiable — without it, the PR conversation becomes a graveyard of stale verdicts.

Use HTML comment markers to identify the comment:

```html
<!-- review-bot-id:pr=<PR_NUMBER> -->
<!-- review-bot-verdict:charted sha=<SHA> confidence=high -->
```

The bot finds its own comment by marker, fetches the current PR HEAD SHA, compares, and edits or appends as needed.

## Contributor commands

Implement at least:

| Command | Effect |
|---|---|
| `@bot re-review` | Queue a fresh review on the current HEAD |
| `@bot explain` | Bot expands on a finding with more context |

Status updates posted to the durable comment, not as new comments:

```html
<!-- bot-command-status:<PR_NUMBER>:re_review:<SHA> -->
<!-- bot-command-progress:start -->
Re-review progress:
- State: Review in progress
- Detail: Targeted re-review run started.
- Run: https://github.com/.../actions/runs/12345
- Updated: 2026-05-25T21:35:02Z
<!-- bot-command-progress:end -->
```

When superseded by a newer review, the progress field updates to "Superseded" so contributors can see what happened without scrolling.

## Maintainer commands

Higher-privilege; gated by maintainer team membership:

| Command | Effect |
|---|---|
| `@bot review` | Force a review run regardless of status |
| `@bot automerge` | Land when all gates pass; bot watches and merges |
| `@bot autofix` | Bot makes the recommended code change; opens follow-up commit |
| `@bot fix ci` | Bot diagnoses + fixes a failing required check (typo, lockfile drift, etc.) |
| `@bot address review` | Bot applies its own findings as a follow-up commit |
| `@bot stop` | Cancel active automation |

These should require explicit team membership checks — never invoke them from contributor comments.

## Avoiding bot/human conflict

- **Re-review supersedes**, doesn't append. The previous run gets cancelled by GitHub concurrency; the durable comment shows "Superseded" state.
- **Force-push triggers re-review automatically**, not on a label change. Label changes go to the labeler, not the reviewer.
- **Bot only edits its own comment.** It never edits the human reviewer's comments or the PR description (except for label-only changes, which aren't comment edits).
- **Bot never closes PRs unilaterally.** Closing is a human-only action.

## Bot ops anti-patterns

- **Spamming the PR with comments.** Use the durable comment + label state instead.
- **Reviewing the diff in isolation.** The bot must read AGENTS.md / scoped guides / sibling code paths. A review that doesn't read context is just a linter.
- **Promising work without evidence.** "I checked X" with no reasoning shown is unhelpful. Bot reviews should include "what I checked" and "what I didn't check" sections.
- **Blocking on bot rating alone.** A lowest-tier (⛔ dry-docked) rating can still be a valid PR if a maintainer overrides; rating is a recommendation, not a vote.
- **Auto-applying labels without justification.** Every label change should have a one-line justification in the verdict comment.

## Migration path from no-bot

If you're starting without a bot:

1. **Labels first.** Define the four axes manually. Maintainers apply by hand. This is the engineering work — get it right before automating.
2. **PR template + parser.** The Real behavior proof workflow (see `01-pr-template.md`) doesn't need a bot; it's just a label-applier script.
3. **Then bot reviews.** Once you have stable labels, point an LLM at the diff + AGENTS.md and have it emit a structured verdict.

Skipping straight to bot reviews without a stable label rubric produces noise.
