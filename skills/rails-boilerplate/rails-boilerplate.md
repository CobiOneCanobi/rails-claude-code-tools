---
name: rails-boilerplate
description: Generate Rails API boilerplate — ApplicationService, ApplicationError, Loggable, base controller, starter Gemfile, Docker, Lefthook, and a seeded CLAUDE.md. Safe to re-run — skips questions if already bootstrapped and only fills gaps.
---

Before generating anything, read the existing project to understand what's already there:

1. Read `.ruby-version` — authoritative Ruby version for this project
2. Read `Gemfile` — Rails version (grep for `gem "rails"`) and all existing gems. Do not run `rails --version` or `ruby --version` — these reflect the host's active rbenv version which may not match the project.
3. Read `app/controllers/application_controller.rb` — note what's already defined
4. Read `app/models/application_record.rb` — note what's already defined
5. Read `config/database.yml` — note the current adapter
6. Read `db/seeds.rb` — note existing content

Confirm the detected versions with the user before proceeding.

Then check whether the project is already bootstrapped by testing for the presence of these files:
- `docker-compose.yml`
- `CLAUDE.md`
- `app/services/application_service.rb`

If all three exist, skip the questions entirely — the answers can be inferred from what's already in the project. Go straight to gap-checking each file below.

The database adapter is always inferred from `config/database.yml` — do not ask. If the adapter is sqlite3, the skill will switch it based on what the user specifies for docker-compose; treat sqlite3 as "not yet chosen" and ask question 1 below. For PostgreSQL or MySQL, proceed without asking.

If any of those files are missing, ask only the questions whose answers are needed to generate the missing files:

1. What database are you using? (PostgreSQL / MySQL) — only ask if `database.yml` shows sqlite3
2. Auth approach? (JWT / Devise / API key / none / undecided) — needed for `Gemfile`, `CLAUDE.md`
3. Background jobs? (Sidekiq / GoodJob / none / undecided) — needed for `Gemfile`, `docker-compose.yml`, `CLAUDE.md`
4. JSON serialization? (jsonapi-serializer / alba / jbuilder / none / undecided) — needed for `Gemfile`, `CLAUDE.md`
5. Do you want Redis included in docker-compose? (yes if Sidekiq or caching — otherwise no) — needed for `docker-compose.yml`, `.env`

Once you have the answers, generate all files below in a single pass. Do not ask further questions — use sensible defaults for anything not covered.

---

## Files to generate

### `Gemfile`

**Do not replace the existing Gemfile.** Add missing gems only. Preserve all existing gems — including Rails 8 defaults like `solid_queue`, `solid_cable`, `solid_cache`, `kamal`, `thruster`. The developer chose these when they ran `rails new` and may be relying on them.

For each gem in the lists below: check if it's already present. Add it if missing. Skip it if already there. Annotate non-obvious additions with a brief inline comment.

If the user chose PostgreSQL or MySQL, remove `gem 'sqlite3'` from the Gemfile if present — it's added by `rails new` by default but is not needed when using a different database.

If the user chose Sidekiq and this is a Rails 8 app (detected from `config.load_defaults 8.x` in `config/application.rb`), also remove `solid_queue`, `solid_cable`, `solid_cache` from the Gemfile and remove their corresponding `require` statements from `config/application.rb`. Rails 8 loads these by default but they conflict with Sidekiq. Show the user exactly what will be removed before writing.

Before writing, show the user a summary of what will be added, what will be removed, and ask for confirmation.

**Always include — core:**
- `pg` or `mysql2` based on DB choice
- `puma`
- `rack-cors`
- `bootsnap`
- `pagy` — pagination

**Always include — development:**
- `lefthook`
- `rubocop-rails-omakase`
- `rubocop-rspec`
- `ruby-lsp`

**Always include — development/test:**
- `dotenv-rails`
- `brakeman`
- `bundler-audit`
- `rspec-rails`
- `factory_bot_rails`
- `faker`
- `shoulda-matchers`
- `webmock` — stub external HTTP requests in tests
- `timecop` — time manipulation in tests
- `simplecov` — test coverage
- `rspec_junit_formatter` — JUnit XML output for CI

