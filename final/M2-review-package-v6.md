# M2 review package v6: cleanup-diagnostics trust boundary, round 2

Date: 2026-11-12 · Branch `feat/landing-and-realm-preview`

> **Status: M2 is NOT complete and NOT ready for deployment.** This round changed only integration-test support code and its offline tests. The hosted Supabase database and Auth integration **remain untested**: `pnpm test:rls` and `pnpm test:integration` stopped on missing configuration (section 5).

## 1. Repository state

| | Value |
|---|---|
| HEAD | `10013c0fb9d7383893dcd6ea3399c938ad4ac21e` (unchanged since before M2 work began) |
| Staged files | 0 before this round and 0 after |
| Working tree | Uncommitted, as before: 28 modified, 3 deleted, 33 untracked entries (`git status --short`). This round's changes are inside the untracked `apps/web/test/` directory. |
| `.next` | `.next/types` snapshot `9f92fb46c33f037e` before and after; `BUILD_ID` `rYrvX0swBrsAr-hHFLhVl` before and after |

## 2. Findings and remediation

All changes are in `apps/web/test/integration/support/diagnostics.ts`, the one wiring line in `auth.test.ts`, and the offline tests.

**Protections kept from v5:**
- The module-private `WeakMap` trust record for errors made by `adminStepError`.
- An exception's `.message` and `.stack` are never read.
- Validation of UUID user ids, labels (`^[a-z0-9-]{1,32}$`), statuses (integers 100 to 599), codes (`^[a-z0-9_]{1,64}$`) and the fixed admin steps.
- Every deletion is attempted, in order, after earlier failures.

### Finding 1: `ADMIN_STEPS` was mutable at runtime and was the validator's source of truth

**Fix:**
- **Private, frozen source of truth.** The authoritative list is a private tuple, `STEPS = Object.freeze([...])`.
- **Lookup does not use `Array.prototype`.** `knownStep()` checks it with a plain indexed `===` loop, not `.includes` or `.indexOf`. Patching `Array.prototype` cannot widen what is accepted, and `===` rejects a `String` object that looks like a step.
- **Exported copy.** `ADMIN_STEPS` is now a **separate frozen copy**, `Object.freeze([...STEPS])`. The validator never reads it. Mutating it throws a `TypeError` (ES modules run in strict mode). Even if it could be mutated, that would not affect validation.
- **Hardened `subject` read.** `adminStepError` also treats a non-object `subject` as `{}` instead of reading from it.

**Tests (new, executed):**
- `ADMIN_STEPS` equals the four steps and `Object.isFrozen` is true.
- `push`, index assignment, `splice` and `Object.defineProperty` on it all throw a `TypeError`, and the list does not contain the injected text afterwards.
- With `Array.prototype.includes` patched to always return true and `Array.prototype.indexOf` to always return 0, an injected step still becomes the fixed `complete an admin step` and contains no sensitive text. The prototypes are restored in `finally`.
- `new String('delete test user')` is not accepted as a step.
- A `null` subject is accepted without being read.

### Finding 2: `diagnosticFor(e, fallback)` echoed caller-provided fallback text

**Fix:**
- The signature is now `diagnosticFor(e, userId)`.
- For a trusted error (in the `WeakMap`) it returns the recorded text.
- For anything else it formats `could not delete test user <safeUserId(userId)> (unexpected exception)` itself. A malformed id becomes the fixed placeholder `invalid-id`.
- No argument of `diagnosticFor` can place free text in its output.

**Tests (new, executed):**
- Nine hostile values passed as the id all produce exactly `could not delete test user invalid-id (unexpected exception)`, with no sensitive text:
  - a diagnostic-looking malicious string;
  - an email address;
  - a JWT-like token;
  - `password=…`;
  - a stack trace;
  - a newline injection carrying a forged second line;
  - an object with `toString`;
  - a number;
  - `undefined`.
- A valid id is formatted as before.
- For a trusted error the recorded text wins, and a hostile id argument is ignored.
- Existing v5 tests now pass a user id instead of the fallback; their expected outputs are unchanged.

### Finding 3: `cleanupFailureReport(failures, total)` interpolated arbitrary strings and totals

**Fix: the report accepts only data the module produced itself.**
- `deleteAllTestUsers(ids, deleteOne)` now returns an **opaque, frozen** `CleanupResult` of `{ attempted, failed }`. Both counts are computed here. The failure lines, each from `diagnosticFor`, are stored in a second module-private `WeakMap`, keyed by that result object, and frozen.
- `cleanupFailureReport(result)` takes **only** such a result:
  - it returns `null` when nothing failed;
  - otherwise it returns the header plus the stored lines;
  - for any other input it returns the fixed text `test-user cleanup report unavailable: the input was not produced by deleteAllTestUsers`, and never any of the input's content.
- There is no `total` parameter any more, so a malformed total cannot be supplied. The total is `ids.length` from the same call.
- `deleteAllTestUsers` rejects an `ids` value that is not an array with a fixed `TypeError` message, before calling `deleteOne`.
- `auth.test.ts` `afterAll` is updated to `cleanupFailureReport(await deleteAllTestUsers(created, deleteTestUser))`.

