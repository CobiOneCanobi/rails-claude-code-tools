---
name: project-claude-md
description: Interview the developer and generate (or fill in) a tailored root CLAUDE.md for a Rails API project. Detects whether a CLAUDE.md already exists — if so, fills in only the [FILL IN: ...] markers; if not, generates a complete file from scratch.
---

## Step 1 — Detect mode

Read `CLAUDE.md` at the project root.

- **If the file exists:** scan it for lines containing `[FILL IN:`. Count them. Tell the user: "Found CLAUDE.md with N unfilled sections. I'll interview you and fill those in — everything else stays as-is." Proceed to **Fill-in mode**.
- **If the file does not exist:** tell the user: "No CLAUDE.md found. I'll read your project files, confirm the stack with you, then generate a complete one." Proceed to **Standalone mode**.

---

## Fill-in mode

### Step 2A — Read existing file

Read the full `CLAUDE.md`. Identify which sections contain `[FILL IN: ...]` markers. The sections that may have markers:

- **Testing** — `[FILL IN: what gets unit tested, what gets integration tested]` or `[FILL IN: what counts as sufficient coverage for a PR]`
- **API Responses** — `[FILL IN: JSON structure, pagination format, error envelope]`
- **Conventions** — `[FILL IN: serializer approach, query objects, additional patterns]`
- **Never-dos** — `[FILL IN: project-specific never-dos]`
- **Background Jobs** — `[FILL IN: queue names, retry strategy, dead letter handling]`
- **Auth** — `[FILL IN: how authentication is enforced across controllers]`

Only proceed with sections that actually have markers. Do not ask about sections that are already filled in.

### Step 3A — Choose sections

List the unfilled sections found and ask which ones the developer wants to fill in now. Example:

> Found 4 unfilled sections: **Testing**, **API Responses**, **Background Jobs**, **Conventions**
> Which would you like to fill in now? (You can name specific ones, or say "all" — anything you skip stays as-is)

Wait for the answer. Only proceed with the sections the developer selected.

### Step 4A — Interview (fill-in)

Ask only about the selected sections. Present all questions at once and wait for answers before proceeding. Do not ask follow-ups or break this into multiple rounds.

**If Testing has markers, ask:**
> **Testing**
> 1. What gets unit-tested vs integration-tested? (e.g., "services unit, controllers integration", "all request specs, unit tests only for complex logic")
> 2. What counts as sufficient coverage for a PR? (e.g., "happy path + main error path", "all public methods", "no threshold — judgment call")
> 3. Do you enforce a SimpleCov minimum? If yes, what percentage?

**If API Responses has markers, ask:**
> **API Responses**
> 1. JSON structure — plain hash, JSONAPI, or a custom envelope? If custom, describe the shape briefly (e.g., `{ data: ..., meta: ... }`).
> 2. Pagination — how is it represented in responses? (e.g., Pagy metadata in a `meta` key, Link headers, cursor-based, not yet decided)
> 3. Error envelope — what does an error response look like? (e.g., `{ error: "message" }`, `{ errors: [...] }`, `{ error: { code: ..., message: ... } }`)

**If Conventions has markers, ask:**
> **Conventions**
> 1. Serializer — what's the approach and naming convention? (e.g., "jsonapi-serializer, named UserSerializer in app/serializers/", "jbuilder partials in app/views/", "not decided yet")
> 2. Query objects — do you use them? If so, where do they live and what's the naming convention?
> 3. Any other patterns to enforce? (e.g., form objects, decorators, service naming conventions, file placement rules)

**If Background Jobs has markers, ask:**
> **Background Jobs**
> 1. Queue names in use and what each handles (e.g., "default, mailers, critical")
> 2. Retry strategy — default retries, or per-worker overrides? Any dead-letter queue handling?
> 3. Anything else workers should or shouldn't do? (e.g., "always idempotent", "never enqueue from within a transaction")

**If Auth has markers, ask:**
> **Auth**
> 1. How is authentication enforced across controllers? (e.g., "before_action :authenticate_user! in ApplicationController with opt-out", "explicit per-controller")
> 2. Any public endpoints that explicitly skip auth? How are they marked?
> 3. Anything else Claude should know about how auth works? (e.g., "admin vs user scopes", "token refresh handled by middleware")

**If Never-dos has markers, ask:**
> **Never-dos**
> What should Claude never do in this project, beyond the general Rails defaults? (e.g., "never touch the billing module without asking", "never run rake tasks directly — use the wrapper scripts", "never change serializer field names without checking the API docs")

### Step 5A — Fill in and show diff

For each section with markers, replace the `[FILL IN: ...]` placeholders with content derived from the developer's answers. Follow these rules:

