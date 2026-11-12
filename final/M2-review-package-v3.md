# M2 review package v3: audit remediation round 2

Date: 2026-11-12 · Branch `feat/landing-and-realm-preview` · Nothing committed, staged, pushed, reset, stashed or discarded.

> **Status: M2 is NOT complete and NOT ready for database deployment.** No migration, SQL test or Auth flow has run against any database or Auth server. The database and Auth integration evidence is still outstanding. Everything below is offline: unit tests with test doubles, a fake psql, static checks and inspection.

Boundaries kept this round: no Supabase connection or configuration, no migrations or SQL tests against a database, no credential files created or modified (`.env.example` is a template with placeholders), no dependency changes (`pnpm-lock.yaml` untouched this round), port 3900 not restarted, `.next` not rebuilt (`.next/types` snapshot `9f92fb46c33f037e` before and after; `BUILD_ID` `rYrvX0swBrsAr-hHFLhVl`), M2 scope unchanged.

---

## 1. Changed files this round (exact list, 20 files)

New (4):

| File | Purpose |
|---|---|
| `apps/web/src/lib/site-origin.ts` | Validates `SOVEREIGN_SITE_URL` into a canonical origin |
| `apps/web/src/lib/site-origin.test.ts` | 23 offline tests |
| `apps/web/src/lib/auth/auth-outcome.ts` | Classifies Auth errors (ok / masked / unavailable); safe log line |
| `apps/web/src/lib/auth/auth-outcome.test.ts` | 14 offline tests |

Modified (16):

| File | Change |
|---|---|
| `apps/web/src/app/auth/actions.ts` | Canonical origin for sign-up and recovery links; header-based `siteOrigin()` removed; reset and sign-in use the classifier; consistent sign-in response |
| `apps/web/src/app/auth/actions.test.ts` | Hostile-header, origin-config, enumeration and logging tests (37 tests, was 16) |
| `apps/web/src/app/auth/callback/route.ts` | Redirects on the canonical origin; 503 when it is not configured |
| `apps/web/src/app/auth/callback/route.test.ts` | Hostile URL/header test and fail-closed test (8 tests, was 6) |
| `apps/web/src/app/auth/sign-in/page.tsx` | New `invalid` wording; `unavailable` message |
| `apps/web/src/app/auth/sign-up/page.tsx` | `unavailable` message |
| `apps/web/src/app/auth/reset-password/page.tsx` | `unavailable` message |
| `apps/web/src/env.d.ts` | Declares `SOVEREIGN_SITE_URL` |
| `apps/web/test/integration/auth.test.ts` | `track()` records each user immediately; cleanup attempts every deletion and fails loudly |
| `apps/web/test/integration/support/admin.ts` | Error messages carry Auth status/code only, never Auth's message |
| `scripts/tap.mjs` | New lexer-based `checkSqlTestFile`; recursive `checkManifest` |
| `scripts/tap.test.mjs` | Tests for every rejected pattern, nested/duplicate/missing/control-in-suite manifests, two new end-to-end cases (113 tests, was 44) |
| `scripts/run-sql-tests.mjs` | Recursive discovery of `*.test.sql`; new `checkManifest` signature |
| `.env.example` | Documents `SOVEREIGN_SITE_URL` with local and deployed examples |
| `apps/web/README.md` | Site-origin paragraph |
| `docs/DECISIONS/0011-hosted-dev-supabase-and-sql-tests.md` | Recursive manifest rule; static check and its limitations |

Unchanged in intent: `manifest.json`, the control `_selftest/deliberate_failure.test.sql`, `identity_signup.test.sql`, `identity_rls.test.sql`, `catalog.test.sql`. None of these files was edited, and all four pass the new check.

---

## 2. Item by item

### Item 1: Auth email origin (high priority)

**Before:** `siteOrigin()` in `actions.ts` built the origin from `x-forwarded-host ?? host` and `x-forwarded-proto`. A client or an intermediary that sets these headers could make Supabase email a confirmation or recovery link pointing at an attacker's host, *if* that host also passed Supabase's redirect allow-list. The callback redirected on `request.url`'s origin.

**Now:**
- `SOVEREIGN_SITE_URL` (server-only, not secret) is the only source of the origin. `parseSiteOrigin()` accepts it only when all of these hold:
  - It is an absolute URL.
  - The scheme is `https:`, or `http:` for `localhost`, `127.0.0.1` or `[::1]` only. This is the explicit local-development allowance; `localhost.evil.example` is rejected.
  - It has no username or password.
  - It is an origin only: no path other than `/`, no query, no fragment, and no `?` or `#` even when empty.
  - It has no surrounding whitespace.

  Rejection reasons never echo the configured value.
- `signUp` uses it for `emailRedirectTo`, and `requestPasswordReset` uses it for `redirectTo`. Neither action reads request headers any more; `next/headers` is no longer imported.
- When the variable is missing or invalid, Auth is never called. Sign-up and password reset redirect with `error=unavailable`, and a log line names the reason only.
- **Extension (flagged in section 5):** the callback also uses the canonical origin for its redirects and returns 503 when it is not configured.

**Tests (executed):**
- `actions.test.ts` mocks `next/headers` with hostile values: `Host: evil.example`, `X-Forwarded-Host: attacker.example`, `X-Forwarded-Proto: http`, and a `Forwarded` header. It asserts that:
  - the sign-up link's origin is exactly `https://sovereign.example`;
  - the recovery link's origin is exactly `https://sovereign.example`;
  - neither link contains `evil` or `attacker`;
  - `headers()` is never called.
- A test proves the explicit `http://localhost:3000` development value is used as given.
- Four invalid values each yield `error=unavailable`, with Auth not called.
- `route.test.ts` requests `http://evil.example/auth/callback` with hostile `Host`/`X-Forwarded-*` headers and gets a redirect to `https://sovereign.example/workspace`. When the variable is unset, the callback returns 503 and makes no exchange.

**Configuration (documented in `.env.example` and `apps/web/README.md`):**

```dotenv
# apps/web/.env.local
SOVEREIGN_SITE_URL=http://localhost:3000          # local development
# SOVEREIGN_SITE_URL=https://your-site.example    # deployed
```

The same origin must be on Supabase Auth's redirect allow-list (URL Configuration). That is a dashboard setting; it was not touched.

### Item 2: Password-reset error handling

`classifyAuthError(error)`:
- `null` or `undefined` → `ok`.
- No numeric `status`, `status === 0` (fetch or network failure in supabase-js), or `status >= 500` → `unavailable`.
- Any 4xx, including 429 → `masked`.

`requestPasswordReset`:
- `ok` and `masked` both give the identical public response, `/auth/reset-password?sent=1`. The page wording is unchanged: "If an account exists for that address, we have sent it a link …".
- `unavailable` (including an exception thrown by the client, now caught) gives `/auth/reset-password?error=unavailable` ("temporarily unavailable"), plus one server log line.

The log line comes from `authFailureLogLine()` and has the form `auth: password-reset request failed (status=500, code=unexpected_failure)`. Its contents are limited to:
- the action name, the numeric status, and the code;
- the code only when it matches `^[a-z0-9_]{1,64}$`, otherwise `none`.

It never includes the email address, Auth's message (which can echo the address), or tokens.

**Why this keeps non-enumeration:**
- An "unavailable" outcome does not depend on the account. A network failure or a 5xx happens whatever address is submitted.
- Every outcome that *can* depend on the account maps to the same response as success. This includes not found, unconfirmed, rate-limited and validation errors.

**Tests (executed):**
- Five outcomes (accepted, 404 user_not_found, 400 email_not_confirmed, 429, 422) give the identical `?sent=1` and log nothing.
- Four failure modes (status 0, 500, 503, thrown exception) give `?error=unavailable`. The log line matches the safe format and contains no `@`, `example.com` or `token`.
- A missing site origin gives `unavailable` without calling Auth.

**Trade-off needing approval:** see section 5.

### Item 3: Sign-in enumeration

**Assessment:** yes, it revealed account state. Before this change, a wrong password produced `?error=invalid`, but `email_not_confirmed` redirected to `/auth/verify-email`.

Supabase checks the password before reporting `email_not_confirmed`. So an attacker could not probe for unconfirmed accounts without the password. But the distinction still leaked two things:
- **Account existence and confirmation state** to anyone holding a guessed or leaked password.
- **A credential-validity oracle.** "Password correct but unconfirmed" looked different from "wrong password".

Low severity, but a real distinction.

**Implemented:** one public response, `?error=invalid`, for every account-dependent failure:
- wrong password
- unknown address
- unconfirmed address
- 4xx including 429

Infrastructure failures (status 0, 5xx) give `?error=unavailable`, with the same safe log line.

**UX change, made visible rather than silent.** The sign-in error text changes:
- Was: "The email address or password is incorrect."
- Now: "We could not sign you in with those details. Check the email address and password. If you have just signed up, confirm your address with the link we emailed you first."

A user who signs in before confirming no longer lands on `/auth/verify-email`. The verify-email page itself is unchanged. It is still used after sign-up (`?sent=1`) and by `requireVerifiedUser`.

**Tests (executed):** four failure types give the identical `?error=invalid`. An explicit test shows that `email_not_confirmed` no longer goes to `/auth/verify-email`. Two infrastructure failures give `?error=unavailable` with no address in the log.

**Approval needed:** see section 5 (UX decision).

Sign-up is unchanged in substance. Its 4xx outcomes still give `?error=signup`, and 5xx or network failures now give `?error=unavailable`. Supabase's own sign-up response for an already-registered address is governed by the project's email-confirmation setting, which is not verified here.

### Item 4: SQL test file validation

`checkSqlTestFile()` no longer runs regexes over comment-stripped text. A small lexer (`lexSql`) produces two same-length views of the file:
- **`code`:** comments blanked, and the contents of `'…'`, `E'…'` (with backslash escapes), `$tag$…$tag$` and `"…"` blanked (quotes kept). Nested `/* */` comments are handled.
- **`literals`:** comments blanked, strings kept. This view is needed because pgTAP functions (`lives_ok`, `throws_ok`, `results_eq`, `is_empty`, …) EXECUTE their string arguments.

Statements are split on `;` and on `\gset` (which sends the query buffer).

Rules, each with its own failing test:

| Rule | Rejected examples (all tested) |
|---|---|
| First statement exactly `BEGIN;` | missing; a statement before it |
| Last statement exactly `ROLLBACK;`, nothing after it | `commit;` last; `rollback` without `;`; a statement or a meta-command after it |
| No transaction control in the body (leading keyword of each statement) | `commit;`, `COMMIT WORK;`, `commit and chain;`, `end;`, `end transaction;`, `abort;`, `rollback;`, `rollback to savepoint s1;`, `begin;`, `start transaction;`, `savepoint s1;`, `release savepoint s1;`, `release s1;`, `prepare transaction 't1';`, `commit prepared 't1';`, `rollback prepared 't1';`, `select 1; commit; select 2;`, `select 1 \gset` followed by `commit;` |
| Forbidden anywhere, strings included | `dblink*`, `pg_background*`, `COMMIT/ROLLBACK PREPARED`, `PREPARE TRANSACTION`, `COPY` (incl. `COPY … TO PROGRAM` and COPY inside `lives_ok($$…$$)`) |
| psql meta-commands: only `\gset` (optionally with one prefix name) | `\c`, `\connect`, `\!`, `\setenv`, `\set`, `\unset`, `\o file`, `\o \|sh`, `\w`, `\i`, `\ir`, `\include`, `\copy`, `\lo_export`, `\gexec`, `\g \|sh`, `\g file`, `\gset x y z`, `\q`, `\if`, `\cd`; also a meta-command after a statement on the same line |
| psql variables only as `:'name'` | `:w`, `from :t`, `:"w"`, a statement starting `:cmd` (casts `::int` and array slices `[1:2]` are not mistaken for variables, tested) |
| Lexing must succeed | unterminated string, dollar string, block comment, quoted identifier |
| Exactly one `plan(` and a `finish()` call, outside strings | `plan(` only inside a string counts as zero |

Positive tests (executed):
- The four committed suite files pass unchanged; one test reads them from disk.
- CRLF line endings, `\gset` with `:'name'` interpolation, keywords inside comments and strings, and nested comments are all accepted.

Two new end-to-end runner cases against the fake psql: a suite file with a mid-file `COMMIT` exits **2**, and psql is never invoked.

**Limitations (also in ADR-0011):**
- It is a lexer plus regexes, **not a SQL parser**. It cannot see SQL built at run time: for example `EXECUTE format(...)`, string concatenation that spells a forbidden word, or a function defined in a migration that commits.
- It relies on PostgreSQL refusing `COMMIT`/`ROLLBACK` inside functions, `DO` blocks and procedures called within an explicit transaction block. It also relies on `ON_ERROR_STOP=1` plus the server rolling back an open transaction when the session ends after an error.
- The string-wide bans are deliberately conservative. They can reject harmless text, such as an assertion description containing the word "copy".
- `DO` and `CALL` statements are allowed, because inside the BEGIN block they cannot end the transaction.
- Test files are reviewed code, not untrusted input. The check guards against mistakes, not a determined malicious author.
- The `--file <rel>` single-file mode joins the given path to the tests directory without the manifest path rule. It is a test seam, used by the runner itself only for the manifest's control.

### Item 5: Manifest validation

Chosen option: **recursive discovery**.
- `run-sql-tests.mjs` lists `*.test.sql` at any depth (`readdirSync(…, { recursive: true })`). It normalises paths to `/`, also on Windows, and sorts them.
- `checkManifest(manifest, filesOnDisk)` now enforces all of the following:
  - Every manifest path is relative, `/`-separated, without `.`/`..` segments or backslashes, and ends in `.test.sql`.
  - The control exists on disk.
  - Every suite entry exists on disk.
  - No suite entry appears twice.
  - The control is not listed in the suite.
  - Every discovered file is either the control or a suite entry, so a nested file under `_selftest/` or any subdirectory cannot be skipped silently.
- The unused `exists` callback parameter was removed.

**Tests (executed):**
- A listed nested file is accepted.
- An unlisted nested file is rejected, both under `rls/` and a second file under `_selftest/`.
- A duplicate is rejected.
- A missing suite file and a missing control are rejected.
- The control listed in the suite is rejected.
- A missing control or suite is rejected.
- Seven malformed paths are rejected.
- End to end: an unlisted nested file makes the runner exit **2** without invoking psql.

### Item 6: Integration-test cleanup

**`auth.test.ts`:**
- `track(user)` pushes the id the moment `createTestUser` returns. This applies in `beforeAll` and in the two recovery tests. A failure creating the second or third user can no longer orphan the first.
- `afterAll` attempts every deletion and collects failures. If any failed, it throws `test-user cleanup failed for N of M users; delete them manually:` followed by one line per user.

**`support/admin.ts`:** every helper error now reports `status=…, code=…` only, plus the label or user id. Previously it included Auth's `error.message`, which can echo an email address.

The report therefore names user ids (random UUIDs of throwaway test users) and Auth status/code. It never names addresses, passwords, keys or tokens.