**Tests (new, executed):**
- Nine untrusted inputs give exactly the fixed text and nothing sensitive:
  - an array of injected strings;
  - an empty array;
  - a string;
  - a look-alike `{ attempted, failed }`;
  - a look-alike carrying `lines`;
  - malformed totals (`-1`, `NaN`);
  - `null`;
  - `undefined`;
  - a number.
- A spread copy and an `Object.create` of a real result are not trusted.
- A real result is frozen. Assigning `attempted` and defining a `lines` property both throw, and the report is unchanged.
- Hostile ids (an email address; a UUID followed by a newline and a forged line) and errors with malicious messages produce only internally generated lines, with `invalid-id` placeholders.
- `ids` that are a string, an array-like object, `null`, `undefined` or a number are rejected with the fixed message, and `deleteOne` is never called.

**Existing coverage kept.** v5's deletion and report tests still pass. They now read lines through the public report, and assertions on result counts were added:
- every deletion attempted in order (`{ attempted: 3, failed: 2 }`);
- arbitrary thrown messages replaced with fixed text;
- legitimate status and code preserved;
- a non-UUID id sanitised;
- no failures gives no report;
- the multiple-failure report.

The same applies to v5's trust-boundary tests:
- the export list (unchanged: `ADMIN_STEPS`, `adminStepError`, `cleanupFailureReport`, `deleteAllTestUsers`, `describeAuthError`, `diagnosticFor`, `safeUserId`);
- an ordinary `Error`, a look-alike class or object, and an object built from a trusted error's prototype are all untrusted;
- rewriting `.message` and `.stack` after creation has no effect;
- the Auth error's message and stack are never read;
- invalid ids, labels, steps, statuses and codes stay sanitised;
- all four legitimate steps;
- the mixed four-failure cleanup.

## 3. Every public API and how text injection is blocked

| Export | Accepts | Output is built from | Injection test |
|---|---|---|---|
| `ADMIN_STEPS` | (value) | frozen copy; not read by any validator | mutation and prototype tampering |
| `safeUserId(id)` | anything | the UUID or `invalid-id` | email address, newline, `undefined` |
| `describeAuthError(error)` | anything | validated status and code, or fixed text | 8 malformed statuses, 8 unsafe codes, string error |
| `adminStepError(step, subject, authError)` | structured | private step or fixed text, validated id or label, `describeAuthError` | injected step, String object, hostile subject, hostile Auth message and stack |
| `diagnosticFor(e, userId)` | error and id | recorded text, or fixed text with `safeUserId` | 9 hostile ids; untrusted errors |
| `deleteAllTestUsers(ids, deleteOne)` | array and callback | counts plus lines from `diagnosticFor` | hostile ids and errors; non-array ids |
| `cleanupFailureReport(result)` | an opaque result | stored lines and counts, or fixed text | 9 forged inputs, copies, mutation |

## 4. Files changed this round (3)

| File | Reason |
|---|---|
| `apps/web/test/integration/support/diagnostics.ts` | Findings 1 to 3: private frozen step list with an `===` lookup, frozen exported copy; `diagnosticFor(e, userId)`; opaque frozen `CleanupResult` plus a private lines `WeakMap`; `cleanupFailureReport(result)`; non-array `ids` rejected |
| `apps/web/test/integration/auth.test.ts` | One line: `cleanupFailureReport(await deleteAllTestUsers(created, deleteTestUser))` (the separate total argument was removed) |
| `apps/web/test/cleanup-diagnostics.test.ts` | Updated for the new signatures with the same assertions, plus 29 new tests. 75 tests, was 46. |

`apps/web/test/integration/support/admin.ts` is unchanged since v5; it is included in section 7 for context. No application source, docs, configuration, lockfile, credential file, SQL or script changed.

## 5. Commands and exit codes

