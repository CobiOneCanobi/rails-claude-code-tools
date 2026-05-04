---
name: ci-cd-config
description: Generate a GitHub Actions CI workflow for a Rails API project — RSpec with JUnit output, RuboCop, Brakeman, bundler-audit, and Docker build validation. Greenfield projects only — assumes db:schema:load. Detects Ruby version and database from existing project files.
---

## Step 1 — Read project files

Read these files to detect the stack. Do not run shell commands.

1. `.ruby-version` — Ruby version
2. `config/database.yml` — database adapter (postgresql / mysql2)
3. `Gemfile` — confirm `rspec_junit_formatter` is present
4. `config/credentials.yml.enc` — note whether it exists (determines if `RAILS_MASTER_KEY` is needed)

If `rspec_junit_formatter` is not in the Gemfile, warn the developer before proceeding:
> `rspec_junit_formatter` is not in your Gemfile — the test job will fail without it. Add it to the `test` group and re-run, or proceed and add it manually.

## Step 2 — Check for existing workflow

Check whether `.github/workflows/ci.yml` exists.

- **If it exists:** read it, generate the new content, show a diff, and ask: "Ready to write this? (yes/no)"
- **If it does not exist:** generate the file, show it in full, and ask: "Ready to write this? (yes/no)"

## Step 3 — Generate `.github/workflows/ci.yml`

Use the detected Ruby version and database adapter. Build the file using the template below.

---

## Workflow template

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      db:
        image: [DB_IMAGE]
        env:
          [DB_ENV]
        ports:
          - [DB_PORT]:[DB_PORT]
        options: >-
          [DB_HEALTH_CHECK]

    env:
      RAILS_ENV: test
      DATABASE_URL: [DATABASE_URL]
      [MASTER_KEY_ENV]

    steps:
      - uses: actions/checkout@v4

      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: .ruby-version
          bundler-cache: true

      - name: Set up database
        run: bundle exec rails db:create db:schema:load

      - name: Run tests
        run: bundle exec rspec --format RspecJunitFormatter --out rspec.xml --format progress

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: rspec-results
          path: rspec.xml

  lint:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: .ruby-version
          bundler-cache: true

      - name: Run RuboCop
        run: bundle exec rubocop

  security:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: .ruby-version
          bundler-cache: true

      - name: Run Brakeman
        run: bundle exec brakeman --no-pager

      - name: Run bundler-audit
        run: bundle exec bundler-audit check --update

  docker:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build --target production .
```

---

## Database substitutions

**PostgreSQL:**
```
DB_IMAGE:      postgres:16-alpine
DB_ENV:
               POSTGRES_PASSWORD: password
DB_PORT:       5432
DB_HEALTH_CHECK:
               --health-cmd pg_isready
               --health-interval 10s
               --health-timeout 5s
               --health-retries 5
DATABASE_URL:  postgresql://postgres:password@localhost:5432/app_test
```

**MySQL:**
```
DB_IMAGE:      mysql:8
DB_ENV:
               MYSQL_ROOT_PASSWORD: password
               MYSQL_DATABASE: app_test
DB_PORT:       3306
DB_HEALTH_CHECK:
               --health-cmd "mysqladmin ping"
               --health-interval 10s
               --health-timeout 5s
               --health-retries 5
DATABASE_URL:  mysql2://root:password@127.0.0.1:3306/app_test
```

Note: MySQL on GitHub Actions requires `127.0.0.1` rather than `localhost` to connect correctly.

**`RAILS_MASTER_KEY` substitution:**

If `config/credentials.yml.enc` exists, include in the test job env:
```
MASTER_KEY_ENV:  RAILS_MASTER_KEY: ${{ secrets.RAILS_MASTER_KEY }}
```

If it does not exist, omit the line entirely.

---

## After writing

Remind the developer:

- Push to a branch and open a PR — all four jobs should go green before merging to main
- If `config/credentials.yml.enc` is present, add `RAILS_MASTER_KEY` as a secret in the repo: Settings → Secrets and variables → Actions → New repository secret
- The `docker` job validates the production build but does not push to a registry — add a deploy job when ready
- If Sidekiq is in the stack, no additional CI config is needed — workers are tested through the app in the `test` job