**Not executed:** the integration suite needs `.env.integration`, which does not exist and was not created. Its fail-closed run is shown in section 3.

---

## 3. Executed checks (exact results)

All runs were from `E:\Sovereign`, after the final source edits. Logs are in `C:\Users\Gamer\AppData\Local\Temp\claude\7858b1e08a6e3505\c502d650-febf-4861-a36a-da9f61d54e28\scratchpad\m2r3\`.

| Command | Exit | Result |
|---|---|---|
| `pnpm lint` | 0 | clean |
| `pnpm run build:libs` | 0 | |
| `pnpm --filter @sovereign/core --filter @sovereign/db --filter @sovereign/worker typecheck` | 0 | |
| `apps/web: tsc -p tsconfig.json --noEmit` (not `pnpm typecheck`, which runs `next typegen` and rewrites `.next/types`) | 0 | first run exited 2 on three typing errors in new test code; fixed, then 0 |
| `pnpm test` | 0 | core 5 · db 23 · worker 8 · web **146** (was 86) · scripts **160** (was 91) |
| `pnpm check:boundaries` | 0 | |
| `node --test scripts/` | 0 | 160 tests, 160 pass, 0 fail |
| `pnpm test:rls` (no configuration) | **2** | `configuration error: SOVEREIGN_SQL_TEST_DB_URL is not set`; no connection attempted (expected fail-closed) |
| `pnpm test:integration` (no credentials) | **1** | `Integration tests need .env.integration …`; 1 file failed, 8 tests skipped, none passed (expected fail-closed) |

Per-file counts for the new or changed web tests (Vitest, executed):

| File | Tests |
|---|---|
| `src/lib/site-origin.test.ts` | 23 |
| `src/lib/auth/auth-outcome.test.ts` | 14 |
| `src/app/auth/actions.test.ts` | 37 |
| `src/app/auth/callback/route.test.ts` | 8 |
| **total** | **82 passed** |

`scripts/tap.test.mjs`: 113 tests, 113 pass (was 44).

`.next/types` snapshot: `9f92fb46c33f037e` before and after. `BUILD_ID`: `rYrvX0swBrsAr-hHFLhVl`.

**Not run this round:**
- No copy build in `.verify-build/web`.
- No `check:bundle` on a fresh build.
- Not repeated, because `.next` must not be rebuilt and the copy build is optional. The server-only `SOVEREIGN_SITE_URL` is read only in `'use server'` actions and a route handler. That it stays out of client bundles is **static inspection only** this round.

### Executed versus static inspection

| Executed (offline) | Static inspection or reasoning only |
|---|---|
| Canonical-origin parsing; the actions and the callback ignore hostile headers (test doubles) | That Supabase Auth honours `emailRedirectTo`/`redirectTo` and rejects origins outside its allow-list |
| Reset and sign-in response mapping for the mocked status/code combinations | Which status/code real Supabase Auth returns for each case (e.g. that SMTP failure is 5xx, that unknown-address reset returns 200) |
| Log lines exclude address, message and tokens for the mocked errors | That no other layer (supabase-js, Next.js) logs the address |
| SQL file checker on the four real files and every rejected pattern | That the four files pass under real psql + pgTAP |
| Manifest rules; runner exit codes against a fake psql | The runner against a real database |
| Integration cleanup code type-checks | Its behaviour against real Auth (suite not executed) |
| | The new sign-in and reset wording in a browser (no server was started) |

---

## 4. Outstanding before M2 can be called complete (unchanged by this round)

- The migrations have not been applied to any database.
- `pnpm test:rls` has not run against a database (pgTAP evidence outstanding).
- `pnpm test:integration` has not run against Auth (Auth evidence outstanding).
- `resetPasswordForEmail` against real Auth is unverified, because it sends a real email. A safe email sink is still needed: custom SMTP to a sink you control, or a local stack with Mailpit.
- `SOVEREIGN_SITE_URL` must be set, and the origin added to the Supabase redirect allow-list, before any Auth email flow can work.

---

## 5. Decisions requiring your approval

1. **New env var `SOVEREIGN_SITE_URL`.**
   - **Behaviour:** server-only and required; unset or invalid means Auth flows fail closed.
   - **Development:** `http://localhost:3000` is the documented default.
   - **Optional addition:** a `next.config` or startup check could fail the build or boot when it is missing. Not implemented.
2. **Callback on the canonical origin (extension beyond the letter of item 1).**
   - `/auth/callback` now redirects to `SOVEREIGN_SITE_URL` rather than to the request's own origin, and returns **503** when it is unset.
   - **Effect:** a deployment reached through any other host name (a preview URL, a raw IP) is sent to the canonical host after confirming. A session cookie set on the other host would not carry over.
   - **Alternative:** keep the request origin in the callback only (it redirects to a validated relative path).
3. **Sign-in UX change.**
   - Unconfirmed users now see the generic "could not sign you in…" message, with a confirmation hint, instead of being sent to `/auth/verify-email`.
   - **Alternative:** keep the old redirect and accept the confirmation-state disclosure to holders of a valid password.
4. **Masking every 4xx.**
   - Rate limits (429) and validation or misconfiguration 4xx look like success on reset and like "invalid" on sign-in. They are visible to operators only through the status/code log line.
   - Users who hit a rate limit are not told to wait.
   - **Alternative:** surface 429 separately. This does not depend on the account in principle, but per-address rate limits could reveal activity on an address.
5. **Regex/lexer SQL check limits** (item 4) and the conservative string-wide bans, which can reject harmless text.
6. **Recursive manifest discovery.** Chosen over rejecting nested files. Subdirectories are allowed as long as every file is listed.

No new security or product decision was taken beyond these six; each is implemented in its most conservative form and can be reverted individually.

---

## 6. Complete revised sources

Full current contents of all 20 files changed this round follow, in the order of the list in section 1.

### `apps/web/src/lib/site-origin.ts`

```ts
// Canonical site origin for every URL that Supabase Auth puts in an email or that the auth callback
// redirects to. It comes from configuration (SOVEREIGN_SITE_URL), never from request headers such as
// Host, X-Forwarded-Host or X-Forwarded-Proto, which a client or an intermediary can set.
//
// Accepted: an absolute URL that is exactly an origin (scheme + host + optional port, optional trailing "/").
//   https://…                      any host
//   http://localhost[:port]        local development only; also http://127.0.0.1 and http://[::1]
// Rejected: other schemes, http for any non-loopback host, credentials, a path, a query, a fragment,
// surrounding whitespace.

export type SiteOriginResult = { readonly ok: true; readonly origin: string } | { readonly ok: false; readonly reason: string };

const LOOPBACK_HOSTS = new Set(['localhost', '127.0.0.1', '[::1]']);

export function parseSiteOrigin(raw: string | undefined): SiteOriginResult {
  if (raw === undefined || raw === '') return { ok: false, reason: 'SOVEREIGN_SITE_URL is not set' };
  if (raw !== raw.trim()) return { ok: false, reason: 'SOVEREIGN_SITE_URL must not contain surrounding whitespace' };
  let url: URL;
  try {
    url = new URL(raw);
  } catch {
    return { ok: false, reason: 'SOVEREIGN_SITE_URL must be an absolute URL' };
  }
  if (url.protocol !== 'https:' && url.protocol !== 'http:') return { ok: false, reason: 'SOVEREIGN_SITE_URL must use https (or http for localhost)' };
  if (url.protocol === 'http:' && !LOOPBACK_HOSTS.has(url.hostname)) {
    return { ok: false, reason: 'SOVEREIGN_SITE_URL may use http only for localhost, 127.0.0.1 or [::1]' };
  }
  if (url.username !== '' || url.password !== '') return { ok: false, reason: 'SOVEREIGN_SITE_URL must not contain credentials' };
  if (url.pathname !== '/' || url.search !== '' || url.hash !== '' || raw.includes('?') || raw.includes('#')) {
    return { ok: false, reason: 'SOVEREIGN_SITE_URL must be an origin only (no path, query or fragment)' };
  }
  return { ok: true, origin: url.origin };
}

/** The configured canonical origin of this deployment. */
export function siteOrigin(): SiteOriginResult {
  return parseSiteOrigin(process.env.SOVEREIGN_SITE_URL);
}
```

### `apps/web/src/lib/site-origin.test.ts`

```ts
import { describe, expect, it } from 'vitest';
import { parseSiteOrigin } from './site-origin';

describe('parseSiteOrigin', () => {
  it.each([
    ['https://sovereign.example', 'https://sovereign.example'],
    ['https://sovereign.example/', 'https://sovereign.example'],
    ['https://sovereign.example:8443', 'https://sovereign.example:8443'],
    ['http://localhost:3000', 'http://localhost:3000'],
    ['http://127.0.0.1:3900', 'http://127.0.0.1:3900'],
    ['http://[::1]:3000', 'http://[::1]:3000'],
  ])('accepts %s', (raw, origin) => {
    expect(parseSiteOrigin(raw)).toEqual({ ok: true, origin });
  });

  it.each([
    [undefined, 'not set'],
    ['', 'not set'],
    [' https://sovereign.example', 'whitespace'],
    ['sovereign.example', 'absolute URL'],
    ['/auth/callback', 'absolute URL'],
    ['ftp://sovereign.example', 'https'],
    ['javascript:alert(1)', 'https'],
    ['http://sovereign.example', 'http only for localhost'],
    ['http://localhost.evil.example', 'http only for localhost'],
    ['https://user:pass@sovereign.example', 'credentials'],
    ['https://user@sovereign.example', 'credentials'],
    ['https://sovereign.example/app', 'origin only'],
    ['https://sovereign.example/?next=/x', 'origin only'],
    ['https://sovereign.example?', 'origin only'],
    ['https://sovereign.example/#frag', 'origin only'],
    ['https://sovereign.example#', 'origin only'],
  ])('rejects %j', (raw, reason) => {
    const r = parseSiteOrigin(raw);
    expect(r.ok).toBe(false);
    expect(r.ok ? '' : r.reason).toContain(reason);
  });

  it('never echoes the configured value in its reasons', () => {
    const r = parseSiteOrigin('https://user:s3cr3t@sovereign.example');
    expect(r.ok ? '' : r.reason).not.toContain('s3cr3t');
  });
});
```

### `apps/web/src/lib/auth/auth-outcome.ts`

```ts
// Classifies a Supabase Auth error from an action whose public response must not reveal whether an
// account exists (password-reset request, sign-in).
//
//   'unavailable'  the request did not reach a working Auth service or the service failed: a network or
//                  fetch failure (status 0), any 5xx, or an error without an HTTP status. These outcomes do
//                  not depend on the account, so telling the user "try again later" reveals nothing.
//   'masked'       any 4xx, including rate limits (429). Some 4xx outcomes can depend on account state, so
//                  they are answered exactly like the normal outcome of that action.
//
// Trade-off (documented in the review package): a 4xx caused by misconfiguration is also masked, so it is
// visible only through the server-side log line, which carries the status and code but never the email
// address, tokens or Auth messages.

export type AuthOutcome = 'ok' | 'unavailable' | 'masked';

export interface AuthErrorShape {
  readonly status?: number | undefined;
  readonly code?: string | undefined;
}

export function classifyAuthError(error: unknown): AuthOutcome {
  if (error === null || error === undefined) return 'ok';
  const status = typeof error === 'object' && 'status' in error ? (error as AuthErrorShape).status : undefined;
  if (typeof status !== 'number' || status === 0 || status >= 500) return 'unavailable';
  return 'masked';
}

/** A log line safe to write: status and code only (never the email, tokens or Auth's message). */
export function authFailureLogLine(action: string, error: unknown): string {
  const e = (typeof error === 'object' && error !== null ? error : {}) as AuthErrorShape;
  const status = typeof e.status === 'number' ? String(e.status) : 'none';
  const code = typeof e.code === 'string' && /^[a-z0-9_]{1,64}$/.test(e.code) ? e.code : 'none';
  return `auth: ${action} failed (status=${status}, code=${code})`;
}
```

### `apps/web/src/lib/auth/auth-outcome.test.ts`

```ts
import { describe, expect, it } from 'vitest';
import { authFailureLogLine, classifyAuthError } from './auth-outcome';

describe('classifyAuthError', () => {
  it('treats no error as ok', () => {
    expect(classifyAuthError(null)).toBe('ok');
    expect(classifyAuthError(undefined)).toBe('ok');
  });

  it.each([
    ['network failure (status 0)', { status: 0, code: undefined }],
    ['500', { status: 500, code: 'unexpected_failure' }],
    ['503', { status: 503 }],
    ['no status', { code: 'x' }],
    ['a thrown non-Auth value', new Error('boom')],
    ['a string', 'boom'],
  ])('reports %s as unavailable', (_name, error) => {
    expect(classifyAuthError(error)).toBe('unavailable');
  });

  it.each([
    ['user not found', { status: 404, code: 'user_not_found' }],
    ['email not confirmed', { status: 400, code: 'email_not_confirmed' }],
    ['invalid credentials', { status: 400, code: 'invalid_credentials' }],
    ['rate limit', { status: 429, code: 'over_email_send_rate_limit' }],
    ['validation', { status: 422, code: 'validation_failed' }],
  ])('masks %s', (_name, error) => {
    expect(classifyAuthError(error)).toBe('masked');
  });
});

describe('authFailureLogLine', () => {
  it('contains only the action, status and code', () => {
    const line = authFailureLogLine('password-reset request', {
      status: 500,
      code: 'unexpected_failure',
      message: 'mail to someone@example.com failed, token abc123',
    });
    expect(line).toBe('auth: password-reset request failed (status=500, code=unexpected_failure)');
    expect(line).not.toMatch(/@|token|abc123/);
  });

  it('drops a code that is not a plain identifier', () => {
    expect(authFailureLogLine('sign-in', { status: 400, code: 'x@y.example' })).toBe('auth: sign-in failed (status=400, code=none)');
    expect(authFailureLogLine('sign-in', 'boom')).toBe('auth: sign-in failed (status=none, code=none)');
  });
});
```

### `apps/web/src/app/auth/actions.ts`

