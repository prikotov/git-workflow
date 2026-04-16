# prikotov/git-workflow

Git workflow standards: branches, commits, PRs, releases, and code review.

## What is it

A set of conventions and rules for Git-based development workflow. Covers branching strategy, commit message format (Conventional Commits), pull request process, code review checklist, release management, and deployment guidelines.

## Contents

- [Index](docs/index.md) — table of contents
- [Branches](docs/branches.md) — branch types, naming, lifecycle
- [Commits](docs/commits.md) — Conventional Commits format and rules
- [Pull Request](docs/pull-request.md) — PR process and requirements
- [Code Review](docs/code-review.md) — review checklist and rules
- [Release](docs/release.md) — SemVer, release model, CHANGELOG
- [Deploy](docs/deploy.md) — deployment rules
- [Release Checklists](docs/release-checklists.md) — checklists for release, hotfix, deploy
- [Release Artifacts](docs/releases/index.md) — release plan template and storage rules

## Usage

Reference these documents from your project's `AGENTS.md` or include them directly:

```
Git workflow rules: vendor/prikotov/git-workflow/docs/
```

Customize scope examples in `docs/commits.md` for your project's module/app structure.

## License

MIT
