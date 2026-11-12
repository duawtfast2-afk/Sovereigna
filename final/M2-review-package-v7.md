# M2 review package v7: cleanup attempt count

Date: 2026-11-12 · Branch `feat/landing-and-realm-preview` · HEAD `10013c0fb9d7383893dcd6ea3399c938ad4ac21e` (unchanged) · Staged files: 0 before and after.

> **Status: M2 is NOT complete and NOT ready for deployment.** This round changed only integration-test support code and its offline tests. The hosted Supabase database and Auth integration **remain untested**: `pnpm test:rls` and `pnpm test:integration` stopped on missing configuration.

## 1. Finding and fix

**Finding (v6 review).** `deleteAllTestUsers()` iterated the caller's live `ids` array and set `attempted = ids.length` after the asynchronous loop. A change to that array while `await deleteOne(id)` was pending could alter both the work list and the reported count.

**Fix, in `deleteAllTestUsers` only:**
- **Snapshot first.** Before the first `await`, the ids are copied into a private `work` array. It is an index loop, using no `Array.prototype` method, consistent with the v6 step lookup.
- **Explicit counter.** The loop runs over `work` and increments an `attempted` counter just before each `deleteOne` call.
- **Counts from the counter.** The result is `Object.freeze({ attempted, failed: lines.length })`; `attempted` no longer comes from the array's length.
- **Unchanged:**
  - ordering and continue-after-failure;
  - diagnostics, still built by `diagnosticFor(e, id)` with the snapshot's id;
  - the non-array `TypeError`;
  - the private `WeakMap` records;
  - the opaque, frozen `CleanupResult` and `cleanupFailureReport(result)`.

## 2. Files changed (2)

| File | Change |
|---|---|
| `apps/web/test/integration/support/diagnostics.ts` | `deleteAllTestUsers`: snapshot of ids, explicit attempt counter, `attempted` taken from the counter |
| `apps/web/test/cleanup-diagnostics.test.ts` | 2 new regression tests (77 tests, was 75); no existing test changed |

Nothing else changed. `auth.test.ts` and `admin.ts` are unchanged since v6.

## 3. New tests (executed, passing)

1. **"processes a snapshot of the ids and counts real attempts when the caller mutates the array mid-deletion".** The caller passes `[ID1, ID2, ID3]`.
   - While deleting `ID1`, after an `await`, the callback appends `ID4`, `ID5` and an email address, overwrites index 1 with a token string, and removes one element with `splice`. It then fails.
   - While deleting `ID2`, it empties the array (`length = 0`) and fails with a trusted `adminStepError`.
   - The test asserts:
     - the callback saw exactly `[ID1, ID2, ID3]`;
     - the result is exactly `{ attempted: 3, failed: 2 }`;
     - the caller's array really is empty afterwards;
     - the report is exactly the header "2 of 3", followed by the `ID1` fallback line and the `ID2` status/code line, with no sensitive text.

   Against the v6 code this test would fail, on both the visited ids and `attempted: 0`. That conclusion is from reading the code: the old code was not re-run.
2. **"counts an attempt even when deleteOne throws synchronously, and still continues".** A callback that throws synchronously for `ID1` still gives `{ attempted: 2, failed: 1 }`. Both ids are attempted, and the only line is the fixed `ID1` fallback.

## 4. Commands and exit codes

