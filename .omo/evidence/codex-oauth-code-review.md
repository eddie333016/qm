# Exact-SHA Code Review: Codex OAuth

- Reviewed SHA: `420afb8ec1d42c0befb8df7dc5a608a9121e8d43`
- Requested ref: `fork/feat/codex-oauth`
- Checkout: `/private/tmp/qm-codex-oauth-review` detached at the reviewed SHA
- Merge base: `7f2c916360f1797a8ff2a77ce2ce40c5fabab087`
- Scope inspected: the full merge-base diff, 22 files, 2,993 additions and 378 deletions. The terminal commit alone changes `src/persistence/pg-pool.ts`.

## Verdict

**PASS**. No CRITICAL or HIGH findings. `codeQualityStatus: WATCH`; `recommendation: APPROVE`.

The PostgreSQL change no longer reaches into pg private `activeQuery`/`cancel` internals. It races queued acquisition against the abort signal, releases a late-acquired client with an error so the pool removes it, and releases an active client only once. Inspection of the installed `pg`/`pg-pool` implementation confirms that error release removes the client and `client.end()` marks the client as ending before it force-destroys a socket with an active query, avoiding the previous unsafe direct cancellation path. The surrounding caller catches timed-out durable-record failures, so the normal turn result remains intact.

OAuth lifecycle and scope handling were traced through startup, per-turn lock acquisition, runtime replacement, and close. The source auth file is copied only to the jailed Codex home, OAuth suppresses ambient OpenAI credential pass-through, source/child account claims are compared before persistence, rotated JWTs are verified, and close/startup-failure paths release their locks and clean the jail.

## Findings

### CRITICAL

None.

### HIGH

None.

### MEDIUM

1. `src/persistence/pg-pool.ts:96-134` has no regression test exercising the actual `pg.Pool` queue and active-query cancellation paths added by the reviewed SHA. `test/codex-harness.test.ts:1744-1789` only proves that the harness passes and aborts an `AbortSignal` to a synthetic recorder; it does not prove that a queued `pool.connect()` late-releases its client, that an active PostgreSQL query rejects cleanly, or that no `EISCONN`/unhandled rejection is emitted. All PostgreSQL tests were skipped without `DATABASE_URL`. Add an integration test using a real Postgres pool with `max: 1` that covers both paths and observes `unhandledRejection`.

### LOW

1. The programming perspective's module-size limit is exceeded by new `src/harness/codex-auth.ts` (417 pure LOC) and expanded `src/harness/codex-app-server.ts` (308 pure LOC); `src/harness/codex-harness.ts` is also now 1,514 pure LOC. This is maintainability debt, not a demonstrated correctness failure in this review. Future work should split by responsibility rather than extend these modules further.

## Skill-perspective check

Ran the required `omo:remove-ai-slops` and `omo:programming` review perspectives before judging maintainability and test relevance.

- `remove-ai-slops`: no deletion-only tests, tests that merely verify a requested removal, tautological tests, or unnecessary production parsing/normalization were found. The durable-record timeout test is behaviorally relevant, but it is insufficient for the Postgres implementation change noted above.
- `programming`: no new `as any`, `@ts-ignore`, `@ts-expect-error`, or brittle prompt-text assertions were introduced. The diff violates the module-size guidance noted in the LOW finding. The production boundary checks serve untrusted auth-file/JSON-RPC inputs and are warranted; they are not needless interior validation.

## Evidence independently checked

- `git ls-remote fork refs/heads/feat/codex-oauth` returned the reviewed SHA.
- Detached `HEAD` exactly matched the reviewed SHA; only pre-existing untracked `.omo/` and `.qa-artifacts/` were present.
- `node --experimental-test-module-mocks --test test/codex-harness.test.ts`: 46 passed.
- `node --experimental-test-module-mocks --test test/dev-cli-lib.test.ts test/model-credential-route.test.ts test/base-model-serviceability.test.ts`: 34 passed.
- `npm run typecheck`: passed.
- ESLint over changed tracked TypeScript files: passed. Repository-wide lint is not usable as branch evidence because pre-existing untracked `.qa-artifacts/*.mjs` have three unused-import errors.
- `git diff --check`: passed.
- A diff-only secret-pattern scan found no credential material.

The full root test command was started but its terminal output was truncated by the execution interface before a final summary; it is therefore not claimed as passing. Its output explicitly showed Postgres tests skipped because `DATABASE_URL` is not configured.

## Blockers

None. The MEDIUM regression-test gap should be addressed before relying on this cancellation path as production-proven, but it does not rise to a demonstrated CRITICAL or HIGH failure at this SHA.
