# Rails failure modes for code review

Read only when the diff affects a Rails application. This is a *review probe*, not a checklist of
things to demand. Every concern needs a changed-line anchor, reachable scenario, impact, and a
check against existing tests, constraints, policies, and framework version. First inspect
`.ruby-version`, `Gemfile.lock`, `config.load_defaults`, `database.yml`, the queue adapter and
relevant gem versions. The checked-out application's configuration wins over this reference;
`rails-guides` documents framework behavior, while `layered-rails` addresses ownership/design.

## Ruby semantics that change behavior

- `0`, `""`, `[]`, and empty relations are truthy. `if relation.count` does not mean rows exist;
  check a real branch before proposing `.exists?` or `count.positive?`.
- Hash symbol/string keys, `**kwargs` forwarding, and ActiveSupport indifferent access are not
  interchangeable. Trace the producer and consumer; don't report a mismatch when the hash is
  `HashWithIndifferentAccess`.
- `Hash.new([])` and `Array.new(n, [])` share one mutable object. `downcase!` and similar bang
  methods may return `nil` for no change. Check whether callers use the returned value or mutate
  the shared value.
- If a value object overrides `eql?`, verify `hash` uses the same equality fields; otherwise
  equal values can miss Hash/Set lookups. Distinguish `==` (value) from `equal?` (identity).
  Check whether this object is actually used as a key before flagging.
- Repeated `&.`/`dig` on a required association can turn a broken invariant into a late `nil`.
  Confirm presence guarantees first; safe navigation is correct for genuinely optional data.
- A block/`ensure` must release acquired files, locks and checked-out connections on exceptions.
  A `return` in `ensure` can suppress the original error; trace the failing path before reporting.
- `map` allocates an array; `each` returns the receiver and is right for side effects. `filter_map`
  fuses filtering and mapping. Flag only when the result/allocations matter to this path.
- `send`, `constantize`, `Marshal.load`, unrestricted YAML and string-based shell commands become
  security issues when **untrusted input** reaches them. Trace the input through any allowlist;
  dynamic dispatch on a fixed internal symbol is not itself a vulnerability.

## Query and association traps

- `includes(:children)` can preload or join depending on conditions. `eager_load` uses a LEFT OUTER
  JOIN. When filtering/ordering through a collection join, test whether parents are duplicated or
  pagination/counting changes; a join is not automatically an N+1 fix.
- `relation.to_a.select` or `sort_by` before limit/pagination loads all rows and can change page
  semantics. `pluck`, `pick`, `exists?`, aggregates or SQL ordering can avoid instantiation, but
  check custom attribute casting and whether the object methods are needed.
- `find_each`/`in_batches` use a cursor and do not preserve arbitrary relation ordering. If the
  business operation needs order, verify it explicitly. A Rails `scope` block returning `nil`
  falls back to a chainable relation; do not flag `nil` as a broken scope without confirming the
  actual caller and Rails version.
- `update_all`, `delete_all`, `insert_all`, `upsert_all` and raw SQL bypass normal model
  validations/callbacks (and potentially timestamp/dependent behavior). Check counter caches,
  audit hooks and cleanup only if this model actually relies on them.
- `default_scope`, enum value reordering, or implicit order can silently change existing reads
  and persisted meanings. Trace queries and stored values before calling a scope change wrong.
- If list ordering requires repeatedly loading every row to derive a Ruby-only sort value,
  consider maintaining an indexed sort key at write time. Verify the recalculation triggers,
  backfill and pagination contract before recommending denormalization; a small bounded list
  does not justify it.
- For nested fragment caches, changing a child does not automatically invalidate the parent's
  key. Check whether `touch: true` or an explicit dependency updates the outer key when rendered
  child data changes. Include user/tenant/locale context only where it affects cached output.

## Writes, constraints and concurrency

- `validates :key, uniqueness: { scope: :account_id }` does not serialize concurrent writers.
  When uniqueness is a **data invariant**, inspect the unique database index and its exact scope,
  case/NULL semantics and soft-delete predicate; don't demand an index for a UI-only hint.
- `exists?` then `create!` is racy across workers without a constraint, atomic write or lock. A
  Ruby mutex protects one process only. Check whether `with_lock` surrounds the state *read* and
  write, or whether a conditional update/unique index already enforces it.