```ts
'use server';
// Auth Server Actions (M2). Every call uses the per-request user client (publishable key + session
// cookies); none uses a privileged key. Errors redirect with a fixed code, never with raw Auth messages.
//
// Links in Auth emails are built from the configured canonical origin (SOVEREIGN_SITE_URL, see
// src/lib/site-origin.ts), never from request headers.
// Account enumeration: the password-reset request and sign-in give the same public response whatever the
// account's state; only failures that do not depend on the account (Auth unreachable or failing) are
// reported separately, as "unavailable" (src/lib/auth/auth-outcome.ts). Nothing here logs an email address,
// a token or an Auth message.
import { redirect } from 'next/navigation';
import { authFailureLogLine, classifyAuthError } from '@/lib/auth/auth-outcome';
import { isAcceptablePassword, isPlausibleEmail, safeNextPath } from '@/lib/auth/next-path';
import { siteOrigin } from '@/lib/site-origin';
import { createSupabaseServerClient } from '@/lib/supabase/server';

const text = (form: FormData, key: string) => {
  const v = form.get(key);
  return typeof v === 'string' ? v.trim() : '';
};
const secret = (form: FormData, key: string) => {
  const v = form.get(key);
  return typeof v === 'string' ? v : '';
};

/** The canonical origin, or null (logged without the configured value) when it is missing or invalid. */
function canonicalOrigin(): string | null {
  const result = siteOrigin();
  if (!result.ok) {
    console.error(`auth: site origin unavailable (${result.reason})`);
    return null;
  }
  return result.origin;
}

export async function signUp(form: FormData): Promise<never> {
  const email = text(form, 'email');
  const password = secret(form, 'password');
  if (!isPlausibleEmail(email) || !isAcceptablePassword(password)) redirect('/auth/sign-up?error=input');
  const origin = canonicalOrigin();
  if (!origin) redirect('/auth/sign-up?error=unavailable');
  const supabase = await createSupabaseServerClient();
  const { error } = await supabase.auth.signUp({
    email,
    password,
    options: { emailRedirectTo: `${origin}/auth/callback?next=/workspace` },
  });
  if (error) {
    if (classifyAuthError(error) === 'unavailable') {
      console.error(authFailureLogLine('sign-up', error));
      redirect('/auth/sign-up?error=unavailable');
    }
    redirect('/auth/sign-up?error=signup');
  }
  redirect('/auth/verify-email?sent=1');
}

export async function signIn(form: FormData): Promise<never> {
  const email = text(form, 'email');
  const password = secret(form, 'password');
  const next = safeNextPath(text(form, 'next'));
  if (!isPlausibleEmail(email) || password.length === 0) redirect('/auth/sign-in?error=invalid');
  const supabase = await createSupabaseServerClient();
  const { error } = await supabase.auth.signInWithPassword({ email, password });
  if (error) {
    if (classifyAuthError(error) === 'unavailable') {
      console.error(authFailureLogLine('sign-in', error));
      redirect('/auth/sign-in?error=unavailable');
    }
    // One response for every account-dependent failure (wrong password, unknown address, unconfirmed
    // address), so the result never shows whether an address has an account or whether it is confirmed.
    redirect('/auth/sign-in?error=invalid');
  }
  redirect(next);
}

export async function signOut(): Promise<never> {
  const supabase = await createSupabaseServerClient();
  await supabase.auth.signOut();
  redirect('/auth/sign-in?signed-out=1');
}

export async function requestPasswordReset(form: FormData): Promise<never> {
  const email = text(form, 'email');
  if (!isPlausibleEmail(email)) redirect('/auth/reset-password?error=input');
  const origin = canonicalOrigin();
  if (!origin) redirect('/auth/reset-password?error=unavailable');
  const supabase = await createSupabaseServerClient();
  let error: unknown;
  try {
    ({ error } = await supabase.auth.resetPasswordForEmail(email, {
      redirectTo: `${origin}/auth/callback?next=/auth/update-password`,
    }));
  } catch (thrown) {
    error = thrown;
  }
  if (classifyAuthError(error) === 'unavailable') {
    console.error(authFailureLogLine('password-reset request', error));
    redirect('/auth/reset-password?error=unavailable');
  }
  // Accepted or masked (any 4xx): the same response, whether or not the address has an account.
  redirect('/auth/reset-password?sent=1');
}

export async function updatePassword(form: FormData): Promise<never> {
  const password = secret(form, 'password');
  if (!isAcceptablePassword(password)) redirect('/auth/update-password?error=input');
  const supabase = await createSupabaseServerClient();
  const { data } = await supabase.auth.getUser();
  if (!data.user) redirect('/auth/sign-in?error=link');
  const { error } = await supabase.auth.updateUser({ password });
  if (error) redirect('/auth/update-password?error=update');
  redirect('/workspace');
}
```

### `apps/web/src/app/auth/actions.test.ts`

```ts
// Offline unit tests for the auth Server Actions. Next.js (headers, redirect) and the per-request user client
// are replaced with test doubles, so these prove the actions' own logic: input validation, fixed error codes,
// safe redirects, the canonical origin of Auth email links, and that the password-reset request and sign-in
// reveal nothing about whether an account exists. Real Supabase Auth behaviour is the integration suite's job.
import { afterEach, beforeEach, describe, expect, it, vi } from 'vitest';

const auth = {
  signUp: vi.fn(),
  signInWithPassword: vi.fn(),
  signOut: vi.fn(),
  resetPasswordForEmail: vi.fn(),
  getUser: vi.fn(),
  updateUser: vi.fn(),
};
// Hostile request headers. The actions must not read them; the mock exists so that, if any code path ever
// did, the tests below would see the attacker's host in the generated link and fail.
const HOSTILE_HEADERS = new Map<string, string>([
  ['host', 'evil.example'],
  ['x-forwarded-host', 'attacker.example'],
  ['x-forwarded-proto', 'http'],
  ['forwarded', 'host=attacker.example;proto=http'],
]);
const headersSpy = vi.fn(() => Promise.resolve({ get: (k: string) => HOSTILE_HEADERS.get(k.toLowerCase()) ?? null }));

vi.mock('@/lib/supabase/server', () => ({ createSupabaseServerClient: () => Promise.resolve({ auth }) }));
vi.mock('next/headers', () => ({ headers: headersSpy }));
vi.mock('next/navigation', () => ({
  redirect: (url: string) => {
    throw new Error(`REDIRECT:${url}`);
  },
}));

const { signUp, signIn, signOut, requestPasswordReset, updatePassword } = await import('./actions');

const form = (fields: Record<string, string>) => {
  const f = new FormData();
  for (const [k, v] of Object.entries(fields)) f.set(k, v);
  return f;
};
/** Runs an action and returns the URL it redirected to. */
const redirectOf = async (p: Promise<unknown>) => {
  try {
    await p;
  } catch (e) {
    const m = /^REDIRECT:(.*)$/.exec((e as Error).message);
    if (m) return m[1];
    throw e;
  }
  throw new Error('the action did not redirect');
};

const SITE = 'https://sovereign.example';
let errorLog: { mock: { calls: unknown[][] }; mockRestore: () => void };
const logged = () => errorLog.mock.calls.map((c: unknown[]) => c.join(' ')).join('\n');

beforeEach(() => {
  for (const fn of Object.values(auth)) fn.mockReset();
  headersSpy.mockClear();
  vi.stubEnv('SOVEREIGN_SITE_URL', SITE);
  errorLog = vi.spyOn(console, 'error').mockImplementation(() => undefined);
});
afterEach(() => {
  vi.unstubAllEnvs();
  errorLog.mockRestore();
});

describe('signUp', () => {
  it('rejects invalid input without calling Auth', async () => {
    expect(await redirectOf(signUp(form({ email: 'nope', password: 'longenough' })))).toBe('/auth/sign-up?error=input');
    expect(await redirectOf(signUp(form({ email: 'a@example.com', password: 'short' })))).toBe('/auth/sign-up?error=input');
    expect(auth.signUp).not.toHaveBeenCalled();
  });

  it('signs up with a confirmation link on the canonical origin, then asks the user to confirm', async () => {
    auth.signUp.mockResolvedValue({ error: null });
    expect(await redirectOf(signUp(form({ email: ' a@example.com ', password: 'longenough' })))).toBe('/auth/verify-email?sent=1');
    expect(auth.signUp).toHaveBeenCalledWith({
      email: 'a@example.com',
      password: 'longenough',
      options: { emailRedirectTo: 'https://sovereign.example/auth/callback?next=/workspace' },
    });
  });

  it('ignores hostile Host / X-Forwarded-* headers when building the link', async () => {
    auth.signUp.mockResolvedValue({ error: null });
    await redirectOf(signUp(form({ email: 'a@example.com', password: 'longenough' })));
    const link = auth.signUp.mock.calls[0]?.[0].options.emailRedirectTo as string;
    expect(new URL(link).origin).toBe(SITE);
    expect(link).not.toMatch(/evil|attacker/);
    expect(headersSpy).not.toHaveBeenCalled();
  });

  it('uses the local development origin when it is configured explicitly', async () => {
    vi.stubEnv('SOVEREIGN_SITE_URL', 'http://localhost:3000');
    auth.signUp.mockResolvedValue({ error: null });
    await redirectOf(signUp(form({ email: 'a@example.com', password: 'longenough' })));
    expect(auth.signUp.mock.calls[0]?.[0].options.emailRedirectTo).toBe('http://localhost:3000/auth/callback?next=/workspace');
  });

  it.each([[''], ['http://sovereign.example'], ['https://sovereign.example/app'], ['not a url']])(
    'refuses to send a link when the site origin is %j, without calling Auth',
    async (value) => {
      vi.stubEnv('SOVEREIGN_SITE_URL', value);
      expect(await redirectOf(signUp(form({ email: 'a@example.com', password: 'longenough' })))).toBe('/auth/sign-up?error=unavailable');
      expect(auth.signUp).not.toHaveBeenCalled();
      expect(logged()).toMatch(/^auth: site origin unavailable/);
    },
  );

  it('reports a generic error code, never the Auth message', async () => {
    auth.signUp.mockResolvedValue({ error: { message: 'internal detail', code: 'x', status: 422 } });
    expect(await redirectOf(signUp(form({ email: 'a@example.com', password: 'longenough' })))).toBe('/auth/sign-up?error=signup');
  });

  it('reports an Auth outage as unavailable, logging status and code only', async () => {
    auth.signUp.mockResolvedValue({ error: { message: 'db down for a@example.com', code: 'unexpected_failure', status: 500 } });
    expect(await redirectOf(signUp(form({ email: 'a@example.com', password: 'longenough' })))).toBe('/auth/sign-up?error=unavailable');
    expect(logged()).toBe('auth: sign-up failed (status=500, code=unexpected_failure)');
  });
});

describe('signIn (one response for every account-dependent failure)', () => {
  it.each([
    ['wrong password', { status: 400, code: 'invalid_credentials' }],
    ['unconfirmed address', { status: 400, code: 'email_not_confirmed' }],
    ['unknown address', { status: 400, code: 'invalid_credentials' }],
    ['rate limit', { status: 429, code: 'over_request_rate_limit' }],
  ])('answers %s with the same invalid-credentials response', async (_name, error) => {
    auth.signInWithPassword.mockResolvedValue({ error });
    expect(await redirectOf(signIn(form({ email: 'a@example.com', password: 'x' })))).toBe('/auth/sign-in?error=invalid');
    expect(errorLog).not.toHaveBeenCalled();
  });

  it('no longer sends an unconfirmed address to the confirmation page', async () => {
    auth.signInWithPassword.mockResolvedValue({ error: { status: 400, code: 'email_not_confirmed' } });
    expect(await redirectOf(signIn(form({ email: 'a@example.com', password: 'x' })))).not.toBe('/auth/verify-email');
  });

  it.each([
    ['network failure', { status: 0 }],
    ['server error', { status: 500, code: 'unexpected_failure' }],
  ])('reports %s as unavailable without the address in the log', async (_name, error) => {
    auth.signInWithPassword.mockResolvedValue({ error });
    expect(await redirectOf(signIn(form({ email: 'a@example.com', password: 'x' })))).toBe('/auth/sign-in?error=unavailable');
    expect(logged()).toMatch(/^auth: sign-in failed \(status=\d+, code=\w+\)$/);
    expect(logged()).not.toContain('a@example.com');
  });

  it('rejects malformed input without calling Auth', async () => {
    expect(await redirectOf(signIn(form({ email: 'nope', password: 'x' })))).toBe('/auth/sign-in?error=invalid');
    expect(await redirectOf(signIn(form({ email: 'a@example.com', password: '' })))).toBe('/auth/sign-in?error=invalid');
    expect(auth.signInWithPassword).not.toHaveBeenCalled();
  });

  it('continues to a same-site next path, and never to another site', async () => {
    auth.signInWithPassword.mockResolvedValue({ error: null });
    expect(await redirectOf(signIn(form({ email: 'a@example.com', password: 'x', next: '/workspace' })))).toBe('/workspace');
    expect(await redirectOf(signIn(form({ email: 'a@example.com', password: 'x', next: 'https://evil.example/' })))).toBe('/workspace');
    expect(await redirectOf(signIn(form({ email: 'a@example.com', password: 'x', next: '//evil.example' })))).toBe('/workspace');
  });
});

describe('signOut', () => {
  it('ends the session and returns to sign-in', async () => {
    auth.signOut.mockResolvedValue({ error: null });
    expect(await redirectOf(signOut())).toBe('/auth/sign-in?signed-out=1');
    expect(auth.signOut).toHaveBeenCalledTimes(1);
  });
});

describe('requestPasswordReset (no account enumeration)', () => {
  it('asks Auth for a recovery link on the canonical origin that returns to update-password', async () => {
    auth.resetPasswordForEmail.mockResolvedValue({ error: null });
    await redirectOf(requestPasswordReset(form({ email: 'a@example.com' })));
    expect(auth.resetPasswordForEmail).toHaveBeenCalledWith('a@example.com', {
      redirectTo: 'https://sovereign.example/auth/callback?next=/auth/update-password',
    });
  });

  it('ignores hostile Host / X-Forwarded-* headers when building the recovery link', async () => {
    auth.resetPasswordForEmail.mockResolvedValue({ error: null });
    await redirectOf(requestPasswordReset(form({ email: 'a@example.com' })));
    const link = auth.resetPasswordForEmail.mock.calls[0]?.[1].redirectTo as string;
    expect(new URL(link).origin).toBe(SITE);
    expect(link).not.toMatch(/evil|attacker/);
    expect(headersSpy).not.toHaveBeenCalled();
  });

  it.each([
    ['accepted', null],
    ['user not found', { status: 404, code: 'user_not_found', message: 'User not found' }],
    ['unconfirmed', { status: 400, code: 'email_not_confirmed' }],
    ['rate limited', { status: 429, code: 'over_email_send_rate_limit' }],
    ['validation', { status: 422, code: 'validation_failed' }],
  ])('gives the same public response when the outcome is %s', async (_name, error) => {
    auth.resetPasswordForEmail.mockResolvedValue({ error });
    expect(await redirectOf(requestPasswordReset(form({ email: 'a@example.com' })))).toBe('/auth/reset-password?sent=1');
    expect(errorLog).not.toHaveBeenCalled();
  });

  it.each([
    ['network failure (status 0)', () => Promise.resolve({ error: { status: 0, name: 'AuthRetryableFetchError' } })],
    ['server error', () => Promise.resolve({ error: { status: 500, code: 'unexpected_failure' } })],
    ['SMTP / service failure', () => Promise.resolve({ error: { status: 503 } })],
    ['a thrown exception', () => Promise.reject(new Error('fetch failed for a@example.com'))],
  ])('reports %s as unavailable, logging nothing about the account', async (_name, impl) => {
    auth.resetPasswordForEmail.mockImplementation(impl);
    expect(await redirectOf(requestPasswordReset(form({ email: 'a@example.com' })))).toBe('/auth/reset-password?error=unavailable');
    const line = logged();
    expect(line).toMatch(/^auth: password-reset request failed \(status=\w+, code=\w+\)$/);
    expect(line).not.toMatch(/@|example\.com|token/);
  });

  it('reports a missing site origin as unavailable without calling Auth', async () => {
    vi.stubEnv('SOVEREIGN_SITE_URL', '');
    expect(await redirectOf(requestPasswordReset(form({ email: 'a@example.com' })))).toBe('/auth/reset-password?error=unavailable');
    expect(auth.resetPasswordForEmail).not.toHaveBeenCalled();
  });

  it('rejects a malformed address without calling Auth', async () => {
    expect(await redirectOf(requestPasswordReset(form({ email: 'nope' })))).toBe('/auth/reset-password?error=input');
    expect(auth.resetPasswordForEmail).not.toHaveBeenCalled();
  });
});

describe('updatePassword', () => {
  it('requires a session validated by getUser()', async () => {
    auth.getUser.mockResolvedValue({ data: { user: null }, error: null });
    expect(await redirectOf(updatePassword(form({ password: 'longenough' })))).toBe('/auth/sign-in?error=link');
    expect(auth.updateUser).not.toHaveBeenCalled();
  });

  it('rejects a short password before calling Auth', async () => {
    expect(await redirectOf(updatePassword(form({ password: 'short' })))).toBe('/auth/update-password?error=input');
    expect(auth.getUser).not.toHaveBeenCalled();
  });

  it('updates the password and continues to the workspace', async () => {
    auth.getUser.mockResolvedValue({ data: { user: { id: 'u1' } }, error: null });
    auth.updateUser.mockResolvedValue({ error: null });
    expect(await redirectOf(updatePassword(form({ password: 'longenough' })))).toBe('/workspace');
    expect(auth.updateUser).toHaveBeenCalledWith({ password: 'longenough' });
  });

  it('reports a failed update with a fixed code', async () => {
    auth.getUser.mockResolvedValue({ data: { user: { id: 'u1' } }, error: null });
    auth.updateUser.mockResolvedValue({ error: { message: 'detail' } });
    expect(await redirectOf(updatePassword(form({ password: 'longenough' })))).toBe('/auth/update-password?error=update');
  });
});
```

