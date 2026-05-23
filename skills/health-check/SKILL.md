---
name: health-check
description: >
  Periodic health check for the ilo grinding pipeline. Pokes stuck background
  subagents, relaunches ones that closed early, picks up new fix tickets to
  keep two in flight, and falls back to reviewing/merging in-review PRs when
  the fix queue is empty. Files any issues encountered as Linear tickets so
  nothing is lost between passes. Invoke directly with `/health-check`, or
  schedule with `/loop 10m /health-check`.
---

# health-check

Run this every ~10 minutes while ilo PR-grinding work is active. Each pass does five things in this order: triage running agents → drain finished agents → top up to two-in-flight → if there is no usable fix work, fall back to PR review/merge → file any new problems as Linear tickets so the next pass picks them up.

Designed for the [ilo-lang/crew](https://github.com/ilo-lang/crew) loop; safe to invoke ad-hoc as well.

## 0. Conventions (load before doing anything)

Environment expectations — the runner already has these set, **never embed credentials in this skill**:

- `LINEAR_API_KEY` — auth for the Linear GraphQL API.
- `GITHUB_TOKEN` (or a logged-in `gh` CLI) — auth for PRs, label edits, merges.

IDs and paths used by the pipeline:

- Linear team: `ilo-lang/ilo`. Look up its id once per pass via `teams(filter:{key:{eq:"ILO"}}){nodes{id}}`; cache for the rest of the pass.
- Label `mini` = the claim on a fix-in-flight. Look up the label id by name (`issueLabels(filter:{name:{eq:"mini"}})`); never hard-code.
- Label `mini-reviewing` = we are reviewing someone else's PR. **Never** apply this to a fix-in-flight.
- States `In Progress`, `In Review`, `Done` — look up state ids by name (`workflowStates`); never hard-code.
- Worktrees live at `/home/<user>/ilo-<num>` with branch `ilo-<num>-<slug>`.
- `CARGO_TARGET_DIR=/home/<user>/.cache/cargo-target-<num>` (NOT `/tmp` — too small).
- `ai.txt` is auto-generated from `SPEC.md` by `build.rs` — never hand-edit.

## 1. Triage running subagents

Use `TaskList` to see every active subagent. For each one that is `running`:

- If duration > 30 minutes with no recent tool activity → `SendMessage`: `status check — give a 3-line update: what you're doing now, what's blocking, ETA.`
- If duration > 90 minutes → assume stuck. Capture the agent id, ticket, and last activity, then surface as one line to the human: `ILO-<n> agent <id> stuck for <mins>m, last: <action>; suggesting cancel.` Do NOT cancel without explicit approval.
- Otherwise leave it alone.

## 2. Drain finished subagents

For each agent in `TaskList` that is `completed` or `failed`:

- If `failed` or closed without a PR URL → **before** relaunching, capture the failure reason from the agent's last message and file a Linear ticket if it surfaced a new problem (see step 5). Then relaunch via `Agent` with the same ticket. Point the new agent at the existing worktree path (state should still be on disk). Tell it: `previous run closed without a PR. Pick up from current worktree state at /home/<user>/ilo-<num>. Don't restart from scratch.`
- If `completed` with a PR URL:
  1. Confirm the PR exists and CI is at least pending: `gh pr view <num> --json statusCheckRollup,labels`.
  2. Re-check that the Linear `mini` label was removed — if it sneaked back, remove again.
  3. **Automatic cleanup (always run, no exceptions):**
     ```
     git -C /home/<user>/ilo worktree remove --force /home/<user>/ilo-<num>
     rm -rf /home/<user>/.cache/cargo-target-<num> /tmp/target-<num>
     ```
     Run these unconditionally when the subagent finishes — whether the PR was opened, the run failed, or the agent self-stopped on a blocker. Stale worktrees and cargo target dirs are the single biggest source of disk-quota failures in the pipeline; never leave them behind.
  4. Scan the agent's final report for any newly-discovered language bug, regression, or block that isn't already tracked → file as a Linear ticket per step 5 before the next pass.

## 3. Top up to two-in-flight

If fewer than two fix-subagents are active:

a. **Find candidates.** Query Linear for backlog/unstarted issues, drop anything tagged `mini`, `mac`, `mini-reviewing`, `mac-reviewing`:

```graphql
{ issues(filter:{state:{type:{in:["backlog","unstarted"]}}}, first:200, orderBy:updatedAt){
    nodes{identifier title labels{nodes{name}} priority}
}}
```

b. **Skip risky / approval-required.**
- Labelled `P0-architectural` (design work, needs human).
- Title contains `Design:`, `Investigate:`, `Decide:`, `RFC` — needs product decisions.
- Body explicitly says "needs human", "ask human", "blocked on decision".
- Priority 0 with no diagnostic-class signal in the body (P0-bug ok; P0-architectural not ok).
- Already has an open PR (`gh pr list --search "in:title ILO-<n>"`).

c. **Of what remains, prefer:**
- P1/P2 over P4.
- Diagnostic / docs / parser-hint scope over runtime bug-hunts.
- Non-overlapping codebase areas vs the currently-running agent (parser vs lexer vs verifier vs registry) to minimise merge friction.

**Diagnostic-code allocation.** If the ticket will introduce a new `ILO-PXXX` / `ILO-TXXX` / `ILO-WXXX` / `ILO-RXXX` code, find the next free integer by looking at **both** the latest `main` *and* every open PR's diff against `main`. Two subagents in parallel can otherwise claim the same code (observed: ILO-460 and ILO-473 both reached for `ILO-P023`, requiring a renumber on the second PR). Tell the subagent the specific reserved-up-to integer in its prompt so it picks the next-free, e.g. "the next free parse code is `ILO-P024` — `ILO-P023` is taken by open PR #773; use `ILO-P024` or later."

d. **Claim and launch.** For each chosen ticket:

```graphql
mutation {
  addLabel: issueAddLabel(id:"<issueId>", labelId:"<miniLabelId>"){success}
  setState: issueUpdate(id:"<issueId>", input:{stateId:"<inProgressStateId>"}){success}
}
```

Then:
```
git -C /home/<user>/ilo fetch --quiet
git -C /home/<user>/ilo worktree add /home/<user>/ilo-<num> -b ilo-<num>-<slug> origin/main
```

Launch a background `Agent` with `run_in_background: true`, `subagent_type: general-purpose`. The prompt **always includes**:

- The ticket id, body, and link.
- The exact design decision for this ticket (you have to think this through — never tell the subagent "pick an approach"; pick it yourself).
- Worktree path + `CARGO_TARGET_DIR=/home/<user>/.cache/cargo-target-<num>`.
- "Do NOT edit ai.txt — it's auto-generated from SPEC.md by build.rs."
- "After tests pass: push, `gh pr create`, remove `mini` label from Linear and PR, do NOT add `mini-reviewing`."
- "If you hit a blocker, stop and report — don't paper over."

## 4. Fallback: no fix work available

If step 3 finds nothing claimable (all backlog tickets are risky / claimed / waiting-on-human), shift to **PR review/merge**:

a. **List PRs we should be reviewing.**
```
gh pr list --state open --json number,title,headRefName,labels,statusCheckRollup,reviewDecision --limit 50
```

For each open PR by our pipeline (head branch matches `ilo-<num>-*`):

i. **Claim it for review.** Add `mini-reviewing` label to the Linear ticket (look it up by branch name → ticket number). Do this only when actually about to review; release if you put it down.

ii. **Run the merge checklist.**
- All required CI checks passed (`statusCheckRollup` all `SUCCESS`). Skip merge if any are `PENDING`; come back next cycle.
- Code coverage didn't regress: pull codecov status from the PR check or `gh pr checks <num>`. If coverage dropped > 0.5% absolute and isn't justified in the PR body, comment asking for tests, then release the review claim and move on.
- Docs updated: check that any `--help`, `SPEC.md`, or `JSON_OUTPUT.md` text the PR body claims to add actually appears in the diff. `gh pr diff <num> | grep -E '^\+.*(SPEC|JSON_OUTPUT|help|--json)'`.
- Skill-tokens budget not blown: if the PR touches `skills/`, trust the CI `check-skill-tokens` job or rerun it locally.

iii. **If all green:**
- `gh pr merge <num> --squash --delete-branch`.
- Linear: set ticket state to `Done`, remove `mini-reviewing`.
- `git -C /home/<user>/ilo fetch --prune` to drop the merged branch ref locally.

iv. **If something fails:**
- Comment on the PR with the specific failure.
- Release the review claim (`mini-reviewing` off Linear, GitHub PR).
- File a Linear ticket (see step 5) if the failure looks like infra or systemic regression.

## 5. File issues as you go (automatic — never optional)

**Every** new problem, regression, or block the pass discovers must become a Linear ticket before the pass ends. This is the canonical recovery mechanism — anything not on the board is lost. Skipping this step is a P0 violation of the loop.

Auto-file a ticket when any of these happen in the pass:

- A subagent's final report mentions a language bug or regression unrelated to its ticket → new ticket capturing the symptom, repro shape, and where the agent encountered it.
- A subagent stopped on a real blocker (acceptance criteria unreachable without product input, design decision required, dependency missing) → new ticket re-scoped around the blocker; close-link the original ticket to it.
- A subagent failed with an infra / harness error (cargo target dir full, git lock, gh token expired, network timeout) → new ticket labelled `infra`; do NOT re-claim the original ticket until the infra ticket has a fix.
- CI infra failing for everyone (codecov offline, runners down) → new ticket labelled `infra`.
- A PR is unmergeable because of a missing required review → new ticket linking the PR.
- A worktree on disk doesn't match any Linear issue (orphan) → new chore ticket to investigate.
- A previously-merged PR introduced a regression discovered while reviewing a later PR → new ticket linked to both PRs, flagged with the regressing commit hash.

**De-dup first.** Search for an open ticket with the same symptom signature before creating one (`issues(filter:{title:{containsIgnoreCase:"<signature>"}, state:{type:{neq:"completed"}}})`); if one exists, comment on it instead of opening a duplicate.

**Create with:**
```graphql
mutation {
  issueCreate(input:{
    title:"<short symptom-led title>",
    description:"<context including ticket / PR links, exact error, repro shape, what was being attempted when discovered>",
    teamId:"<teamId>",
    priority: <0..4>,
    labelIds:[<one or more of: bug, infra, regression, block, chore>]
  }){ success issue{ identifier url } }
}
```

Resolve label ids by name; never hard-code. If a label doesn't exist yet, create it (`issueLabelCreate`) and proceed.

**Always quote-link to the discovering ticket / PR** in the new ticket's description so the next pass can trace the chain backward.

Never silently swallow a problem. If you don't have the context to write a good title, file the ticket with the shortest accurate description and a `triage` label rather than dropping it.

## 6. Report

End the pass with one block, max 10 lines, addressed to the human running the pipeline:

```
health-check @ <UTC time>
  running: ILO-461 (45m), ILO-457 (3m)
  drained: ILO-470 → PR 759 (clean), ILO-466 → PR 758 (clean)
  picked up: ILO-457
  pr review: PR 758 merged (CI green, cov +0.1%), PR 759 waiting on CI
  filed: ILO-481 (codecov upload timing out — infra)
  blocked: ILO-462 needs human (P0-architectural), skipped
```

If nothing changed this pass, say `quiet — 2 in flight, nothing to drain` and stop.

## Hard rules

- Never embed credentials in this file; read `LINEAR_API_KEY` / `GITHUB_TOKEN` from env.
- Never `--no-verify` or skip hooks.
- Never force-push, never amend; new commits only.
- Never cancel a running subagent without surfacing first.
- Never merge a PR with a red required check.
- Never touch a worktree that isn't the one for the ticket you're working.
- Never apply `mini-reviewing` to a fix-in-flight (it's review-only).
- Never edit `ai.txt`.
- Never invent a ticket priority / state if the Linear API call failed — fail loud, don't fall back to defaults.
