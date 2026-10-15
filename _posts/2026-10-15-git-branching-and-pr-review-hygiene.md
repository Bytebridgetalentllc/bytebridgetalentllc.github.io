---
layout: post
title: "Git Branching and Pull Request Hygiene: A Checklist That Fits on One Screen"
date: 2026-10-15 11:05:00 +0530
description: "A one-screen checklist for Git branching and pull requests: short-lived branches, safe rebases, small PRs, labeled review comments and branch protection."
---

Most review pain doesn't come from the code. It comes from the shape of the change. Here's the workflow our team at ByteBridge Talent LLC keeps coming back to, in checklist form.

![Short-lived branches](/assets/git-short-lived-branches.png)

### Branches

- Branch from an up-to-date `main`, and keep branches short-lived (days, not weeks).
- Use a predictable name: `feat/login-rate-limit` or `fix/null-total-on-empty-cart`.
- Rebase onto `main` before asking for review, rather than merging `main` into your branch over and over:

```bash
git fetch origin
git rebase origin/main
# after a rebase, update the remote branch safely:
git push --force-with-lease
```

`--force-with-lease` refuses to overwrite the remote if someone else has pushed to it since your last fetch. Plain `--force` doesn't check.

### Commits

- One logical change per commit. Separate refactors from behaviour changes.
- Use an imperative subject line of about 50 characters, then a body that explains *why*.
- Tidy up work-in-progress commits before review:

```bash
git commit --fixup=<sha>          # mark a fix for an earlier commit
git rebase -i --autosquash origin/main
```

### The pull request

- Keep it small enough to review in one sitting. If the description needs the word "also", consider splitting it.
- The description should cover what changed, why, how it was tested and any risk or rollback notes.
- Draft PRs are for early feedback. Mark the PR ready only when CI passes.
- Add a template so nobody has to remember all this:

```markdown
<!-- .github/pull_request_template.md -->
## What
## Why
## How tested
## Risk / rollback
```

### Reviewing

- Review the design first, then the details. Don't nitpick a PR whose approach is wrong.
- Label your comments: `blocking:`, `suggestion:`, `nit:`. Authors then know what has to change.
- Let tools argue about formatting. A pre-commit hook or CI linter beats a human comment about whitespace.
- Approve when it's better than `main` and safe, not only when it's perfect.

### Guardrails on `main`

Turn on branch protection: required status checks, at least one approval, and no direct pushes. A `CODEOWNERS` file routes reviews automatically:

```text
# .github/CODEOWNERS
/infra/        @your-org/platform
/api/          @your-org/backend
*.md           @your-org/docs
```

### Merging

Pick one strategy and stick to it. Squash-merge gives one commit per PR on `main`, which keeps `git log` and `git revert` simple. If your commits are already clean and meaningful, rebase-merge keeps them. Delete the branch after merging.

Good hygiene here is mostly small habits, applied every time.

---

## Who's behind this blog

This checklist comes from the engineering side of ByteBridge Talent LLC. Since early 2025 we've worked out of Grand Island, Nebraska, placing cloud, DevOps and software engineers with teams that need to move faster. Some join on contract, some start contract-to-hire, and others are hired directly from day one. As a direct vendor to Medica Health Plans, we've helped staff more than 60 technology roles over about two years. You'll find more about us at [bytebridgetalent.com](https://bytebridgetalent.com).