### `apps/web/src/app/auth/callback/route.ts`

```ts
// Target of the links Supabase emails (sign-up confirmation, password recovery). Exchanges the one-time
// code (PKCE) or token hash for a session cookie, then continues to a validated same-site path on the
// configured canonical origin (SOVEREIGN_SITE_URL), never on an origin taken from request headers.
import { NextResponse, type NextRequest } from 'next/server';
import { safeNextPath } from '@/lib/auth/next-path';
import { siteOrigin } from '@/lib/site-origin';
import { createSupabaseServerClient } from '@/lib/supabase/server';

const OTP_TYPES = ['signup', 'recovery', 'email', 'email_change', 'invite', 'magiclink'] as const;
type OtpType = (typeof OTP_TYPES)[number];
const isOtpType = (v: string | null): v is OtpType => OTP_TYPES.includes(v as OtpType);

export async function GET(request: NextRequest): Promise<NextResponse> {
  const origin = siteOrigin();
  if (!origin.ok) {
    console.error(`auth: callback unavailable (${origin.reason})`);
    return new NextResponse('Sign-in is temporarily unavailable.', { status: 503 });
  }

  const params = request.nextUrl.searchParams;
  const code = params.get('code');
  const tokenHash = params.get('token_hash');
  const type = params.get('type');
  const next = safeNextPath(params.get('next'));

  const supabase = await createSupabaseServerClient();
  let ok = false;
  if (code) {
    ok = !(await supabase.auth.exchangeCodeForSession(code)).error;
  } else if (tokenHash && isOtpType(type)) {
    ok = !(await supabase.auth.verifyOtp({ type, token_hash: tokenHash })).error;
  }
  return NextResponse.redirect(new URL(ok ? next : '/auth/sign-in?error=link', origin.origin));
}
```

### `apps/web/src/app/auth/callback/route.test.ts`

```ts
// Offline unit tests for the email-link callback: code / token-hash exchange, failure handling, the
// open-redirect guard, and that redirects use the configured canonical origin, not request headers.
// The user client is a test double.
import { NextRequest } from 'next/server';
import { afterEach, beforeEach, describe, expect, it, vi } from 'vitest';

const auth = { exchangeCodeForSession: vi.fn(), verifyOtp: vi.fn() };
vi.mock('@/lib/supabase/server', () => ({ createSupabaseServerClient: () => Promise.resolve({ auth }) }));

const { GET } = await import('./route');
const SITE = 'https://sovereign.example';
const call = async (query: string, init?: { url?: string; headers?: Record<string, string> }) => {
  const res = await GET(new NextRequest(`${init?.url ?? SITE}/auth/callback${query}`, { headers: init?.headers ?? {} }));
  return { status: res.status, location: res.headers.get('location') };
};

beforeEach(() => {
  auth.exchangeCodeForSession.mockReset();
  auth.verifyOtp.mockReset();
  vi.stubEnv('SOVEREIGN_SITE_URL', SITE);
});
afterEach(() => {
  vi.unstubAllEnvs();
  vi.restoreAllMocks();
});

describe('GET /auth/callback', () => {
  it('exchanges a valid code and continues to the requested same-site path', async () => {
    auth.exchangeCodeForSession.mockResolvedValue({ error: null });
    expect(await call('?code=abc&next=/auth/update-password')).toEqual({ status: 307, location: `${SITE}/auth/update-password` });
    expect(auth.exchangeCodeForSession).toHaveBeenCalledWith('abc');
  });

  it('sends a failed exchange back to sign-in with an error code', async () => {
    auth.exchangeCodeForSession.mockResolvedValue({ error: { message: 'expired' } });
    expect((await call('?code=abc')).location).toBe(`${SITE}/auth/sign-in?error=link`);
  });

  it('never redirects off-site, even after a successful exchange', async () => {
    auth.exchangeCodeForSession.mockResolvedValue({ error: null });
    expect((await call('?code=abc&next=https://evil.example/')).location).toBe(`${SITE}/workspace`);
    expect((await call('?code=abc&next=//evil.example')).location).toBe(`${SITE}/workspace`);
  });

  it('redirects to the canonical origin whatever the request URL and Host / X-Forwarded-* headers say', async () => {
    auth.exchangeCodeForSession.mockResolvedValue({ error: null });
    const r = await call('?code=abc&next=/workspace', {
      url: 'http://evil.example',
      headers: { host: 'evil.example', 'x-forwarded-host': 'attacker.example', 'x-forwarded-proto': 'http' },
    });
    expect(r.location).toBe(`${SITE}/workspace`);
  });

  it('fails closed (503, no exchange) when the site origin is not configured', async () => {
    vi.stubEnv('SOVEREIGN_SITE_URL', '');
    const errorLog = vi.spyOn(console, 'error').mockImplementation(() => undefined);
    const r = await call('?code=abc');
    expect(r).toEqual({ status: 503, location: null });
    expect(auth.exchangeCodeForSession).not.toHaveBeenCalled();
    expect(errorLog.mock.calls.join(' ')).toMatch(/^auth: callback unavailable/);
  });

  it('verifies a token hash of an allowed type', async () => {
    auth.verifyOtp.mockResolvedValue({ error: null });
    expect((await call('?token_hash=h1&type=recovery&next=/auth/update-password')).location).toBe(`${SITE}/auth/update-password`);
    expect(auth.verifyOtp).toHaveBeenCalledWith({ type: 'recovery', token_hash: 'h1' });
  });

  it('ignores a token hash with an unknown type', async () => {
    expect((await call('?token_hash=h1&type=admin')).location).toBe(`${SITE}/auth/sign-in?error=link`);
    expect(auth.verifyOtp).not.toHaveBeenCalled();
  });

  it('treats a request without code or token as a failed link', async () => {
    expect((await call('')).location).toBe(`${SITE}/auth/sign-in?error=link`);
    expect(auth.exchangeCodeForSession).not.toHaveBeenCalled();
  });
});
```

### `apps/web/src/app/auth/sign-in/page.tsx`

```ts
import type { Metadata } from 'next';
import Link from 'next/link';
import { signIn } from '../actions';
import { AuthShell, param, type ShellMessage } from '../AuthShell';
import { safeNextPath } from '@/lib/auth/next-path';

export const metadata: Metadata = { title: 'Sign in', robots: { index: false, follow: false } };

type Props = { searchParams: Promise<Record<string, string | string[] | undefined>> };

export default async function SignInPage({ searchParams }: Props) {
  const params = await searchParams;
  const error = param(params, 'error');
  const message: ShellMessage | null =
    error === 'invalid'
      ? {
          tone: 'error',
          text: 'We could not sign you in with those details. Check the email address and password. If you have just signed up, confirm your address with the link we emailed you first.',
        }
      : error === 'unavailable'
        ? { tone: 'error', text: 'Sign-in is temporarily unavailable. Please try again in a few minutes.' }
        : error === 'link'
          ? { tone: 'error', text: 'That link is invalid or has expired. Please try again.' }
          : param(params, 'signed-out')
            ? { tone: 'info', text: 'You are signed out.' }
            : null;

  return (
    <AuthShell
      eyebrow="Welcome back"
      title="Sign in"
      message={message}
      footer={
        <>
          <Link href="/auth/sign-up">Create an account</Link>
          <Link href="/auth/reset-password">Forgot your password?</Link>
        </>
      }
    >
      <form className="auth-form" action={signIn}>
        <input type="hidden" name="next" value={safeNextPath(param(params, 'next'))} />
        <label className="auth-field">
          Email address
          <input type="email" name="email" autoComplete="email" required />
        </label>
        <label className="auth-field">
          Password
          <input type="password" name="password" autoComplete="current-password" required />
        </label>
        <button type="submit" className="button">
          Sign in
        </button>
      </form>
    </AuthShell>
  );
}
```

### `apps/web/src/app/auth/sign-up/page.tsx`

```ts
import type { Metadata } from 'next';
import Link from 'next/link';
import { signUp } from '../actions';
import { AuthShell, param, type ShellMessage } from '../AuthShell';
import { PASSWORD_MAX, PASSWORD_MIN } from '@/lib/auth/next-path';

export const metadata: Metadata = { title: 'Create an account', robots: { index: false, follow: false } };

type Props = { searchParams: Promise<Record<string, string | string[] | undefined>> };

export default async function SignUpPage({ searchParams }: Props) {
  const error = param(await searchParams, 'error');
  const message: ShellMessage | null =
    error === 'input'
      ? { tone: 'error', text: `Enter a valid email address and a password of ${PASSWORD_MIN} to ${PASSWORD_MAX} characters.` }
      : error === 'signup'
        ? { tone: 'error', text: 'We could not create the account. Check the details and try again.' }
        : error === 'unavailable'
          ? { tone: 'error', text: 'Sign-up is temporarily unavailable. Please try again in a few minutes.' }
          : null;

  return (
    <AuthShell
      eyebrow="Join Sovereign"
      title="Create an account"
      message={message}
      footer={<Link href="/auth/sign-in">Already have an account? Sign in</Link>}
    >
      <form className="auth-form" action={signUp}>
        <label className="auth-field">
          Email address
          <input type="email" name="email" autoComplete="email" required />
        </label>
        <label className="auth-field">
          Password ({PASSWORD_MIN} characters or more)
          <input type="password" name="password" autoComplete="new-password" minLength={PASSWORD_MIN} maxLength={PASSWORD_MAX} required />
        </label>
        <button type="submit" className="button">
          Create account
        </button>
      </form>
      <p className="muted" style={{ margin: 0, fontSize: 13 }}>
        We will email you a link to confirm your address. Running agents requires a confirmed address.
      </p>
    </AuthShell>
  );
}
```

### `apps/web/src/app/auth/reset-password/page.tsx`

```ts
import type { Metadata } from 'next';
import Link from 'next/link';
import { requestPasswordReset } from '../actions';
import { AuthShell, param, type ShellMessage } from '../AuthShell';

export const metadata: Metadata = { title: 'Reset your password', robots: { index: false, follow: false } };

type Props = { searchParams: Promise<Record<string, string | string[] | undefined>> };

export default async function ResetPasswordPage({ searchParams }: Props) {
  const params = await searchParams;
  const message: ShellMessage | null = param(params, 'sent')
    ? { tone: 'info', text: 'If an account exists for that address, we have sent it a link to reset the password.' }
    : param(params, 'error') === 'input'
      ? { tone: 'error', text: 'Enter a valid email address.' }
      : param(params, 'error') === 'unavailable'
        ? { tone: 'error', text: 'Password reset is temporarily unavailable. Please try again in a few minutes.' }
        : null;

  return (
    <AuthShell eyebrow="Account recovery" title="Reset your password" message={message} footer={<Link href="/auth/sign-in">Back to sign in</Link>}>
      <form className="auth-form" action={requestPasswordReset}>
        <label className="auth-field">
          Email address
          <input type="email" name="email" autoComplete="email" required />
        </label>
        <button type="submit" className="button">
          Send reset link
        </button>
      </form>
    </AuthShell>
  );
}
```

### `apps/web/src/env.d.ts`

```ts
// Public Supabase settings, inlined into the browser bundle by Next.js (they are public by design).
// Declared explicitly so they can be read with dot access, which is what Next.js inlines.
// The service-role/secret key is deliberately NOT declared here: app code must never read it (ADR-0011).
declare namespace NodeJS {
  interface ProcessEnv {
    NEXT_PUBLIC_SUPABASE_URL?: string;
    /** Publishable key (sb_publishable_…); preferred when the project offers it. */
    NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY?: string;
    /** Legacy anon key; used only when no publishable key is configured. */
    NEXT_PUBLIC_SUPABASE_ANON_KEY?: string;
    /**
     * Canonical origin of this site (server-only), e.g. https://sovereign.example or, for local
     * development only, http://localhost:3000. Links in Auth emails and the callback's redirects are built
     * from it, never from Host / X-Forwarded-* headers. Validated by src/lib/site-origin.ts.
     */
    SOVEREIGN_SITE_URL?: string;
  }
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
  // Attempt every deletion, then fail loudly if any did not succeed. The report names the user ids and the
  // Auth status/code only (deleteTestUser's message), never an address, password, key or token.
  const failures: string[] = [];
  for (const id of created) {
    try {
      await deleteTestUser(id);
    } catch (e) {
      failures.push(e instanceof Error ? e.message : `could not delete test user ${id}`);
    }
  }
  if (failures.length > 0) {
    throw new Error(`test-user cleanup failed for ${failures.length} of ${created.length} users; delete them manually:\n${failures.join('\n')}`);
  }
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

### `apps/web/test/integration/support/admin.ts`

```ts
// THE ONLY service-role client in the repository (ADR-0011). Used solely by the integration tests to create,
// confirm and delete throwaway users through the Auth admin API; never for table access, never imported by
// application code (a lint rule and test/service-role-isolation.test.ts enforce both).
import { createClient, type SupabaseClient } from '@supabase/supabase-js';
import { integrationEnv } from './env';

let admin: SupabaseClient | undefined;

