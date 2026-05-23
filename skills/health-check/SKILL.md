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

- If `failed` or closed without a PR URL → relaunch via `Agent` with the same ticket. Point the new agent at the existing worktree path (state should still be on disk). Tell it: `previous run closed without a PR. Pick up from current worktree state at /home/<user>/ilo-<num>. Don't restart from scratch.`
- If `completed` with a PR URL:
  1. Confirm the PR exists and CI is at least pending: `gh pr view <num> --json statusCheckRollup,labels`.
  2. Re-check that the Linear `mini` label was removed — if it sneaked back, remove again.
  3. Clean up: `git worktree remove --force /home/<user>/ilo-<num>; rm -rf /home/<user>/.cache/cargo-target-<num> /tmp/target-<num>`.

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

## 5. File issues as you go

Any problem the pass discovers that isn't already tracked → file a Linear ticket immediately so the next pass can act on it. Examples:

- A subagent reported a real language bug while doing a different ticket → new ticket on the new bug.
- CI infra failing for everyone (codecov offline, runners down) → new ticket, label `infra`.
- A PR can't be merged because of a missing review from a specific reviewer → new ticket linking the PR.
- A worktree on disk doesn't match any Linear issue (orphan) → new chore ticket to investigate.

Create with:
```graphql
mutation {
  issueCreate(input:{
    title:"<short>",
    description:"<context including ticket / PR links and what was observed>",
    teamId:"<teamId>",
    priority: <0..4>
  }){ success issue{ identifier url } }
}
```

Never silently swallow a problem.

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