All runs were from `E:\Sovereign`, after the final edit. Logs are in `C:\Users\Gamer\AppData\Local\Temp\claude\7858b1e08a6e3505\c502d650-febf-4861-a36a-da9f61d54e28\scratchpad\m2r7\`.

| Command | Exit | Result |
|---|---|---|
| `apps/web: npx vitest run test/cleanup-diagnostics.test.ts` (focused) | 0 | 77 passed |
| `pnpm lint` (`eslint . --max-warnings 0 && prettier --check .`) | 0 | clean |
| `pnpm run build:libs` | 0 | |
| `pnpm --filter @sovereign/core --filter @sovereign/db --filter @sovereign/worker typecheck` | 0 | |
| `apps/web: ./node_modules/.bin/tsc -p tsconfig.json --noEmit` | 0 | |
| `pnpm test` | 0 | core 5 · db 23 · worker 8 · web **280** (v6: 278) · scripts 204 |
| `pnpm check:boundaries` | 0 | |
| `node --test scripts/` | 0 | 204 pass, 0 fail, 0 skipped |
| `pnpm test:rls` | **2** | `configuration error: SOVEREIGN_SQL_TEST_DB_URL is not set`; no connection attempted |
| `pnpm test:integration` | **1** | `Integration tests need .env.integration …`; **8 tests skipped**, 0 passed |

**Still untested:**
- the pgTAP suite;
- the 8 Auth integration tests, including the real cleanup path;
- `resetPasswordForEmail` against real Auth;
- browser checks;
- a copy build plus `check:bundle`.

## 5. State and prohibited operations

- **Repository:** HEAD is unchanged. There are 0 staged files before and after. The working tree is uncommitted as before: 28 modified, 3 deleted and 33 untracked entries.
- **`.next`:** the `.next/types` snapshot is `9f92fb46c33f037e` before and after, and `BUILD_ID` is `rYrvX0swBrsAr-hHFLhVl` before and after.
- **Supabase:** not contacted, linked, inspected or configured.
- **Databases:** no migrations were run.
- **Credentials:** no credential or environment file was read, created or modified.
- **Dependencies:** none changed.
- **Git:** no operations performed.
- **Servers:** none started or restarted.
- **`.next`:** not rebuilt.
- **Hosted tests:** none run.

## 6. Revised source

### `apps/web/test/integration/support/diagnostics.ts` (complete)

```ts
// Sanitised diagnostics for the integration-test helpers and their cleanup. Pure, no Supabase import, so it
// is unit-tested offline (test/cleanup-diagnostics.test.ts). Only these values ever reach a diagnostic:
//   - a user id, and only when it is a UUID (ids of throwaway test users);
//   - a test-user label, and only when it matches ^[a-z0-9-]{1,32}$;
//   - an HTTP status, and only when it is an integer from 100 to 599;
//   - an Auth error code, and only when it matches ^[a-z0-9_]{1,64}$;
//   - counts computed here, and fixed text written here (including the private list of admin steps).
// Never an exception's message or stack, an email address, a password, a key, a token or a raw response.
//
// Trust boundary: no public function accepts diagnostic text.
//   - adminStepError() builds its message here from structured, validated inputs and records it in a
//     module-private WeakMap; only errors in that map are trusted, and the recorded text (not the error's
//     current .message) is what gets reported.
//   - The authoritative step list is a private frozen tuple, checked with a plain === loop; the exported
//     ADMIN_STEPS is a separate frozen copy, so nothing done to it can change what is accepted.
//   - diagnosticFor() takes a user id, not fallback text, and formats the fallback itself.
//   - deleteAllTestUsers() returns an opaque, frozen result whose failure lines live in a second private
//     WeakMap; cleanupFailureReport() accepts only such a result, and anything else yields fixed text.

const UUID = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
const CODE = /^[a-z0-9_]{1,64}$/;
const LABEL = /^[a-z0-9-]{1,32}$/;

const STEPS = Object.freeze(['create test user', 'confirm test user', 'generate a recovery link', 'delete test user'] as const);
export type AdminStep = (typeof STEPS)[number];
/** A frozen copy for callers; the validator never reads it. */
export const ADMIN_STEPS: readonly AdminStep[] = Object.freeze([...STEPS]);

/** The step itself when it is one of the private steps (compared with ===), otherwise null. */
function knownStep(step: unknown): AdminStep | null {
  for (let i = 0; i < STEPS.length; i++) if (STEPS[i] === step) return STEPS[i] as AdminStep;
  return null;
}

/** The id when it is a UUID, otherwise a fixed placeholder. */
export const safeUserId = (id: unknown): string => (typeof id === 'string' && UUID.test(id) ? id : 'invalid-id');

const safeLabel = (label: unknown): string => (typeof label === 'string' && LABEL.test(label) ? label : 'invalid-label');

/** Status and code of an Auth error, each validated; never its message. */
export function describeAuthError(error: unknown): string {
  if (error === null || error === undefined) return 'no data returned';
  if (typeof error !== 'object') return 'status=invalid, code=invalid';
  const { status, code } = error as { status?: unknown; code?: unknown };
  const s = status === undefined ? 'none' : typeof status === 'number' && Number.isInteger(status) && status >= 100 && status <= 599 ? String(status) : 'invalid';
  const c = code === undefined ? 'none' : typeof code === 'string' && CODE.test(code) ? code : 'invalid';
  return `status=${s}, code=${c}`;
}

// Errors created by adminStepError, mapped to the text built for them at creation time.
const trustedErrors = new WeakMap<object, string>();

/**
 * The error an admin helper throws when a step fails. Every part of its message is validated or fixed here:
 * the step must be one of the private steps, the subject is a UUID user id or a plain label, and the Auth
 * error contributes only its validated status and code.
 */
export function adminStepError(step: AdminStep, subject: { readonly userId?: unknown; readonly label?: unknown }, authError: unknown): Error {
  const fixedStep = knownStep(step) ?? 'complete an admin step';
  const s = typeof subject === 'object' && subject !== null ? subject : {};
  const who = 'userId' in s ? ` ${safeUserId(s.userId)}` : 'label' in s ? ` ${safeLabel(s.label)}` : '';
  const text = `could not ${fixedStep}${who} (${describeAuthError(authError)})`;
  const error = new Error(text);
  trustedErrors.set(error, text);
  return error;
}