**Auth gems — based on user's choice:**
- JWT: `jwt` + `bcrypt`
- Devise: `devise`
- API key: `bcrypt` only
- None/undecided: omit, add a comment noting auth is not yet configured

**Background job gems — based on user's choice:**
- Sidekiq: `gem 'sidekiq', '~> 7.0'` and `gem 'connection_pool', '< 3.0'` — connection_pool 3.0.x has a SyntaxError on Ruby 3.3 that breaks Sidekiq. Pin until connection_pool 3.1+ is released.
- GoodJob: `good_job`
- None/undecided: omit

**Serialization — based on user's choice:**
- jsonapi-serializer: `jsonapi-serializer`
- alba: `alba`
- jbuilder: `jbuilder`
- None/undecided: omit, add a comment noting serialization approach is not yet decided

---

### `app/services/application_service.rb`

If the file already exists, skip it and note "no changes needed." Otherwise, create `app/services/` directory if it doesn't exist. Service objects are plain Ruby classes. `.call` is the public interface — it instantiates and delegates to the instance method of the same name. Subclasses implement `#call`.

```ruby
class ApplicationService
  def self.call(...)
    new(...).call
  end
end
```

### `app/errors/application_error.rb`

If the file already exists, skip it and note "no changes needed." Otherwise, create `app/errors/` directory if it doesn't exist. Base error class for expected application failures. Carries an HTTP status code and optional metadata for structured error responses. Subclass for specific error types.

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

### `app/controllers/concerns/loggable.rb`

If the file already exists, skip it and note "no changes needed." Otherwise, create it. Structured JSON logging concern. Including this in a class gives it `log_info`, `log_warn`, and `log_error` methods that emit consistent, parseable log lines. The class name is included automatically for traceability.

```ruby
module Loggable
  extend ActiveSupport::Concern

  def log_info(message, **context)
    Rails.logger.info(log_payload(message, **context))
  end

  def log_warn(message, **context)
    Rails.logger.warn(log_payload(message, **context))
  end

  def log_error(message, **context)
    Rails.logger.error(log_payload(message, **context))
  end

  private

  def log_payload(message, **context)
    { class: self.class.name, message: message, **context }.to_json
  end
end
```

### `app/controllers/application_controller.rb`

Read the existing file first. Add only what's missing:
- `include Loggable` — add if not already present
- `rescue_from ApplicationError` block — add if not already present

Do not remove any existing content. If the file already includes both, skip it and note "no changes needed."

### `config/database.yml`

Read the existing file first. If the adapter is already PostgreSQL or MySQL, add `DATABASE_URL` support only if not already present — do not replace the file. If the adapter is sqlite3 (the Rails default), replace the file entirely since we're switching adapters. In either case, show the user what will change before writing.

**PostgreSQL:**
```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  url: <%= ENV["DATABASE_URL"] %>

development:
  <<: *default
  database: <%= ENV.fetch("DB_NAME", "app_development") %>

test:
  <<: *default
  database: <%= ENV.fetch("DB_NAME_TEST", "app_test") %>

production:
  <<: *default
```

**MySQL:**
```yaml
default: &default
  adapter: mysql2
  encoding: utf8mb4
  collation: utf8mb4_unicode_ci
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  url: <%= ENV["DATABASE_URL"] %>

development:
  <<: *default
  database: <%= ENV.fetch("DB_NAME", "app_development") %>

test:
  <<: *default
  database: <%= ENV.fetch("DB_NAME_TEST", "app_test") %>

production:
  <<: *default
```

### `app/models/application_record.rb`

Read the existing file first. Add only what's missing:
- `self.inheritance_column = :_type_disabled` — add if not already present

`primary_abstract_class` is already in the Rails default — do not add it again. If the file already has `inheritance_column` disabled, skip it and note "no changes needed."

### `db/seeds.rb`

Read the existing file first. If the production guard (`abort` or equivalent) is already present, skip it. If the file is empty or only has comments, prepend the following and preserve any existing content below it:

```ruby
# frozen_string_literal: true

abort("Seeds should not be run in production.") if Rails.env.production?

# Keep seeds idempotent — use find_or_create_by over create
# Group seeds by model, ordered by dependency (no FKs violated)

puts "Seeding complete."
```