All runs were from `E:\Sovereign`, after the final edit. Logs are in `C:\Users\Gamer\AppData\Local\Temp\claude\7858b1e08a6e3505\c502d650-febf-4861-a36a-da9f61d54e28\scratchpad\m2r6\`.

| Command | Exit | Result |
|---|---|---|
| `pnpm lint` (`eslint . --max-warnings 0 && prettier --check .`) | 0 | clean |
| `pnpm run build:libs` | 0 | |
| `pnpm --filter @sovereign/core --filter @sovereign/db --filter @sovereign/worker typecheck` | 0 | |
| `apps/web: ./node_modules/.bin/tsc -p tsconfig.json --noEmit` | 0 | type-checks `admin.ts` and `auth.test.ts` against the new API |
| `pnpm test` | 0 | core 5 · db 23 · worker 8 · web **278** (v5: 249; +29) · scripts 204 |
| `pnpm check:boundaries` | 0 | |
| `node --test scripts/` | 0 | 204 tests, 204 pass, 0 fail, 0 skipped |
| `pnpm test:rls` | **2** | `configuration error: SOVEREIGN_SQL_TEST_DB_URL is not set`; no connection attempted |
| `pnpm test:integration` | **1** | `Integration tests need .env.integration …`; 1 file failed, **8 tests skipped**, 0 passed |

During development: `npx vitest run test/cleanup-diagnostics.test.ts` passed 75 tests, and web `tsc --noEmit` exited 0.

**Hosted integration remains untested:**
- the pgTAP suite: control, `identity_signup`, `identity_rls`, `catalog`;
- the 8 Auth integration tests, including the real cleanup path through `adminStepError`, `deleteAllTestUsers` and `cleanupFailureReport`;
- `resetPasswordForEmail` against real Auth;
- browser checks;
- a copy build plus `check:bundle`.

## 6. Prohibited operations: none performed

- **Supabase:** no project was contacted, linked, inspected or configured, and no `supabase` CLI command was run.
- **Databases:** no migrations were applied or run against any database.
- **Credentials:** no credential or environment file was created, read or modified.
- **Dependencies:** nothing installed, removed or updated.
- **Git:** no stage, commit, push, reset, stash or discard. Staged files are 0 before and after, and HEAD is unchanged.
- **Servers:** none started or restarted.
- **`.next`:** not rebuilt or modified; the hashes are in section 1.
- **Hosted tests:** none run. The two commands above stop on missing configuration before connecting.

**Limitations:**
- Trust holds within one loaded instance of the module. A result or error from a duplicated module instance gets fixed text: less detail, never leaked text.
- The `Array.prototype` tampering test patches globals inside one test and restores them in `finally`.
- v4's remaining limitations are unchanged.
- No new security or product decision was made this round.

## 7. Complete revised sources

The full current contents of the three changed files follow, with `admin.ts` (unchanged) for context.

### `apps/web/test/integration/support/diagnostics.ts`

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
  const lines: string[] = [];
  for (const id of ids) {
    try {
      await deleteOne(id);
    } catch (e) {
      lines.push(diagnosticFor(e, id));
    }
  }
  const result: CleanupResult = Object.freeze({ attempted: ids.length, failed: lines.length });
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

### `apps/web/test/integration/auth.test.ts`

```ts
// Supabase Auth flows and the server-side verified-email gate (M2), against the isolated development
// project. This layer does NOT prove RLS correctness; the pgTAP suite (pnpm test:rls) does that.
// Test users are created and deleted through the admin helper; no confirmation or reset email is sent.
import { afterAll, beforeAll, describe, expect, it } from 'vitest';
import { checkVerifiedUser } from '@/lib/auth/gate';
import { TEST_PASSWORD, confirmTestUser, createTestUser, deleteTestUser, generateRecoveryToken } from './support/admin';
import { cleanupFailureReport, deleteAllTestUsers } from './support/diagnostics';
import { publicClient } from './support/user';

// Every user is recorded the moment createTestUser returns, so a later failure in beforeAll or in a test
// can never leave an untracked user behind.
const created: string[] = [];
const track = <T extends { id: string }>(user: T): T => {
  created.push(user.id);
  return user;
};
let unverified: { id: string; email: string };
let verifiedA: { id: string; email: string };
let verifiedB: { id: string; email: string };

beforeAll(async () => {
  unverified = track(await createTestUser('unverified', false));
  verifiedA = track(await createTestUser('a', true));
  verifiedB = track(await createTestUser('b', true));
});

afterAll(async () => {
  // Attempt every deletion, then fail loudly if any did not succeed. The report carries only validated user
  // ids, validated Auth status/code and fixed text (support/diagnostics.ts); an unknown exception's message
  // is never read.
  const report = cleanupFailureReport(await deleteAllTestUsers(created, deleteTestUser));
  if (report) throw new Error(report);
});

describe('sign-in and the verified-email requirement', () => {
  it('rejects password sign-in for an unconfirmed address (no session is issued)', async () => {
    const client = publicClient();
    const { data, error } = await client.auth.signInWithPassword({ email: unverified.email, password: TEST_PASSWORD });
    expect(data.session).toBeNull();
    expect(error?.code).toBe('email_not_confirmed');
    expect(await checkVerifiedUser(client)).toEqual({ status: 'signed-out' });
  });

  it('allows sign-in once the address is confirmed, and the gate reports verified', async () => {
    await confirmTestUser(unverified.id);
    const client = publicClient();
    const { data, error } = await client.auth.signInWithPassword({ email: unverified.email, password: TEST_PASSWORD });
    expect(error).toBeNull();
    expect(data.session).not.toBeNull();
    expect(await checkVerifiedUser(client)).toMatchObject({ status: 'verified', userId: unverified.id });
  });

  it('rejects a wrong password', async () => {
    const { data, error } = await publicClient().auth.signInWithPassword({ email: verifiedA.email, password: 'definitely-wrong-1' });
    expect(data.session).toBeNull();
    expect(error).not.toBeNull();
  });
});

describe('the gate trusts only tokens the Auth server validates', () => {
  it('treats a forged access token as signed out', async () => {
    const forged = [
      Buffer.from(JSON.stringify({ alg: 'HS256', typ: 'JWT' })).toString('base64url'),
      Buffer.from(JSON.stringify({ sub: verifiedA.id, role: 'authenticated', email_confirmed_at: new Date().toISOString(), exp: 4102444800 })).toString('base64url'),
      'not-a-real-signature',
    ].join('.');
    const client = publicClient(forged);
    expect(await checkVerifiedUser({ auth: { getUser: () => client.auth.getUser(forged) } })).toEqual({ status: 'signed-out' });
  });

  it('treats a signed-out session as signed out', async () => {
    const client = publicClient();
    const { data } = await client.auth.signInWithPassword({ email: verifiedA.email, password: TEST_PASSWORD });
    const token = data.session?.access_token;
    expect(token).toBeTruthy();
    await client.auth.signOut();
    expect(await checkVerifiedUser(client)).toEqual({ status: 'signed-out' });
    const stale = publicClient(token);
    expect(await checkVerifiedUser({ auth: { getUser: () => stale.auth.getUser(token) } })).toEqual({ status: 'signed-out' });
  });
});