- Preserve the section header and surrounding content exactly — only replace the `[FILL IN: ...]` lines themselves
- Write in the same register as the rest of the file — brief, imperative, no prose
- If the developer said "not decided yet" or equivalent for a section, replace the marker with: `[Not yet decided — fill in before writing code that depends on this]`
- Do not add new sections, do not reorder sections, do not reformat the file

Show a diff of the changes before writing — old lines marked with `-`, new lines with `+`. Ask: "Ready to write this? (yes/no)"

If the user says yes, write the file. If no, ask what to change and repeat.

---

## Standalone mode

### Step 2B — Read project files

Read these files to infer the stack. Do not run shell commands — read the files directly.

1. `.ruby-version` — Ruby version
2. `Gemfile` — Rails version (grep for `gem "rails"`), database adapter (`pg`, `mysql2`, `sqlite3`), auth (`devise`, `jwt`, `bcrypt`), background jobs (`sidekiq`, `good_job`), serialization (`jsonapi-serializer`, `alba`, `jbuilder`)
3. `config/database.yml` — confirm database adapter

Also check whether these files exist (do not read their contents — existence is enough):
- `app/services/application_service.rb` → note as **has_application_service**
- `app/errors/application_error.rb` → note as **has_application_error**
- `app/controllers/concerns/loggable.rb` → note as **has_loggable**

Summarize what you detected. Example:

```
Detected:
- Ruby 3.3.0 (from .ruby-version)
- Rails 8.0.1 (from Gemfile)
- PostgreSQL (from database.yml)
- Devise (from Gemfile)
- Sidekiq (from Gemfile)
- No serializer gem found
- ApplicationService, ApplicationError, Loggable present
```

Ask the developer to confirm or correct this before proceeding.

### Step 3B — Interview (standalone)

After confirming the stack, ask the project-specific questions. Group them and present all at once — wait for answers before proceeding.

**Always ask:**

> **Testing**
> 1. What gets unit-tested vs integration-tested? (e.g., "services unit, controllers integration", "all request specs, unit tests only for complex logic")
> 2. What counts as sufficient coverage for a PR? (e.g., "happy path + main error path", "all public methods", "no threshold — judgment call")
> 3. Do you enforce a SimpleCov minimum? If yes, what percentage?

> **API Responses**
> 1. JSON structure — plain hash, JSONAPI, or a custom envelope? If custom, describe the shape briefly (e.g., `{ data: ..., meta: ... }`).
> 2. Pagination — how is it represented in responses? (e.g., Pagy metadata in a `meta` key, Link headers, cursor-based, not yet decided)
> 3. Error envelope — what does an error response look like? (e.g., `{ error: "message" }`, `{ errors: [...] }`, `{ error: { code: ..., message: ... } }`)

> **Conventions**
> 1. Serializer — what's the approach and naming convention? (e.g., "jsonapi-serializer, named UserSerializer in app/serializers/", "jbuilder partials in app/views/", "not decided yet")
> 2. Query objects — do you use them? If so, where do they live and what's the naming convention?
> 3. Any other patterns to enforce? (e.g., form objects, decorators, service naming conventions, file placement rules)

> **Never-dos**
> What should Claude never do in this project, beyond the general Rails defaults? (e.g., "never touch the billing module without asking", "never run rake tasks directly — use the wrapper scripts", "never change serializer field names without checking the API docs")

**Ask only if Sidekiq or GoodJob was detected:**

> **Background Jobs**
> 1. Queue names in use and what each handles (e.g., "default, mailers, critical")
> 2. Retry strategy — default retries, or per-worker overrides? Any dead-letter queue handling?
> 3. Anything else workers should or shouldn't do? (e.g., "always idempotent", "never enqueue from within a transaction")

**Ask only if Devise, JWT, or bcrypt was detected:**

> **Auth**
> 1. How is authentication enforced across controllers? (e.g., "before_action :authenticate_user! in ApplicationController with opt-out", "explicit per-controller")
> 2. Any public endpoints that explicitly skip auth? How are they marked?
> 3. Anything else Claude should know about how auth works? (e.g., "admin vs user scopes", "token refresh handled by middleware")

### Step 4B — Generate and confirm

Generate the full `CLAUDE.md` using the instructions below. Substitute all detected and answered values. For anything the developer said was "not decided yet", use: `[Not yet decided — fill in before writing code that depends on this]`

Show the complete generated file and ask: "Ready to write this? (yes/no)"

If yes, write it. If no, ask what to change and repeat.

---

## How to assemble the standalone CLAUDE.md

Build the file section by section in this order. Include or omit sections based on the conditions below.

### Always include: Stack

```markdown
## Stack
- Ruby [RUBY_VERSION], Rails [RAILS_VERSION], API-only mode
- Database: [DB_CHOICE]
- Testing: RSpec + FactoryBot
- Background jobs: [BACKGROUND_JOBS_CHOICE or "none"]
- Auth: [AUTH_CHOICE or "none"]
- Serialization: [SERIALIZATION_CHOICE or "none"]
```

