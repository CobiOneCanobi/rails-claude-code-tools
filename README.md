# rails-claude-code-tools

Rails API projects have a lot of day-one setup that's repetitive and easy to get wrong — project structure, Gemfile decisions, Docker config, CI pipelines, CLAUDE.md conventions. These skills encode the right decisions upfront so Claude works with context from the start, rather than re-explaining your stack every session.

Each skill is independently adoptable — pick what's relevant to your project's stage.

---

## How it works

Skills are Markdown files that Claude Code loads as slash commands. Copy the `.md` file from any tool directory into your project's `.claude/commands/` folder and invoke it in a session.

Here's what running `/rails-boilerplate` on a fresh Rails app looks like:

```
Claude: A few questions before generating:
        1. Database — PostgreSQL or MySQL?
        2. Auth — JWT / Devise / API key / none / undecided?
        3. Background jobs — Sidekiq / GoodJob / none / undecided?
        4. Serialization — jsonapi-serializer / jbuilder / none / undecided?

You:    PostgreSQL, JWT, Sidekiq, jsonapi-serializer
```

Generated in a single pass:

```
Gemfile                                    — pg, jwt, bcrypt, sidekiq, pagy, rspec-rails,
                                             factory_bot_rails, brakeman, bundler-audit, and more
app/services/application_service.rb
app/errors/application_error.rb
app/controllers/concerns/loggable.rb
app/controllers/application_controller.rb  — rescue_from ApplicationError, Loggable included
config/database.yml                        — DATABASE_URL-first, matches your DB choice
Dockerfile                                 — multi-stage: development + production targets
docker-compose.yml                         — app, db, redis services
.env + .env.example                        — pre-filled from your answers
bin/check_env_completeness                 — pre-commit script: flags ENV keys missing from .env.example
lefthook.yml                               — RuboCop on commit, ENV check on commit, RSpec on push
spec/spec_helper.rb + rails_helper.rb
spec/support/                              — FactoryBot, Shoulda Matchers, WebMock, Timecop
CLAUDE.md                                  — seeded with your stack choices and convention markers
DOCKER.md                                  — Docker command reference
```

Safe to re-run — checks for existing files and fills gaps only.

---

## New Rails API project

Start with:

```bash
rails new my_app \
  --api \
  --database=postgresql \
  --skip-test \
  --skip-jbuilder \
  --skip-action-mailer \
  --skip-action-mailbox \
  --skip-action-text \
  --skip-active-storage
cd my_app
```

`--api` strips middleware and view layers. `--skip-test` omits Minitest — the boilerplate installs RSpec instead. `--skip-jbuilder` avoids a duplicate if you choose jbuilder during `/rails-boilerplate`. Adjust `--database` to `mysql` if needed. The `--skip-*` flags on the last line drop email, rich text, and file upload infrastructure — remove any you intend to use.

Then run these skills in order:

| Step | Skill                | What it does                                                                                                          |
| ---- | -------------------- | --------------------------------------------------------------------------------------------------------------------- |
| 1    | `/rails-boilerplate` | ApplicationService, ApplicationError, Loggable, curated Gemfile, Dockerfile, docker-compose, Lefthook, base CLAUDE.md |
| 2    | `/project-claude-md` | Interview-driven generator for a tailored root CLAUDE.md — conventions, patterns, never-dos                           |
| 3    | `/ci-cd-config`      | GitHub Actions: RSpec with JUnit output, RuboCop, Brakeman, bundler-audit, Docker build                               |

---

## Existing Rails project

Skip straight to the skills catalog. The worker generator and worker conventions template are designed to drop into a codebase that already has patterns in place — no boilerplate required.

---

## Skills

| Tool                                                                                                                                                                                                                          | Scale      |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- |
| [Rails boilerplate generator](skills/rails-boilerplate/) — production-ready base layer for a Rails API project                                                                                                                | greenfield |
| [Project CLAUDE.md generator](skills/project-claude-md/) — tailored CLAUDE.md from a developer interview                                                                                                                      | greenfield |
| [CI/CD config](skills/ci-cd-config/) — GitHub Actions pipeline wired for Rails                                                                                                                                                | greenfield |
| [Sidekiq worker generator](skills/worker-generator/) — scalar args, service delegation, retry options, spec stub. Includes a [worker conventions template](skills/worker-generator/workers-CLAUDE.md) for `app/workers/CLAUDE.md` | at scale |

**Labels:**
- `greenfield` — useful from day one
- `at scale` — pays off once the codebase has grown

---

## Also worth knowing

**Claude Code built-ins** — no installation required:

- `/review` — PR review and description
- `/security-review` — security scan before merging

**MCP servers** — external, require installation:

- [GitHub MCP](https://github.com/github/github-mcp-server) — issues, PRs, and branches as native session actions
- [rails-mcp-server](https://github.com/maquina-app/rails-mcp-server) — structured access to routes, models, and schema

**Rails tooling** — gems and tools that pair well with these skills:

- **[Brakeman](https://brakemanscanner.org/)** — static security analysis. Catches mass assignment, SQL injection, hardcoded secrets. Wired into CI by `/ci-cd-config`.
- **[bundler-audit](https://github.com/rubysec/bundler-audit)** — checks Gemfile.lock against known CVEs. Also in CI via `/ci-cd-config`.
- **[strong_migrations](https://github.com/ankane/strong_migrations)** — catches dangerous migrations before they run: adding columns with defaults, removing columns, missing indexes on large tables.
- **[database_consistency](https://github.com/djezzzl/database_consistency)** — schema health checks: missing FK indexes, nullable columns without defaults, missing FK constraints.
- **[RuboCop](https://github.com/rubocop/rubocop) + [rubocop-rails-omakase](https://github.com/rails/rubocop-rails-omakase)** — code style enforcement. Pre-commit via Lefthook, generated by `/rails-boilerplate`.
- **[Bullet](https://github.com/flyerhzm/bullet)** — detects N+1 queries and missing eager loading at runtime.
- **[rack-mini-profiler](https://github.com/MiniProfiler/rack-mini-profiler)** — request performance profiling. Pairs well with Bullet for tracking down slow endpoints.
- **[debride](https://github.com/seattlerb/debride)** — dead code detection. Rails metaprogramming creates false positives — treat output as a starting point.