### `.env` and `.env.example`

If either file already exists, skip it and note "no changes needed." Otherwise, generate both files in the same pass.

**`.env`** — not committed to the repo. Pre-fill all values from what you know:
- `DATABASE_URL` — use the docker-compose service name, adapter, and default credentials matching the DB choice
- `REDIS_URL` — include if Redis was selected, using the docker-compose service name
- `SIDEKIQ_CONCURRENCY` — include if Sidekiq was selected, default to `5`
- `RAILS_MASTER_KEY` — read the contents of `config/master.key` and populate it automatically

PostgreSQL example:
```
DATABASE_URL=postgresql://postgres:password@db:5432/app_development
DB_NAME=app_development
DB_NAME_TEST=app_test
REDIS_URL=redis://redis:6379/0
RAILS_MASTER_KEY=<contents of config/master.key>
```

MySQL example:
```
DATABASE_URL=mysql2://root:password@db:3306/app_development
DB_NAME=app_development
DB_NAME_TEST=app_test
RAILS_MASTER_KEY=<contents of config/master.key>
```

**`.env.example`** — committed to the repo. Same structure as `.env` but with values blanked out or set to obvious placeholders. Documents what variables are required without exposing real values.

```
DATABASE_URL=
DB_NAME=app_development
DB_NAME_TEST=app_test
REDIS_URL=                        # Required if using Redis
SIDEKIQ_CONCURRENCY=5             # Required if using Sidekiq
RAILS_MASTER_KEY=                 # Copy from config/master.key — never commit the real value
```

Also confirm these are present in `.gitignore` — add any that are missing:
- `.env` — should be in Rails' default `.gitignore` but make it explicit with a comment if not already there
- `spec/examples.txt` — generated by RSpec to track example status between runs, should not be committed

### `Dockerfile`

If the file already exists, skip it and note "no changes needed." Otherwise, generate a multi-stage build. Use the Ruby version provided. Stages:

**base** — system dependencies only (libpq-dev or default-libmysqlclient-dev depending on DB choice, curl, build-essential). Update RubyGems and install the latest Bundler explicitly — Ruby base images often ship with an older Bundler that's incompatible with Rails 8: `RUN gem update --system && gem install bundler`. Set `WORKDIR /app`.

**development** — inherits base. Use `COPY Gemfile Gemfile.lock* ./` — the `*` makes `Gemfile.lock` optional so the image builds cleanly on first run before `Gemfile.lock` exists. Copies app code. No CMD — docker-compose sets it.

**production** — inherits base. Copies Gemfile and Gemfile.lock. Runs `bundle config set --local without 'development test' && bundle install`. Copies app code. Sets `RAILS_ENV=production`, `RAILS_LOG_TO_STDOUT=true`. Exposes port 3000. CMD: `bundle exec rails server -b 0.0.0.0`.

### `docker-compose.yml`

If the file already exists, skip it and note "no changes needed." Otherwise, generate a local development setup. Use the database the user chose.

Services:
- **app** — builds from `Dockerfile` target `development`. Mounts project root to `/app`. Mounts `bundle_cache` to `/usr/local/bundle` — persists installed gems between restarts and ensures the correct Bundler version is used. Command: `bash -c "bundle install && bundle exec rails server -b 0.0.0.0"`. Depends on `db` (and `redis` if requested). Sets `DATABASE_URL` and (if Redis) `REDIS_URL` using compose service names. Port `3000:3000`.
- **db** — PostgreSQL (`postgres:16-alpine`) or MySQL (`mysql:8`). Named volume. Set a default password via environment variable. Port exposed for local access.
- **redis** (if requested) — `redis:7-alpine`. Named volume. Port `6379:6379`.
- **sidekiq** (if Sidekiq was selected) — same build as app. Mounts project root and `bundle_cache`. Command: `bundle exec sidekiq`. Depends on `db` and `redis`. Sets same `DATABASE_URL` and `REDIS_URL` environment variables as app service.

Use named volumes for all data services, including `bundle_cache` for the app's gem installation path.

### `DOCKER.md`

If the file already exists, skip it and note "no changes needed." Otherwise, generate a Docker command reference for this project. Generated once, not loaded into every Claude session.