function adminClient(): SupabaseClient {
  if (!admin) {
    const { url, serviceRoleKey } = integrationEnv();
    admin = createClient(url, serviceRoleKey, { auth: { autoRefreshToken: false, persistSession: false } });
  }
  return admin;
}

/** Status and code of an Auth error, never its message (which can echo an address or other detail). */
const describeError = (error: { status?: number | undefined; code?: string | undefined } | null | undefined) =>
  error ? `status=${error.status ?? 'none'}, code=${error.code ?? 'none'}` : 'no data returned';

const RUN = `${Date.now().toString(36)}${Math.random().toString(36).slice(2, 6)}`;
/** Addresses on the reserved example.com domain; the admin API sends no email for them. */
export const testEmail = (label: string) => `sovereign-it-${RUN}-${label}@example.com`;
export const TEST_PASSWORD = `It-${RUN}-Passw0rd!`;

export async function createTestUser(label: string, confirmed: boolean): Promise<{ id: string; email: string }> {
  const email = testEmail(label);
  const { data, error } = await adminClient().auth.admin.createUser({ email, password: TEST_PASSWORD, email_confirm: confirmed });
  if (error || !data.user) throw new Error(`could not create test user ${label} (${describeError(error)})`);
  return { id: data.user.id, email };
}

export async function confirmTestUser(id: string): Promise<void> {
  const { error } = await adminClient().auth.admin.updateUserById(id, { email_confirm: true });
  if (error) throw new Error(`could not confirm test user ${id} (${describeError(error)})`);
}

/**
 * Creates a password-recovery token for a test user WITHOUT sending any email: the Auth admin API's
 * generateLink returns the token for the caller to deliver ("to be sent via a custom email provider").
 * This exercises the recovery flow; it does not exercise the app's own resetPasswordForEmail() request.
 */
export async function generateRecoveryToken(email: string, redirectTo: string): Promise<{ tokenHash: string; type: string }> {
  const { data, error } = await adminClient().auth.admin.generateLink({ type: 'recovery', email, options: { redirectTo } });
  if (error || !data.properties) throw new Error(`could not generate a recovery link (${describeError(error)})`);
  return { tokenHash: data.properties.hashed_token, type: data.properties.verification_type };
}

export async function deleteTestUser(id: string): Promise<void> {
  const { error } = await adminClient().auth.admin.deleteUser(id);
  if (error) throw new Error(`could not delete test user ${id} (${describeError(error)})`);
}
```

### `scripts/tap.mjs`

```js
// Strict TAP evaluation for the psql-based pgTAP runner (ADR-0011). Pure functions, no I/O, so every
// failure rule is unit-tested offline (scripts/tap.test.mjs). A file passes only when ALL of these hold:
//   - psql exited 0, did not time out, and wrote nothing to stderr;
//   - the first meaningful line is exactly one plan "1..N" with N >= 1;
//   - exactly N result lines follow, numbered 1..N in order, each "ok K" or "not ok K" (optional " - text");
//   - no result carries a SKIP or TODO directive, and pgTAP reported no "Looks like ..." summary;
//   - every other line is a "#" diagnostic; anything else is unexpected output.

/** @typedef {{ n: number, ok: boolean, text: string }} TapResult */
/**
 * @typedef {object} FileVerdict
 * @property {'pass' | 'assertion-failure' | 'error'} kind
 *   pass: valid TAP, every assertion ok. assertion-failure: valid TAP and a clean psql run, but one or more
 *   assertions are "not ok" (nothing else wrong). error: anything else (SQL/connection error, stderr output,
 *   timeout, malformed or incomplete TAP, unexpected output).
 * @property {number | null} planned
 * @property {TapResult[]} results
 * @property {number[]} failed   numbers of the "not ok" assertions
 * @property {string[]} reasons  human-readable reasons for a non-pass verdict (never contains secrets)
 */

const PLAN = /^1\.\.(\d+)$/;
const RESULT = /^(not )?ok (\d+)(?: - (.*))?$/;
const DIRECTIVE = /#\s*(SKIP|TODO)\b/i;

/** Parses psql's -t -A output of a pgTAP file. Structural problems are returned as reasons. */
export function parseTap(stdout) {
  const reasons = [];
  /** @type {TapResult[]} */
  const results = [];
  let planned = null;
  let planCount = 0;
  let sawMeaningful = false;

  for (const raw of stdout.split(/\r?\n/)) {
    const line = raw.trimEnd();
    if (line === '') continue;
    if (line === '#' || line.startsWith('# ')) {
      if (/Looks like you/i.test(line)) reasons.push(`pgTAP summary: ${line.slice(2)}`);
      continue;
    }
    const plan = PLAN.exec(line);
    if (plan) {
      planCount += 1;
      if (planCount > 1) reasons.push(`more than one plan line ("${line}")`);
      else if (sawMeaningful) reasons.push('plan line is not the first TAP line');
      else planned = Number(plan[1]);
      sawMeaningful = true;
      continue;
    }
    const res = RESULT.exec(line);
    if (res) {
      sawMeaningful = true;
      const n = Number(res[2]);
      const text = res[3] ?? '';
      if (DIRECTIVE.test(text)) reasons.push(`assertion ${n} uses a SKIP/TODO directive (not allowed)`);
      results.push({ n, ok: !res[1], text });
      continue;
    }
    sawMeaningful = true;
    reasons.push(`unexpected output: "${line.length > 120 ? `${line.slice(0, 120)}…` : line}"`);
  }

  if (planCount === 0) reasons.push('missing plan line');
  else if (planned !== null && planned < 1) reasons.push('plan declares zero tests');
  if (planned !== null) {
    if (results.length !== planned) reasons.push(`planned ${planned} assertions but ran ${results.length}`);
    results.forEach((r, i) => {
      if (r.n !== i + 1) reasons.push(`assertion numbering broken at position ${i + 1} (got ${r.n})`);
    });
  }
  return { planned, results, reasons };
}

/**
 * Combines the psql process outcome and the TAP output into one verdict.
 * @param {{ exitCode: number | null, stdout: string, stderr: string, timedOut?: boolean }} run
 * @returns {FileVerdict}
 */
export function evaluateRun(run) {
  const { planned, results, reasons } = parseTap(run.stdout ?? '');
  const failed = results.filter((r) => !r.ok).map((r) => r.n);
  const processReasons = [];
  if (run.timedOut) processReasons.push('psql timed out');
  if (run.exitCode !== 0) processReasons.push(`psql exited with code ${run.exitCode}`);
  const stderr = (run.stderr ?? '').trim();
  if (stderr) processReasons.push(`psql wrote to stderr: ${stderr.split(/\r?\n/)[0]?.slice(0, 200) ?? ''}`);

  // "Looks like you failed N tests" is pgTAP's own summary of not-ok assertions, not a structural problem.
  const structural = reasons.filter((r) => !(failed.length > 0 && /^pgTAP summary: Looks like you failed/i.test(r)));
  if (processReasons.length || structural.length) {
    return { kind: 'error', planned, results, failed, reasons: [...processReasons, ...structural] };
  }
  if (failed.length) {
    return { kind: 'assertion-failure', planned, results, failed, reasons: [`assertions not ok: ${failed.join(', ')}`] };
  }
  return { kind: 'pass', planned, results, failed, reasons: [] };
}

/**
 * The deliberate-failure control is confirmed only when the runner exited 1 AND the cause is exactly the
 * control's single failing assertion: kind assertion-failure, plan 1..1, one result, "not ok 1".
 * @param {number | null} exitCode exit code of the runner process that ran the control file alone
 * @param {FileVerdict | undefined} verdict the verdict that run reported
 */
export function controlConfirmed(exitCode, verdict) {
  if (exitCode === 0) return { confirmed: false, reason: 'runner returned 0 for a deliberately failing assertion' };
  if (exitCode !== 1) return { confirmed: false, reason: `runner returned ${exitCode} (expected 1); control is inconclusive` };
  if (!verdict) return { confirmed: false, reason: 'control run produced no report' };
  if (verdict.kind !== 'assertion-failure') {
    return { confirmed: false, reason: `control failed for the wrong reason: ${verdict.reasons.join('; ')}` };
  }
  if (verdict.planned !== 1 || verdict.results.length !== 1 || verdict.failed.join() !== '1') {
    return { confirmed: false, reason: 'control must be exactly one planned assertion, reported as "not ok 1"' };
  }
  return { confirmed: true, reason: 'runner returned 1 for the deliberate "not ok 1"' };
}

/**
 * Lexes SQL just far enough to tell code from comments and literals. Returns two views of the file, both the
 * same length as the input so positions line up:
 *   code      comments blanked, and the contents of '…', E'…', $tag$…$tag$ and "…" blanked (quotes kept);
 *             what is left is the text psql and the server read as statements, keywords and meta-commands.
 *   literals  comments blanked, literals kept: used for patterns that are dangerous even inside a string,
 *             because pgTAP functions (lives_ok, throws_ok, results_eq, is_empty, …) EXECUTE string arguments.
 * @returns {{ code: string, literals: string, problem: string | null }}
 */
export function lexSql(sql) {
  let code = '';
  let literals = '';
  const blank = (s) => s.replace(/[^\n]/g, ' ');
  const keep = (s) => {
    code += s;
    literals += s;
  };
  let i = 0;
  while (i < sql.length) {
    const c = sql[i];
    const two = sql.slice(i, i + 2);
    if (two === '--') {
      const end = sql.indexOf('\n', i);
      const stop = end === -1 ? sql.length : end;
      const s = blank(sql.slice(i, stop));
      code += s;
      literals += s;
      i = stop;
    } else if (two === '/*') {
      let depth = 0;
      let j = i;
      while (j < sql.length) {
        if (sql.startsWith('/*', j)) {
          depth++;
          j += 2;
        } else if (sql.startsWith('*/', j)) {
          depth--;
          j += 2;
          if (depth === 0) break;
        } else j++;
      }
      if (depth !== 0) return { code, literals, problem: 'unterminated /* comment' };
      const s = blank(sql.slice(i, j));
      code += s;
      literals += s;
      i = j;
    } else if (c === "'") {
      const escapes = /[eE]$/.test(sql.slice(0, i)) && !/[\w$]$/.test(sql.slice(0, i - 1));
      let j = i + 1;
      for (;;) {
        if (j >= sql.length) return { code, literals, problem: 'unterminated string literal' };
        if (escapes && sql[j] === '\\') j += 2;
        else if (sql[j] === "'" && sql[j + 1] === "'") j += 2;
        else if (sql[j] === "'") break;
        else j++;
      }
      const raw = sql.slice(i, j + 1);
      code += `'${blank(raw.slice(1, -1))}'`;
      literals += raw;
      i = j + 1;
    } else if (c === '"') {
      let j = i + 1;
      for (;;) {
        if (j >= sql.length) return { code, literals, problem: 'unterminated quoted identifier' };
        if (sql[j] === '"' && sql[j + 1] === '"') j += 2;
        else if (sql[j] === '"') break;
        else j++;
      }
      const raw = sql.slice(i, j + 1);
      code += `"${blank(raw.slice(1, -1))}"`;
      literals += raw;
      i = j + 1;
    } else if (c === '$' && !/[\w$]$/.test(sql.slice(0, i))) {
      const tag = /^\$([A-Za-z_][A-Za-z0-9_]*)?\$/.exec(sql.slice(i));
      if (!tag) {
        keep(c);
        i++;
        continue;
      }
      const end = sql.indexOf(tag[0], i + tag[0].length);
      if (end === -1) return { code, literals, problem: `unterminated ${tag[0]} string` };
      const stop = end + tag[0].length;
      const raw = sql.slice(i, stop);
      code += tag[0] + blank(raw.slice(tag[0].length, -tag[0].length)) + tag[0];
      literals += raw;
      i = stop;
    } else {
      keep(c);
      i++;
    }
  }
  return { code, literals, problem: null };
}

// Statements that end, split or escape the file's single transaction. Matched against the first word(s) of
// each top-level statement.
const TRANSACTION_CONTROL = [
  [/^commit\b/i, 'COMMIT'],
  [/^end\b/i, 'END (ends the transaction)'],
  [/^abort\b/i, 'ABORT'],
  [/^rollback\b/i, 'ROLLBACK before the last statement'],
  [/^begin\b/i, 'a second BEGIN'],
  [/^start\s+transaction\b/i, 'START TRANSACTION'],
  [/^savepoint\b/i, 'SAVEPOINT'],
  [/^release\b/i, 'RELEASE (SAVEPOINT)'],
  [/^prepare\s+transaction\b/i, 'PREPARE TRANSACTION'],
  [/^copy\b/i, 'COPY'],
];
// Dangerous anywhere, including inside strings that pgTAP executes: work done outside this session's
// transaction (dblink, pg_background), two-phase commit, and COPY (which can read or write server files and
// run programs).
const FORBIDDEN_ANYWHERE = [
  [/\bdblink\w*/i, 'dblink (separate connection outside this transaction)'],
  [/\bpg_background\w*/i, 'pg_background (separate session outside this transaction)'],
  [/\b(?:commit|rollback)\s+prepared\b/i, 'COMMIT/ROLLBACK PREPARED'],
  [/\bprepare\s+transaction\b/i, 'PREPARE TRANSACTION'],
  [/\bcopy\b/i, 'COPY'],
];
// psql meta-commands: only \gset (store a query result in psql variables) is allowed. Everything else can
// change the connection (\c, \connect), run shell commands (\!, \setenv, \g |cmd, \o |cmd), read or write
// files (\i, \ir, \include, \o, \w, \copy, \lo_*), or execute generated SQL (\gexec).
const ALLOWED_META = /^\\gset(?:\s+[A-Za-z_][A-Za-z0-9_]*)?\s*$/;

/**
 * Static shape every pgTAP file must have so it can never leave changes behind. It is a regex-and-lexer
 * check, NOT a SQL parser (limitations: docs/DECISIONS/0011 and the review package):
 *   - the first statement is BEGIN and the last is ROLLBACK, and nothing follows ROLLBACK;
 *   - no statement in between ends or splits the transaction (COMMIT, END, ABORT, ROLLBACK, BEGIN,
 *     START TRANSACTION, SAVEPOINT, RELEASE, PREPARE TRANSACTION) and none is COPY;
 *   - no dblink, pg_background, COMMIT/ROLLBACK PREPARED, PREPARE TRANSACTION or COPY anywhere, strings included;
 *   - no psql meta-command except \gset, and psql variables are interpolated only as :'name' (a quoted literal);
 *   - exactly one plan( and a finish() call, outside strings.
 * @returns {string[]} problems (empty when the file is acceptable)
 */
