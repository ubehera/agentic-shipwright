# PR Template + Real Behavior Proof Schema

Drop this in `.github/PULL_REQUEST_TEMPLATE.md` and pair it with a workflow that parses the structured fields.

## The template

```markdown
## Summary

<!-- 1-3 sentences. What changed and why. Link the issue. -->

## Changes

- <!-- bulleted list of concrete edits -->

## Real behavior proof

> Required for behavior-changing PRs. Use `proof: override` label + a written reason if not applicable (e.g., docs-only, internal refactor with no behavior change).

- **Behavior addressed:** <!-- one sentence stating what user-observable behavior changed; mention the issue # if any -->
- **Real environment tested:** <!-- "Local macOS 15.5 Node 22.19+ on branch HEAD <SHA>", "Sandbox run <ID>", "Live provider <NAME>", etc. -->
- **Exact steps or command run after this patch:**

  ```sh
  <!-- the command(s) you ran. Paste them verbatim. -->
  ```

- **Evidence after fix:** <!-- terminal output (in a code block), screenshot link, gist link, sandbox artifact URL, or test summary. NOT "tests pass" — actual output. -->
- **Observed result after fix:** <!-- one sentence: what you saw, what you expected, do they match -->
- **What was not tested:** <!-- be honest. "Live multi-region failover", "Production-scale workload", or "Specific edge case from issue #N" or similar. "None" is acceptable if you tested everything. -->

## Risk

<!-- Optional. Use if your change touches state, auth, public API, or fallback behavior. -->
- [ ] Touches `merge-risk: 🚨 session-state` surface — explain how this fix preserves existing invariants
- [ ] Touches `merge-risk: 🚨 availability` surface — explain how operator action / cleanup paths still complete
- [ ] Touches `merge-risk: 🚨 compatibility` surface — additive change OR named compat metadata is present

## Sibling surface check

<!-- Did you check whether sibling code paths have the same bug? Provider A vs Provider B, Channel X vs Channel Y, etc. One-sided fixes need explicit "verified siblings unaffected" or "follow-up issue filed". -->

## Changelog

<!-- Required for user-facing fix/feat/perf changes. Format: single-line bullet under the active version's "### Fixes" / "### Changes" section. Contributor PRs: leave this for the maintainer if you don't have access to CHANGELOG.md conventions. -->
```

## The parsing workflow

Create `.github/workflows/real-behavior-proof.yml`:

```yaml
name: Real behavior proof

on:
  pull_request_target:
    types: [opened, edited, synchronize, reopened, ready_for_review, labeled, unlabeled]

permissions:
  pull-requests: write
  contents: read

concurrency:
  group: real-behavior-proof-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  parse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          ref: ${{ github.event.pull_request.head.sha }}
      - uses: actions/setup-node@v5
        with: { node-version: '22' }
      - run: node scripts/github/real-behavior-proof-policy.mjs
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
          PR_BODY: ${{ github.event.pull_request.body }}
```

## The parsing script

`scripts/github/real-behavior-proof-policy.mjs` — sketch (adapt to your needs):

```javascript
#!/usr/bin/env node
// Parses PR body for Real behavior proof fields. Applies labels based on
// what it finds. Required fields missing → triage: needs-real-behavior-proof.
// Found + non-trivial → proof: supplied. Live commands/screenshots/logs → proof: sufficient.

const FIELD_ALIASES = {
  behaviorAddressed: [
    /^[\-\*\s]*\*\*Behavior addressed:\*\*/im,
    /^[\-\*\s]*\*\*Issue addressed:\*\*/im,
  ],
  realEnvironment: [
    /^[\-\*\s]*\*\*Real environment tested:\*\*/im,
    /^[\-\*\s]*\*\*Environment tested:\*\*/im,
  ],
  exactSteps: [
    /^[\-\*\s]*\*\*Exact steps or command run after this patch:\*\*/im,
    /^[\-\*\s]*\*\*Steps after this patch:\*\*/im,
    /^[\-\*\s]*\*\*Command run after fix:\*\*/im,
  ],
  evidence: [
    /^[\-\*\s]*\*\*Evidence after fix:\*\*/im,
    /^[\-\*\s]*\*\*Evidence:\*\*/im,
  ],
  observedResult: [
    /^[\-\*\s]*\*\*Observed result after fix:\*\*/im,
    /^[\-\*\s]*\*\*Observed result:\*\*/im,
  ],
  whatWasNotTested: [
    /^[\-\*\s]*\*\*What was not tested:\*\*/im,
  ],
};

// Patterns that indicate evidence is live / real, not mock
const LIVE_EVIDENCE_PATTERNS = [
  /```\s*sh|```\s*bash|```\s*shell/i,        // shell output
  /\$\s+\w+/,                                  // a command prompt
  /https:\/\/[\w-]+\.s3\.amazonaws\.com\//,    // sandbox artifact
  /github\.com\/[\w-]+\/[\w-]+\/actions\/runs/, // CI run link
  /pass|fail|exit \d/i,                        // result indicators
];

const MOCK_ONLY_PATTERNS = [
  /^mocked /im,
  /unit tests? pass/i,                          // "unit tests pass" alone is not enough
];

// ... parse body, classify, apply labels via gh api ...
```

A production-grade parser can grow to ~40 patterns over time. Don't try to write all of them upfront — start strict, loosen as you learn what real PRs look like.

## Label outcomes

| Label | Meaning |
|---|---|
| `triage: needs-real-behavior-proof` | Required fields missing or empty |
| `proof: supplied` | Fields present, contains text |
| `proof: sufficient` | Live commands / screenshots / sandbox artifacts present, not mock-only |
| `proof: override` | Maintainer applied explicitly; PR body has a written reason in the Summary |

Required-for-merge: `proof: sufficient` OR `proof: override`. Configure via GitHub branch protection or your merge bot.

## Anti-patterns this prevents

- **"Tests pass"** as evidence — pattern matchers downgrade to `proof: supplied` only.
- **Description left blank** — `triage: needs-real-behavior-proof` blocks merge.
- **Stale proof after force-push** — re-parse on `synchronize`. The label gets stripped if the SHA changes and the body wasn't updated.

## Edge cases worth handling early

- **Stack-on-top-of-PR**: if your repo uses stacked PRs, the proof should reference the top of the stack, not the merge base.
- **Multiple proof sections** when re-pushing: append "Refreshed proof for HEAD `<SHA>`" subsection rather than replacing — preserves history of what was verified when.
- **Docs-only PRs**: pre-filter in the workflow (skip if `git diff --stat` only touches `docs/**` and `*.md`). Or use the `proof: override` path.
- **Internal refactor**: same — `proof: override` with a written rationale.
