# Workers

Workers live in `app/workers/`, named `<Action>Worker` (e.g. `SendWelcomeEmailWorker`, `ProcessPaymentWorker`).

## Arguments

- `perform` arguments must be scalar — strings, integers, booleans only
- Never pass ActiveRecord objects, hashes, or arrays of objects
- Always pass a record's ID and look it up inside the worker — never pass the record itself
- If a worker operates on multiple records, pass their IDs as separate arguments

## Structure

- Always set `sidekiq_options` explicitly — never rely on Sidekiq defaults
- Delegate business logic to a service — the worker calls one service and returns
- Keep `perform` thin: find the record, call the service, handle expected errors

```ruby
def perform(user_id)
  user = User.find(user_id)
  SendWelcomeEmailService.call(user: user)
end
```

## Error handling

- Rescue `ActiveRecord::RecordNotFound` explicitly when a missing record is expected and unrecoverable — let Sidekiq discard rather than retry infinitely
- Let unexpected errors propagate — Sidekiq will retry according to `sidekiq_options retry:`
- Never rescue `StandardError` broadly in a worker

## Idempotency

- Workers must be safe to run more than once — Sidekiq retries on failure
- Use database constraints or guard clauses to prevent duplicate side effects

## Queue and retry

- Queue names: `[FILL IN: list queue names and what each handles]`
- Default retry count: `[FILL IN: default retry count]`
- Dead letter handling: `[FILL IN: what happens after retries are exhausted]`
