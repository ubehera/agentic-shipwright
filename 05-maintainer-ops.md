# Maintainer Operations Runbook

The operational patterns maintainers use to land PRs efficiently: force-push touch-ups, pre-merge validation reports, automerge triggers, area-owner routing. These exist to make merging safe and fast for the maintainer, and to keep an audit trail of why each merge happened.

## The takeover pattern

**Contributors enable `Allow edits by maintainers` on their PRs.** This is a checkbox on the PR creation form in GitHub UI; for fork-based PRs it allows the upstream maintainer team to push to the contributor's branch.

Once enabled, the maintainer can:

1. Rebase the PR onto current `main` (resolving conflicts they understand)
2. Restructure or add small commits (tests, formatting, lint fixes, conventions)
3. Force-push the result to the contributor's branch

GitHub shows the push as "Maintainer pushed N commits" with the maintainer as **committer** and the contributor preserved as **author** in each commit.

Why this pattern: removes the back-and-forth ping-pong of "please rebase, please add a test, please format with our convention." The maintainer can do those edits themselves in 2 minutes instead of asking the contributor to take an hour to do them.

**Discipline:**
- Don't change the contributor's *intent*. If you'd want to rewrite the whole approach, that's a review comment, not a push.
- Keep authorship as the contributor in any commit you didn't write from scratch (use `git commit --author=<contributor>` if cherry-picking).
- If you add a substantive new commit, sign it as your own (committer + author both you).

## The pre-merge validation report

Before clicking Merge, the maintainer posts a structured comment documenting what they verified:

```markdown
Maintainer verification for head `<SHA>` rebased on base `<BASE_SHA>`:

- **Focused regression tests:** `node scripts/run-vitest.mjs src/agents/foo.test.ts -- --reporter=dot`
  → N files / M tests passed.
- **Diff sanity:** `git diff --check origin/main...HEAD` → passed.
- **Structured review:** `<autoreview command>` → clean, no accepted/actionable findings.
- **Broad changed gate:** `<crabbox/testbox command>` → passed on `<sandbox_id>`, run `<run_id>`, exit 0.
- **GitHub checks:** `gh pr checks <PR#> --repo <repo>` → all current checks passing, including `<critical-check-name>`.

Note: <any infrastructure flakes or known-flaky checks called out as unrelated>.
```

Then merge.

Why this pattern: future-you (or another maintainer) reviewing a regression six months later can see exactly what was checked before landing. If a regression escapes despite the validation, the report identifies which check should have caught it.

Codify this rule in your AGENTS.md / CONTRIBUTING.md: *"PR verification: before merge, post exact local commands, CI/sandbox run IDs, before/after proof when used, and known proof gaps."*

## Force-push hygiene

When you rebase a PR:

```sh
# Fetch latest
git fetch upstream main
git fetch origin <pr-branch>

# Check out the PR branch (fork or upstream)
git checkout <pr-branch>

# Rebase onto current main
git rebase upstream/main

# Resolve conflicts deliberately — keep contributor's intent
# Run the focused test suite to make sure rebase didn't break anything
node scripts/run-vitest.mjs <affected-tests>

