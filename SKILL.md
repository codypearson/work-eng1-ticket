---
name: work-eng1-ticket
description: Work ENG1 Jira tickets end-to-end — assign, transition, branch, plan, implement, subtasks, commit, and open a GitHub PR. Use when the user explicitly invokes this skill to start, pick up, or work an ENG1 ticket.
disable-model-invocation: true
---

# Work ENG1 Ticket

Personal workflow for ENG1 Jira tickets across repos. Follow steps in order. Do not skip gates.

## Prerequisites

- Atlassian MCP available (`plugin-atlassian-atlassian`).
- Prefer GitHub MCP server `github` when available and authenticated. GitHub CLI is acceptable as a fallback if the GitHub MCP is not available.

## Setup (once per invocation)

1. Resolve `cloudId` via `getAccessibleAtlassianResources`.
2. Resolve current user via `atlassianUserInfo` (account id for assignee).
3. Load the parent ticket with `getJiraIssue` — include description, comments, issuetype, assignee, status.
4. List existing subtasks via `searchJiraIssuesUsingJql` (`parent = <ticket key>`). Match by summary before creating anything.
5. Discover ENG1 types/fields via `getJiraProjectIssueTypesMetadata` + `getJiraIssueTypeMetaWithFields` so **Deploy** issue type and **Release Note** custom field id are known at runtime (do not hardcode field ids).

## Subtask strategy

These four subtasks may already exist (often created with the parent). **Check first; never duplicate.** If a matching subtask exists, edit/transition it. Only `createJiraIssue` when missing.

| Summary | Behavior |
|---------|----------|
| **Development** | **In Progress** while implementing; **Done** when ready for review. |
| **Code Review** | Stays **To Do**. Description = testing instructions for the reviewing developer (code review **and** automated/manual testing). |
| **Stakeholder Review** | Decision point — ask the user. If needed: stay **To Do**, description = simple reproduction steps for stakeholders. If not: add a comment explaining why it is not needed, then set to **Done**. |
| **Deploy** | Stays **To Do**. Same as before (issue type **Deploy**, Release Note required). |

Match existing issues by summary (`Development`, `Code Review`, `Stakeholder Review`, `Deploy`). Deploy may also be identified by issue type **Deploy**.

## Workflow checklist

Copy and track progress:

```
ENG1 Ticket Progress:
- [ ] 1. Assign to me if not already
- [ ] 2. Set In Progress
- [ ] 3. Read description + comments
- [ ] 4. Checkout main/master and update
- [ ] 5. If Bug: reproduce (HTTP, or Debug + user for browser)
- [ ] 6. Create branch from default (ticket key lowercase)
- [ ] 7. Draft fix plan
- [ ] 8. HARD STOP — wait for user plan approval
- [ ] 9. Ensure Development → In Progress; implement
- [ ] 10. Test the fix
- [ ] 11. Development → Done (ready for review)
- [ ] 12–15. Code Review + Stakeholder Review decision + Deploy
- [ ] 16. Commit and push
- [ ] 17. Create GitHub PR (ticket key in title)
```

### 1–3. Ticket prep

- If assignee is not the current user, assign with `editJiraIssue`.
- Transition to **In Progress** via `getTransitionsForJiraIssue` → `transitionJiraIssue`.
- Read the full description and all comments before coding.

### 4–6. Repo prep and Bug reproduction

- Prefer `main`, else `master`. Checkout and pull so the default branch is up to date.
- Create a new branch off the default branch named with the ticket key in **lowercase** (e.g. `ENG1-123` → `eng1-123`). Create it before instrumentation when a browser-based Bug repro is needed.

If issuetype is **Bug**, determine the cause before drafting a fix plan:

- Prefer direct HTTP when that fully covers the issue (no browser required).
- If a **browser-based** reproduction is needed (do **not** use automated browser control):
  1. Switch to **Debug mode** (ask the user to switch if you cannot).
  2. Instrument the relevant code so the reproduction will leave readable evidence in logs.
  3. Give the user clear, step-by-step instructions to reproduce the issue manually.
  4. Wait for the user to reproduce; then read the logs / instrumentation output to determine what happened.
  5. Once the cause is determined, continue the workflow as before (plan → approval → implement).

