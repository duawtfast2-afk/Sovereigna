# M2 review package v5: cleanup-diagnostics trust boundary

Date: 2026-11-12 · Branch `feat/landing-and-realm-preview` · HEAD `10013c0` (unchanged) · Staged files: 0 before and after.

> **Status: M2 is NOT complete and NOT ready for deployment.** This round changed only test-support code and its offline tests. Hosted Supabase database and Auth integration **remain untested**: `pnpm test:rls` and `pnpm test:integration` stopped on missing configuration, as before (section 4).

## 1. Finding and remediation

**Finding (v4 review).** `SafeDiagnosticError` was exported with a constructor that accepted arbitrary text. `diagnosticFor()` then trusted any instance of it and forwarded its `.message`. A mistakenly built `SafeDiagnosticError` could therefore carry an email address, token or stack into a cleanup report.

**Remediation (`apps/web/test/integration/support/diagnostics.ts`):**
- **`SafeDiagnosticError` is removed.** No exported class or function accepts diagnostic text.
- **Structured input only.** The only way to make a trusted diagnostic is `adminStepError(step, subject, authError)`.
  - `step` must be one of the fixed `ADMIN_STEPS`: `create test user`, `confirm test user`, `generate a recovery link`, `delete test user`. Any other value, for example one forced in by a type cast, becomes the fixed text `complete an admin step`.
  - `subject` is `{ userId }`, `{ label }` or `{}`. A `userId` is shown only if it is a UUID, otherwise `invalid-id`. A `label` is shown only if it matches `^[a-z0-9-]{1,32}$`, otherwise `invalid-label`.
  - `authError` contributes only `describeAuthError()`. Status must be an integer from 100 to 599, otherwise `invalid`. Code must match `^[a-z0-9_]{1,64}$`, otherwise `invalid`. The Auth error's message and stack are never read.
- **The message is assembled inside the module.** The final text is recorded in a module-private `WeakMap`, keyed by the error object.
- **What `diagnosticFor(e, fallback)` reports.** It returns the **recorded** text for an error that `adminStepError` created, and the fixed fallback for anything else. It never reads `e.message` or `e.stack`. This rules out impersonation by:
  - an ordinary `Error`, even one whose text looks exactly like a real diagnostic;
  - a locally defined class named `SafeDiagnosticError`;
  - an error-like object literal;
  - an object created from a trusted error's prototype;
  - a trusted error whose `.message` or `.stack` was rewritten after creation. The original recorded text is still what gets reported.
- **Cleanup behaviour is unchanged.** `deleteAllTestUsers()` still attempts every deletion in order and continues after failures. It still reports `could not delete test user <safe id> (unexpected exception)` for any untrusted failure.
- **`admin.ts` only switches calls.** Its four throw sites now call `adminStepError(...)` with structured arguments. Nothing else in it changed.
- **Unchanged files.** `auth.test.ts` (`afterAll` uses `deleteAllTestUsers` and `cleanupFailureReport`, as in v4) and all application code are unchanged.

## 2. Files changed this round (3)

| File | Reason |
|---|---|
| `apps/web/test/integration/support/diagnostics.ts` | Replace the exported `SafeDiagnosticError` with `adminStepError` (structured, validated) and a private `WeakMap` trust record; `diagnosticFor` reports only recorded text |
| `apps/web/test/integration/support/admin.ts` | The four throw sites use `adminStepError(step, subject, error)`; the `SafeDiagnosticError`, `describeAuthError` and `safeUserId` imports are replaced by `adminStepError` |
| `apps/web/test/cleanup-diagnostics.test.ts` | Two v4 tests that constructed `SafeDiagnosticError` now use `adminStepError`, with the same assertions. 18 new trust-boundary tests. 46 tests, was 28. |

No other file changed: no application source, docs, configuration, lockfile, credentials, SQL or scripts.

## 3. Tests

All tests in `apps/web/test/cleanup-diagnostics.test.ts` are offline; none uses Supabase. 46 pass.

**Kept from v4 (assertions unchanged):**
- **Status and code display:** valid status and code are shown; absent values show as none or no data; 8 malformed statuses and 8 unsafe codes become `invalid`; the 64-character code limit is enforced; a non-object error is never read.
- **`safeUserId`:** sanitises non-UUIDs.
- **Untrusted failures:** every deletion is attempted in order; arbitrary thrown messages are replaced with fixed text (an `Error` with an email, key-like string, token and stack; a string; an object with a password; `undefined`; a `TypeError`); a non-UUID id in the fallback text is sanitised.
- **Reports:** no failures gives no report; the multiple-failure report lists every failure.

