# Singularity Review Environment

Prepares a Singularity git worktree for local validation by aligning Node, Corepack, pnpm, and dependencies with the versions declared in the repository.

Use this skill before running build, lint, typecheck, or test commands during a Singularity PR review.

## Effective Prompt

For best results, include this in your prompt:

```text
/singularity-review-env for preparing the current git worktree for building and running tests
```
