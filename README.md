# prikotov/git-workflow

Git workflow standards: branches, commits, PRs, releases, and code review.

## What is it

A set of conventions and rules for Git-based development workflow. Covers branching strategy, commit message format (Conventional Commits), pull request process, code review checklist, release management, and deployment guidelines.

## Contents

- [Index](doc/index.md) — table of contents
- [Branches](doc/branches.md) — branch types, naming, lifecycle
- [Commits](doc/commits.md) — Conventional Commits format and rules
- [Pull Request](doc/pull-request.md) — PR process and requirements
- [Code Review](doc/code-review.md) — review checklist and rules
- [Release](doc/release.md) — SemVer, release model, CHANGELOG
- [Deploy](doc/deploy.md) — deployment rules
- [Release Checklists](doc/release-checklists.md) — checklists for release, hotfix, deploy

## Usage

Reference these documents from your project's `AGENTS.md` or include them directly:

```
Git workflow rules: vendor/prikotov/git-workflow/doc/
```

Customize scope examples in `doc/commits.md` for your project's module/app structure.

## License

MIT