export function checkSqlTestFile(sql) {
  const { code, literals, problem } = lexSql(sql);
  if (problem) return [problem];
  const problems = [];

  for (const m of code.matchAll(/\\[^\s]*[^\n]*/g)) {
    const meta = m[0].trim();
    if (!ALLOWED_META.test(meta)) problems.push(`psql meta-command not allowed: ${meta.split(/\s/)[0]}`);
  }
  if (/(?<!:):(?![:=])\s*"/.test(code) || /(?<![:\w]):(?![:=])[A-Za-z_]/.test(code)) {
    problems.push("psql variables may be interpolated only as :'name'");
  }
  for (const [re, name] of FORBIDDEN_ANYWHERE) if (re.test(literals)) problems.push(`not allowed anywhere: ${name}`);

  // Statements end at ";" or at a \gset meta-command (which sends the query buffer).
  const statements = [];
  const terminators = [];
  let current = '';
  for (let i = 0; i < code.length; i++) {
    if (code[i] === ';') {
      statements.push(current.trim());
      terminators.push(';');
      current = '';
    } else if (code[i] === '\\') {
      const end = code.indexOf('\n', i);
      statements.push(current.trim());
      terminators.push('\\');
      current = '';
      i = end === -1 ? code.length : end;
    } else current += code[i];
  }
  const trailing = current.trim();
  const nonEmpty = statements.map((s, k) => ({ s, t: terminators[k] })).filter((x) => x.s !== '');

  if (nonEmpty.length === 0 || !/^begin$/i.test(nonEmpty[0].s) || nonEmpty[0].t !== ';') {
    problems.push('first statement must be BEGIN;');
  }
  const last = nonEmpty[nonEmpty.length - 1];
  if (trailing !== '' || !last || !/^rollback$/i.test(last.s) || last.t !== ';' || nonEmpty.length < 2) {
    problems.push('last statement must be ROLLBACK; with nothing after it');
  }
  for (const { s } of nonEmpty.slice(1, -1)) {
    for (const [re, name] of TRANSACTION_CONTROL) {
      if (re.test(s)) {
        problems.push(`statement not allowed inside the test transaction: ${name}`);
        break;
      }
    }
  }

  const plans = code.match(/\bplan\s*\(/gi) ?? [];
  if (plans.length !== 1) problems.push(`must call plan() exactly once (found ${plans.length})`);
  if (!/\bfinish\s*\(\s*\)/i.test(code)) problems.push('must call finish()');
  return problems;
}

/** A manifest path: relative, "/"-separated, no "." or ".." segments, ending in .test.sql. */
const isManifestPath = (p) =>
  typeof p === 'string' && /^[A-Za-z0-9_-]+(?:\/[A-Za-z0-9_-]+)*\.test\.sql$/.test(p);

/**
 * Validates the manifest against every *.test.sql file found RECURSIVELY under the tests directory, so no
 * test can be silently skipped: each discovered file is either the declared control or listed in the suite
 * exactly once, every listed file was discovered, and the control is never also listed in the suite.
 * @param {{ control?: unknown, suite?: unknown }} manifest
 * @param {string[]} filesOnDisk every *.test.sql under the tests directory, as "/"-separated relative paths
 */
export function checkManifest(manifest, filesOnDisk) {
  const problems = [];
  const onDisk = new Set(filesOnDisk);
  const control = manifest.control;
  if (typeof control !== 'string' || !control) problems.push('manifest.control must name the control file');
  else if (!isManifestPath(control)) problems.push(`manifest.control is not a relative *.test.sql path: ${control}`);
  else if (!onDisk.has(control)) problems.push(`control file not found: ${control}`);

  if (!Array.isArray(manifest.suite) || manifest.suite.length === 0) problems.push('manifest.suite must list the test files');
  const suite = Array.isArray(manifest.suite) ? manifest.suite : [];
  const seen = new Set();
  for (const f of suite) {
    if (!isManifestPath(f)) problems.push(`manifest.suite entry is not a relative *.test.sql path: ${String(f)}`);
    else if (!onDisk.has(f)) problems.push(`suite file not found: ${f}`);
    if (f === control) problems.push(`the control file must not be listed in the suite: ${f}`);
    if (seen.has(f)) problems.push(`manifest.suite lists a file twice: ${String(f)}`);
    seen.add(f);
  }
  for (const f of filesOnDisk) {
    if (f !== control && !seen.has(f)) problems.push(`test file not in manifest: ${f}`);
  }
  return problems;
}
```

### `scripts/tap.test.mjs`

```js
// Offline tests for the pgTAP runner (ADR-0011): parser rules, control logic, static/manifest checks, and the
// runner end to end against a fake psql (exit codes, control-then-suite flow, secret handling).
import assert from 'node:assert/strict';
import { spawnSync } from 'node:child_process';
import { mkdtempSync, mkdirSync, readFileSync, rmSync, writeFileSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { dirname, join } from 'node:path';
import { describe, it } from 'node:test';
import { fileURLToPath } from 'node:url';
import { checkManifest, checkSqlTestFile, controlConfirmed, evaluateRun, parseTap } from './tap.mjs';

const ok = (stdout) => ({ exitCode: 0, stdout, stderr: '' });
const GOOD = '1..2\nok 1 - first\nok 2 - second\n';

describe('evaluateRun: passing output', () => {
  it('passes a complete, all-ok run', () => {
    const v = evaluateRun(ok(GOOD));
    assert.equal(v.kind, 'pass');
    assert.equal(v.planned, 2);
    assert.deepEqual(v.failed, []);
  });

  it('accepts # diagnostics and blank lines between results', () => {
    assert.equal(evaluateRun(ok('1..2\n\n# setting up\nok 1\n# note\nok 2 - b\n')).kind, 'pass');
  });

  it('accepts CRLF line endings', () => {
    assert.equal(evaluateRun(ok('1..1\r\nok 1 - a\r\n')).kind, 'pass');
  });
});

describe('evaluateRun: every failure mode fails', () => {
  const cases = [
    ['a not ok assertion', ok('1..2\nok 1\nnot ok 2 - b\n# Looks like you failed 1 test of 2\n'), 'assertion-failure'],
    ['a missing plan', ok('ok 1\nok 2\n'), 'error'],
    ['an empty output', ok(''), 'error'],
    ['a zero plan', ok('1..0\n'), 'error'],
    ['a short run (fewer results than planned)', ok('1..3\nok 1\nok 2\n'), 'error'],
    ['extra results beyond the plan', ok('1..1\nok 1\nok 2\n'), 'error'],
    ['duplicated numbering', ok('1..2\nok 1\nok 1\n'), 'error'],
    ['skipped numbering', ok('1..2\nok 1\nok 3\n'), 'error'],
    ['two plan lines', ok('1..1\n1..1\nok 1\n'), 'error'],
    ['a plan after results', ok('ok 1\n1..1\n'), 'error'],
    ['unexpected output (stray query row)', ok('1..1\nINSERT 0 1\nok 1\n'), 'error'],
    ['a SKIP directive', ok('1..1\nok 1 - a # SKIP not ready\n'), 'error'],
    ['a TODO directive', ok('1..1\nnot ok 1 - a # TODO later\n'), 'error'],
    ['a pgTAP planned/ran mismatch summary', ok('1..2\nok 1\nok 2\n# Looks like you planned 3 tests but ran 2\n'), 'error'],
    ['a non-zero psql exit (SQL error, ON_ERROR_STOP)', { exitCode: 3, stdout: '1..2\nok 1\n', stderr: '' }, 'error'],
    ['stderr output even with exit 0', { exitCode: 0, stdout: GOOD, stderr: 'WARNING: something' }, 'error'],
    ['a timeout', { exitCode: null, stdout: GOOD, stderr: '', timedOut: true }, 'error'],
    ['a connection failure', { exitCode: 2, stdout: '', stderr: 'psql: error: connection failed' }, 'error'],
  ];
  for (const [name, run, kind] of cases) {
    it(`fails on ${name}`, () => {
      const v = evaluateRun(run);
      assert.equal(v.kind, kind);
      assert.notEqual(v.kind, 'pass');
      assert.ok(v.reasons.length > 0);
    });
  }

  it('reports exactly which assertions failed', () => {
    assert.deepEqual(evaluateRun(ok('1..3\nnot ok 1\nok 2\nnot ok 3\n')).failed, [1, 3]);
  });

  it('a not-ok plus a structural problem is an error, not an assertion failure', () => {
    assert.equal(evaluateRun(ok('1..2\nnot ok 1\n')).kind, 'error');
  });
});

describe('parseTap', () => {
  it('keeps assertion text', () => {
    assert.deepEqual(parseTap('1..1\nok 1 - user A sees only its workspace\n').results, [
      { n: 1, ok: true, text: 'user A sees only its workspace' },
    ]);
  });
});

describe('controlConfirmed', () => {
  const failing1 = evaluateRun(ok('1..1\nnot ok 1 - deliberate\n# Looks like you failed 1 test of 1\n'));
  it('is confirmed only for exit 1 caused by exactly "not ok 1"', () => {
    assert.equal(controlConfirmed(1, failing1).confirmed, true);
  });
  it('is rejected when the runner returned 0', () => {
    assert.equal(controlConfirmed(0, failing1).confirmed, false);
  });
  it('is rejected for exit 2 (configuration) or any other code', () => {
    assert.equal(controlConfirmed(2, failing1).confirmed, false);
    assert.equal(controlConfirmed(null, failing1).confirmed, false);
  });
  it('is rejected when exit 1 has the wrong cause', () => {
    assert.equal(controlConfirmed(1, evaluateRun({ exitCode: 3, stdout: '', stderr: 'ERROR' })).confirmed, false);
    assert.equal(controlConfirmed(1, evaluateRun(ok('1..2\nnot ok 1\nok 2\n'))).confirmed, false);
    assert.equal(controlConfirmed(1, evaluateRun(ok('1..1\nok 1\n'))).confirmed, false);
    assert.equal(controlConfirmed(1, undefined).confirmed, false);
  });
});

describe('checkSqlTestFile', () => {
  const valid = '-- header\nbegin;\nselect plan(1);\nselect ok(true);\nselect * from finish();\nrollback;\n';
  /** The valid file with extra statements inserted after plan(). */
  const withBody = (extra) => valid.replace('select ok(true);', () => `${extra}\nselect ok(true);`);
  const rejects = (sql, pattern) => {
    const problems = checkSqlTestFile(sql);
    assert.ok(problems.length > 0, 'expected the file to be rejected');
    if (pattern) assert.match(problems.join('\n'), pattern);
  };

  it('accepts the required shape', () => assert.deepEqual(checkSqlTestFile(valid), []));
  it("accepts CRLF line endings, \\gset and :'name' interpolation", () => {
    const sql = withBody("select id as w from t \\gset\nselect is(:'w'::uuid, :'w'::uuid, 'same');").replace(/\n/g, '\r\n');
    assert.deepEqual(checkSqlTestFile(sql), []);
  });
  it('accepts the four committed suite files', () => {
    for (const f of ['_selftest/deliberate_failure', 'identity_signup', 'identity_rls', 'catalog']) {
      const sql = readFileSync(fileURLToPath(new URL(`../packages/db/supabase/tests/${f}.test.sql`, import.meta.url)), 'utf8');
      assert.deepEqual(checkSqlTestFile(sql), [], f);
    }
  });
  it('ignores keywords inside comments and ordinary strings', () => {
    assert.deepEqual(checkSqlTestFile(valid.replace('-- header', '-- calls plan( once; commit; \\c other')), []);
    assert.deepEqual(checkSqlTestFile(withBody("/* commit; /* nested */ rollback; */ select is('a;commit;b', 'a;commit;b', 'x');")), []);
    assert.deepEqual(checkSqlTestFile(withBody("select ok($$ end $$ <> '', 'case ... end in a string');")), []);
  });

  it('requires BEGIN first', () => {
    rejects(valid.replace('begin;', ''), /first statement must be BEGIN/);
    rejects(valid.replace('begin;', 'select 1;\nbegin;'), /first statement must be BEGIN/);
  });
  it('requires exactly one plan()', () => {
    rejects(valid.replace('select plan(1);', ''), /plan\(\) exactly once \(found 0\)/);
    rejects(valid.replace('select plan(1);', 'select plan(1); select plan(1);'), /found 2/);
  });
  it('does not count plan( inside a string', () => rejects(valid.replace('select plan(1);', "select 'plan(1)';"), /found 0/));
  it('requires finish()', () => rejects(valid.replace('select * from finish();', ''), /finish\(\)/));
  it('requires ROLLBACK as the last statement (never COMMIT)', () => {
    rejects(valid.replace('rollback;', 'commit;'), /last statement must be ROLLBACK/);
    rejects(valid.replace('rollback;', 'rollback'), /last statement must be ROLLBACK/);
  });
  it('rejects anything after the final ROLLBACK', () => {
    rejects(`${valid}select 1;\n`, /last statement must be ROLLBACK/);
    rejects(`${valid}select 1`, /nothing after it/);
    rejects(`${valid}\\! echo hi\n`, /meta-command/);
  });

  describe('rejects transaction control inside the body', () => {
    for (const [stmt, pattern] of [
      ['commit;', /COMMIT/],
      ['COMMIT WORK;', /COMMIT/],
      ['commit and chain;', /COMMIT/],
      ['end;', /END/],
      ['end transaction;', /END/],
      ['abort;', /ABORT/],
      ['rollback;', /ROLLBACK before the last/],
      ['rollback to savepoint s1;', /ROLLBACK before the last/],
      ['begin;', /second BEGIN/],
      ['start transaction;', /START TRANSACTION/],
      ['savepoint s1;', /SAVEPOINT/],
      ['release savepoint s1;', /RELEASE/],
      ['release s1;', /RELEASE/],
      ["prepare transaction 't1';", /PREPARE TRANSACTION/],
      ["commit prepared 't1';", /PREPARED/],
      ["rollback prepared 't1';", /PREPARED/],
      ['select 1; commit; select 2;', /COMMIT/],
      ['select 1 \\gset\ncommit;', /COMMIT/],
    ]) {
      it(`rejects ${JSON.stringify(stmt)}`, () => rejects(withBody(stmt), pattern));
    }
  });

  describe('rejects work outside the transaction, anywhere in the file (strings included)', () => {
    for (const [stmt, pattern] of [
      ["select dblink_exec('dbname=postgres', 'delete from t');", /dblink/],
      ["select lives_ok($$ select dblink_exec('x', 'commit') $$);", /dblink/],
      ["select pg_background_launch('delete from t');", /pg_background/],
      ["select lives_ok('commit prepared ''t1''');", /PREPARED/],
      ["select lives_ok($q$ prepare transaction 't1' $q$);", /PREPARE TRANSACTION/],
      ["copy t to '/tmp/x';", /COPY/],
      ["copy t to program 'rm -rf /';", /COPY/],
      ["select lives_ok($$copy t from program 'id'$$);", /COPY/],
    ]) {
      it(`rejects ${JSON.stringify(stmt)}`, () => rejects(withBody(stmt), pattern));
    }
  });

  describe('rejects psql meta-commands other than \\gset', () => {
    for (const meta of [
      '\\c other_db',
      '\\connect postgres://evil.example/db',
      '\\! rm -rf /',
      '\\setenv PGHOST evil.example',
      '\\set ON_ERROR_STOP 0',
      '\\unset ON_ERROR_STOP',
      '\\o /tmp/out',
      '\\o |sh',
      '\\w /tmp/buffer',
      '\\i other.sql',
      '\\ir other.sql',
      '\\include other.sql',
      '\\copy t to /tmp/x',
      '\\lo_export 1 /tmp/x',
      '\\gexec',
      '\\g |sh',
      '\\g /tmp/out',
      '\\gset x y z',
      '\\q',
      '\\if true',
      '\\cd /',
    ]) {
      it(`rejects ${meta}`, () => rejects(withBody(`select 1 ${meta}`), /meta-command not allowed/));
    }
    it('rejects a meta-command after a statement on the same line', () => rejects(withBody('select 1; \\c other'), /meta-command not allowed: \\c/));
  });

  describe('rejects unsafe psql variable interpolation', () => {
    for (const stmt of ['select :w;', 'select * from :t;', 'select :"w";', ':cmd;']) {
      it(`rejects ${JSON.stringify(stmt)}`, () => rejects(withBody(stmt), /interpolated only as/));
    }
    it('does not mistake ::casts or array slices for variables', () => {
      assert.deepEqual(checkSqlTestFile(withBody("select ok(1::int = 1 and (array[1,2,3])[1:2] = array[1,2], 'casts');")), []);
    });
  });

  describe('rejects input it cannot lex', () => {
    for (const [name, stmt] of [
      ['an unterminated string', "select 'oops;"],
      ['an unterminated dollar string', 'select $$ oops;'],
      ['an unterminated block comment', '/* oops'],
      ['an unterminated quoted identifier', 'select "oops;'],
    ]) {
      it(`rejects ${name}`, () => rejects(withBody(stmt), /unterminated/));
    }
  });
});

describe('checkManifest', () => {
  const disk = ['_selftest/c.test.sql', 'a.test.sql', 'b.test.sql'];
  const base = { control: '_selftest/c.test.sql', suite: ['a.test.sql', 'b.test.sql'] };
  const rejects = (manifest, onDisk, pattern) => assert.match(checkManifest(manifest, onDisk).join('\n'), pattern);

  it('accepts a complete manifest', () => assert.deepEqual(checkManifest(base, disk), []));
  it('accepts a nested file that is listed', () => {
    assert.deepEqual(checkManifest({ ...base, suite: [...base.suite, 'rls/x.test.sql'] }, [...disk, 'rls/x.test.sql']), []);
  });
  it('fails when a top-level test file on disk is missing from the manifest', () => {
    rejects({ ...base, suite: ['a.test.sql'] }, disk, /test file not in manifest: b\.test\.sql/);
  });
  it('fails when a nested test file on disk is missing from the manifest', () => {
    rejects(base, [...disk, 'rls/hidden.test.sql'], /test file not in manifest: rls\/hidden\.test\.sql/);
    rejects(base, [...disk, '_selftest/second.test.sql'], /test file not in manifest: _selftest\/second\.test\.sql/);
  });
  it('fails on a duplicate suite entry', () => rejects({ ...base, suite: ['a.test.sql', 'b.test.sql', 'a.test.sql'] }, disk, /twice: a\.test\.sql/));
  it('fails on a missing suite file or control file', () => {
    rejects({ ...base, suite: [...base.suite, 'x.test.sql'] }, disk, /suite file not found: x\.test\.sql/);
    rejects({ ...base, control: '_selftest/missing.test.sql' }, disk, /control file not found/);
  });
  it('fails when the control file is also listed in the suite', () => {
    rejects({ ...base, suite: [...base.suite, '_selftest/c.test.sql'] }, disk, /control file must not be listed in the suite/);
  });
  it('fails without a control or a suite', () => {
    rejects({ suite: base.suite }, disk, /manifest\.control must name/);
    rejects({ control: base.control, suite: [] }, disk, /manifest\.suite must list/);
  });
  it('fails on paths that are absolute, escape the directory, use backslashes or are not *.test.sql', () => {
    for (const bad of ['/etc/x.test.sql', '../x.test.sql', 'rls/../a.test.sql', 'rls\\x.test.sql', './a.test.sql', 'a.sql', 42]) {
      rejects({ ...base, suite: [...base.suite, bad] }, disk, /not a relative \*\.test\.sql path/);
    }
  });
});

// ---------------------------------------------------------------- the runner end to end, with a fake psql
const RUNNER = fileURLToPath(new URL('./run-sql-tests.mjs', import.meta.url));
const FAKE = fileURLToPath(new URL('./fixtures/fake-psql.mjs', import.meta.url));
const SECRET = 'pw-Fake-Not-Real-123';
const SQL = 'begin;\nselect plan(1);\nselect ok(true);\nselect * from finish();\nrollback;\n';

/** Builds a throwaway tests directory and runs the runner against the fake psql. */
function runRunner(map, { args = [], url = `postgresql://tester:${SECRET}@db.example.test:5432/postgres`, extra = {} } = {}) {
  const dir = mkdtempSync(join(tmpdir(), 'sovereign-sqltest-'));
  try {
    mkdirSync(join(dir, '_selftest'));
    writeFileSync(join(dir, '_selftest', 'control.test.sql'), SQL);
    for (const f of ['a.test.sql', 'b.test.sql', 'c.test.sql']) writeFileSync(join(dir, f), SQL);
    for (const [rel, body] of Object.entries(extra)) {
      mkdirSync(dirname(join(dir, rel)), { recursive: true });
      writeFileSync(join(dir, rel), body);
    }
    writeFileSync(join(dir, 'manifest.json'), JSON.stringify({ control: '_selftest/control.test.sql', suite: ['a.test.sql', 'b.test.sql', 'c.test.sql'] }));
    const log = join(dir, 'calls.log');
    const env = { ...process.env, PSQL_PATH: FAKE, FAKE_PSQL_MAP: JSON.stringify(map), FAKE_PSQL_LOG: log, SOVEREIGN_SQL_TESTS_DIR: dir };
    if (url === null) delete env['SOVEREIGN_SQL_TEST_DB_URL'];
    else env['SOVEREIGN_SQL_TEST_DB_URL'] = url;
    const r = spawnSync(process.execPath, [RUNNER, ...args], { env, encoding: 'utf8' });
    let calls = [];
    try {
      calls = readFileSync(log, 'utf8').trim().split('\n').filter(Boolean).map((l) => JSON.parse(l));
    } catch {
      calls = [];
    }
    return { code: r.status, out: `${r.stdout}${r.stderr}`, calls };
  } finally {
    rmSync(dir, { recursive: true, force: true });
  }
}

const PASS = { stdout: '1..1\nok 1 - fine\n', code: 0 };
const CONTROL_FAILS = { stdout: '1..1\nnot ok 1 - deliberate\n# Looks like you failed 1 test of 1\n', code: 0 };

describe('runner end to end (fake psql)', () => {
  it('a single passing file exits 0; a deliberately failing assertion exits 1', () => {
    assert.equal(runRunner({ 'a.test.sql': PASS }, { args: ['--file', 'a.test.sql'] }).code, 0);
    assert.equal(runRunner({ 'a.test.sql': CONTROL_FAILS }, { args: ['--file', 'a.test.sql'] }).code, 1);
  });

  it('confirmed control, passing suite: runs all three files and exits 0', () => {
    const r = runRunner({ 'control.test.sql': CONTROL_FAILS, 'a.test.sql': PASS, 'b.test.sql': PASS, 'c.test.sql': PASS });
    assert.equal(r.code, 0, r.out);
    assert.deepEqual(r.calls.map((c) => c.file), ['control.test.sql', 'a.test.sql', 'b.test.sql', 'c.test.sql']);
    assert.match(r.out, /confirmed/);
  });

  it('a control that passes (runner would return 0) stops everything with exit 1 and runs no suite file', () => {
    const r = runRunner({ 'control.test.sql': PASS, 'a.test.sql': PASS, 'b.test.sql': PASS, 'c.test.sql': PASS });
    assert.equal(r.code, 1);
    assert.deepEqual(r.calls.map((c) => c.file), ['control.test.sql']);
    assert.match(r.out, /NOT CONFIRMED/);
  });

  it('a control failing for the wrong reason (SQL error) stops everything', () => {
    const r = runRunner({ 'control.test.sql': { stdout: '', stderr: 'ERROR: boom', code: 3 }, 'a.test.sql': PASS });
    assert.equal(r.code, 1);
    assert.deepEqual(r.calls.map((c) => c.file), ['control.test.sql']);
  });

  it('a failing suite file does not stop the others, and the run exits 1', () => {
    const r = runRunner({ 'control.test.sql': CONTROL_FAILS, 'a.test.sql': CONTROL_FAILS, 'b.test.sql': PASS, 'c.test.sql': { stdout: '1..2\nok 1\n', code: 0 } });
    assert.equal(r.code, 1);
    assert.deepEqual(r.calls.map((c) => c.file), ['control.test.sql', 'a.test.sql', 'b.test.sql', 'c.test.sql']);
    assert.match(r.out, /2 of 3 files failed/);
  });

  it('missing configuration exits 2 without running psql', () => {
    const r = runRunner({}, { url: null });
    assert.equal(r.code, 2);
    assert.equal(r.calls.length, 0);
  });

  it('a nested test file missing from the manifest exits 2 without running psql', () => {
    const r = runRunner({}, { extra: { 'rls/hidden.test.sql': SQL } });
    assert.equal(r.code, 2);
    assert.equal(r.calls.length, 0);
    assert.match(r.out, /test file not in manifest: rls\/hidden\.test\.sql/);
  });

  it('a suite file that commits mid-file exits 2 without running psql', () => {
    const r = runRunner({}, { extra: { 'a.test.sql': SQL.replace('select ok(true);', 'commit;\nselect ok(true);') } });
    assert.equal(r.code, 2);
    assert.equal(r.calls.length, 0);
    assert.match(r.out, /a\.test\.sql: .*COMMIT/);
  });

  it('passes ON_ERROR_STOP, keeps the URL out of argv and output, and uses libpq environment variables', () => {
    const r = runRunner({ 'control.test.sql': CONTROL_FAILS, 'a.test.sql': PASS, 'b.test.sql': PASS, 'c.test.sql': PASS });
    for (const c of r.calls) {
      assert.ok(c.args.includes('ON_ERROR_STOP=1'));
      assert.ok(!c.args.join(' ').includes(SECRET));
      assert.deepEqual(c.env, ['PGHOST', 'PGUSER', 'PGPASSWORD', 'PGDATABASE', 'PGSSLMODE']);
    }
    assert.ok(!r.out.includes(SECRET));
  });
});
```

### `scripts/run-sql-tests.mjs`

```js
// pgTAP runner over psql (ADR-0011). No dependencies; runs on plain Node.js.
//
//   pnpm test:rls                                  control first, then the whole suite
//   node scripts/run-sql-tests.mjs --file <rel> --json    one file; prints a JSON verdict line
//
// Configuration (environment; the URL is never printed or passed on a command line):
//   SOVEREIGN_SQL_TEST_DB_URL  postgres URL of the isolated development database (root .env, gitignored)
//   PSQL_PATH                  optional psql executable; a path ending in .mjs is run with Node (test seam)
//
// Exit codes: 0 every file passed · 1 a test failed (or the control was not confirmed) · 2 configuration problem.
// Control-then-suite: the deliberate-failure control runs alone, in a separate runner process, and is accepted
// only when that process exits 1 because of its single "not ok 1". Exit 0 (failures undetected) or any other
// cause stops the command before the suite. Once confirmed, every suite file runs, even after a failure.
import { spawnSync } from 'node:child_process';
import { existsSync, readFileSync, readdirSync } from 'node:fs';
import { dirname, join, resolve, sep } from 'node:path';
import { fileURLToPath } from 'node:url';
import { checkManifest, checkSqlTestFile, controlConfirmed, evaluateRun } from './tap.mjs';

const SCRIPT = fileURLToPath(import.meta.url);
const ROOT = resolve(dirname(SCRIPT), '..');
const TESTS_DIR = process.env['SOVEREIGN_SQL_TESTS_DIR'] ?? join(ROOT, 'packages', 'db', 'supabase', 'tests');
const FILE_TIMEOUT_MS = 120_000;

class ConfigError extends Error {}

/** libpq environment for psql, derived from the URL; nothing secret ever reaches argv or output. */
function psqlEnvironment() {
  const raw = process.env['SOVEREIGN_SQL_TEST_DB_URL'];
  if (!raw) throw new ConfigError('SOVEREIGN_SQL_TEST_DB_URL is not set (root .env, see .env.example)');
  let url;
  try {
    url = new URL(raw);
  } catch {
    throw new ConfigError('SOVEREIGN_SQL_TEST_DB_URL is not a valid URL');
  }
  if (url.protocol !== 'postgres:' && url.protocol !== 'postgresql:') {
    throw new ConfigError('SOVEREIGN_SQL_TEST_DB_URL must use postgres:// or postgresql://');
  }
  return {
    ...process.env,
    PGHOST: url.hostname,
    PGPORT: url.port || '5432',
    PGUSER: decodeURIComponent(url.username),
    PGPASSWORD: decodeURIComponent(url.password),
    PGDATABASE: decodeURIComponent(url.pathname.replace(/^\//, '')) || 'postgres',
    PGSSLMODE: url.searchParams.get('sslmode') ?? 'require',
    PGCONNECT_TIMEOUT: '15',
    PGAPPNAME: 'sovereign-sql-tests',
    PGOPTIONS: '-c client_min_messages=warning',
  };
}

function psqlCommand() {
  const p = process.env['PSQL_PATH'] ?? 'psql';
  return p.endsWith('.mjs') ? { cmd: process.execPath, pre: [p] } : { cmd: p, pre: [] };
}

function readTestFile(rel) {
  const abs = join(TESTS_DIR, rel);
  if (!existsSync(abs)) throw new ConfigError(`test file not found: ${rel}`);
  const problems = checkSqlTestFile(readFileSync(abs, 'utf8'));
  if (problems.length) throw new ConfigError(`${rel}: ${problems.join('; ')}`);
  return abs;
}

/** Runs one file through psql and returns its strict verdict. */
function runFile(rel, env) {
  const abs = readTestFile(rel);
  const { cmd, pre } = psqlCommand();
  const args = [...pre, '-X', '-q', '-t', '-A', '-v', 'ON_ERROR_STOP=1', '-f', abs];
  const r = spawnSync(cmd, args, { env, encoding: 'utf8', timeout: FILE_TIMEOUT_MS, windowsHide: true });
  if (r.error && r.error.code === 'ENOENT') throw new ConfigError(`psql not found (${cmd}); install it or set PSQL_PATH`);
  return evaluateRun({
    exitCode: r.status,
    stdout: r.stdout ?? '',
    stderr: r.stderr ?? '',
    timedOut: r.error?.code === 'ETIMEDOUT' || r.signal === 'SIGTERM',
  });
}

function loadManifest() {
  const path = join(TESTS_DIR, 'manifest.json');
  if (!existsSync(path)) throw new ConfigError('manifest.json not found in the tests directory');
  const manifest = JSON.parse(readFileSync(path, 'utf8'));
  // Every *.test.sql at any depth, as "/"-separated relative paths (also on Windows).
  const onDisk = readdirSync(TESTS_DIR, { recursive: true, encoding: 'utf8' })
    .map((f) => f.split(sep).join('/'))
    .filter((f) => f.endsWith('.test.sql'))
    .sort();
  const problems = checkManifest(manifest, onDisk);
  if (problems.length) throw new ConfigError(`manifest: ${problems.join('; ')}`);
  return manifest;
}

function printVerdict(rel, v) {
  const count = v.planned === null ? '?' : `${v.results.filter((r) => r.ok).length}/${v.planned}`;
  console.log(`${v.kind === 'pass' ? 'PASS' : 'FAIL'}  ${rel}  (${count})`);
  if (v.kind !== 'pass') {
    for (const r of v.results.filter((x) => !x.ok)) console.log(`      not ok ${r.n}${r.text ? ` - ${r.text}` : ''}`);
    for (const reason of v.reasons) console.log(`      ${reason}`);
  }
}

function singleFile(rel, json) {
  const verdict = runFile(rel, psqlEnvironment());
  if (json) console.log(JSON.stringify({ file: rel, verdict }));
  else printVerdict(rel, verdict);
  return verdict.kind === 'pass' ? 0 : 1;
}

function fullRun() {
  const manifest = loadManifest();
  for (const rel of [manifest.control, ...manifest.suite]) readTestFile(rel);
  psqlEnvironment(); // fail fast on configuration before spending a connection

  console.log(`control  ${manifest.control}`);
  const child = spawnSync(process.execPath, [SCRIPT, '--file', manifest.control, '--json'], {
    env: process.env,
    encoding: 'utf8',
    timeout: FILE_TIMEOUT_MS + 10_000,
    windowsHide: true,
  });
  if (child.status === 2) {
    console.error((child.stderr ?? '').trim());
    return 2;
  }
  let report;
  try {
    report = JSON.parse((child.stdout ?? '').trim().split(/\r?\n/).pop() ?? '');
  } catch {
    report = undefined;
  }
  const control = controlConfirmed(child.status, report?.verdict);
  console.log(`         ${control.confirmed ? 'confirmed' : 'NOT CONFIRMED'}: ${control.reason}`);
  if (!control.confirmed) {
    console.log('Stopping: the runner cannot be trusted to report failures, so the suite was not run.');
    return 1;
  }

  const env = psqlEnvironment();
  let failures = 0;
  for (const rel of manifest.suite) {
    const verdict = runFile(rel, env);
    printVerdict(rel, verdict);
    if (verdict.kind !== 'pass') failures += 1;
  }
  console.log(failures ? `${failures} of ${manifest.suite.length} files failed` : `all ${manifest.suite.length} files passed`);
  return failures ? 1 : 0;
}

function main(argv) {
  try {
    const i = argv.indexOf('--file');
    if (i !== -1) {
      const rel = argv[i + 1];
      if (!rel) throw new ConfigError('--file needs a path relative to the tests directory');
      return singleFile(rel, argv.includes('--json'));
    }
    return fullRun();
  } catch (e) {
    if (e instanceof ConfigError) {
      console.error(`configuration error: ${e.message}`);
      return 2;
    }
    throw e;
  }
}

process.exitCode = main(process.argv.slice(2));
```

### `.env.example`

```dotenv
# =============================================================================
# Sovereign — environment template
# Copy to `.env` (root) or the app-specific `.env.local` and fill in locally.
# NEVER commit real values. NEVER paste real values into chat, issues or docs.
# See docs/SECURITY.md (secrets), docs/ARCHITECTURE.md (§F.11) and docs/DATA-MODEL.md (§G.6).
# =============================================================================

# Which file gets which value (ADR-0011). All of these files are gitignored.
#   apps/web/.env.local   → the two public NEXT_PUBLIC_ values and SOVEREIGN_SITE_URL below (read by Next.js)
#   .env (root)           → SOVEREIGN_SQL_TEST_DB_URL (read by `pnpm test:rls` only)
#   .env.integration      → the public values again + SUPABASE_SERVICE_ROLE_KEY (read by
#                           `pnpm test:integration` and `pnpm check:bundle` only; never by Next.js)
# Use the isolated DEVELOPMENT project only. Never a production project.

# -----------------------------------------------------------------------------
# apps/web — values safe to expose to the browser (public by design)
# Prefer the publishable key (sb_publishable_…); the legacy anon key also works.
# -----------------------------------------------------------------------------
NEXT_PUBLIC_SUPABASE_URL=https://your-dev-project-ref.supabase.co
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=replace-with-dev-publishable-key
# NEXT_PUBLIC_SUPABASE_ANON_KEY=replace-with-dev-anon-key

# -----------------------------------------------------------------------------
# apps/web — canonical site origin (server-only; not a secret, not exposed to the browser)
# Links in Auth emails (sign-up confirmation, password recovery) and the /auth/callback
# redirects are built from this value, never from Host / X-Forwarded-* request headers.
# Rules (apps/web/src/lib/site-origin.ts): absolute URL, origin only (no path, query,
# fragment or credentials); https, or http only for localhost / 127.0.0.1 / [::1].
# Unset or invalid → sign-up and password reset report "temporarily unavailable" and the
# callback returns 503. It must also be listed in Supabase Auth → URL Configuration.
#   Local development:  SOVEREIGN_SITE_URL=http://localhost:3000
#   Deployed:           SOVEREIGN_SITE_URL=https://your-site.example
# -----------------------------------------------------------------------------
SOVEREIGN_SITE_URL=http://localhost:3000

# -----------------------------------------------------------------------------
# Integration tests ONLY — goes in the root .env.integration, nowhere else.
# Used solely by apps/web/test/integration/support/admin.ts to create and delete
# test users. Application code must never read it (lint rule + structural test).
# The worker must NOT use this key either (see docs/DATA-MODEL.md §G.6).
# -----------------------------------------------------------------------------
SUPABASE_SERVICE_ROLE_KEY=replace-with-dev-service-role-key

# -----------------------------------------------------------------------------
# SQL tests (pgTAP via psql) — root .env. Session-pooler connection string of the
# DEVELOPMENT project (Dashboard → Connect → Session pooler). Never printed.
# -----------------------------------------------------------------------------
SOVEREIGN_SQL_TEST_DB_URL=postgresql://postgres.your-dev-project-ref:replace-with-dev-db-password@aws-0-region.pooler.supabase.com:5432/postgres
# PSQL_PATH=C:\Program Files\PostgreSQL\18\bin\psql.exe

# -----------------------------------------------------------------------------
# apps/worker — SERVER ONLY
# Direct Postgres connection as the restricted `sovereign_worker` role.
# -----------------------------------------------------------------------------
WORKER_DATABASE_URL=postgresql://sovereign_worker:replace-with-local-password@127.0.0.1:54322/postgres
WORKER_ID=local-worker-1

# LLM provider (operator-funded; see docs/PRD.md §C.1). Leave empty until M5.
ANTHROPIC_API_KEY=

# Web search provider for the web_search tool (chosen at M5). Leave empty.
SEARCH_PROVIDER=
SEARCH_API_KEY=
```

### `apps/web/README.md`

```markdown
# @sovereign/web

**Responsibility:** the Next.js application and all user interface (docs/ARCHITECTURE.md §F.4, docs/PRD.md §D).
- Route groups: `(public)` (landing, Realm, Orders, Oracle, Codex, Chronicle, auth), `(app)/w/[workspace]` (Citadel, Forge, Quests, Guilds, Watchtower, Settings), `(admin)`.
- Server Components for reads, and Server Actions that call `@sovereign/core` services for mutations.
- **Composition root** (`src/server/composition.ts`, server-only): per request, it builds the user-session Supabase client, wraps it in `@sovereign/db` adapters, and injects them into core services. This is the only place where core and db meet in this app (ADR-0009).
- Realtime client hooks (private Broadcast topics, notify-then-fetch).
- The design system: tokens, shadcn/ui primitives, Sovereign components, goddess artwork slots.

**Must not contain:** business rules, permission logic, or budget logic. It calls `@sovereign/core`. RLS in the database is the second line of defence, not a substitute for those calls. No direct SQL.

**Credentials:** the browser and the server user clients get only `NEXT_PUBLIC_SUPABASE_URL` and the publishable (or anon) key, and act as the signed-in user under RLS. `SUPABASE_SERVICE_ROLE_KEY` is never read by application code: it lives only in the root `.env.integration` for the integration-test admin helper (ADR-0011). A lint rule, a structural test and `pnpm check:bundle` enforce this.

**Site origin:** `SOVEREIGN_SITE_URL` (server-only, in `apps/web/.env.local`) is the canonical origin used for links in Auth emails and for the `/auth/callback` redirects; request headers (Host, X-Forwarded-*) are never used for them. It must be an origin only, https, or http for localhost / 127.0.0.1 / [::1] in local development, e.g. `SOVEREIGN_SITE_URL=http://localhost:3000`. When it is unset or invalid, sign-up and password reset report "temporarily unavailable" and the callback returns 503. See `.env.example`.

**Allowed internal dependencies:** `@sovereign/core`, `@sovereign/db` (adapters and clients, used only in the composition root).

**Server-only convention:** `src/server/` (including the composition root) is server code. Until the `server-only` package arrives at M2, nothing enforces this at build time, so client components must not import from `src/server/`.

**Status (M0.4):** minimal App Router scaffold (`src/app/layout.tsx`, `src/app/page.tsx`) and `GET /api/health`, which checks configuration only and returns pass/fail per check, never reasons or configuration (docs/ARCHITECTURE.md §H.1). `next build` passes on Node 26.13.0 with Next.js 16.4.5. No design system or product UI yet (M1).
```

### `docs/DECISIONS/0011-hosted-dev-supabase-and-sql-tests.md`

```markdown
# 0011 — Hosted development Supabase, service-role isolation and the psql pgTAP runner
- Status: Accepted (approved for M2, 2026-11-12)
- Date: 2026-11-12
- Baseline refs: §G.5, §G.8, §I.5, §I.8; ADR-0001, ADR-0006

## Context
M2 (identity and tenant isolation) needs a Supabase database. Docker is not available on the development machine, so `supabase start` cannot run. On Windows, `supabase test db` also needs Docker even against a hosted project, because the CLI has no native `pg_prove` and starts it in a container. PostgreSQL 18's `psql` client is installed. M2 also introduces the first privileged credential (the service-role key), and its handling had to be decided before any code used it.

## Decision
1. **Hosted development project.** M2 runs against a dedicated, isolated Supabase **development** project created by the owner. Never a production project. Migrations are applied with `supabase db push`; types are generated with `supabase gen types --linked`. Neither needs Docker.
2. **Service-role isolation.**
   - Browser and server user clients (`apps/web/src/lib/supabase`) use only the project URL and the publishable/anon key, with the user's session. Every query runs under RLS.
   - The verified-email gate (`src/lib/auth/gate.ts`, `src/server/auth.ts`) trusts only `supabase.auth.getUser()`, which the Auth server validates; never `getSession()` or client metadata.
   - The service-role key lives only in the root `.env.integration`, which Next.js never loads. Its only user is `apps/web/test/integration/support/admin.ts`, which creates, confirms and deletes test users through the Auth admin API.
   - Enforcement: ESLint `no-restricted-syntax` / `no-restricted-imports` in application and package source; `apps/web/test/service-role-isolation.test.ts` (offline, part of `pnpm test`); `pnpm check:bundle` after each build (scans `.next/static` for the key names, the `sb_secret_` prefix and, when available, the key value, without printing it).
3. **pgTAP through psql** (`scripts/run-sql-tests.mjs`, parser `scripts/tap.mjs`), run by `pnpm test:rls`.
   - **Files** come from `packages/db/supabase/tests/manifest.json`. Every `*.test.sql` at any depth under the tests directory must be either the declared control or listed in the suite exactly once; the control must not also be in the suite. Each file must start with `BEGIN;`, call `plan()` once, call `finish()` and end with `ROLLBACK;` with nothing after it, so tests never leave changes behind. pgTAP itself is created inside each transaction.
   - **Static file check** (`checkSqlTestFile` in `scripts/tap.mjs`), run before any connection: a small lexer masks comments, quoted strings, dollar-quoted strings and quoted identifiers, then rejects transaction control between `BEGIN` and the final `ROLLBACK` (COMMIT, END, ABORT, ROLLBACK, BEGIN, START TRANSACTION, SAVEPOINT, RELEASE, PREPARE TRANSACTION), COPY, every psql meta-command except `\gset`, unquoted psql variable interpolation, and, anywhere including strings, dblink, pg_background, COMMIT/ROLLBACK PREPARED, PREPARE TRANSACTION and COPY. **Limitations:** it is not a SQL parser. It cannot see SQL built at run time (for example `EXECUTE format(...)` inside a function, or a function defined elsewhere that commits); it relies on PostgreSQL refusing COMMIT/ROLLBACK inside functions, DO blocks and procedures called within an explicit transaction block, and on `ON_ERROR_STOP=1` plus disconnect-rollback when a statement fails; and its string-wide bans can reject harmless text (an assertion description containing the word "copy"). Files are reviewed code, not untrusted input: the check guards against mistakes, not a malicious author.
   - **Execution:** `psql -X -q -t -A -v ON_ERROR_STOP=1 -f <file>`, `client_min_messages=warning`, 120 s per file. The URL comes from `SOVEREIGN_SQL_TEST_DB_URL` (root `.env`) and reaches psql only as libpq environment variables; it is never in argv or output.
   - **A file passes only if:** psql exits 0, does not time out and writes nothing to stderr; the first TAP line is one plan `1..N` (N ≥ 1); exactly N results follow, numbered 1..N; all are `ok`; none has a SKIP/TODO directive; pgTAP printed no "Looks like …" summary; every other line is a `#` diagnostic. Any other line is unexpected output.
   - **Exit codes:** 0 all passed; 1 a test failed or the control was not confirmed; 2 configuration problem.
   - **Control then suite:** `_selftest/deliberate_failure.test.sql` (one false assertion) runs first, alone, in a separate runner process. It is confirmed only when that process exits 1 because of exactly `not ok 1`. Exit 0 (failures undetected), exit 2, or exit 1 for any other cause stops the command before the suite. After confirmation, every suite file runs, even after a failure, and all are reported.
   - **Offline proof:** `scripts/tap.test.mjs` covers every failure rule, the control logic and the runner end to end against a fake psql (`scripts/fixtures/fake-psql.mjs`, selected with `PSQL_PATH`; a tests directory can be substituted with `SOVEREIGN_SQL_TESTS_DIR`).
4. **Separate test layers, both required.** pgTAP proves PostgreSQL enforcement (trigger, constraints, RLS as anon and two users, privileges, catalog). The integration suite (`pnpm test:integration`) proves Supabase Auth flows and the gate. Its single cross-tenant read only shows the app's client carries the user's session; it is not evidence of isolation.

## Consequences
- The SQL and integration suites need the development project's credentials and are not part of the offline `pnpm test`.
- Automated tests never send email: test users are created through the admin API on the reserved `example.com` domain. Sign-up confirmation and password reset are checked manually in a browser with the owner's address.
- Moving to local Supabase later (Docker) changes only where migrations and tests run; the runner, files and isolation rules stay the same.
```
