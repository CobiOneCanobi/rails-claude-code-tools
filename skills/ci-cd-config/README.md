# CI/CD Config Generator

**Scale:** greenfield — set up before the first PR

Generates a GitHub Actions workflow for a Rails API project: test suite with JUnit output, RuboCop lint, Brakeman + bundler-audit security checks, and a Docker build validation.

---

## When to use it

Day one or before the first pull request. Gets CI running correctly for your stack without copy-pasting from old projects or fighting with service container configuration.

For existing projects, the generated config is a starting point only — the database setup step (`db:schema:load`) assumes a greenfield project with no migrations yet, and any existing CI config with custom jobs or deployment steps will need manual adjustment.

---

## Prerequisites

- GitHub repository
- `rspec_junit_formatter` in your Gemfile — added automatically by `/rails-boilerplate`
- Docker installed locally (for validating the generated workflow before pushing)

---

## Installation

Copy the skill file into your project's `.claude/commands/` directory:

```bash
mkdir -p .claude/commands
cp ci-cd-config.md .claude/commands/ci-cd-config.md
```

---

## Usage

In a Claude Code session:

```
/ci-cd-config
```

Claude detects your Ruby version and database from existing project files — no questions needed in most cases. Shows the generated file before writing.

---

## What gets generated

```
.github/
  workflows/
    ci.yml
```

Four jobs, all running on `ubuntu-latest`:

| Job | What it runs |
|---|---|
| `test` | RSpec with JUnit XML output, uploaded as a build artifact. Spins up a database service container matching your stack (PostgreSQL or MySQL). |
| `lint` | RuboCop |
| `security` | Brakeman + bundler-audit |
| `docker` | `docker build --target production` — validates the Dockerfile builds cleanly without pushing |

All jobs use `ruby/setup-ruby` with `bundler-cache: true` for fast gem installs.

---

## Triggers

- Push to `main`
- Pull requests (all branches)

---

## Re-running

Safe to re-run. If `.github/workflows/ci.yml` already exists, shows a diff of what would change and asks for confirmation before writing.