### 7–8. Plan (hard stop)

1. Draft a concrete plan to fix the issue (for Bugs: only after the cause is known).
2. Present it to the user.
3. **Do not continue** until the user explicitly approves the plan.

### 9–11. Implement

- Ensure **Development** exists (create only if missing). Transition it to **In Progress**.
- Complete the necessary code changes (remove temporary instrumentation unless it belongs in the permanent fix).
- Test the fix. For browser-facing Bugs, prefer the same Debug + user-driven verification pattern (instructions for the user, then read logs) rather than automated browser control.
- When work is ready for review, transition **Development** to **Done**.

### 12–15. Closeout subtasks

Ensure each of the following (edit if present; create only if missing):

1. **Code Review** — stays **To Do**. Set description to detailed testing instructions targeted at the reviewing developer. Cover code review plus automated and/or manual testing needed to verify the work.
2. **Stakeholder Review** — **ask the user** whether stakeholder review is needed.
   - If **yes**: stay **To Do**; set description to simple reproduction steps targeted at stakeholders.
   - If **no**: add a comment explaining why stakeholder review is not needed, then transition the subtask to **Done**.
3. **Deploy** — use issue type **Deploy** (`issueTypeName: Deploy`) when creating. Stays **To Do**.
   - If deploy needs manual intervention (untracked config, log checks, new cron entries, etc.), document that in the Deploy description; otherwise leave description empty.
   - Set the **Release Note** field on the Deploy issue (required) via discovered field id in `additional_fields` / `editJiraIssue`.

### 16–17. Ship

- Commit (allowed when this skill is active) following the repo’s commit message style.
- `git push -u` the branch.
- Create the PR — prefer GitHub MCP `create_pull_request`; use `gh pr create` if the GitHub MCP is not available:
  - Derive `owner` and `repo` from `git remote get-url origin`.
  - MCP required args: `owner`, `repo`, `title`, `head`, `base` (plus body). **Never** call `create_pull_request` without those five args.
  - PR **title must include the ticket key** (e.g. `ENG1-123: …`).
- Return the PR URL to the user.

## Tool mapping

| Area | Tools |
|------|--------|
| Jira | `atlassianUserInfo`, `getAccessibleAtlassianResources`, `getJiraIssue`, `searchJiraIssuesUsingJql`, `editJiraIssue`, `getTransitionsForJiraIssue`, `transitionJiraIssue`, `createJiraIssue`, `addCommentToJiraIssue`, `getJiraProjectIssueTypesMetadata`, `getJiraIssueTypeMetaWithFields` |
| Git | checkout/pull, branch, commit, `git push -u` |
| GitHub | Prefer MCP `create_pull_request` on server `github`; fallback `gh pr create` |
| Bugs | Debug mode + code instrumentation + user-driven reproduction; read logs. Prefer HTTP when sufficient. Do **not** use automated browser control. |

## Rules

- Do not continue past planning without explicit user approval.
- Never create a duplicate of Development / Code Review / Stakeholder Review / Deploy — check existing subtasks first and edit.
- Development → **In Progress** while working → **Done** when ready for review.
- Code Review and Deploy stay **To Do**; Stakeholder Review stays **To Do** only if the user says it is needed, otherwise comment why it is not needed and set it to **Done**.
- Deploy description only when manual intervention is required; otherwise empty.
- Release Note is required on Deploy.
- Determine Bug causes before drafting a fix plan. Prefer HTTP when sufficient; for browser-based Bugs use Debug mode, instrument, user-driven repro, then read logs — never automated browser control.
- Prefer existing repo commit conventions; always include ticket key in PR title.
- Prefer GitHub MCP; GitHub CLI is acceptable as a fallback if the GitHub MCP is not available.
