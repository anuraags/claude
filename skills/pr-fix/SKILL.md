---
name: pr-fix
description: Skill for fixing pull request based on reviews.
version: 1.0.0
---
 
# Pull Request Fix Skill

There should be a PR on the current branch containing an automated code review in a comment.

The comment should contain a header string "## Automated Code Review"

Please fix all the issues raised in the comment. Do not commit any code. I would like to review, commit, and push the fixes myself.

Once I have committed and fixed the push, update the comment in the PR by adding a rocket emoji next to the issue to indicate it has been addressed. Also include the fix that was applied at the bottom of the issue description.