**Changed construction, same assertions:**
- "keeps the status and code of a legitimate structured diagnostic". Was "of a SafeDiagnosticError". Now uses `adminStepError(...)`, and passes an Auth error whose message contains an email address.
- The multiple-failure report test now makes its trusted failure with `adminStepError`.

**New: trust boundary (18 tests):**

| Requirement | Tests |
|---|---|
| No public constructor for arbitrary text | The module's export list is exactly `ADMIN_STEPS, adminStepError, cleanupFailureReport, deleteAllTestUsers, describeAuthError, diagnosticFor, safeUserId`; `SafeDiagnosticError` is not exported |
| A malicious message cannot become trusted | An ordinary `Error` with a malicious message, and one with a perfectly formatted fake diagnostic, both give the fixed fallback |
| No impersonation by ordinary or hand-made objects | A local `class SafeDiagnosticError extends Error`, an `{ name, message, stack }` object, an object from a trusted error's prototype, and an `Error` with a trusted error's own properties copied onto it all give the fallback |
| Email, token, password or stack cannot get through | A trusted error whose `.message` and `.stack` were rewritten afterwards still reports its original text; an Auth error whose message and stack contain an email address, token and password contributes only its status and code |
| Invalid ids, statuses and codes stay sanitised | An email as user id → `invalid-id`; a newline in the user id → `invalid-id`; an email as label → `invalid-label`; a 33-character label → `invalid-label`; a step outside the list → fixed text; 6 status/code cases (status 1000, status `'500'`, a token-like code, an email code, a string error, `null`) |
| Legitimate diagnostics stay useful | All four admin steps produce their exact expected text with status and code; `diagnosticFor` returns the trusted text, or the fallback for an `Error` and for `undefined` |
| Every deletion still attempted | Four ids mixing a trusted failure, an untrusted `Error` with a malicious message, an impersonating object, and a success: all four are attempted in order, and the report shows exactly the trusted status/code line plus two fixed fallback lines, with no sensitive text |

## 4. Commands run and exit codes

All runs were from `E:\Sovereign`, after the final edit. Logs are in `C:\Users\Gamer\AppData\Local\Temp\claude\7858b1e08a6e3505\c502d650-febf-4861-a36a-da9f61d54e28\scratchpad\m2r5\`.

| Command | Exit | Result |
|---|---|---|
| `pnpm lint` | 0 | clean |
| `pnpm run build:libs` | 0 | |
| `pnpm --filter @sovereign/core --filter @sovereign/db --filter @sovereign/worker typecheck` | 0 | |
| `apps/web: ./node_modules/.bin/tsc -p tsconfig.json --noEmit` | 0 | type-checks `admin.ts` and `auth.test.ts` against the new API |
| `pnpm test` | 0 | core 5 · db 23 · worker 8 · web **249** (v4: 231; +18) · scripts 204 |
| `pnpm check:boundaries` | 0 | |
| `node --test scripts/` | 0 | 204 tests, 204 pass, 0 fail, 0 skipped |
| `pnpm test:rls` | **2** | `configuration error: SOVEREIGN_SQL_TEST_DB_URL is not set`; no connection attempted |
| `pnpm test:integration` | **1** | `Integration tests need .env.integration …`; 1 file failed, **8 tests skipped**, 0 passed |

During development, also run: `npx vitest run test/cleanup-diagnostics.test.ts` (46 passed) and web `tsc --noEmit` (exit 0).

## 5. Hosted integration remains untested

| Not run | Why |
|---|---|
| pgTAP suite (`pnpm test:rls`): control, plus `identity_signup`, `identity_rls` and `catalog` | No database URL; hosted database out of scope |
| Auth integration suite (`pnpm test:integration`): 8 tests, including the real cleanup path through `adminStepError` and `deleteAllTestUsers` | No `.env.integration`; hosted Auth out of scope |
| `resetPasswordForEmail` against real Auth | It would send a real email; no safe sink exists |
| Browser checks; copy build plus `check:bundle` | No server may be started or restarted; `.next` must not be rebuilt |

## 6. Prohibited operations: none performed

- **Supabase:** no project was contacted, linked, inspected or configured. No `supabase` CLI command was run. No migrations were applied, and no database was written.
- **Credentials:** none created or modified. `.env`, `.env.local`, `.env.integration`, `apps/web/.env.local` and `apps/web/.env` do not exist; this was checked by listing only, and nothing was read.
- **Dependencies:** nothing installed, removed or updated.
- **Git:** no stage, commit, push, reset, stash or discard. Staged files are 0 before and after, and HEAD is `10013c0` before and after.
- **Servers:** none started or restarted. Port 3900 was not touched.
- **`.next`:** not rebuilt or modified. The `.next/types` snapshot is `9f92fb46c33f037e` before and after, and `BUILD_ID` is `rYrvX0swBrsAr-hHFLhVl` before and after.

## 7. Limitations

- The trust record is a module-level `WeakMap`. Trust therefore holds within one module instance, which is how Vitest loads it within a test file. An error created by a second copy of the module (for example, if a bundler duplicated it) would be reported with the fixed fallback. That fails safe: less detail, never leaked text.
- Labels come from the test file's own constants. Any label outside `^[a-z0-9-]{1,32}$` is shown as `invalid-label`.
- Everything listed in v4's remaining limitations still applies unchanged. No new product or security decision was made this round.

## 8. Complete revised sources

The full current contents of the three changed files follow. `auth.test.ts` is included unchanged, so the cleanup wiring can be reviewed in context.

### `apps/web/test/integration/support/diagnostics.ts`

```ts
// Sanitised diagnostics for the integration-test helpers and their cleanup. Pure, no Supabase import, so it
// is unit-tested offline (test/cleanup-diagnostics.test.ts). Only these values ever reach a report:
//   - a user id, and only when it is a UUID (ids of throwaway test users);
//   - a test-user label, and only when it matches ^[a-z0-9-]{1,32}$;
//   - an HTTP status, and only when it is an integer from 100 to 599;
//   - an Auth error code, and only when it matches ^[a-z0-9_]{1,64}$;
//   - fixed text written here (including the fixed list of admin steps).
// Never an exception's message or stack, an email address, a password, a key, a token or a raw response.
//
// Trust boundary: there is no public constructor that accepts diagnostic text. adminStepError() builds the
// message here from structured, validated inputs and records it in a module-private WeakMap. Only errors in
// that map are trusted, and the recorded text (not the error's current .message) is what gets reported, so
// an ordinary Error, an error-like object, an object made from a trusted error's prototype, or a trusted
// error whose .message was changed afterwards cannot inject text into a cleanup report.

