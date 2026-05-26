# shipwright

**A CI/CD template for multi-team production-critical projects.**

A paradigm + scaffolding for CI/CD at projects with real users, multiple maintainers, and code you can't easily un-ship. Covers: bot-mediated review, structured proof discipline, sandboxed broad gates, run-ID-gated releases, and area-owner routing.

Named after [shipwrights](https://en.wikipedia.org/wiki/Shipwright) — the craft of building vessels you trust to ship.

## Documents

- **[00-design.md](./00-design.md)** — The seven principles + rationale + anti-patterns + adaptation guide. Read this first.
- **[01-pr-template.md](./01-pr-template.md)** — PR description schema + Real behavior proof parser workflow.
- **[02-ci-workflow-scaffold.md](./02-ci-workflow-scaffold.md)** — Three-tier CI (fast / broad / release) with concrete YAML examples and concurrency patterns.
- **[03-review-automation.md](./03-review-automation.md)** — Label rubric (priority + risk + proof + status axes), bot review contract, durable comment pattern, command surface.
- **[04-release-pipeline.md](./04-release-pipeline.md)** — Version scheme, validation chain, run-ID-gated publish, prerelease/stable/full profiles.
- **[05-maintainer-ops.md](./05-maintainer-ops.md)** — Force-push touch-up pattern, pre-merge validation report, area-owner routing, command discipline.
- **[06-project-specific-extensions.md](./06-project-specific-extensions.md)** — Build-your-own guidance for the project-shape-specific pieces: live QA harness, sandbox runner infra, review bot, public-SDK compat contract, CODEOWNERS routing.

## How to apply this to a new project

1. **Read `00-design.md`.** Pick which of the seven principles apply to your project's scale and skip the rest. Smaller projects skip more.
2. **Start with `01-pr-template.md`.** PR template + parser is cheap and high-value; it works even without anything else.
3. **Add `02-ci-workflow-scaffold.md`'s fast lane.** Define your changed-paths classifier and wire `check:changed` / `test:changed`.
4. **Add labels per `03-review-automation.md`.** Start with manual labeling; automate later.
5. **Layer in `04-release-pipeline.md` and `05-maintainer-ops.md`** when you're ready for the release rigor and team patterns.

Each layer assumes the previous ones exist. Don't add structured proof before you have lanes; don't add bot review before you have labels; don't add release gates before you have validation.

## Project-specific extensions

The pieces in `06-project-specific-extensions.md` cover *how to build* the project-shape-specific layers:

| Extension | Solves | When to add |
|---|---|---|
| **Live QA harness** | Real-environment proof for external systems / hardware | When mocks aren't enough and you ship to real users |
| **Sandbox runner** | Reproducible remote runner with explicit env contract | When broad gates burden dev machines |
| **Review bot** | Structured first-pass review with rubric and labels | When review attention is a bottleneck |
| **Compat contract** | Stability guarantee for SDK/API consumers | When you ship a public SDK / API |
| **Area-owner routing** | Right reviewers see the right PRs | When you have 3+ regular reviewers |

Each is independently valuable; add them in the order your bottlenecks dictate.

## What this template is NOT

- **Not a one-size-fits-all spec.** Adapt aggressively to your project shape.
- **Not turnkey scaffolding.** Examples are illustrative. You'll need to write your own `scripts/changed-lanes.mjs`, customize the labeler config, build the proof parser to match your PR template, etc.
- **Not a complete enumeration.** Some patterns at very large scale (custom static-analysis matrices, multi-product release orchestration, federated docs sync) aren't covered here. Build what you need; this template is the spine, not the skeleton.

## Estimated implementation effort

For a project moving from "no formal CI/CD" to the rigor described in this template:

| Layer | Effort | Required? | Covered in |
|---|---|---|---|
| Fast lane (`check:changed`, lanes classifier) | 1-2 weeks | Yes | 00, 02 |
| PR template + Real behavior proof parser | 3-5 days | Yes | 01 |
| CI workflow tiers (`ci.yml` with shards) | 1-2 weeks | Yes | 02 |
| Label rubric (4 axes) + GitHub config | 2-3 days | Yes | 03 |
| Release pipeline (run-ID-gated) | 1-2 weeks | Yes for shipped software | 04 |
| Maintainer ops + docs | 1 week | Yes | 05 |
| CODEOWNERS / area routing | 1-2 days | Yes for 3+ reviewers | 06 §5 |
| Sandbox infra (if not using GH Actions runners) | 2-6 weeks | Optional | 06 §2 |
| Review bot integration | 2-4 weeks | Optional | 06 §3 |
| Live QA harness | 1-3 weeks | Optional | 06 §1 |
| Public SDK compat contract | 1 week to design + ongoing discipline | Yes if shipping SDK | 06 §4 |

So: **6-12 weeks of focused work** to get the core in place. The optional extensions in doc 06 add weeks but only when their respective bottlenecks bite — don't build them upfront.

The payoff: fewer regressions in production, faster review turnaround, less time spent on "did you test this?" arguments, and an audit trail for every merge.

## Source

This template was synthesized from research into one mature production codebase's CI/CD setup (56 GitHub workflows, custom review bot, sandboxed runner infrastructure, structured proof discipline, run-ID-gated releases). The principles and patterns are intentionally generalized; the source project's specific tool names and people have been replaced with role-neutral placeholders. See conversation history in this directory if you need the original research reports for context.
