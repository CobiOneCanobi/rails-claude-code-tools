# Project CLAUDE.md Generator

**Scale:** greenfield — run early in a new project, before writing application code

Interviews the developer and generates (or fills in) a tailored root `CLAUDE.md` for a Rails API project.

---

## When to use it

Two modes, detected automatically:

**Fill-in mode** — you ran `/rails-boilerplate` and have a `CLAUDE.md` with `[FILL IN: ...]` markers. This skill interviews you and replaces each marker with real content. The structural sections the boilerplate wrote (Service Objects, Error Handling, Logging, Docker) are left untouched.

**Standalone mode** — no `CLAUDE.md` exists yet, or you started a project without the boilerplate. This skill reads your project files to detect the stack, interviews you, and generates a complete `CLAUDE.md` from scratch.

In both modes, run it before writing any application code. The conventions section is what Claude reads to stay consistent across every future session.

---

## Prerequisites

- A Rails API project (existing `Gemfile` and `app/` directory)
- You don't need all decisions made upfront — in fill-in mode the skill shows you the unfilled sections and asks which ones you're ready to tackle. You can fill in one section now and come back for the rest later.

---

## Installation

Copy the skill file into your project's `.claude/commands/` directory:

```bash
mkdir -p .claude/commands
cp project-claude-md.md .claude/commands/project-claude-md.md
```

---

## Usage

In a Claude Code session:

```
/project-claude-md
```

Claude will detect the mode, read your project files, ask targeted questions, then show you the result before writing.

---

## What it asks

**In fill-in mode** — only the sections that have `[FILL IN: ...]` markers:

| Section | Questions |
|---|---|
| Testing | What gets unit-tested vs integration-tested? What's sufficient coverage for a PR? Any SimpleCov threshold? |
| API Responses | JSON structure (plain hash / JSONAPI / custom envelope)? Pagination format? Error envelope shape? |
| Conventions | Serializer approach and naming? Any query objects or form objects? Other patterns to enforce? |
| Background Jobs | Queue names and purpose? Retry strategy? Any worker rules (idempotency, no enqueue-in-transaction)? |
| Auth | How is auth enforced across controllers? Public endpoints? Token/session approach? |
| Never-dos | Any project-specific commands or actions Claude should never take? |

**In standalone mode** — everything above, plus detected stack details you confirm:

| Section | Source |
|---|---|
| Ruby version | Read from `.ruby-version` |
| Rails version | Read from `Gemfile` |
| Database | Read from `config/database.yml` |
| Auth approach | Read from `Gemfile` (devise / jwt / bcrypt / none) |
| Background jobs | Read from `Gemfile` (sidekiq / good_job / none) |
| Serialization | Read from `Gemfile` (jsonapi-serializer / alba / jbuilder / none) |
| ApplicationService / ApplicationError / Loggable | Checked by file existence — sections only included if present |

---

## What gets generated

In standalone mode, sections are conditional:

| Section | Condition |
|---|---|
| Stack | Always |
| Service Objects | Only if `app/services/application_service.rb` exists |
| Error Handling | Only if `app/errors/application_error.rb` exists |
| Logging | Only if `app/controllers/concerns/loggable.rb` exists |
| Auth | Only if Devise / JWT / bcrypt detected |
| Migrations | Always — pre-filled with universal Rails migration rules |
| Background Jobs | Only if Sidekiq or GoodJob detected |
| CORS | Always |
| What Claude Should Never Do | Always |
| Testing | Always — filled from interview |
| API Responses | Always — filled from interview |
| Conventions | Always — filled from interview |

**Fill-in mode** — lists the unfilled sections found, asks which ones you want to tackle now, interviews you on those only, then shows a diff before writing. Sections you skip stay as `[FILL IN:]` markers — re-run the skill when you're ready for them.

**Standalone mode** — shows the full generated file, asks for confirmation before writing.

---

## Relationship to rails-boilerplate

`/rails-boilerplate` generates a `CLAUDE.md` shell — structural sections pre-filled, project-specific sections left as `[FILL IN: ...]` markers. It ends by telling you to fill those in before writing code.

`/project-claude-md` is the natural next step: it fills those in. If you didn't run the boilerplate, it generates a complete file instead.

The two skills do not conflict. The boilerplate intentionally defers the project-specific decisions to this skill.
