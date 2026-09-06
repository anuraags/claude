---
name: checking-changes-against-jira
description: Use when reviewing your own code changes before committing or opening a PR, to confirm they're linked to a JIRA ticket, that the ticket has a clear scope and definition of done, and that the diff doesn't exceed what the ticket describes.
---

# Checking Changes Against JIRA

## Overview

Verify code changes trace back to a well-specified JIRA ticket and stay inside its
scope. Uses the `acli` CLI (`acli jira workitem ...`) already installed on PATH.
This is a **report-only** check — surface findings, never auto-fix, amend, or commit.

## When to Use

- Before committing, or before opening/updating a PR, on a feature branch.
- NOT on a shared/integration branch (`master`, `main`, `qa`, `develop`, `staging`)
  with no local changes — there is no single "the changes" to check. Ask the user
  which commit range or ticket they mean instead of guessing from history.

## Step 1: Determine the diff to check

In order, first match wins:
1. Uncommitted work: `git status` + `git diff HEAD` (staged and unstaged). If
   non-empty, this is "the changes."
2. If the working tree is clean and the current branch is a feature branch (not one
   of the shared branches above): `git merge-base <default-branch> HEAD`, then
   `git diff <merge-base>...HEAD`.
3. If the working tree is clean AND the current branch is itself a shared branch:
   stop and ask the user what to compare (a commit range, PR number, or ticket key).
   Do not go digging through git log / GitHub PR history to reconstruct intent —
   that's expensive and produces guesses, not answers.

## Step 2: Find the ticket key

In order, first match wins:
1. Ticket key explicitly given by the user.
2. Current branch name, e.g. `AP-123-short-desc` → key matches `^[A-Z]+-\d+`.
3. Commit subjects within the diff's commit range (`git log <range> --oneline`),
   matching `AP-123:` or `[AP-123]` style prefixes.
4. None found → report **"changes are not linked to a JIRA ticket"** and stop there.
   Do not fall back to keyword/JQL search across the Jira project to guess a ticket
   — an unconfirmed guess is worse than reporting "not linked."

## Step 3: Fetch the ticket

Run both — plain mode renders the description as readable text (JSON mode returns
raw ADF, not usable directly):

```
acli jira workitem view <KEY> --fields summary,description,issuetype,status
acli jira workitem view <KEY> --fields issuetype,parent,subtasks,labels --json
```

| Need | Field | Source |
|---|---|---|
| Readable description | `description` | plain-mode output |
| Is it an Epic / has an Epic parent | `issuetype`, `parent` | `--json` |
| Broken into subtasks (scope signal) | `subtasks` | `--json` |

## Step 4: Judge ticket quality

Flag the ticket rather than proceeding if any of these hold:

- **Underspecified**: description is empty, or under ~2 sentences, or has no
  identifiable outcome/acceptance-criteria statement (no "as a user I want...",
  no bullet list of criteria, no done/complete condition — just a title restated).
- **Scope too large**: `issuetype` is Epic, or the ticket has one or more
  `subtasks`, or the description enumerates multiple independent deliverables
  (e.g. several unrelated features/areas in one description).

Name the specific missing element or signal — not just "looks thin."

## Step 5: Compare the diff to ticket scope

For each changed file/hunk in the Step 1 diff, judge whether it's plausibly covered
by the ticket's description. Flag ones that aren't — e.g. changes to an unrelated
module, config, or feature area the ticket never mentions. Stay inside the diff
determined in Step 1; don't widen the check to unrelated history.

## Report Format

Always report in this order:

1. **Ticket**: `KEY — summary` (or "not linked to a ticket" per Step 2.4, then stop).
2. **Ticket quality**: `pass` / `underspecified: <missing element>` /
   `scope-too-large: <signal>`.
3. **Out-of-scope changes**: `file → why it's outside the ticket's stated scope`,
   or `None`.

## Boundaries

- Report only. Never edit code, amend commits, or push based on this check.
- Never search Jira by free-text/JQL to guess a ticket when none is linked.
- Never reconstruct scope from broad git/GitHub archaeology — use the bounded diff
  from Step 1 only.