describe('password recovery flow (no email is sent)', () => {
  // The recovery token comes from the Auth admin API (generateLink), which never sends email. The app's own
  // request step, supabase.auth.resetPasswordForEmail(), is NOT exercised here because it sends a real email;
  // that acceptance criterion stays open until a safe email sink exists (see the M2 review package).
  it('a recovery token signs the user in, the new password works and the old one stops working', async () => {
    const recovering = track(await createTestUser('recovery', true));
    const { tokenHash, type } = await generateRecoveryToken(recovering.email, 'http://localhost:3000/auth/callback?next=/auth/update-password');
    expect(type).toBe('recovery');

    const client = publicClient();
    const { data: verified, error: verifyError } = await client.auth.verifyOtp({ type: 'recovery', token_hash: tokenHash });
    expect(verifyError).toBeNull();
    expect(verified.session?.user.id).toBe(recovering.id);
    expect(await checkVerifiedUser(client)).toMatchObject({ status: 'verified', userId: recovering.id });

    const newPassword = `${TEST_PASSWORD}-new`;
    expect((await client.auth.updateUser({ password: newPassword })).error).toBeNull();

    const old = await publicClient().auth.signInWithPassword({ email: recovering.email, password: TEST_PASSWORD });
    expect(old.data.session).toBeNull();
    const fresh = await publicClient().auth.signInWithPassword({ email: recovering.email, password: newPassword });
    expect(fresh.error).toBeNull();
  });

  it('a recovery token cannot be used twice', async () => {
    const reuse = track(await createTestUser('recovery-reuse', true));
    const { tokenHash } = await generateRecoveryToken(reuse.email, 'http://localhost:3000/auth/callback?next=/auth/update-password');
    expect((await publicClient().auth.verifyOtp({ type: 'recovery', token_hash: tokenHash })).error).toBeNull();
    const second = await publicClient().auth.verifyOtp({ type: 'recovery', token_hash: tokenHash });
    expect(second.error).not.toBeNull();
    expect(second.data.session).toBeNull();
  });
});

describe('client wiring (not an RLS proof)', () => {
  it("a signed-in user's client carries that user's session to the database", async () => {
    const client = publicClient();
    await client.auth.signInWithPassword({ email: verifiedA.email, password: TEST_PASSWORD });
    const { data, error } = await client.from('workspaces').select('created_by, kind');
    expect(error).toBeNull();
    expect(data).toEqual([{ created_by: verifiedA.id, kind: 'personal' }]);
    expect(data?.some((w) => w.created_by === verifiedB.id)).toBe(false);
  });
});
```

### `apps/web/test/cleanup-diagnostics.test.ts`

```ts
// Offline tests for the integration suite's sanitised diagnostics (test/integration/support/diagnostics.ts).
// No Supabase client is involved; deletions are test doubles.
import { describe, expect, it, vi } from 'vitest';
import * as diagnostics from './integration/support/diagnostics';
import {
  ADMIN_STEPS,
  type AdminStep,
  type CleanupResult,
  adminStepError,
  cleanupFailureReport,
  deleteAllTestUsers,
  describeAuthError,
  diagnosticFor,
  safeUserId,
} from './integration/support/diagnostics';