```markdown
# Docker Reference

## Daily use
docker-compose up -d                          — start all services
docker-compose down                           — stop all services
docker-compose exec app bash                  — shell into app container (use for all Ruby/Rails commands)
docker-compose run --rm app bash              — one-off container if app isn't running
docker-compose logs -f app                    — tail app logs
docker-compose logs -f db                     — tail DB logs
docker-compose logs -f                        — tail all logs
docker-compose ps                             — check running services

## When things change
docker-compose build app                      — rebuild after Gemfile or Dockerfile changes
docker-compose restart app                    — restart app without full down/up

## Debugging
docker attach $(docker-compose ps -q app)     — attach for debugger statements (Ctrl+P Ctrl+Q to detach, never Ctrl+C)

## Database access
docker-compose exec db psql -U postgres       — PostgreSQL shell
docker-compose exec db mysql -u root -p       — MySQL shell

## Cleanup
docker system prune                           — remove stopped containers, unused networks, dangling images
docker system prune -a                        — also removes unused images (frees significant disk space)
docker volume prune                           — remove unused volumes
docker-compose down --rmi local               — remove locally built images

## Destructive
docker-compose down -v                        — removes volumes, wipes local database and bundle cache
```

### `.rubocop.yml`

If the file already exists, skip it and note "no changes needed." Otherwise, generate a minimal config to wire `rubocop-rails-omakase` and `rubocop-rspec` together. No custom rules — just inheritance and plugin activation.

```yaml
plugins:
  - rubocop-rspec

inherit_gem:
  rubocop-rails-omakase: rubocop.yml

AllCops:
  NewCops: enable
  Exclude:
    - 'db/schema.rb'
    - 'bin/**/*'
    - 'vendor/**/*'
    - 'node_modules/**/*'
```

Note: `plugins` is used for `rubocop-rspec` (RuboCop 1.72+ extension API) — `inherit_gem` handles loading `rubocop-rails-omakase` implicitly.

### `config/initializers/good_job.rb` (only if GoodJob was selected)

If the file already exists, skip it and note "no changes needed."

```ruby
GoodJob.configure do |config|
  config.execution_mode = :external
  config.max_threads = ENV.fetch("GOOD_JOB_MAX_THREADS", 5).to_i
  config.queues = ENV.fetch("GOOD_JOB_QUEUES", "*")
end
```

Also update `config/application.rb` to set the queue adapter:

```ruby
config.active_job.queue_adapter = :good_job
```

### `config/initializers/sidekiq.rb` (only if Sidekiq was selected)

If the file already exists, skip it and note "no changes needed."

```ruby
Sidekiq.configure_server do |config|
  config.redis = { url: ENV.fetch("REDIS_URL", "redis://localhost:6379/0") }
end

Sidekiq.configure_client do |config|
  config.redis = { url: ENV.fetch("REDIS_URL", "redis://localhost:6379/0") }
end
```

### `spec/spec_helper.rb`

If the file already exists, skip it and note "no changes needed."

```ruby
RSpec.configure do |config|
  config.expect_with :rspec do |expectations|
    expectations.include_chain_clauses_in_custom_matcher_descriptions = true
  end

  config.mock_with :rspec do |mocks|
    mocks.verify_partial_doubles = true
  end

  config.shared_context_metadata_behavior = :apply_to_host_groups
  config.filter_run_when_matching :focus
  config.example_status_persistence_file_path = "spec/examples.txt"
  config.disable_monkey_patching!
  config.order = :random
  Kernel.srand config.seed

  config.profile_examples = 10
end
```

### `spec/rails_helper.rb`

If the file already exists, skip it and note "no changes needed."

```ruby
require "spec_helper"
ENV["RAILS_ENV"] ||= "test"
require_relative "../config/environment"
abort("The Rails environment is running in production mode!") if Rails.env.production?
require "rspec/rails"
require "simplecov"

SimpleCov.start "rails" do
  add_filter "/spec/"
  add_filter "/config/"
  # Set a minimum_coverage threshold once baseline coverage is established
end

Dir[Rails.root.join("spec/support/**/*.rb")].each { |f| require f }

RSpec.configure do |config|
  config.fixture_paths = [ Rails.root.join("spec/fixtures") ]
  config.use_transactional_fixtures = true
  config.infer_spec_type_from_file_location!
  config.filter_rails_from_backtrace!
end
```