# Force-push with lease (refuses if someone else pushed in the meantime)
git push --force-with-lease
```

**Never use plain `--force`.** `--force-with-lease` checks that your local view of the remote ref matches the actual remote; if the contributor pushed an update while you were rebasing, the lease refuses and you go re-fetch.

### When rebase loses code (rare but important)

Watch for this scenario:
- You rebase a PR that has a "revert X" commit
- The revert + previous commits get squashed during interactive rebase
- The revert's effect propagates incorrectly, deleting code that should stay

Defense:
- Always `git diff <base>..HEAD --stat` after rebase to see the net change
- Compare against the PR's previous diff stat; if numbers look very different, investigate
- Especially watch for missing files or sudden line deletions in files the PR shouldn't touch

## Area-owner routing

The `.github/labeler.yml` config maps file paths to labels. Labels imply area ownership. Maintainers monitor "their" labels via GitHub notification filters:

```
is:open is:pr label:"channel: discord"  -reviewed-by:@me
```

Auto-mention via CODEOWNERS:

```
# .github/CODEOWNERS
src/agents/                        @agents-team
extensions/discord/                @discord-area-owners
docs/                              @docs-team
src/plugin-sdk/                    @sdk-maintainers
```

GitHub automatically requests review from CODEOWNERS when a PR touches their paths. This is the primary mechanism for "the right person sees this PR."

If you don't have multiple maintainers, skip CODEOWNERS but keep the labeler — the area labels still help with prioritization ("which P1 is in my area today?").

## Maintainer commands (bot-mediated)

These should be gated by team membership in your review bot config:

| Command | Action | Required gate |
|---|---|---|
| `@bot review` | Force a fresh bot review | Maintainer team |
| `@bot automerge` | Land when all gates pass; bot watches for the green state | Maintainer team |
| `@bot autofix` | Bot applies its own recommended fix as a follow-up commit | Maintainer team |
| `@bot fix ci` | Bot diagnoses + fixes a failing required check (lockfile drift, type stub, etc.) | Maintainer team |
| `@bot address review` | Bot applies findings from its previous review as a commit | Maintainer team |
| `@bot stop` | Cancel active automation | Maintainer team |
| `@bot explain` | Bot expands on a finding with more context | Anyone |
| `@bot re-review` | Queue a fresh review (no other actions) | PR author or maintainer |

The split between "fresh-review commands" (anyone) and "mutation commands" (maintainer) is important — anyone can ask for a re-check, but only maintainers can land or rewrite code.

## Closing duplicates

When a maintainer decides a PR or issue is a duplicate / not-planned / superseded:

1. Comment with the decision rationale, the supported alternative, and what evidence would change the decision
2. Search for associated issues/PRs (linked, mentioned, or matching the area) and close them together with the same rationale
3. Apply `not-planned` or `duplicate-of-#XXX` labels

A useful rule to put in your AGENTS.md: *"Maintainer decision closes the cluster: if deciding reported behavior/proposed fix is not planned, comment+close all directly associated open issues/PRs unless explicitly told to keep one open."*

Don't leave associated issues open "in case." Close with rationale; if new evidence appears later, the original reporter can reopen.

## Release commit etiquette

Releases are dispatched via `release-publish.yml` (see `04-release-pipeline.md`). Maintainer's role:

1. Confirm the `preflight_run_id` and `full_validation_run_id` both point at the ref they want to release
2. Confirm `CHANGELOG.md` head section is updated for the version being shipped
3. Trigger the workflow with the right `release_profile`
4. Watch the workflow until publish completes; if it fails partway (e.g., plugins published but core failed), follow the recovery runbook

After shipping:
- Comment on the issues that the release fixes, with the commit SHA and release URL
- Comment on the merged PR with the release version it shipped in: *"Landed via GitHub rebase merge onto main. Source head: <SHA>. Landed commits: <SHA>. Gate: GitHub checks green at source head, including <critical-checks>. Local sync: main fast-forwarded to <SHA>. Thanks @<contributor>!"*

## What NOT to do as a maintainer

- **Force-push without `--force-with-lease`.** Race-loses contributor's pushes.
- **Merge with red CI.** Even one red required check. The bot's "this check is flaky" excuse should be a workflow-level fix, not a merge-time override.
- **Skip the pre-merge validation report.** Future maintainers have nothing to audit.
- **Land without rebase onto current main.** Even if GitHub allows it, you're shipping the state that was validated against an old base; new regressions could land via merge resolution.
- **Disable `Allow edits by maintainers` on your own PRs.** You're a maintainer; if a teammate wants to push a fix, let them.
- **Use `git push --no-verify`.** If a pre-commit hook fails, fix the underlying issue. Skipping hooks defeats the safety net.

## Operational metrics worth tracking

If your team has time for this, track:
- **Time from PR open → first bot review** (target: < 5 min)
- **Time from `ready for maintainer look` → merge** (target: < 24h for P1, < 1 week for P2)
- **Force-push count per PR** (a high count suggests review feedback isn't landing in one pass)
- **Rebase conflicts at merge time** (high count suggests `main` is moving faster than PR iteration)
- **Post-merge incident count per release** (the true measure of CI/CD quality)

Don't track:
- **Lines of code added** (encourages bloat)
- **PR throughput** (encourages churn, discourages careful work)
- **Bot rating distribution** (bot tunes to the curve; not a quality signal)