const ID1 = '00000000-0000-4000-8000-000000000001';
const ID2 = '00000000-0000-4000-8000-000000000002';
const ID3 = '00000000-0000-4000-8000-000000000003';
const SENSITIVE = /@|eyJ|token|secret|password|Passw0rd|privileged-key|stack|at \w+ \(/i;

describe('describeAuthError', () => {
  it('shows a valid status and code', () => {
    expect(describeAuthError({ status: 404, code: 'user_not_found', message: 'no user a@example.com' })).toBe('status=404, code=user_not_found');
  });

  it('marks absent values as none and a missing error as no data', () => {
    expect(describeAuthError({})).toBe('status=none, code=none');
    expect(describeAuthError(null)).toBe('no data returned');
    expect(describeAuthError(undefined)).toBe('no data returned');
  });

  it.each([
    ['a string', '500'],
    ['a fraction', 404.5],
    ['NaN', Number.NaN],
    ['Infinity', Number.POSITIVE_INFINITY],
    ['below 100', 0],
    ['above 599', 600],
    ['negative', -1],
    ['an object', { toString: () => 'a@example.com' }],
  ])('marks a malformed status (%s) as invalid', (_name, status) => {
    expect(describeAuthError({ status, code: 'x' })).toBe('status=invalid, code=x');
  });

  it.each([
    ['an email address', 'a@example.com'],
    ['a token', 'eyJhbGciOiJIUzI1NiJ9.payload'],
    ['upper case', 'User_Not_Found'],
    ['spaces', 'user not found'],
    ['a newline', 'code\ninjected line'],
    ['too long', 'a'.repeat(65)],
    ['empty', ''],
    ['not a string', 42],
  ])('marks an unsafe code (%s) as invalid', (_name, code) => {
    expect(describeAuthError({ status: 400, code })).toBe('status=400, code=invalid');
  });

  it('accepts a code at the 64-character limit', () => {
    expect(describeAuthError({ status: 400, code: 'a'.repeat(64) })).toBe(`status=400, code=${'a'.repeat(64)}`);
  });

  it('never reads a non-object error', () => {
    expect(describeAuthError('a@example.com token=eyJ')).toBe('status=invalid, code=invalid');
  });
});

describe('safeUserId', () => {
  it('keeps a UUID and replaces anything else', () => {
    expect(safeUserId(ID1)).toBe(ID1);
    expect(safeUserId('a@example.com')).toBe('invalid-id');
    expect(safeUserId(`${ID1}\ninjected`)).toBe('invalid-id');
    expect(safeUserId(undefined)).toBe('invalid-id');
  });
});

/** The failure lines of a cleanup result, read back through the public report (empty when nothing failed). */
const linesOf = (result: CleanupResult): string[] => cleanupFailureReport(result)?.split('\n').slice(1) ?? [];

describe('deleteAllTestUsers', () => {
  it('attempts every deletion even after failures, in order', async () => {
    const deleteOne = vi.fn((id: string) => (id === ID3 ? Promise.resolve() : Promise.reject(new Error('boom'))));
    const result = await deleteAllTestUsers([ID1, ID2, ID3], deleteOne);
    expect(deleteOne.mock.calls.map((c) => c[0])).toEqual([ID1, ID2, ID3]);
    expect(result).toEqual({ attempted: 3, failed: 2 });
    expect(linesOf(result)).toHaveLength(2);
  });

  it('replaces arbitrary thrown messages with fixed text', async () => {
    const leaky = new Error('delete failed for a@example.com with key privileged-key-abc token=eyJxyz');
    leaky.stack = 'Error: secret\n    at adminClient (admin.ts:12)';
    const thrown: unknown[] = [leaky, 'a@example.com', { message: 'password=It-x-Passw0rd!' }, undefined, new TypeError('fetch failed: secret')];
    const ids = thrown.map((_, i) => `00000000-0000-4000-8000-00000000000${i}`);
    const failures = linesOf(await deleteAllTestUsers(ids, (id) => Promise.reject(thrown[ids.indexOf(id)])));
    expect(failures).toEqual(ids.map((id) => `could not delete test user ${id} (unexpected exception)`));
    for (const line of failures) expect(line).not.toMatch(SENSITIVE);
  });

  it('keeps the status and code of a legitimate structured diagnostic', async () => {
    const result = await deleteAllTestUsers([ID1], (id) =>
      Promise.reject(adminStepError('delete test user', { userId: id }, { status: 500, code: 'unexpected_failure', message: 'a@example.com' })),
    );
    expect(linesOf(result)).toEqual([`could not delete test user ${ID1} (status=500, code=unexpected_failure)`]);
  });

  it('sanitises an id that is not a UUID in the fallback text', async () => {
    expect(linesOf(await deleteAllTestUsers(['a@example.com'], () => Promise.reject(new Error('x'))))).toEqual([
      'could not delete test user invalid-id (unexpected exception)',
    ]);
  });

  it('returns no failures when every deletion succeeds', async () => {
    const result = await deleteAllTestUsers([ID1, ID2], () => Promise.resolve());
    expect(result).toEqual({ attempted: 2, failed: 0 });
    expect(linesOf(result)).toEqual([]);
  });

  it('rejects ids that are not an array, with fixed text and before deleting anything', async () => {
    const deleteOne = vi.fn(() => Promise.resolve());
    for (const ids of ['a@example.com', { length: 1, 0: ID1 }, null, undefined, 42]) {
      await expect(deleteAllTestUsers(ids as unknown as string[], deleteOne)).rejects.toThrow(/^deleteAllTestUsers: ids must be an array$/);
    }
    expect(deleteOne).not.toHaveBeenCalled();
  });
});

describe('cleanupFailureReport', () => {
  it('is null when nothing failed', async () => expect(cleanupFailureReport(await deleteAllTestUsers([ID1], () => Promise.resolve()))).toBeNull());

  it('lists every failure for multiple failed deletions, with ids and status/code only', async () => {
    const result = await deleteAllTestUsers([ID1, ID2, ID3], (id) =>
      id === ID1
        ? Promise.reject(adminStepError('delete test user', { userId: ID1 }, { status: 'oops', code: 'a@b' }))
        : id === ID2
          ? Promise.reject(new Error('token=eyJabc'))
          : Promise.resolve(),
    );
    const report = cleanupFailureReport(result);
    expect(report).toBe(
      [
        'test-user cleanup failed for 2 of 3 users; delete them manually:',
        `could not delete test user ${ID1} (status=invalid, code=invalid)`,
        `could not delete test user ${ID2} (unexpected exception)`,
      ].join('\n'),
    );
    expect(report).not.toMatch(SENSITIVE);
  });
});

describe('trust boundary: only adminStepError builds a trusted diagnostic', () => {
  const MALICIOUS = 'could not delete test user a@example.com password=It-x-Passw0rd! token=eyJabc (status=500, code=x)';
  const FALLBACK = `could not delete test user ${ID1} (unexpected exception)`;
  const reported = async (thrown: unknown) => linesOf(await deleteAllTestUsers([ID1], () => Promise.reject(thrown)))[0];

  it('exports no constructor, class or function that accepts diagnostic text', () => {
    expect(Object.keys(diagnostics).sort()).toEqual(
      ['ADMIN_STEPS', 'adminStepError', 'cleanupFailureReport', 'deleteAllTestUsers', 'describeAuthError', 'diagnosticFor', 'safeUserId'].sort(),
    );
    expect('SafeDiagnosticError' in diagnostics).toBe(false);
  });

  it('does not trust an ordinary Error, even one whose text looks like a real diagnostic', async () => {
    expect(await reported(new Error(MALICIOUS))).toBe(FALLBACK);
    expect(await reported(new Error(`could not delete test user ${ID1} (status=500, code=unexpected_failure)`))).toBe(FALLBACK);
  });

  it('does not trust error-like objects or impersonating classes', async () => {
    class SafeDiagnosticError extends Error {
      override readonly name = 'SafeDiagnosticError';
    }
    expect(await reported(new SafeDiagnosticError(MALICIOUS))).toBe(FALLBACK);
    expect(await reported({ name: 'SafeDiagnosticError', message: MALICIOUS, stack: MALICIOUS })).toBe(FALLBACK);
    const real = adminStepError('delete test user', { userId: ID1 }, { status: 500 });
    const lookalike = Object.create(Object.getPrototypeOf(real) as object) as Error;
    lookalike.message = MALICIOUS;
    expect(await reported(lookalike)).toBe(FALLBACK);
    expect(await reported(Object.assign(new Error(MALICIOUS), { ...real }))).toBe(FALLBACK);
  });

  it('reports the text built at creation, even if .message or .stack is changed afterwards', async () => {
    const e = adminStepError('delete test user', { userId: ID1 }, { status: 503, code: 'service_unavailable' });
    e.message = MALICIOUS;
    e.stack = `Error: ${MALICIOUS}\n    at secret (admin.ts:1)`;
    expect(await reported(e)).toBe(`could not delete test user ${ID1} (status=503, code=service_unavailable)`);
  });

  it('never reads the message or stack of the Auth error it is given', async () => {
    const authError = Object.assign(new Error('mail to a@example.com failed, token=eyJabc password=hunter22'), { status: 400, code: 'bad_request' });
    authError.stack = 'Error: secret\n    at adminClient (admin.ts:12)';
    const line = await reported(adminStepError('delete test user', { userId: ID1 }, authError));
    expect(line).toBe(`could not delete test user ${ID1} (status=400, code=bad_request)`);
    expect(line).not.toMatch(SENSITIVE);
  });

  it.each([
    ['an email as the user id', { userId: 'a@example.com' }, 'could not delete test user invalid-id (status=none, code=none)'],
    ['a newline in the user id', { userId: `${ID1}\ntoken=eyJ` }, 'could not delete test user invalid-id (status=none, code=none)'],
    ['an email as the label', { label: 'a@example.com' }, 'could not delete test user invalid-label (status=none, code=none)'],
    ['an over-long label', { label: 'a'.repeat(33) }, 'could not delete test user invalid-label (status=none, code=none)'],
  ])('sanitises %s', async (_name, subject, expected) => {
    expect(await reported(adminStepError('delete test user', subject, {}))).toBe(expected);
  });

  it('replaces a step outside the fixed list', async () => {
    const step = 'leak a@example.com token=eyJabc' as unknown as AdminStep;
    expect(await reported(adminStepError(step, { userId: ID1 }, { status: 500 }))).toBe(`could not complete an admin step ${ID1} (status=500, code=none)`);
  });

  it.each([
    [{ status: 1e3, code: 'ok_code' }, 'status=invalid, code=ok_code'],
    [{ status: '500', code: 'x' }, 'status=invalid, code=x'],
    [{ status: 500, code: 'eyJhbGciOi.payload' }, 'status=500, code=invalid'],
    [{ status: 500, code: 'a@example.com' }, 'status=500, code=invalid'],
    ['a@example.com token=eyJ', 'status=invalid, code=invalid'],
    [null, 'no data returned'],
  ])('keeps invalid status and code values sanitised (%j)', async (authError, expected) => {
    expect(await reported(adminStepError('delete test user', { userId: ID1 }, authError))).toBe(`could not delete test user ${ID1} (${expected})`);
  });

  it('builds legitimate diagnostics for every admin step', () => {
    expect(adminStepError('create test user', { label: 'recovery-reuse' }, { status: 422, code: 'email_exists' }).message).toBe(
      'could not create test user recovery-reuse (status=422, code=email_exists)',
    );
    expect(adminStepError('confirm test user', { userId: ID2 }, { status: 404, code: 'user_not_found' }).message).toBe(
      `could not confirm test user ${ID2} (status=404, code=user_not_found)`,
    );
    expect(adminStepError('generate a recovery link', {}, null).message).toBe('could not generate a recovery link (no data returned)');
    expect(diagnosticFor(adminStepError('delete test user', { userId: ID3 }, { status: 429 }), ID1)).toBe(
      `could not delete test user ${ID3} (status=429, code=none)`,
    );
    expect(diagnosticFor(new Error('x'), ID1)).toBe(FALLBACK);
    expect(diagnosticFor(undefined, ID1)).toBe(FALLBACK);
  });

  it('still attempts every deletion when trusted, untrusted and impersonating failures are mixed', async () => {
    const ids = [ID1, ID2, ID3, '00000000-0000-4000-8000-000000000004'];
    const deleteOne = vi.fn((id: string) => {
      if (id === ID1) return Promise.reject(adminStepError('delete test user', { userId: id }, { status: 500, code: 'unexpected_failure' }));
      if (id === ID2) return Promise.reject(new Error(MALICIOUS));
      if (id === ID3) return Promise.reject({ name: 'SafeDiagnosticError', message: MALICIOUS });
      return Promise.resolve();
    });
    const report = cleanupFailureReport(await deleteAllTestUsers(ids, deleteOne));
    expect(deleteOne.mock.calls.map((c) => c[0])).toEqual(ids);
    expect(report).toBe(
      [
        'test-user cleanup failed for 3 of 4 users; delete them manually:',
        `could not delete test user ${ID1} (status=500, code=unexpected_failure)`,
        `could not delete test user ${ID2} (unexpected exception)`,
        `could not delete test user ${ID3} (unexpected exception)`,
      ].join('\n'),
    );
    expect(report).not.toMatch(SENSITIVE);
  });
});

describe('v6 finding 1: the step list cannot be changed at runtime', () => {
  const INJECTED = 'leak a@example.com token=eyJabc';

  it('exports a frozen copy of the steps', () => {
    expect(ADMIN_STEPS).toEqual(['create test user', 'confirm test user', 'generate a recovery link', 'delete test user']);
    expect(Object.isFrozen(ADMIN_STEPS)).toBe(true);
  });

  it('refuses mutation of the exported list (strict-mode TypeError), and the list is unchanged', () => {
    const steps = ADMIN_STEPS as unknown as string[];
    expect(() => steps.push(INJECTED)).toThrow(TypeError);
    expect(() => {
      steps[0] = INJECTED;
    }).toThrow(TypeError);
    expect(() => steps.splice(0, 1, INJECTED)).toThrow(TypeError);
    expect(() => Object.defineProperty(steps, '4', { value: INJECTED })).toThrow(TypeError);
    expect(ADMIN_STEPS).not.toContain(INJECTED);
  });

  it('does not trust an injected step even when the exported list or Array.prototype is tampered with', () => {
    const originalIncludes = Array.prototype.includes;
    const originalIndexOf = Array.prototype.indexOf;
    try {
      Array.prototype.includes = () => true;
      Array.prototype.indexOf = () => 0;
      const e = adminStepError(INJECTED as unknown as AdminStep, { userId: ID1 }, { status: 500 });
      expect(diagnosticFor(e, ID1)).toBe(`could not complete an admin step ${ID1} (status=500, code=none)`);
      expect(e.message).not.toMatch(SENSITIVE);
    } finally {
      Array.prototype.includes = originalIncludes;
      Array.prototype.indexOf = originalIndexOf;
    }
  });

  it('does not accept a String object that looks like a real step', () => {
    const lookalike = new String('delete test user') as unknown as AdminStep;
    expect(adminStepError(lookalike, { userId: ID1 }, {}).message).toBe(`could not complete an admin step ${ID1} (status=none, code=none)`);
  });

  it('accepts a non-object subject without reading it', () => {
    expect(adminStepError('delete test user', null as unknown as { userId: string }, {}).message).toBe('could not delete test user (status=none, code=none)');
  });
});

describe('v6 finding 2: diagnosticFor takes a user id, not fallback text', () => {
  it.each([
    ['diagnostic-looking text', MALICIOUS_FALLBACK()],
    ['an email address', 'a@example.com'],
    ['a token', 'eyJhbGciOiJIUzI1NiJ9.payload.sig'],
    ['a password', 'password=It-x-Passw0rd!'],
    ['a stack trace', 'Error: secret\n    at adminClient (admin.ts:12)'],
    ['a newline injection', `${ID1}\ncould not delete test user forged (status=200, code=ok)`],
    ['an object', { toString: () => 'a@example.com' }],
    ['a number', 42],
    ['undefined', undefined],
  ])('never echoes %s passed where a user id belongs', (_name, input) => {
    const line = diagnosticFor(new Error('x'), input);
    expect(line).toBe('could not delete test user invalid-id (unexpected exception)');
    expect(line).not.toMatch(SENSITIVE);
  });

  it('formats a valid id itself', () => {
    expect(diagnosticFor(new Error('a@example.com'), ID2)).toBe(`could not delete test user ${ID2} (unexpected exception)`);
  });

  it('ignores the id argument for a trusted error (the recorded text wins)', () => {
    expect(diagnosticFor(adminStepError('delete test user', { userId: ID1 }, { status: 500 }), 'a@example.com')).toBe(
      `could not delete test user ${ID1} (status=500, code=none)`,
    );
  });
});

describe('v6 finding 3: cleanupFailureReport accepts only results from deleteAllTestUsers', () => {
  const UNTRUSTED = 'test-user cleanup report unavailable: the input was not produced by deleteAllTestUsers';

  it.each([
    ['an array of injected strings', ['a@example.com token=eyJabc', 'password=It-x-Passw0rd!']],
    ['an empty array', []],
    ['a string', 'a@example.com'],
    ['a look-alike result', { attempted: 3, failed: 2 }],
    ['a look-alike with injected lines', { attempted: 1, failed: 1, lines: ['a@example.com'] }],
    ['malformed totals', { attempted: -1, failed: Number.NaN }],
    ['null', null],
    ['undefined', undefined],
    ['a number', 7],
  ])('gives fixed text for %s', (_name, input) => {
    const report = cleanupFailureReport(input as unknown as CleanupResult);
    expect(report).toBe(UNTRUSTED);
    expect(report).not.toMatch(SENSITIVE);
  });

  it('does not trust a copy of a real result', async () => {
    const real = await deleteAllTestUsers([ID1], () => Promise.reject(new Error('x')));
    expect(cleanupFailureReport({ ...real })).toBe(UNTRUSTED);
    expect(cleanupFailureReport(Object.create(real) as CleanupResult)).toBe(UNTRUSTED);
  });

  it('a real result is frozen: its counts cannot be altered to change the report', async () => {
    const real = await deleteAllTestUsers([ID1, ID2], (id) => (id === ID1 ? Promise.reject(new Error('x')) : Promise.resolve()));
    expect(Object.isFrozen(real)).toBe(true);
    expect(() => {
      (real as { attempted: number }).attempted = 999;
    }).toThrow(TypeError);
    expect(() => Object.defineProperty(real, 'lines', { value: ['a@example.com'] })).toThrow(TypeError);
    expect(cleanupFailureReport(real)).toBe(`test-user cleanup failed for 1 of 2 users; delete them manually:\n${FALLBACK_ID1}`);
  });

  it('carries only internally generated lines, even when ids and errors are hostile', async () => {
    const hostileIds = ['a@example.com', `${ID1}\nforged line`, ID2];
    const result = await deleteAllTestUsers(hostileIds, () => Promise.reject(new Error(MALICIOUS_FALLBACK())));
    expect(cleanupFailureReport(result)).toBe(
      [
        'test-user cleanup failed for 3 of 3 users; delete them manually:',
        'could not delete test user invalid-id (unexpected exception)',
        'could not delete test user invalid-id (unexpected exception)',
        `could not delete test user ${ID2} (unexpected exception)`,
      ].join('\n'),
    );
  });
});

function MALICIOUS_FALLBACK() {
  return 'could not delete test user a@example.com password=It-x-Passw0rd! token=eyJabc (status=500, code=x)';
}
const FALLBACK_ID1 = `could not delete test user ${ID1} (unexpected exception)`;
```

### `apps/web/test/integration/support/admin.ts` (unchanged since v5, for context)

```ts
// THE ONLY service-role client in the repository (ADR-0011). Used solely by the integration tests to create,
// confirm and delete throwaway users through the Auth admin API; never for table access, never imported by
// application code (a lint rule and test/service-role-isolation.test.ts enforce both).
import { createClient, type SupabaseClient } from '@supabase/supabase-js';
import { adminStepError } from './diagnostics';
import { integrationEnv } from './env';

let admin: SupabaseClient | undefined;

function adminClient(): SupabaseClient {
  if (!admin) {
    const { url, serviceRoleKey } = integrationEnv();
    admin = createClient(url, serviceRoleKey, { auth: { autoRefreshToken: false, persistSession: false } });
  }
  return admin;
}

const RUN = `${Date.now().toString(36)}${Math.random().toString(36).slice(2, 6)}`;
/** Addresses on the reserved example.com domain; the admin API sends no email for them. */
export const testEmail = (label: string) => `sovereign-it-${RUN}-${label}@example.com`;
export const TEST_PASSWORD = `It-${RUN}-Passw0rd!`;

export async function createTestUser(label: string, confirmed: boolean): Promise<{ id: string; email: string }> {
  const email = testEmail(label);
  const { data, error } = await adminClient().auth.admin.createUser({ email, password: TEST_PASSWORD, email_confirm: confirmed });
  if (error || !data.user) throw adminStepError('create test user', { label }, error);
  return { id: data.user.id, email };
}

export async function confirmTestUser(id: string): Promise<void> {
  const { error } = await adminClient().auth.admin.updateUserById(id, { email_confirm: true });
  if (error) throw adminStepError('confirm test user', { userId: id }, error);
}

/**
 * Creates a password-recovery token for a test user WITHOUT sending any email: the Auth admin API's
 * generateLink returns the token for the caller to deliver ("to be sent via a custom email provider").
 * This exercises the recovery flow; it does not exercise the app's own resetPasswordForEmail() request.
 */
export async function generateRecoveryToken(email: string, redirectTo: string): Promise<{ tokenHash: string; type: string }> {
  const { data, error } = await adminClient().auth.admin.generateLink({ type: 'recovery', email, options: { redirectTo } });
  if (error || !data.properties) throw adminStepError('generate a recovery link', {}, error);
  return { tokenHash: data.properties.hashed_token, type: data.properties.verification_type };
}

export async function deleteTestUser(id: string): Promise<void> {
  const { error } = await adminClient().auth.admin.deleteUser(id);
  if (error) throw adminStepError('delete test user', { userId: id }, error);
}
```
