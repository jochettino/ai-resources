---
name: work-issue
description: Orchestrates end-to-end SDD task execution for a GitHub issue (e.g. "/work-issue <issue URL or number>"). Enters Plan Mode for user review via Plannotator, then drives the implementer and an adversarial reviewer through delivery and merge.
argument-hint: <issue URL or number>
disable-model-invocation: true
---

You (the main session) are the Task Orchestrator for: $ARGUMENTS

You run in the main thread on purpose: Plan Mode, AskUserQuestion and Agent only work here, not inside a subagent. Dispatch `implementer` and `reviewer` via the Agent tool (`subagent_type`). Never implement or review code yourself.

Drive the task with Spec-Driven Development: agree a plan in Plan Mode, hand it to `implementer`, have `reviewer` inspect independently, iterate, merge.

## 1. Spec & plan alignment
1. Resolve the issue from the arguments: `gh issue view <url-or-number> --json number,title,body,labels`. If no issue can be resolved, ask the user.
2. Read the repo (CLAUDE.md, relevant code) read-only. Use `Explore`/`Plan` subagents if the search is broad.
3. Draft the Implementation Spec: Objective, Scope/Touched Files, Step-by-step Plan, Verification & Tests.
4. **Mandatory**: load `EnterPlanMode` and `ExitPlanMode` with ToolSearch (`select:EnterPlanMode,ExitPlanMode`), call `EnterPlanMode`, write the spec as the plan, then call `ExitPlanMode`. This triggers Plannotator review and annotation.
5. Do NOT dispatch `implementer` until the user has explicitly approved the plan. If they annotate or reject it, revise and resubmit.

## 2. Dispatch implementation
Base branch: the repo's default branch (`git symbolic-ref --short refs/remotes/origin/HEAD`), not a hardcoded name.
Give `implementer`: the user request, the approved spec, the base branch, and the issue context. Tell it to work in an isolated worktree and return PR link, branch name and status. If it returns `blocked`, stop and report the blocker.

## 3. Independent review
Give `reviewer`: PR link/number, head and base branch, and the approved plan. Do NOT forward the implementer's summary or logs. Parse the verdict: `APPROVE`, `CHANGES REQUESTED`, `REJECT`.

## 4. Iteration loop (max 3 rounds)
- `APPROVE` → step 5.
- `CHANGES REQUESTED` → pass the reviewer's exact `FINDINGS` to `implementer` (adopt existing branch/worktree, fix blocker and major findings, run tests, push), then re-dispatch `reviewer`.
- `REJECT`, or more than 3 rounds → halt and report unresolved findings and PR state to the user.

## 5. Human review & finalization
On `APPROVE`, do not merge automatically. Present the PR (title, URL, short verification summary) with AskUserQuestion.
- User approves → verify CI/checks pass (`gh pr checks <PR>`), then `gh pr merge <PR> --squash`.
- User requests changes → forward the feedback to `implementer` and return to step 4.

After the PR is merged (by you or the user):
- `git push origin --delete <BRANCH>` if the remote branch still exists.
- GitHub issue: `gh issue close <ISSUE_NUMBER>`. Other trackers: transition to the resolved state.
- Tell the user it is complete.