### `spec/support/factory_bot.rb`

If the file already exists, skip it and note "no changes needed."

```ruby
RSpec.configure do |config|
  config.include FactoryBot::Syntax::Methods
end
```

### `spec/support/shoulda_matchers.rb`

If the file already exists, skip it and note "no changes needed."

```ruby
Shoulda::Matchers.configure do |config|
  config.integrate do |with|
    with.test_framework :rspec
    with.library :rails
  end
end
```

### `spec/support/webmock.rb`

If the file already exists, skip it and note "no changes needed."

```ruby
require "webmock/rspec"

WebMock.disable_net_connect!(allow_localhost: true)
```

### `spec/support/timecop.rb`

If the file already exists, skip it and note "no changes needed."

```ruby
RSpec.configure do |config|
  config.after { Timecop.return }
end
```

### `spec/factories/.keep`

If the file already exists, skip it and note "no changes needed." Otherwise create an empty `.keep` file to commit the directory.

### `bin/check_env_completeness`

If the file already exists, skip it and note "no changes needed."

Copy the contents of `check_env_completeness.rb` from the `env-completeness` hook in rails-claude-tools verbatim. Then make it executable:

```bash
chmod +x bin/check_env_completeness
```

### `lefthook.yml`

If the file already exists, skip it and note "no changes needed."

```yaml
pre-commit:
  commands:
    rubocop:
      glob: "*.rb"
      run: bundle exec rubocop --autocorrect {staged_files}
    env_completeness:
      run: ruby bin/check_env_completeness

pre-push:
  commands:
    rspec:
      run: bundle exec rspec
```

### `CLAUDE.md`

Read the existing `CLAUDE.md` if it exists. Check for each section below by its `##` heading. Add any sections whose heading is not present. Never remove or overwrite existing content. If all sections are already present, skip the file and note "no changes needed."

If the file does not exist, generate it at the project root.

In both cases, use what you know from the user's answers (or from inferring the stack). Use `[FILL IN: ...]` markers for anything that requires project-specific decisions.

Build the file section by section. Include or omit sections based on the conditions below.

**Always include:**

```markdown
# CLAUDE.md

## Stack
- Ruby [RUBY_VERSION], Rails [RAILS_VERSION], API-only mode
- Database: [DB_CHOICE]
- Testing: RSpec + FactoryBot
- Background jobs: [BACKGROUND_JOBS_CHOICE or "undecided"]
- Auth: [AUTH_CHOICE or "undecided"]
- Serialization: [SERIALIZATION_CHOICE or "undecided"]

## Service Objects
- Location: `app/services/`
- Inherit from `ApplicationService`
- Public interface: `ServiceName.call(args)`
- Implement logic in `#call`
- One responsibility per service

## Error Handling
- Raise `ApplicationError` subclasses for expected failures
- Set `status_code` to the appropriate HTTP status code
- `ApplicationController` rescues `ApplicationError` automatically and renders JSON
- Do not rescue `StandardError` or `Exception` at the application level