const UUID = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
const CODE = /^[a-z0-9_]{1,64}$/;
const LABEL = /^[a-z0-9-]{1,32}$/;

/** The admin steps a trusted diagnostic can describe. */
export const ADMIN_STEPS = ['create test user', 'confirm test user', 'generate a recovery link', 'delete test user'] as const;
export type AdminStep = (typeof ADMIN_STEPS)[number];

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
const trusted = new WeakMap<object, string>();

/**
 * The error an admin helper throws when a step fails. Every part of its message is validated or fixed here:
 * the step must be one of ADMIN_STEPS, the subject is a UUID user id or a plain label, and the Auth error
 * contributes only its validated status and code.
 */
export function adminStepError(step: AdminStep, subject: { readonly userId?: unknown; readonly label?: unknown }, authError: unknown): Error {
  const fixedStep = (ADMIN_STEPS as readonly unknown[]).includes(step) ? step : 'complete an admin step';
  const who = 'userId' in subject ? ` ${safeUserId(subject.userId)}` : 'label' in subject ? ` ${safeLabel(subject.label)}` : '';
  const text = `could not ${fixedStep}${who} (${describeAuthError(authError)})`;
  const error = new Error(text);
  trusted.set(error, text);
  return error;
}

/** The trusted text of an error made by adminStepError, or the fixed fallback for anything else. */
export const diagnosticFor = (e: unknown, fallback: string): string =>
  (typeof e === 'object' && e !== null ? trusted.get(e) : undefined) ?? fallback;

/**
 * Attempts to delete every user, even after failures, and returns one sanitised line per failed deletion.
 * Unknown exceptions are reported with fixed text; their messages are never read.
 */
export async function deleteAllTestUsers(ids: readonly string[], deleteOne: (id: string) => Promise<void>): Promise<string[]> {
  const failures: string[] = [];
  for (const id of ids) {
    try {
      await deleteOne(id);
    } catch (e) {
      failures.push(diagnosticFor(e, `could not delete test user ${safeUserId(id)} (unexpected exception)`));
    }
  }
  return failures;
}

