# Rails Boilerplate Generator

**Scale:** greenfield — run once at the start of a new Rails API project

Generates the base layer for a Rails API project: service object pattern, structured error handling, structured logging, Docker setup, Lefthook config, and a seeded CLAUDE.md. Run it after `rails new`, before writing any application code.

---

## When to use it

Day one of a new Rails API project, or any time you want to add missing pieces to an existing one. Reads existing files before making changes, adds what's missing, and preserves everything already there including Rails 8 defaults (`solid_queue`, `solid_cable`, `kamal`, etc.). Safe to re-run — if the project is already bootstrapped it skips the questions entirely and only fills gaps.

---

## Prerequisites

- Ruby and Rails installed locally — this is the only host dependency. Used for `rails new` and detecting versions. Install via [rbenv](https://github.com/rbenv/rbenv) or [mise](https://mise.jdx.dev/).
- Rails API project created: `rails new my-app --api`
- Docker installed locally — everything else runs in containers

---

## Installation

Copy the skill file into your project's `.claude/commands/` directory:

```bash
mkdir -p .claude/commands
cp rails-boilerplate.md .claude/commands/rails-boilerplate.md
```

---

## Usage

In a Claude Code session:

```
/rails-boilerplate
```

Claude will detect your Ruby and Rails versions automatically. If the project isn't bootstrapped yet, it asks up to five questions (DB, auth, background jobs, serialization, Redis) — skipping any it can infer from existing files. If the project is already bootstrapped, it skips the questions entirely and goes straight to gap-checking. All files are generated or updated in a single pass.

---

## What gets generated

```
app/
  services/
    application_service.rb        # Base class — .call class method, #call instance method
  controllers/
    application_controller.rb     # Rescues ApplicationError, includes Loggable
    concerns/
      loggable.rb                 # Structured JSON logging: log_info, log_warn, log_error
  errors/
    application_error.rb          # Base error with status_code and metadata
  models/
    application_record.rb         # STI disabled by default

config/
  database.yml                    # DATABASE_URL-first, conditional on DB choice

db/
  seeds.rb                        # Production guard, idempotency conventions

spec/
  spec_helper.rb                  # RSpec core config — random ordering, focus filter, slow example profiling
  rails_helper.rb                 # Rails + SimpleCov + support file loader
  support/
    factory_bot.rb                # FactoryBot::Syntax::Methods included in all specs
    shoulda_matchers.rb           # Shoulda Matchers wired to RSpec + Rails
    webmock.rb                    # External HTTP disabled by default in tests
    timecop.rb                    # Timecop.return after each example — prevents time leaking between specs
  factories/
    .keep                         # Placeholder — add factories here

Gemfile                           # Curated gems based on your stack choices
.env.example                      # Required environment variables, committed to repo
Dockerfile                        # Multi-stage: development + production targets
docker-compose.yml                # app + db + redis (optional)
bin/
  check_env_completeness           # pre-commit script: flags ENV keys missing from .env.example
lefthook.yml                      # pre-commit: RuboCop + ENV completeness, pre-push: RSpec
DOCKER.md                         # Docker command reference — separate from CLAUDE.md to reduce session token usage
CLAUDE.md                         # Structural sections pre-filled; Auth + Background Jobs conditional; Testing/API/Conventions left as [FILL IN:] markers
```

---

## Patterns generated

### Service objects

```ruby
class ApplicationService
  def self.call(...)
    new(...).call
  end
end
```

Subclasses inherit and implement `#call`. Called via `ServiceName.call(args)`. Using `call` (not `perform`) keeps service objects distinct from Sidekiq workers, which use `perform` by convention.

### Error handling

```ruby
class ApplicationError < StandardError
  attr_reader :status_code, :metadata

  def initialize(message = nil, status_code: 422, metadata: {})
    super(message)
    @status_code = status_code
    @metadata = metadata
  end
end
```

Subclass for specific error types. `ApplicationController` rescues `ApplicationError` automatically and renders a JSON response with the status code. The `metadata` hash passes structured context through to the response.

### Logging

```ruby
module Loggable
  extend ActiveSupport::Concern

  def log_info(message, **context)
    Rails.logger.info(log_payload(message, **context))
  end

  # ...

  private

  def log_payload(message, **context)
    { class: self.class.name, message: message, **context }.to_json
  end
end
```

Emits structured JSON log lines. The class name is included automatically. Include in any class that needs logging — not just controllers.

---

## Customization notes

**`call` vs `perform`:** `call` is the Ruby-idiomatic interface for callable objects. Sidekiq workers use `perform` — keeping service objects on `call` avoids confusion when a service is invoked from a worker.

**ApplicationError default status:** defaults to `422 Unprocessable Entity`. Override in subclasses for specific error types (`404`, `403`, `500`, etc.).

**Gemfile:** curated based on your answers. Always includes a production-ready baseline (pagination, rate limiting, security scanning, structured test setup). Undecided or deferred choices are omitted with a comment — add them when you actually need them.

**`.env`:** generated and pre-filled from your answers — `DATABASE_URL`, `REDIS_URL` (if Redis), `SIDEKIQ_CONCURRENCY` (if Sidekiq), and `RAILS_MASTER_KEY` read from `config/master.key`. Verify the values before running the checklist. Never commit this file.

**`.env.example`:** same structure as `.env` but with values blanked out. Committed to the repo as documentation of required variables.

**Redis in docker-compose:** optional. Include it if you're planning Sidekiq, Action Cable, or caching. Easy to add later if not needed upfront.

**Lefthook:** run `lefthook install` after adding it to the Gemfile to activate the git hooks. The generated config runs RuboCop on staged files and an ENV completeness check pre-commit, and RSpec pre-push. The ENV completeness check scans staged files for `ENV['KEY']` and `ENV.fetch('KEY')` references and fails the commit if any key is missing from `.env.example`.

**CLAUDE.md:** generated with structural sections pre-filled (Service Objects, Error Handling, Logging, Migrations, CORS, Docker) and `[FILL IN: ...]` markers for project-specific decisions (Testing, API Responses, Conventions). Auth and Background Jobs sections are included conditionally — only if those choices were made. Fill in the markers before writing application code using `/project-claude-md`.

## Recommended next steps

After the boilerplate verification checklist passes:

1. `/project-claude-md` — fill in the `[FILL IN: ...]` sections in `CLAUDE.md` before writing any application code
2. `/ci-cd-config` — generate a GitHub Actions workflow (RSpec, RuboCop, Brakeman, Docker build)
