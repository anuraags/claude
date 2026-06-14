---
name: pr-summariser
description: Update a PR description with a minimal, concrete summary of what the PR implements and what needs testing. Use when the user asks to summarise a PR, write/update a PR description, or prepare a PR for review.
---

# PR Summariser

Write or update the description of a pull request in as few words as possible, while staying concrete. The reader is a junior developer who has not seen this code before.

## Steps

1. Identify the PR. If the user gave a PR number or URL, use it. Otherwise find the PR for the current branch:
   ```
   gh pr view --json number,title,body,baseRefName,headRefName,url
   ```
   If no PR exists, tell the user and stop.

2. Read the actual changes — do not rely on commit messages, the PR title, or the existing description alone:
   ```
   gh pr diff <number>
   ```
   For large diffs, look at the file list first (`gh pr diff <number> --name-only`) and read the substantive files.

3. Prompt for a JIRA ticket number from the user to identify the scope of the PR. You can use the atlassian cli to fetch the ticket title and description.

3. Write the description following the rules and format below.

4. Show the proposed description to the user, then update the PR:
   ```
   gh pr edit <number> --body "..."
   ```
   Preserve any sections the existing description has that you didn't write (e.g. ticket links, checklists required by the repo template) — keep them, don't delete them.

## Writing rules

- **Two paragraphs followed by bullets.** The first paragraph is a one-line summary of the problem or goal. The second is a one-line summary of the solution. Each paragraph should be no longer than 8 sentences. Then the bullets are the specific changes, implementation, and testing steps.
- **Fewest words that still inform.** Cut filler ("This PR aims to...", "In order to..."). Every sentence must tell the reader something they'd otherwise have to find in the diff.
- **No abstractions or vague verbs.** Avoid "refactors the data layer", "improves robustness", "enhances the workflow". Instead say what actually changed: name the files, functions, tables, endpoints, or flags. "Adds a `property_locator_token` column to `jobs` and populates it in `create_job()`" beats "adds token persistence".
- **Assume a junior developer.** Spell out names rather than internal shorthand. If a domain term is unavoidable, define it in half a sentence. Don't assume they know why the change was needed — state the problem in one line.
- **Plain sentences and short bullets.** No marketing tone, no emoji, no nested heading hierarchy.

## Description format

```markdown
## What does this PR do?

Two paragraphs: the problem or goal and the solution. Each paragraph should be no longer than 8 sentences. Be concrete, not abstract. Avoid filler and vague verbs. Assume the reader is a junior developer who hasn't seen this code before.

Then 2-8 bullets, each naming a concrete change (file/function/endpoint/schema/behaviour).

## What needs to be tested?

2-5 bullets: the specific behaviours a reviewer should exercise or watch for.
Include edge cases the diff touches (null paths, migrations, error handling),
and the exact command/endpoint/page to use where known.

Closes ticket [TICKET-123](https://example.atlassian.net/browse/TICKET-123).

```

Omit anything that doesn't fit those two sections unless the repo's PR template requires it.

## Example

Bad (abstract):
> This PR enhances the job pipeline by introducing a robust token persistence mechanism for improved property resolution.

Good (concrete):
> ## What does this PR do?
> Jobs lost their property locator token on retry, so retried jobs resolved to the wrong property.
> - Adds `property_locator_token` column to the `jobs` table (migration `0042`).
> - `create_job()` now stores the token; `retry_job()` reads it back instead of re-deriving it.
>
> ## How to test
> - Create a job, retry it, confirm it resolves to the same property as the original run.
> - Run the migration against a copy of staging — `jobs` rows predating it have a NULL token; retrying one of those should fall back to the old derivation path.