/** The cleanup failure report, or null when every deletion succeeded. */
export function cleanupFailureReport(failures: readonly string[], total: number): string | null {
  if (failures.length === 0) return null;
  return `test-user cleanup failed for ${failures.length} of ${total} users; delete them manually:\n${failures.join('\n')}`;
}
```

### `apps/web/test/integration/support/admin.ts`

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

### `apps/web/test/cleanup-diagnostics.test.ts`

```ts
// Offline tests for the integration suite's sanitised diagnostics (test/integration/support/diagnostics.ts).
// No Supabase client is involved; deletions are test doubles.
import { describe, expect, it, vi } from 'vitest';
import * as diagnostics from './integration/support/diagnostics';
import {
  type AdminStep,
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

describe('deleteAllTestUsers', () => {
  it('attempts every deletion even after failures, in order', async () => {
    const deleteOne = vi.fn((id: string) => (id === ID3 ? Promise.resolve() : Promise.reject(new Error('boom'))));
    const failures = await deleteAllTestUsers([ID1, ID2, ID3], deleteOne);
    expect(deleteOne.mock.calls.map((c) => c[0])).toEqual([ID1, ID2, ID3]);
    expect(failures).toHaveLength(2);
  });

  it('replaces arbitrary thrown messages with fixed text', async () => {
    const leaky = new Error('delete failed for a@example.com with key privileged-key-abc token=eyJxyz');
    leaky.stack = 'Error: secret\n    at adminClient (admin.ts:12)';
    const thrown: unknown[] = [leaky, 'a@example.com', { message: 'password=It-x-Passw0rd!' }, undefined, new TypeError('fetch failed: secret')];
    const ids = thrown.map((_, i) => `00000000-0000-4000-8000-00000000000${i}`);
    const failures = await deleteAllTestUsers(ids, (id) => Promise.reject(thrown[ids.indexOf(id)]));
    expect(failures).toEqual(ids.map((id) => `could not delete test user ${id} (unexpected exception)`));
    for (const line of failures) expect(line).not.toMatch(SENSITIVE);
  });

  it('keeps the status and code of a legitimate structured diagnostic', async () => {
    const failures = await deleteAllTestUsers([ID1], (id) =>
      Promise.reject(adminStepError('delete test user', { userId: id }, { status: 500, code: 'unexpected_failure', message: 'a@example.com' })),
    );
    expect(failures).toEqual([`could not delete test user ${ID1} (status=500, code=unexpected_failure)`]);
  });

  it('sanitises an id that is not a UUID in the fallback text', async () => {
    expect(await deleteAllTestUsers(['a@example.com'], () => Promise.reject(new Error('x')))).toEqual([
      'could not delete test user invalid-id (unexpected exception)',
    ]);
  });

  it('returns no failures when every deletion succeeds', async () => {
    expect(await deleteAllTestUsers([ID1, ID2], () => Promise.resolve())).toEqual([]);
  });
});

describe('cleanupFailureReport', () => {
  it('is null when nothing failed', () => expect(cleanupFailureReport([], 3)).toBeNull());

  it('lists every failure for multiple failed deletions, with ids and status/code only', async () => {
    const failures = await deleteAllTestUsers([ID1, ID2, ID3], (id) =>
      id === ID1
        ? Promise.reject(adminStepError('delete test user', { userId: ID1 }, { status: 'oops', code: 'a@b' }))
        : id === ID2
          ? Promise.reject(new Error('token=eyJabc'))
          : Promise.resolve(),
    );
    const report = cleanupFailureReport(failures, 3);
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
  const reported = async (thrown: unknown) => (await deleteAllTestUsers([ID1], () => Promise.reject(thrown)))[0];

  it('exports no constructor or class that accepts diagnostic text', () => {
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
    expect(diagnosticFor(adminStepError('delete test user', { userId: ID3 }, { status: 429 }), 'fallback')).toBe(
      `could not delete test user ${ID3} (status=429, code=none)`,
    );
    expect(diagnosticFor(new Error('x'), 'fallback')).toBe('fallback');
    expect(diagnosticFor(undefined, 'fallback')).toBe('fallback');
  });

  it('still attempts every deletion when trusted, untrusted and impersonating failures are mixed', async () => {
    const ids = [ID1, ID2, ID3, '00000000-0000-4000-8000-000000000004'];
    const deleteOne = vi.fn((id: string) => {
      if (id === ID1) return Promise.reject(adminStepError('delete test user', { userId: id }, { status: 500, code: 'unexpected_failure' }));
      if (id === ID2) return Promise.reject(new Error(MALICIOUS));
      if (id === ID3) return Promise.reject({ name: 'SafeDiagnosticError', message: MALICIOUS });
      return Promise.resolve();
    });
    const report = cleanupFailureReport(await deleteAllTestUsers(ids, deleteOne), ids.length);
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
```

### `apps/web/test/integration/auth.test.ts` (unchanged this round, for context)

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
  const report = cleanupFailureReport(await deleteAllTestUsers(created, deleteTestUser), created.length);
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