/**
 * The diagnostic for a failed deletion of `userId`: the recorded text of an error made by adminStepError,
 * otherwise fixed text naming the validated id. Never reads the error's message or stack.
 */
export function diagnosticFor(e: unknown, userId: unknown): string {
  const recorded = typeof e === 'object' && e !== null ? trustedErrors.get(e) : undefined;
  return recorded ?? `could not delete test user ${safeUserId(userId)} (unexpected exception)`;
}

/** Opaque outcome of deleteAllTestUsers: counts only; the failure lines are held privately. */
export interface CleanupResult {
  readonly attempted: number;
  readonly failed: number;
}
const cleanupLines = new WeakMap<object, readonly string[]>();

/**
 * Attempts to delete every user, even after failures, and returns an opaque result. Each failed deletion is
 * described by diagnosticFor(); unknown exceptions are reported with fixed text.
 */
export async function deleteAllTestUsers(ids: readonly string[], deleteOne: (id: string) => Promise<void>): Promise<CleanupResult> {
  if (!Array.isArray(ids)) throw new TypeError('deleteAllTestUsers: ids must be an array');
  // Snapshot before the first await: the caller's array may change while a deletion is pending, and must
  // change neither the work list nor the counts.
  // Copied with an index loop (no Array.prototype method), like the step lookup above.
  const work: unknown[] = [];
  for (let i = 0, n = ids.length; i < n; i++) work[i] = ids[i];
  const lines: string[] = [];
  let attempted = 0;
  for (let i = 0; i < work.length; i++) {
    const id = work[i] as string;
    attempted++;
    try {
      await deleteOne(id);
    } catch (e) {
      lines.push(diagnosticFor(e, id));
    }
  }
  const result: CleanupResult = Object.freeze({ attempted, failed: lines.length });
  cleanupLines.set(result, Object.freeze(lines));
  return result;
}

/**
 * The cleanup failure report for a result returned by deleteAllTestUsers, or null when every deletion
 * succeeded. Any other input (strings, arrays, look-alike objects) yields fixed text, never its content.
 */
export function cleanupFailureReport(result: CleanupResult): string | null {
  const lines = typeof result === 'object' && result !== null ? cleanupLines.get(result) : undefined;
  if (!lines) return 'test-user cleanup report unavailable: the input was not produced by deleteAllTestUsers';
  if (lines.length === 0) return null;
  return `test-user cleanup failed for ${lines.length} of ${result.attempted} users; delete them manually:\n${lines.join('\n')}`;
}
```

### `apps/web/test/cleanup-diagnostics.test.ts` (the two new tests, lines 118 to 163; the rest of the file is unchanged since v6)

```ts
  it('processes a snapshot of the ids and counts real attempts when the caller mutates the array mid-deletion', async () => {
    const ID4 = '00000000-0000-4000-8000-000000000004';
    const ID5 = '00000000-0000-4000-8000-000000000005';
    const ids = [ID1, ID2, ID3];
    const seen: string[] = [];
    const deleteOne = async (id: string) => {
      seen.push(id);
      if (id === ID1) {
        // While the first deletion is pending: grow, rewrite and shrink the caller's array.
        await Promise.resolve();
        ids.push(ID4, ID5, 'a@example.com');
        ids[1] = 'token=eyJabc';
        ids.splice(2, 1);
        throw new Error('first deletion failed');
      }
      if (id === ID2) {
        ids.length = 0; // emptied mid-run: must not cut the work short or change the counts
        throw adminStepError('delete test user', { userId: id }, { status: 500, code: 'unexpected_failure' });
      }
    };
    const result = await deleteAllTestUsers(ids, deleteOne);
    expect(seen).toEqual([ID1, ID2, ID3]);
    expect(result).toEqual({ attempted: 3, failed: 2 });
    expect(ids).toEqual([]);
    const report = cleanupFailureReport(result);
    expect(report).toBe(
      [
        'test-user cleanup failed for 2 of 3 users; delete them manually:',
        `could not delete test user ${ID1} (unexpected exception)`,
        `could not delete test user ${ID2} (status=500, code=unexpected_failure)`,
      ].join('\n'),
    );
    expect(report).not.toMatch(SENSITIVE);
  });

  it('counts an attempt even when deleteOne throws synchronously, and still continues', async () => {
    const deleteOne = vi.fn((id: string) => {
      if (id === ID1) throw new Error('sync failure a@example.com');
      return Promise.resolve();
    });
    const result = await deleteAllTestUsers([ID1, ID2], deleteOne);
    expect(deleteOne).toHaveBeenCalledTimes(2);
    expect(result).toEqual({ attempted: 2, failed: 1 });
    expect(linesOf(result)).toEqual([`could not delete test user ${ID1} (unexpected exception)`]);
  });

```
