# prikotov/git-workflow

Git workflow standards: branches, commits, PRs, releases, and code review.

## What is it

A set of conventions and rules for Git-based development workflow. Covers branching strategy, commit message format (Conventional Commits), pull request process, code review checklist, release management, and deployment guidelines.

## Contents

- [Index](docs/git-workflow/index.md) — table of contents
- [Branches](docs/git-workflow/branches.md) — branch types, naming, lifecycle
- [Commits](docs/git-workflow/commits.md) — Conventional Commits format and rules
- [Pull Request](docs/git-workflow/pull-request.md) — PR process and requirements
- [Code Review](docs/git-workflow/code-review.md) — review checklist and rules
- [Release](docs/git-workflow/release.md) — SemVer, release model, CHANGELOG
- [Deploy](docs/git-workflow/deploy.md) — deployment rules
- [Release Checklists](docs/git-workflow/release-checklists.md) — checklists for release, hotfix, deploy
- [Release Artifacts](docs/git-workflow/releases/index.md) — release plan template and storage rules

## Installation

```bash
composer require --dev prikotov/git-workflow
```

## Initialisation

To copy git workflow documentation into your project:

```bash
php vendor/bin/git-workflow-init
```

Or specify a target directory:

```bash
php vendor/bin/git-workflow-init /path/to/project
```

This copies `docs/` into `docs/git-workflow/` in your project. Existing files are never overwritten.

> **Note:** Re-running the command is safe.

## Reference (in package)