### Include only if has_application_service

```markdown
## Service Objects
- Location: `app/services/`
- Inherit from `ApplicationService`
- Public interface: `ServiceName.call(args)`
- Implement logic in `#call`
- One responsibility per service
```

### Include only if has_application_error

```markdown
## Error Handling
- Raise `ApplicationError` subclasses for expected failures
- Set `status_code` to the appropriate HTTP status code
- `ApplicationController` rescues `ApplicationError` automatically and renders JSON
- Do not rescue `StandardError` or `Exception` at the application level
```

### Include only if has_loggable

```markdown
## Logging
- Include `Loggable` in any class that needs structured logging
- Use `log_info`, `log_warn`, `log_error` with keyword context: `log_info("order created", order_id: order.id)`
- Never log sensitive data (passwords, tokens, PII)
```

### Include only if Devise, JWT, or bcrypt was detected

```markdown
## Auth
[AUTH_CONTENT — filled from interview answers]
```

Fill [AUTH_CONTENT] from the developer's answers, e.g.:
```
- `before_action :authenticate_user!` in ApplicationController — opt out per action with `skip_before_action`
- Public endpoints are marked with `# public endpoint` comment and explicit skip
- Admin vs user scopes handled via Pundit policies
```

### Always include: Migrations

```markdown
## Migrations
- Always write reversible migrations — use `change` with reversible operations, or explicit `up`/`down`
- Always add an index when adding a foreign key column
- Never remove a column in the same migration that removes it from the model — deprecate first
- Never run migrations without a rollback plan
```

### Include only if Sidekiq or GoodJob was detected

For Sidekiq:
```markdown
## Background Jobs (Sidekiq)
- Workers live in `app/workers/`, named `<Action>Worker`
- `perform` arguments must be scalar (string, integer, boolean) — never pass ActiveRecord objects or hashes
- Delegate business logic to a service — workers call one service and return
- [FILL IN: queue names, retry strategy, dead letter handling — filled from interview answers]
```

For GoodJob:
```markdown
## Background Jobs (GoodJob)
- Jobs live in `app/jobs/`, named `<Action>Job`, inherit from `ApplicationJob`
- `perform` arguments must be serializable — use Global IDs or scalar values
- Delegate business logic to a service — jobs call one service and return
- [FILL IN: queue names, retry strategy, concurrency settings — filled from interview answers]
```

Replace the `[FILL IN: ...]` line with the content from the developer's interview answers.

### Always include: CORS

```markdown
## CORS
- Configured in `config/initializers/cors.rb`
- Set allowed origins before connecting a frontend — do not use `*` in production
```

### Always include: What Claude Should Never Do

```markdown
## What Claude Should Never Do
- Never run `db:migrate`, `db:reset`, `db:drop`, or `db:purge` without explicit instruction
- Never run commands with `RAILS_ENV=production` — connect to production environments manually
- Never commit secrets, tokens, or credentials
- Never bypass strong params
[PROJECT_SPECIFIC_NEVER_DOS]
```

Add one bullet per project-specific never-do from the developer's answers. If they had nothing to add, omit [PROJECT_SPECIFIC_NEVER_DOS].

### Always include: Testing

```markdown
## Testing
- Framework: RSpec + FactoryBot
[TESTING_CONTENT]
```

Fill [TESTING_CONTENT] from the developer's answers, e.g.:
```
- Unit tests: services, models, policy rules
- Integration tests: all request specs (happy path + main error paths)
- A PR is sufficient when: all new public methods are unit tested and the happy path has a request spec
- SimpleCov minimum: 90%
```

### Always include: API Responses

```markdown
## API Responses
[API_RESPONSES_CONTENT]
```

Fill [API_RESPONSES_CONTENT] from answers, e.g.:
```
- JSON structure: `{ data: <resource>, meta: <pagination> }`
- Pagination: Pagy metadata in `meta` key — `{ count:, page:, items:, pages: }`
- Error envelope: `{ error: { code: <string>, message: <string> } }`
- HTTP status codes are always set explicitly — never rely on Rails defaults
```

### Always include: Conventions

```markdown
## Conventions
[CONVENTIONS_CONTENT]
```

Fill [CONVENTIONS_CONTENT] from answers, e.g.:
```
- Serializers: jsonapi-serializer, named `<Model>Serializer`, located in `app/serializers/`
- Query objects: `app/queries/`, named `<Model>Query`, single public method `.call(scope, **filters)`
- No form objects — use strong params and services
```

---

## After writing

Confirm the file was written. Then remind the developer:

- If any sections were left as `[Not yet decided — ...]`, fill those in before writing application code that depends on them
- Run `/rails-boilerplate` first if you haven't already set up the service object pattern, error handling, and Docker config
