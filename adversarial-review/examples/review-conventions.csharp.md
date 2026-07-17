# Review conventions — C#/.NET example

Copy this file to `<your-repo>/.claude/review-conventions.md` and edit. The adversarial-review
skill reads it (if present) and passes a summary to every persona. Everything here is
*additive* review depth — the skill works without it.

<!-- Optional: pin the persona model so the skill stops asking. One of: inherit, sonnet, haiku -->
persona-model: inherit

## Language pitfalls to hunt (C#/.NET)

- `async void` anywhere outside a UI event handler; missing `await` (fire-and-forget);
  `.Result`/`.Wait()`/`.GetAwaiter().GetResult()` on async code paths (deadlock risk).
- `IDisposable`/`IAsyncDisposable` not disposed on all paths — especially `HttpResponseMessage`,
  DB connections/readers, and anything returned early or thrown past. `using` declarations preferred.
- `HttpClient` constructed per-request instead of via `IHttpClientFactory` (socket exhaustion).
- LINQ multiple enumeration of `IEnumerable<T>` backed by a query or generator.
- `DateTime.Now` where `DateTimeOffset.UtcNow` is meant; implicit local-time arithmetic.
- Culture traps: `ToString()`/`Parse` without `CultureInfo.InvariantCulture` on data meant for machines.
- `ConfigureAwait(false)` policy: required in library code here, not in app code.
- Exceptions: never catch bare `Exception` to continue; `throw;` not `throw ex;` (stack trace loss).
- Concurrency: prefer `Interlocked`/`Channel<T>`/immutable snapshots over `lock` on hot paths;
  no `lock (this)` or lock on public objects.

## Project invariants (edit for your repo)

- All public API surface changes require an interface-first design and a semver note in the PR.
- Data access goes through the repository layer — no inline SQL outside `*/Data/`.
- Feature flags gate all new externally-visible behavior.

## Severity calibration (edit for your repo)

- Anything that can corrupt persisted data is P0 regardless of likelihood.
- Missing tests on `Core/` projects is P1; on tooling/scripts it is P3.