- Database transactions roll back database changes, not outgoing HTTP calls, emails, files or
  enqueued work on an external queue. `after_commit` runs after persistence and can itself fail;
  enqueue-after-transaction-commit depends on this job's settings, Rails config and queue topology.
  Ask how a failed enqueue after commit is recovered before recommending an outbox.
- On PostgreSQL, rescuing a statement error **inside** a transaction and continuing SQL leaves
  that transaction unusable until rollback. Verify the actual adapter and rescue scope.
- Backfills/DDL: inspect table size, existing data, adapter, transaction policy, lock time and
  deploy order. Adding `NOT NULL` before backfill or a unique index before dedupe can fail deploy;
  dropping a used column in the same rollout can break old app workers. Reversible migrations
  are not automatically safe rollback for data already transformed.

## HTTP boundary and ownership

- A global `Model.find(params[:id])` is not inherently an IDOR: inspect the actual policy check,
  tenant scope and action. In tenant flows, test a real cross-account ID through read, update,
  destroy, download and background enqueue paths. A scope may be enforced by policy after lookup;
  report only a reachable breach.
- `params.expect(order: {})`, `permit!`, and `to_unsafe_h` broaden the input contract; check the
  exact attributes assigned. Rails 8 `params.expect` has specific nested-array syntax (`[[...]]`)
  and is not a mandatory replacement for correct `require(...).permit(...)` in older apps.
- `render json: model` may expose newly added sensitive columns, depending on serializers/model
  methods. `redirect_to` with attacker-controlled external targets, `html_safe`, unsafe SQL
  fragments/`Arel.sql`, and JSON endpoints using cookie auth without CSRF defense need a concrete
  input-to-sink trace, not a keyword hit.
- `current_user`/`Current.account` propagation into jobs, cache keys, Active Storage links and
  ActionCable channels can affect tenant isolation. Confirm what each subsystem serializes or
  scopes rather than assuming request context survives execution.

## Jobs and external effects

- Active Job may serialize records with GlobalID and load them *at execution time*. Deletion can
  raise `ActiveJob::DeserializationError` before `perform`; when absence is normal, evaluate an
  ID-based lookup with a narrow missing-record path. Do not blanket-discard deserialization
  failures if other missing records would mean lost work.
- Retried/double-delivered jobs can repeat external side effects after a worker dies between the
  external success and local update. Check stable operation IDs, gateway idempotency contracts,
  persisted state transitions and recovery. A local `paid?` flag alone does not prove the gateway
  was not already charged.
- `retry_on`, backend retries and manual retry loops may stack. Inspect actual queue adapter and
  error classes, bound retry count, and whether retry replays a non-idempotent write. Jobs should
  log enough safe identifiers to reconcile a stuck state without exposing secrets.

## Design and tests: ask, don't prescribe

- Rails models/associations/scopes, controllers, concerns, callbacks, jobs and RESTful resources
  can be sufficient. A service, domain operation, form or repository must solve a concrete boundary
  or repeated complex behavior, not a method-length threshold or theoretical layering concern.
  Inspect the repo's chosen domain pattern before suggesting a rewrite; don't import another
  team's style into a coherent existing architecture.
- A boolean status may be sufficient. Consider a separate record only when the changed
  requirement needs the actor, timestamp, lifecycle or association/query semantics that the
  boolean cannot express cleanly; otherwise a new model adds needless state and joins.
- For each suspected issue, ask: would the existing test fail if this behavior regressed? Prefer
  a request test for ownership/parameter filtering, an integration test for side effects, a
  concurrency test for a race-prone invariant, or a job test for duplicate execution only when
  the changed contract warrants it. Do not request tests of framework behavior or mock echoes.

## Source checks

Use the installed `rails-guides` **matching the repository's Rails version** to verify details:
`references/action_controller_overview.md` (expect/strong params),
`references/active_record_querying.md` (batches/eager loading/scopes),
`references/active_record_callbacks.md` (callback order/after_commit), and
`references/active_job_basics.md` (enqueue timing/GlobalID). The local copies are snapshots;
check installed gem source or versioned official API docs when a version-sensitive claim matters.
This playbook adds failure scenarios and disproof checks, not a second set of universal rules.
