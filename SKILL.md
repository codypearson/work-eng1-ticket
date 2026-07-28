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
4. Discover ENG1 types/fields via `getJiraProjectIssueTypesMetadata` + `getJiraIssueTypeMetaWithFields` so **Deploy** issue type and **Release Note** custom field id are known at runtime (do not hardcode field ids).

## Workflow checklist

Copy and track progress:

```
ENG1 Ticket Progress:
- [ ] 1. Assign to me if not already
- [ ] 2. Set In Progress
- [ ] 3. Read description + comments
- [ ] 4. Checkout main/master and update
- [ ] 5. If Bug: reproduce (browser or HTTP)
- [ ] 6. Create branch from default (ticket key lowercase)
- [ ] 7. Draft fix plan
- [ ] 8. HARD STOP — wait for user plan approval
- [ ] 9. Create work subtasks; keep statuses current → Done
- [ ] 10. Implement the fix
- [ ] 11. Test the fix
- [ ] 12–15. Create Review & Test + Deploy (To Do); Release Note on Deploy
- [ ] 16. Commit and push
- [ ] 17. Create GitHub PR (ticket key in title)
```

### 1–3. Ticket prep

- If assignee is not the current user, assign with `editJiraIssue`.
- Transition to **In Progress** via `getTransitionsForJiraIssue` → `transitionJiraIssue`.
- Read the full description and all comments before coding.

### 4–5. Repo prep

- Prefer `main`, else `master`. Checkout and pull so the default branch is up to date.
- If issuetype is **Bug**, reproduce before changing code:
  - Prefer direct HTTP when that fully covers the issue.
  - If the ticket reproduction includes a browser UI aspect, exercise it via a controlled browser.
  - If a test account is needed, locate a random account and reset the password if necessary to enable login.

### 6. Branch

Create a new branch off the default branch named with the ticket key in **lowercase** (e.g. `ENG1-123` → `eng1-123`).

### 7–8. Plan (hard stop)

1. Draft a concrete plan to fix the issue.
2. Present it to the user.
3. **Do not continue** until the user explicitly approves the plan.

### 9–11. Implement

- Create work subtasks on the parent with `createJiraIssue` (`projectKey: ENG1`, `parent: <ticket key>`, appropriate issue type).
- Transition work subtasks as work proceeds; they should end **Done**.
- Complete the necessary code changes.
- Test the fix (same browser/HTTP approach as reproduction when applicable).

### 12–15. Closeout subtasks

Create two subtasks that remain in **To Do**:

1. **Review & Test** — description must include detailed testing instructions for verifying the work.
2. **Deploy** — use issue type **Deploy** (`issueTypeName: Deploy`).
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
| Jira | `atlassianUserInfo`, `getAccessibleAtlassianResources`, `getJiraIssue`, `editJiraIssue`, `getTransitionsForJiraIssue`, `transitionJiraIssue`, `createJiraIssue`, `getJiraProjectIssueTypesMetadata`, `getJiraIssueTypeMetaWithFields` |
| Git | checkout/pull, branch, commit, `git push -u` |
| GitHub | Prefer MCP `create_pull_request` on server `github`; fallback `gh pr create` |
| Browser | Controlled browser (cursor-ide-browser) when reproduction/verification has a UI aspect |

## Rules

- Do not continue past planning without explicit user approval.
- Work subtasks → **Done**; Review & Test and Deploy stay **To Do**.
- Deploy description only when manual intervention is required; otherwise empty.
- Release Note is required on Deploy.
- Reproduce Bugs before coding.
- If the ticket reproduction includes a browser UI aspect, exercise it via a controlled browser.
- If a test account is needed, locate a random account and reset the password if necessary to enable login.
- Prefer existing repo commit conventions; always include ticket key in PR title.
- Prefer GitHub MCP; GitHub CLI is acceptable as a fallback if the GitHub MCP is not available.
