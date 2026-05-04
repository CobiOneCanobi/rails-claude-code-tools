---
name: worker-generator
description: Scaffold a Sidekiq worker with correct conventions — scalar args only, service delegation, explicit sidekiq_options, RecordNotFound rescue, and a spec stub.
---

## Step 1 — Check prerequisites

Read `Gemfile`. If `sidekiq` is not present, stop and tell the developer:
> Sidekiq is not in your Gemfile. Add it before generating a worker.

## Step 2 — Interview

Ask all four questions at once and wait for answers:

> 1. **Worker name** — action only, no "Worker" suffix (e.g. `SendWelcomeEmail`, `ProcessPayment`, `SyncInventory`)
> 2. **Queue** — which queue should this run on? (e.g. `default`, `mailers`, `critical`)
> 3. **Retry count** — how many times should Sidekiq retry on failure? (default: 5)
> 4. **Primary argument** — what record does this worker operate on? Give the model name and the ID param name (e.g. `User, user_id` / `Order, order_id`). Always use the record's ID — never pass the record object itself. If the worker doesn't operate on a single record, list the scalar args it needs (strings, integers, booleans only — no objects or hashes).

## Step 3 — Check for existing files

Derive file paths from the worker name:
- Worker: `app/workers/<snake_case_name>_worker.rb`
- Spec: `spec/workers/<snake_case_name>_worker_spec.rb`

For each file:
- If it already exists, show a diff of what would change and ask for confirmation before writing
- If it does not exist, generate it

## Step 4 — Generate worker file

**If the primary argument is a record ID:**

```ruby
# frozen_string_literal: true

class [WorkerName]Worker
  include Sidekiq::Worker
  include Loggable

  sidekiq_options queue: :[QUEUE], retry: [RETRY_COUNT]

  def perform([PRIMARY_ARG])
    [model] = [MODEL_CLASS].find([PRIMARY_ARG])
  end
end
```

**If the primary argument is a non-ID scalar (or multiple scalars):**

```ruby
# frozen_string_literal: true

class [WorkerName]Worker
  include Sidekiq::Worker
  include Loggable

  sidekiq_options queue: :[QUEUE], retry: [RETRY_COUNT]

  def perform([ARGS])
  end
end
```

Substitutions:
- `[WorkerName]` — PascalCase worker name without "Worker" suffix (e.g. `SendWelcomeEmail`)
- `[QUEUE]` — queue name as a symbol (e.g. `:mailers`)
- `[RETRY_COUNT]` — integer from the developer's answer
- `[PRIMARY_ARG]` — snake_case ID param name (e.g. `user_id`)
- `[MODEL_CLASS]` — PascalCase model name (e.g. `User`)
- `[model]` — snake_case model name (e.g. `user`)
- `[ARGS]` — comma-separated scalar arg names for the non-ID case

## Step 5 — Generate spec file

**If the primary argument is a record ID:**

```ruby
# frozen_string_literal: true

require "rails_helper"

RSpec.describe [WorkerName]Worker, type: :worker do
  describe "#perform" do
    let(:[model]) { create(:[model]) }

    it "performs without error" do
      expect { described_class.new.perform([model].id) }.not_to raise_error
    end
  end
end
```

**If the primary argument is a non-ID scalar (or multiple scalars):**

```ruby
# frozen_string_literal: true

require "rails_helper"

RSpec.describe [WorkerName]Worker, type: :worker do
  describe "#perform" do
    it "performs without error" do
      expect { described_class.new.perform([EXAMPLE_ARGS]) }.not_to raise_error
    end
  end
end
```

## Step 6 — Create directories if needed

If `app/workers/` does not exist, create it.
If `spec/workers/` does not exist, create it.

## After generating

Remind the developer:

- Rename the service call if the generated service name doesn't match an existing service
- If `[QUEUE]` is a new queue, add it to `config/sidekiq.yml` or your Sidekiq initializer
- Run `bundle exec rspec spec/workers/[snake_case_name]_worker_spec.rb` to confirm the spec passes