## Logging
- Include `Loggable` in any class that needs structured logging
- Use `log_info`, `log_warn`, `log_error` with keyword context: `log_info("order created", order_id: order.id)`
- Never log sensitive data (passwords, tokens, PII)
```

**Include only if auth is not "none" or "undecided":**

For JWT:
```markdown
## Auth
- Tokens passed as `Authorization: Bearer <token>` header
- [FILL IN: how token validation is wired into controllers]
- [FILL IN: public endpoints and how they opt out of auth]
```

For Devise (API mode):
```markdown
## Auth
- Devise handles authentication — users authenticated via token or session depending on config
- [FILL IN: how authenticate_user! is applied across controllers]
- [FILL IN: public endpoints and how they opt out of auth]
```

For API key:
```markdown
## Auth
- API key authentication
- [FILL IN: how keys are validated and wired into controllers]
- [FILL IN: public endpoints and how they opt out of auth]
```

**Always include:**

```markdown
## Migrations
- Always write reversible migrations — use `change` with reversible operations, or explicit `up`/`down`
- Always add an index when adding a foreign key column
- Never remove a column in the same migration that removes it from the model — deprecate first
- Never run migrations without a rollback plan
```

**Include only if Sidekiq was selected:**

```markdown
## Background Jobs (Sidekiq)
- Workers live in `app/workers/`, named `<Action>Worker`
- `perform` arguments must be scalar (string, integer, boolean) — never pass ActiveRecord objects or hashes
- Delegate business logic to a service — workers call one service and return
- [FILL IN: queue names and purpose, retry strategy, dead letter handling]
```

**Include only if GoodJob was selected:**

```markdown
## Background Jobs (GoodJob)
- Jobs live in `app/jobs/`, named `<Action>Job`, inherit from `ApplicationJob`
- `perform` arguments must be serializable — use Global IDs or scalar values
- Delegate business logic to a service — jobs call one service and return
- [FILL IN: queue names and purpose, retry strategy, concurrency settings]
```

**Always include:**

```markdown
## CORS
- Configured in `config/initializers/cors.rb`
- Set allowed origins before connecting a frontend — do not use `*` in production

## Docker
See `DOCKER.md` for command reference.

## What Claude Should Never Do
- Never run `db:migrate`, `db:reset`, `db:drop`, or `db:purge` without explicit instruction
- Never commit secrets, tokens, or credentials
- Never bypass strong params
- [FILL IN: project-specific never-dos]

## Testing
- Framework: RSpec + FactoryBot
- [FILL IN: what gets unit tested, what gets integration tested]
- [FILL IN: what counts as sufficient coverage for a PR]

## API Responses
- [FILL IN: JSON structure, pagination format, error envelope]

## Conventions
- [FILL IN: serializer approach, query objects, additional patterns]
```

---

## After generating

Confirm which files were created, which were modified (and what was added), and which were skipped because no changes were needed.

Then automatically run a syntax check on every generated Ruby file using the local Ruby installation:

```bash
ruby -c app/services/application_service.rb
ruby -c app/errors/application_error.rb
ruby -c app/controllers/concerns/loggable.rb
ruby -c app/controllers/application_controller.rb
ruby -c app/models/application_record.rb
ruby -c db/seeds.rb
```

Report any failures. If all pass, output the following verification checklist for the developer to work through:

---

**Verification checklist — run in order:**

```bash
# On your host machine:
rm Gemfile.lock                               # Remove host-generated lockfile — will be regenerated inside the container with the correct Bundler version
docker-compose up -d                          # Start all services — bundle install and rails server run automatically

# Watch startup logs — wait until you see Puma running before proceeding:
docker-compose logs -f app                    # Ctrl+C once you see "Listening on..."

# Confirm server started cleanly — no errors in output:
docker-compose logs app

# Shell into the app container for all remaining commands:
docker-compose exec app bash

# Inside the container:
bundle exec lefthook install                  # Register git hooks — works because docker-compose mounts .git into the container
bundle exec rails db:create                   # Confirms database.yml connects correctly
bundle exec rails db:migrate                  # No migrations yet — confirms it runs cleanly
bundle exec rails runner 'puts "Rails #{Rails.version} booted on Ruby #{RUBY_VERSION}"'  # Confirms app loads
bundle exec rubocop                           # Confirms generated files pass linting
bundle exec rspec                             # No specs yet — confirms test environment initialises
bundle exec brakeman --no-pager               # Security baseline on generated code
bundle exec bundler-audit check --update      # Check for vulnerable gem versions — container has internet access
```

If Sidekiq was selected, also run inside the container:
```bash
bundle exec sidekiq                           # Confirm Sidekiq connects to Redis — Ctrl+C after it starts
```

If GoodJob was selected, also run inside the container:
```bash
bundle exec good_job start                    # Confirm GoodJob connects to the database — Ctrl+C after it starts
```

---

Finally, remind the user:
- `.env` has been generated and pre-filled — verify the values look correct before running the checklist
- Fill in the `[FILL IN: ...]` sections in `CLAUDE.md` before writing any application code — these sections are the most important part of the file
