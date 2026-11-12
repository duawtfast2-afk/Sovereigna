# M2 review package v4: audit remediation round 3

Date: 2026-11-12 · Branch `feat/landing-and-realm-preview` · HEAD `10013c0` (unchanged) · Staged files: 0 before and after.

> **Status: M2 is NOT complete and NOT ready for deployment.** Every result here is offline: unit tests with test doubles, a fake psql, static checks and code inspection. No migration, SQL test or Auth flow has run against any database or Auth server. The hosted database and Auth integration evidence is still outstanding.

## 0. Safety confirmation

| Constraint | Status |
|---|---|
| Hosted Supabase project contacted, inspected or configured | **No.** No `supabase` CLI command was run. The only network-capable test commands (`pnpm test:rls`, `pnpm test:integration`) stopped on missing configuration before any connection (section 5). |
| `supabase link`, `db push`, migrations, or any database write | **None.** |
| Credential files created, modified or populated | **None.** `.env`, `.env.local`, `.env.integration`, `apps/web/.env.local` and `apps/web/.env` do not exist (checked by listing only; nothing was read). `.env.example` was not touched this round. |
| Dependencies installed, removed or updated | **None.** `package.json` and `pnpm-lock.yaml` were not edited this round. |
| Git staging or history | **None.** No stage, commit, push, reset, stash or discard. `git diff --cached` is empty before and after; HEAD is `10013c0` before and after. |
| Servers restarted or started | **None.** Port 3900 was not touched, and no dev server was started. Browser verification was therefore not done. |
| `.next` rebuilt or modified | **No.** `.next/types` snapshot is `9f92fb46c33f037e` before and after; `BUILD_ID` is `rYrvX0swBrsAr-hHFLhVl` before and after. Web typecheck used `tsc --noEmit`, not `pnpm typecheck`. |

---

## 1. Summary of the five remediations, plus the checker edge case

1. **Callback origin mismatch.**
   - `/auth/callback` compares the request's normalised origin (`new URL(request.nextUrl.href).origin`) with the canonical `SOVEREIGN_SITE_URL` origin.
   - On any mismatch it returns **400** with fixed text, no `Location` header and no `Set-Cookie`.
   - The check runs before the Supabase client is created and before `exchangeCodeForSession()` or `verifyOtp()`. A mismatched host therefore never touches the code or token, and no session cookie is written for it.
   - The same-site `next` handling (`safeNextPath`) is unchanged on the accepted path.
2. **Strict raw `SOVEREIGN_SITE_URL`.**
   - The raw string itself must match `scheme://host[:port][/]`, with a lowercase DNS name or `[::1]` as the host.
   - The raw string (minus one optional trailing `/`) must also equal the parser's `url.origin` exactly.
   - So anything the WHATWG URL parser would rewrite is rejected:
     - dot-segments: `/..`, `/foo/..`, `%2e%2e`, `.%2e`;
     - other percent-encoding and backslashes;
     - `https:/x` and `https:x` forms;
     - upper-case schemes and hosts, and trailing dots;
     - an explicit default port, or an empty port;
     - shortened IPv4 (`127.1`) and expanded IPv6;
     - empty userinfo.
   - The https-only rule for non-loopback hosts is unchanged.
3. **Thrown exceptions in sign-in and sign-up.**
   - Client creation and the Auth call now run inside `attemptAuth()`. It turns any thrown value into a fixed `{ thrown: true }` marker and never reads the exception's message or stack.
   - Sign-up, sign-in and the password-reset request all treat a thrown exception as **unavailable**. They log the fixed line `auth: <action> failed (unexpected exception)` and redirect with `error=unavailable`.
   - Returned (non-thrown) errors keep the existing classification. Sign-in still gives the same `error=invalid` for every account-dependent 4xx, and reset still gives the same `sent=1`.
   - The reset request's client creation is now inside the guard too. In v3 it was outside the try.
4. **SQL runner `--file` containment.**
   - `containedTestPath()` resolves the path to an absolute path and requires it to stay inside the tests directory, checked with `path.relative`. It rejects traversal of any depth, paths on another drive or a UNC share, the directory itself, and anything not ending in `*.test.sql`.
   - The runner checks this lexically, then again on real paths (`realpathSync`), so a symlink or junction inside the directory cannot lead outside it.
   - Both checks run before the file is read or passed to psql. Nested files and absolute paths inside the directory still work.
5. **Cleanup diagnostics.**
   - A new pure module `support/diagnostics.ts` is the only producer of cleanup text. It includes only:
     - user ids that are UUIDs;
     - HTTP statuses that are integers from 100 to 599;
     - error codes matching `^[a-z0-9_]{1,64}$`;
     - fixed text.
   - `admin.ts` throws a `SafeDiagnosticError` whose message is built from those parts only.
   - Cleanup (`deleteAllTestUsers`) attempts every deletion and keeps the message only of a `SafeDiagnosticError`. Any other exception becomes `could not delete test user <id> (unexpected exception)`; its message is never read.

**Checker edge case.** Previously `:{?name}` was **not** caught: the interpolation rule only looked at `:name` and `:"name"`. It is now rejected, together with the `: {?name}` spelling, and is documented in ADR-0011. It is harmless in practice (it expands to TRUE or FALSE), but the policy is a single form, `:'name'`, so it gets no exception. No runner capability was added.

---

## 2. Files changed this round (17) and why

| # | File | Status | Reason |
|---|---|---|---|
| 1 | `apps/web/src/app/auth/callback/route.ts` | modified | Finding 1: reject a non-canonical request origin before any exchange |
| 2 | `apps/web/src/app/auth/callback/route.test.ts` | modified | Finding 1 tests (14 tests, was 8) |
| 3 | `apps/web/src/lib/site-origin.ts` | modified | Finding 2: raw-form check |
| 4 | `apps/web/src/lib/site-origin.test.ts` | modified | Finding 2 tests (57 tests, was 23) |
| 5 | `apps/web/src/lib/auth/auth-outcome.ts` | modified | Finding 3: `attemptAuth`, `authExceptionLogLine` |
| 6 | `apps/web/src/lib/auth/auth-outcome.test.ts` | modified | Finding 3 tests (17 tests, was 14) |
| 7 | `apps/web/src/app/auth/actions.ts` | modified | Finding 3: sign-up, sign-in and reset guarded by `attemptAuth` |
| 8 | `apps/web/src/app/auth/actions.test.ts` | modified | Finding 3 tests (51 tests, was 37). The v3 reset case "a thrown exception" moved into the new thrown-exception matrix, with the new fixed log line. |
| 9 | `scripts/tap.mjs` | modified | Finding 4: `containedTestPath`. Edge case: reject `:{?name}`. |
| 10 | `scripts/run-sql-tests.mjs` | modified | Finding 4: containment before reading, lexical and on real paths |
| 11 | `scripts/tap.test.mjs` | modified | Finding 4 and edge-case tests (157 tests, was 113). The `runRunner` helper now builds `<base>/tests`, so outside files can sit next to the tests directory. |
| 12 | `apps/web/test/integration/support/diagnostics.ts` | **new** | Finding 5: pure sanitiser and cleanup loop |
| 13 | `apps/web/test/integration/support/admin.ts` | modified | Finding 5: `SafeDiagnosticError`, validated status, code and id |
| 14 | `apps/web/test/integration/auth.test.ts` | modified | Finding 5: cleanup through `deleteAllTestUsers` and `cleanupFailureReport` |
| 15 | `apps/web/test/cleanup-diagnostics.test.ts` | **new** | Finding 5: offline tests (28) |
| 16 | `docs/DECISIONS/0011-hosted-dev-supabase-and-sql-tests.md` | modified | Documents `:{?name}` and `--file` containment |
| 17 | `apps/web/README.md` | modified | Documents the raw-form rule and the callback's origin check |

Not changed: `.env.example`, the sign-in, sign-up and reset pages, `manifest.json`, all `*.test.sql` files, migrations, `package.json`, `pnpm-lock.yaml`, and every Landing/Realm file.

---

## 3. Findings in detail

### Finding 1: callback host and canonical-origin mismatch

**Order of operations in `GET /auth/callback`:**
1. Read `SOVEREIGN_SITE_URL`. If it is unset or invalid, return 503 (unchanged from v3).
2. Normalise the request origin with `new URL(request.nextUrl.href).origin`. This lowercases the scheme and host and drops a default port. Compare it with the canonical origin. If they are not equal, return 400 with the fixed text "This link must be opened on the site it was sent from." The response has no `Location` and no `Set-Cookie`, and the fixed log line `auth: callback rejected (request origin is not the configured site origin)` contains no host, code or token.
3. Only then create the Supabase server client and exchange the code or verify the token hash.
4. Redirect to `safeNextPath(next)` on the canonical origin (unchanged).

Because the rejection happens before `createSupabaseServerClient()`, the cookie adapter is never instantiated on a mismatched host. Nothing can call `setAll` there.

A redirect to the canonical host carrying the code was not chosen. A PKCE code verifier lives in a cookie on the host where the flow started, so forwarding the code would only fail later, and it would move a live code across hosts.

**Tests (`route.test.ts`, executed, 14 pass):**
- Five mismatches, each with both a `code` request and a `token_hash` request, give 400, no `Location`, no `Set-Cookie`, a body without the code, token or hosts, a log without them, and **zero** calls to `exchangeCodeForSession` and `verifyOtp`:
  - another host;
  - http instead of https;
  - another port;
  - a subdomain;
  - a raw IP.
- Normalisation: `https://SOVEREIGN.example:443/...` is accepted and exchanged, because origins are compared after normalisation.
- The canonical flow still works: code exchange, token-hash verification, a failed link, unknown OTP type, no code, off-site `next` replaced with `/workspace`, and Host / X-Forwarded-* headers that do not change the redirect.
- Unset configuration gives 503 with no exchange.

### Finding 2: strict raw-input validation of `SOVEREIGN_SITE_URL`

**Checks, in order:** set; no surrounding whitespace; parses as an absolute URL; http(s) only; http only for loopback; no credentials; then the new raw-form check:

```ts
const LABEL = '[a-z0-9](?:[a-z0-9-]{0,61}[a-z0-9])?';
const RAW_ORIGIN = new RegExp(`^https?://(?:${LABEL}(?:\\.${LABEL})*|\\[::1\\])(?::[0-9]{1,5})?/?$`);
if (!RAW_ORIGIN.test(raw) || raw.replace(/\/$/, '') !== url.origin) → reject ('… origin only …, written in canonical form')
```

The regex rejects:
- any `/` beyond one trailing slash, so `/..`, `/.`, `//`, `/foo/..` all fail;
- any `%` (encoded dot-segments, `%2e` in the host, `%20`);
- `\`, `?`, `#` and `@`;
- upper-case letters, trailing dots, non-ASCII hosts, and IPv6 other than `[::1]`.

The equality with `url.origin` additionally rejects values the regex admits but the parser would rewrite: `:443` on https, `127.1`, an empty port.

**Tests (`site-origin.test.ts`, executed, 57 pass):**
- Accepted: `https://sovereign.example`, the same with a trailing `/`, `https://sovereign.example:8443`, `http://localhost:3000`, `http://127.0.0.1:3900`, `http://[::1]:3000`.
- Rejected, all required cases included:
  - dot-segments: `https://sovereign.example/..`, `/.`, `/foo/..`, `/foo/../`;
  - encoded dot-segments: `/%2e%2e`, `/%2E%2E/`, `/.%2e`, `/%2e`;
  - other paths: `//`, `/%20`, `https://sovereign.example/app`;
  - backslashes: `https://sovereign.example\`, `https://sovereign.example\..`, `https:\\sovereign.example`;
  - malformed scheme separators: `https:/sovereign.example`, `https:sovereign.example`, `https:///sovereign.example`;
  - case and form: `HTTPS://…`, `https://Sovereign.Example`, a trailing-dot host, `:443`, an empty port `https://sovereign.example:`, `https://sovereign%2eexample`, non-ASCII `https://sövereign.example`;
  - loopback rewrites: `http://127.1:3000`, `http://LOCALHOST:3000`, `http://localhost:3000/..`, `http://[0:0:0:0:0:0:0:1]:3000`;
  - whitespace: a tab, an embedded newline;
  - credentials: `https://user:pass@…`, `https://user@…`, `https://@…`, `https://:@…`;
  - queries and fragments: `?next=/x`, `?`, `#frag`, `#`, `/..?x`, `/..#x`;
  - schemes: `ftp:`, `javascript:`, and http for a non-loopback host (`http://sovereign.example`, `http://localhost.evil.example`).
- No reason echoes the configured value.

**Stricter than before (intended):** the value must be written exactly as the canonical origin: a lower-case host, no explicit default port, and a punycode (`xn--`) host for international names.

### Finding 3: thrown exceptions in sign-in and sign-up

```ts
export async function attemptAuth<T>(step: () => Promise<{ error: T }>): Promise<{ error: T } | { thrown: true }> {
  try { return await step(); } catch { return { thrown: true }; }
}
export const authExceptionLogLine = (action: string) => `auth: ${action} failed (unexpected exception)`;
```

**How each action uses it:** the step is `async () => (await createSupabaseServerClient()).auth.<call>(…)`, so both client creation and the Auth call are covered.
- On `{ thrown: true }`: log the fixed line, then redirect to `error=unavailable`.
- Otherwise: the v3 logic, unchanged. `redirect()` is called outside the guard, so Next.js's redirect signal is never swallowed.

**Tests (executed):**
- `actions.test.ts` runs a matrix of 3 actions × 5 cases (15 tests):
  - the Auth call rejects with an `Error` whose message and stack contain an address, a JWT-like token and the password;
  - it throws synchronously;
  - it rejects with a non-Error string;
  - it rejects with an Auth-shaped 4xx object;
  - `createSupabaseServerClient()` rejects.

  Each case asserts the exact `…?error=unavailable` URL, that the URL contains no sensitive text, and that the only log line is exactly `auth: <action> failed (unexpected exception)`.
- The existing generic-response tests still pass: sign-in `invalid` for four account-dependent errors, and reset `sent=1` for five outcomes.
- `auth-outcome.test.ts` adds 3 tests: returned results pass through; any thrown value becomes `{ thrown: true }` with nothing from the exception; the fixed log line.

**Decision note:** a *thrown* object shaped like a 4xx Auth error is treated as unavailable, not masked. The finding asked for thrown exceptions to be "service-unavailable outcomes". supabase-js returns Auth errors rather than throwing them, so a throw indicates a client or runtime failure. This does not depend on the account.

### Finding 4: SQL runner `--file` path containment

```js
export function containedTestPath(testsDir, rel, p = path) {
  if (typeof rel !== 'string' || rel === '' || rel.includes('\0')) → reject
  if (!rel.endsWith('.test.sql')) → reject
  const root = p.resolve(testsDir);
  const abs = p.resolve(root, rel);
  const inside = p.relative(root, abs);
  if (inside === '' || inside === '..' || inside.startsWith(`..${p.sep}`) || p.isAbsolute(inside)) → reject 'outside the tests directory'
  return { ok: true, abs };
}
```

**In `readTestFile`:**
1. Lexical containment. A failure throws a configuration error (exit 2) before `existsSync`, `readFileSync` or psql.
2. Existence check.
3. Containment again on real paths: `realpathSync(TESTS_DIR)` and `realpathSync(abs)`. A link that escapes is rejected before reading.
4. Only then read the file, run the static checks and run psql.

Full runs go through the same function for the control and every suite file.

**Tests (executed):**
- Unit (`containedTestPath`, `path.posix` and `path.win32`):

  | | posix | win32 |
  |---|---|---|
  | Accepted | `a.test.sql`, `rls/x.test.sql`, `_selftest/c.test.sql`, `rls/../a.test.sql`, `./a.test.sql`, and an absolute path inside the directory | `a.test.sql`, `rls\x.test.sql`, `rls/x.test.sql`, `rls\..\a.test.sql`, and an absolute path inside the directory |
  | Rejected | `../../x.test.sql`, sibling `../tests2/x.test.sql`, `rls/../../x.test.sql`, `rls/../../../etc/x.test.sql`, `/etc/x.test.sql`, the parent directory's file, `..`, `.`, empty, `a.sql`, `rls`, and a NUL byte | `..\..\x.test.sql`, `../../x.test.sql`, `rls\..\..\x.test.sql`, mixed `rls/..\../x.test.sql`, `C:\Windows\x.test.sql`, `D:\x.test.sql`, drive-relative `D:x.test.sql`, UNC `\\server\share\x.test.sql`, root-relative `\x.test.sql`, and the sibling `tests2` prefix trap |

- End to end (real runner, fake psql). Each rejected path must exit **2** with "outside the tests directory", with **zero psql invocations**:
  - `../outside.test.sql`;
  - `../../outside.test.sql`;
  - nested `rls/../../outside.test.sql`;
  - the same written with the platform separator;
  - an absolute path to the outside file;
  - a junction inside the tests directory that points outside it.

  The outside file contains a mid-file `COMMIT`. Had it been read, the error would name COMMIT, so the tests also assert that the output does **not** mention COMMIT. This is evidence the file was not read, not only that it was not executed.
- The junction case **ran** on this Windows machine. It was not skipped; it is written to skip only where link creation is not permitted.
- Valid paths: `--file a.test.sql` exits 0 and runs exactly that file. A nested `rls/x.test.sql`, both relative and as an absolute path inside the directory, exits 0 with one psql call.

**Edge case `:{?name}`:** rejected (see section 1). Three new tests: `select :{?w};`, `select ok(:{?w}, 'defined');`, `select : {?w};`. The four committed suite files still pass, and that test re-reads them from disk.

### Finding 5: integration cleanup error sanitation

**Rules in `support/diagnostics.ts`:**
- `describeAuthError(error)`:
  - `null` or `undefined` → `no data returned`;
  - a non-object → `status=invalid, code=invalid`;
  - status → `none` if absent; the number if it is an integer from 100 to 599; else `invalid`;
  - code → `none` if absent; the code if it matches `^[a-z0-9_]{1,64}$`; else `invalid`;
  - the message is never read.
- `safeUserId(id)`: the id if it is a UUID, else `invalid-id`.
- `SafeDiagnosticError`: thrown by `admin.ts`, with messages built only from the parts above plus fixed text and the test's own constant labels.
- `deleteAllTestUsers(ids, deleteOne)`:
  - tries every id in order and continues after failures;
  - for a `SafeDiagnosticError`, keeps its message;
  - for anything else, uses `could not delete test user <safe id> (unexpected exception)`.
- `cleanupFailureReport(failures, total)`: null when nothing failed; otherwise a header line plus one line per failure.

**`auth.test.ts` `afterAll`:**
```ts
const report = cleanupFailureReport(await deleteAllTestUsers(created, deleteTestUser), created.length);
if (report) throw new Error(report);
```

**Tests (`test/cleanup-diagnostics.test.ts`, offline, executed, 28 pass):**
- Valid status and code are shown.
- Absent values show as `none`.
- **Malformed statuses** become `invalid`: a string, a fraction, NaN, Infinity, 0, 600, −1, and an object with a `toString`.
- **Unsafe codes** become `invalid`: an email address, a token, upper case, spaces, a newline, 65 characters, empty, and a number. A 64-character code is accepted.
- **Arbitrary thrown messages** become fixed text: an `Error` with an address, a key-like string, a token and a stack; a string; an object with a password message; `undefined`; a `TypeError`.
- **Multiple failures:** every deletion is attempted in order, and the report lists exactly both failures with sanitised ids and status/code, and no sensitive text.
- A `SafeDiagnosticError` keeps its sanitised message.
- A non-UUID id becomes `invalid-id`.
- All deletions succeeding gives no report.

---

## 4. Tests added or modified per finding

| Finding | Test file | Count now (was) | Executed |
|---|---|---|---|
| 1 | `apps/web/src/app/auth/callback/route.test.ts` | 14 (8) | yes, 14 pass |
| 2 | `apps/web/src/lib/site-origin.test.ts` | 57 (23) | yes, 57 pass |
| 3 | `apps/web/src/app/auth/actions.test.ts` | 51 (37) | yes, 51 pass |
| 3 | `apps/web/src/lib/auth/auth-outcome.test.ts` | 17 (14) | yes, 17 pass |
| 4 and edge case | `scripts/tap.test.mjs` | 157 (113) | yes, 157 pass |
| 5 | `apps/web/test/cleanup-diagnostics.test.ts` (new) | 28 | yes, 28 pass |
| 5 | `apps/web/test/integration/auth.test.ts` (cleanup wiring) | 8 | **not run: needs credentials** (type-checked only) |

---

## 5. Commands run and exit codes

All runs were from `E:\Sovereign`, after the final source edit. The one edit made after this run was to `apps/web/README.md` (Markdown, which no check reads). Logs are in `C:\Users\Gamer\AppData\Local\Temp\claude\7858b1e08a6e3505\c502d650-febf-4861-a36a-da9f61d54e28\scratchpad\m2r4\`.

| Command | Exit | Result |
|---|---|---|
| `pnpm lint` | 0 | clean |
| `pnpm run build:libs` | 0 | |
| `pnpm --filter @sovereign/core --filter @sovereign/db --filter @sovereign/worker typecheck` | 0 | |
| `apps/web: ./node_modules/.bin/tsc -p tsconfig.json --noEmit` | 0 | |
| `pnpm test` | 0 | core 5 · db 23 · worker 8 · web **231** (v3: 146) · scripts **204** (v3: 160) |
| `pnpm check:boundaries` | 0 | |
| `node --test scripts/` | 0 | 204 tests, 204 pass, 0 fail, 0 skipped |
| `pnpm test:rls` | **2** | `configuration error: SOVEREIGN_SQL_TEST_DB_URL is not set`; no connection attempted |
| `pnpm test:integration` | **1** | `Integration tests need .env.integration …`; 1 file failed, 8 tests skipped, 0 passed |

Also run during development (all passing at the end):
- `npx vitest run` on the five changed web test files, with the verbose reporter for the per-file counts above;
- `node --test scripts/tap.test.mjs`.

## 6. Tests not run (credentials or a hosted environment unavailable)

| Not run | Why | What it would prove |
|---|---|---|
| `pnpm test:rls`: control plus `identity_signup` (23), `identity_rls` (27) and `catalog` (30) | No `SOVEREIGN_SQL_TEST_DB_URL`; hosted database out of scope | Migrations, RLS, privileges and triggers on real PostgreSQL with pgTAP |
| `pnpm test:integration`: 8 tests in `auth.test.ts`, including the new cleanup wiring | No `.env.integration`; hosted Auth out of scope | Real sign-in, the verified-email gate, the recovery flow, client wiring, real cleanup |
| `resetPasswordForEmail` against real Auth | It would send a real email; no safe sink exists | The request step of password recovery |
| Browser check of the callback 400 / 503 and the auth pages | No server may be started or restarted | Rendering and real Next.js request handling (`request.nextUrl` behind the real server) |
| Copy build plus `pnpm check:bundle` | `.next` must not be rebuilt; the optional copy build was not repeated | That `SOVEREIGN_SITE_URL` and the integration helpers stay out of client bundles (static inspection only) |

No result above claims hosted database or Auth integration has passed.

---

## 7. Remaining limitations and unresolved items

1. **The callback trusts how Next.js builds `request.nextUrl`.** It is derived from the Host the server sees. Behind a reverse proxy that rewrites Host, every legitimate callback would get 400. The proxy must preserve Host, or the configured origin must be what the app sees. `localhost` and `127.0.0.1` are different origins. This is documented in the README but not verified on a running server.
2. **The callback check supersedes v3 decision 2's alternative.** "Keep the request origin in the callback" is no longer possible: a mismatched request is refused outright.
3. **`signOut` and `updatePassword` were not changed.** They are out of this finding's scope. A thrown exception there still reaches Next.js's error handling: the user sees the generic error page, with no message shown in production.
4. **The `--file` check has a time-of-check/time-of-use gap.** psql re-opens the file by path after the check. Someone able to swap files inside the tests directory between the two steps could defeat it. This is a local developer tool run on reviewed files.
5. **The SQL checker is still a lexer plus regexes, not a SQL parser.** The v3 limitations stand (ADR-0011).
6. **`SOVEREIGN_SITE_URL` is stricter than v3.** Values that v3 accepted, such as an upper-case host or `:443`, are now refused, so Auth email flows fail closed until the value is corrected.
7. **Credentials are still not configured.** All hosted evidence listed in section 6 is outstanding.

Decisions that need your approval are only those carried over from v3, unchanged:
- the new env var;
- the sign-in UX wording;
- masking every 4xx;
- the SQL checker limits;
- recursive manifest discovery.

Plus three that are new here:
- **400 for a mismatched callback origin**, rather than a redirect;
- **thrown 4xx-shaped objects treated as unavailable**;
- **the stricter canonical-form rule** for `SOVEREIGN_SITE_URL`.

---

## 8. Complete revised sources

Full current contents of all 17 files changed this round follow, in the order of the table in section 2.

### `apps/web/src/app/auth/callback/route.ts`

```ts
// Target of the links Supabase emails (sign-up confirmation, password recovery). Exchanges the one-time
// code (PKCE) or token hash for a session cookie, then continues to a validated same-site path on the
// configured canonical origin (SOVEREIGN_SITE_URL), never on an origin taken from request headers.
// A request that arrives on any other origin is refused before any exchange (400, no cookie written).
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

  // Exchange only on the canonical host. On any other host (a preview URL, a raw IP, a hostile Host header)
  // the session cookie would be written for the wrong site, so reject before creating the Auth client or
  // touching the code / token. Origins are compared after URL normalisation (scheme, host, port).
  let requestOrigin: string | null;
  try {
    requestOrigin = new URL(request.nextUrl.href).origin;
  } catch {
    requestOrigin = null;
  }
  if (requestOrigin !== origin.origin) {
    console.error('auth: callback rejected (request origin is not the configured site origin)');
    return new NextResponse('This link must be opened on the site it was sent from.', { status: 400 });
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

  it('ignores Host / X-Forwarded-* headers on a canonical-origin request', async () => {
    auth.exchangeCodeForSession.mockResolvedValue({ error: null });
    const r = await call('?code=abc&next=/workspace', {
      headers: { host: 'evil.example', 'x-forwarded-host': 'attacker.example', 'x-forwarded-proto': 'http' },
    });
    expect(r.location).toBe(`${SITE}/workspace`);
  });

  it.each([
    ['another host', 'https://evil.example'],
    ['http instead of https', 'http://sovereign.example'],
    ['another port', 'https://sovereign.example:8443'],
    ['a subdomain', 'https://www.sovereign.example'],
    ['a raw IP', 'https://203.0.113.7'],
  ])('refuses %s before any exchange and writes no cookie', async (_name, url) => {
    const errorLog = vi.spyOn(console, 'error').mockImplementation(() => undefined);
    for (const query of ['?code=abc&next=/workspace', '?token_hash=h1&type=recovery&next=/auth/update-password']) {
      const res = await GET(new NextRequest(`${url}/auth/callback${query}`));
      expect(res.status).toBe(400);
      expect(res.headers.get('location')).toBeNull();
      expect(res.headers.get('set-cookie')).toBeNull();
      expect(await res.text()).not.toMatch(/abc|h1|sovereign|evil/);
    }
    expect(auth.exchangeCodeForSession).not.toHaveBeenCalled();
    expect(auth.verifyOtp).not.toHaveBeenCalled();
    expect(errorLog.mock.calls.join(' ')).not.toMatch(/abc|h1|evil|203\.0/);
  });

  it('compares normalised origins: default port and host case still match', async () => {
    auth.exchangeCodeForSession.mockResolvedValue({ error: null });
    const res = await GET(new NextRequest('https://SOVEREIGN.example:443/auth/callback?code=abc&next=/workspace'));
    expect(res.headers.get('location')).toBe(`${SITE}/workspace`);
    expect(auth.exchangeCodeForSession).toHaveBeenCalledWith('abc');
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
// surrounding whitespace, and any value the URL parser would rewrite (dot-segments, percent-encoding,
// backslashes, upper-case hosts, an explicit default port such as :443, shortened IPv4 such as 127.1).

export type SiteOriginResult = { readonly ok: true; readonly origin: string } | { readonly ok: false; readonly reason: string };

const LOOPBACK_HOSTS = new Set(['localhost', '127.0.0.1', '[::1]']);
// scheme://host[:port][/] with a lowercase DNS name (or [::1]) and nothing else: no userinfo, path, query,
// fragment, percent-encoding or backslash.
const LABEL = '[a-z0-9](?:[a-z0-9-]{0,61}[a-z0-9])?';
const RAW_ORIGIN = new RegExp(`^https?://(?:${LABEL}(?:\\.${LABEL})*|\\[::1\\])(?::[0-9]{1,5})?/?$`);

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
  // The URL parser normalises before we can look (dot-segments such as "/..", "/foo/..", "%2e%2e";
  // backslashes; host case; default ports; shortened IPv4), so the RAW value must already be the exact
  // serialised origin, optionally followed by one "/". Anything the parser would have rewritten is rejected.
  if (!RAW_ORIGIN.test(raw) || raw.replace(/\/$/, '') !== url.origin) {
    return { ok: false, reason: 'SOVEREIGN_SITE_URL must be an origin only (no path, query or fragment), written in canonical form' };
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
    // Dot-segments and encodings that the URL parser would normalise away to "/".
    ['https://sovereign.example/..', 'origin only'],
    ['https://sovereign.example/.', 'origin only'],
    ['https://sovereign.example/foo/..', 'origin only'],
    ['https://sovereign.example/foo/../', 'origin only'],
    ['https://sovereign.example/%2e%2e', 'origin only'],
    ['https://sovereign.example/%2E%2E/', 'origin only'],
    ['https://sovereign.example/.%2e', 'origin only'],
    ['https://sovereign.example/%2e', 'origin only'],
    ['https://sovereign.example//', 'origin only'],
    ['https://sovereign.example/%20', 'origin only'],
    // Backslashes and other parser rewrites.
    ['https://sovereign.example\\', 'origin only'],
    ['https://sovereign.example\\..', 'origin only'],
    ['https:\\\\sovereign.example', 'origin only'],
    ['https:/sovereign.example', 'origin only'],
    ['https:sovereign.example', 'origin only'],
    ['https:///sovereign.example', 'origin only'],
    ['HTTPS://sovereign.example', 'origin only'],
    ['https://Sovereign.Example', 'origin only'],
    ['https://sovereign.example.', 'origin only'],
    ['https://sovereign.example:443', 'origin only'],
    ['https://sovereign.example:', 'origin only'],
    ['https://sovereign%2eexample', 'origin only'],
    ['https://xn--sovereign-.example/x', 'origin only'],
    ['https://sövereign.example', 'origin only'],
    ['http://127.1:3000', 'origin only'],
    ['http://LOCALHOST:3000', 'origin only'],
    ['http://localhost:3000/..', 'origin only'],
    ['http://[0:0:0:0:0:0:0:1]:3000', 'origin only'],
    ['https://sovereign.example\t', 'whitespace'],
    ['https://sove\nreign.example', 'origin only'],
    // Credentials, queries and fragments that hide behind a parser rewrite.
    ['https://@sovereign.example', 'origin only'],
    ['https://:@sovereign.example', 'origin only'],
    ['https://sovereign.example/..?x', 'origin only'],
    ['https://sovereign.example/..#x', 'origin only'],
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

/**
 * Runs an Auth step (client creation included) and turns a thrown exception into a fixed marker, so callers
 * treat it as 'unavailable' without ever touching the exception's message or stack.
 */
export async function attemptAuth<T>(step: () => Promise<{ error: T }>): Promise<{ error: T } | { thrown: true }> {
  try {
    return await step();
  } catch {
    return { thrown: true };
  }
}

/** Fixed log line for a thrown exception: the action only, never the exception's message or stack. */
export const authExceptionLogLine = (action: string): string => `auth: ${action} failed (unexpected exception)`;

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
import { attemptAuth, authExceptionLogLine, authFailureLogLine, classifyAuthError } from './auth-outcome';

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

describe('attemptAuth', () => {
  it('passes a returned result through unchanged', async () => {
    expect(await attemptAuth(() => Promise.resolve({ error: null }))).toEqual({ error: null });
    const error = { status: 400, code: 'invalid_credentials' };
    expect(await attemptAuth(() => Promise.resolve({ error }))).toEqual({ error });
  });

  it('turns any thrown value into a fixed marker that carries nothing from the exception', async () => {
    for (const thrown of [new Error('a@example.com token=eyJ'), 'a@example.com', { status: 400 }, undefined]) {
      const r = await attemptAuth(() => Promise.reject(thrown));
      expect(r).toEqual({ thrown: true });
      expect(JSON.stringify(r)).not.toMatch(/@|eyJ|400/);
    }
    expect(
      await attemptAuth(() => {
        throw new Error('sync');
      }),
    ).toEqual({ thrown: true });
  });

  it('has a fixed log line for exceptions', () => {
    expect(authExceptionLogLine('sign-in')).toBe('auth: sign-in failed (unexpected exception)');
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
// a token, an Auth message or an exception's message. A thrown exception (client creation included) is
// always reported as "unavailable".
import { redirect } from 'next/navigation';
import { attemptAuth, authExceptionLogLine, authFailureLogLine, classifyAuthError } from '@/lib/auth/auth-outcome';
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
  const result = await attemptAuth(async () =>
    (await createSupabaseServerClient()).auth.signUp({
      email,
      password,
      options: { emailRedirectTo: `${origin}/auth/callback?next=/workspace` },
    }),
  );
  if ('thrown' in result) {
    console.error(authExceptionLogLine('sign-up'));
    redirect('/auth/sign-up?error=unavailable');
  }
  const { error } = result;
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
  const result = await attemptAuth(async () => (await createSupabaseServerClient()).auth.signInWithPassword({ email, password }));
  if ('thrown' in result) {
    console.error(authExceptionLogLine('sign-in'));
    redirect('/auth/sign-in?error=unavailable');
  }
  const { error } = result;
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
  const result = await attemptAuth(async () =>
    (await createSupabaseServerClient()).auth.resetPasswordForEmail(email, {
      redirectTo: `${origin}/auth/callback?next=/auth/update-password`,
    }),
  );
  if ('thrown' in result) {
    console.error(authExceptionLogLine('password-reset request'));
    redirect('/auth/reset-password?error=unavailable');
  }
  const { error } = result;
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

const createClient = vi.fn(() => Promise.resolve({ auth }));
vi.mock('@/lib/supabase/server', () => ({ createSupabaseServerClient: () => createClient() }));
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
  createClient.mockReset();
  createClient.mockImplementation(() => Promise.resolve({ auth }));
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

// A secret-looking message and stack that must never reach the user or the log.
const leaky = () => {
  const e = new Error('connect failed for a@example.com token=eyJhbGciOi password=longenough');
  e.stack = 'Error: a@example.com at secret-frame (eyJhbGciOi)';
  return e;
};
const SENSITIVE = /a@example\.com|eyJ|token|password=|longenough|secret-frame|connect failed/;

describe.each([
  ['sign-up', () => signUp(form({ email: 'a@example.com', password: 'longenough' })), () => auth.signUp, '/auth/sign-up?error=unavailable'],
  ['sign-in', () => signIn(form({ email: 'a@example.com', password: 'longenough' })), () => auth.signInWithPassword, '/auth/sign-in?error=unavailable'],
  [
    'password-reset request',
    () => requestPasswordReset(form({ email: 'a@example.com' })),
    () => auth.resetPasswordForEmail,
    '/auth/reset-password?error=unavailable',
  ],
])('%s: thrown exceptions are "unavailable"', (action, run, method, unavailable) => {
  it.each([
    ['the Auth call rejects with an Error', () => method().mockRejectedValue(leaky())],
    ['the Auth call throws synchronously', () => method().mockImplementation(() => { throw leaky(); })],
    ['the Auth call rejects with a non-Error value', () => method().mockRejectedValue('a@example.com token=eyJ')],
    ['the Auth call rejects with an Auth-shaped 4xx object', () => method().mockRejectedValue({ status: 400, code: 'invalid_credentials', message: 'a@example.com' })],
    ['client creation fails', () => createClient.mockRejectedValue(leaky())],
  ])('when %s', async (_name, arrange) => {
    arrange();
    const url = await redirectOf(run());
    expect(url).toBe(unavailable);
    expect(url).not.toMatch(SENSITIVE);
    expect(logged()).toBe(`auth: ${action} failed (unexpected exception)`);
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

### `scripts/tap.mjs`

```js
// Strict TAP evaluation for the psql-based pgTAP runner (ADR-0011). Pure functions, no I/O, so every
// failure rule is unit-tested offline (scripts/tap.test.mjs). A file passes only when ALL of these hold:
//   - psql exited 0, did not time out, and wrote nothing to stderr;
//   - the first meaningful line is exactly one plan "1..N" with N >= 1;
//   - exactly N result lines follow, numbered 1..N in order, each "ok K" or "not ok K" (optional " - text");
//   - no result carries a SKIP or TODO directive, and pgTAP reported no "Looks like ..." summary;
//   - every other line is a "#" diagnostic; anything else is unexpected output.

import path from 'node:path';

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

/**
 * Resolves a --file / manifest path and proves it stays inside the tests directory, before anything reads it.
 * Pure path arithmetic (no file access); `p` is node:path by default, or path.win32 / path.posix in tests.
 * The runner repeats the check on real paths (symlinks, junctions) once the file is known to exist.
 * @returns {{ ok: true, abs: string } | { ok: false, reason: string }}
 */
export function containedTestPath(testsDir, rel, p = path) {
  if (typeof rel !== 'string' || rel === '' || rel.includes('\0')) return { ok: false, reason: 'test path is empty or invalid' };
  if (!rel.endsWith('.test.sql')) return { ok: false, reason: 'test path must name a *.test.sql file' };
  const root = p.resolve(testsDir);
  const abs = p.resolve(root, rel);
  const inside = p.relative(root, abs);
  if (inside === '' || inside === '..' || inside.startsWith(`..${p.sep}`) || p.isAbsolute(inside)) {
    return { ok: false, reason: 'test path is outside the tests directory' };
  }
  return { ok: true, abs };
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
  // Unquoted :name, identifier-quoted :"name", and the :{?name} existence test are all rejected: only :'name'
  // (always a quoted literal) is allowed. :{?name} expands to TRUE/FALSE and is harmless today, but the
  // policy is one interpolation form, so it is not an exception.
  if (/(?<!:):(?![:=])\s*"/.test(code) || /(?<![:\w]):(?![:=])[A-Za-z_]/.test(code) || /(?<!:):\s*\{/.test(code)) {
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

### `scripts/run-sql-tests.mjs`

```js
// pgTAP runner over psql (ADR-0011). No dependencies; runs on plain Node.js.
//
//   pnpm test:rls                                  control first, then the whole suite
//   node scripts/run-sql-tests.mjs --file <rel> --json    one file inside the tests directory; prints a JSON verdict line
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
import { existsSync, readFileSync, readdirSync, realpathSync } from 'node:fs';
import { dirname, join, relative, resolve, sep } from 'node:path';
import { fileURLToPath } from 'node:url';
import { checkManifest, checkSqlTestFile, containedTestPath, controlConfirmed, evaluateRun } from './tap.mjs';

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

/** Reads a test file only after proving it lies inside TESTS_DIR, lexically and then on real paths. */
function readTestFile(rel) {
  const contained = containedTestPath(TESTS_DIR, rel);
  if (!contained.ok) throw new ConfigError(`${contained.reason}; refusing to read it`);
  const { abs } = contained;
  if (!existsSync(abs)) throw new ConfigError(`test file not found: ${rel}`);
  // A symlink or junction inside the directory must not lead outside it either.
  const real = containedTestPath(realpathSync(TESTS_DIR), relative(realpathSync(TESTS_DIR), realpathSync(abs)));
  if (!real.ok) throw new ConfigError(`${real.reason} (after resolving links); refusing to read it`);
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

### `scripts/tap.test.mjs`

```js
// Offline tests for the pgTAP runner (ADR-0011): parser rules, control logic, static/manifest checks, and the
// runner end to end against a fake psql (exit codes, control-then-suite flow, secret handling).
import assert from 'node:assert/strict';
import { spawnSync } from 'node:child_process';
import { mkdtempSync, mkdirSync, readFileSync, rmSync, symlinkSync, writeFileSync } from 'node:fs';
import { tmpdir } from 'node:os';
import path, { dirname, join, sep } from 'node:path';
import { describe, it } from 'node:test';
import { fileURLToPath } from 'node:url';
import { checkManifest, checkSqlTestFile, containedTestPath, controlConfirmed, evaluateRun, parseTap } from './tap.mjs';

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
    for (const stmt of ['select :w;', 'select * from :t;', 'select :"w";', ':cmd;', 'select :{?w};', "select ok(:{?w}, 'defined');", 'select : {?w};']) {
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

describe('containedTestPath', () => {
  for (const [name, p, dir, cases] of [
    [
      'posix',
      path.posix,
      '/repo/packages/db/supabase/tests',
      {
        ok: ['a.test.sql', 'rls/x.test.sql', '_selftest/c.test.sql', 'rls/../a.test.sql', './a.test.sql', '/repo/packages/db/supabase/tests/rls/x.test.sql'],
        bad: ['../../x.test.sql', '../tests2/x.test.sql', 'rls/../../x.test.sql', 'rls/../../../etc/x.test.sql', '/etc/x.test.sql', '/repo/packages/db/supabase/x.test.sql', '..', '.', '', 'a.sql', 'rls', 'a.test.sql\0'],
      },
    ],
    [
      'win32',
      path.win32,
      'C:\\repo\\packages\\db\\supabase\\tests',
      {
        ok: ['a.test.sql', 'rls\\x.test.sql', 'rls/x.test.sql', 'rls\\..\\a.test.sql', 'C:\\repo\\packages\\db\\supabase\\tests\\rls\\x.test.sql'],
        bad: ['..\\..\\x.test.sql', '../../x.test.sql', 'rls\\..\\..\\x.test.sql', 'rls/..\\../x.test.sql', 'C:\\Windows\\x.test.sql', 'D:\\x.test.sql', 'D:x.test.sql', '\\\\server\\share\\x.test.sql', '\\x.test.sql', 'C:\\repo\\packages\\db\\supabase\\tests2\\x.test.sql'],
      },
    ],
  ]) {
    describe(name, () => {
      for (const rel of cases.ok) {
        it(`accepts ${JSON.stringify(rel)}`, () => {
          const r = containedTestPath(dir, rel, p);
          assert.equal(r.ok, true);
          assert.ok(r.abs.startsWith(dir + p.sep));
        });
      }
      for (const rel of cases.bad) {
        it(`rejects ${JSON.stringify(rel)}`, () => assert.equal(containedTestPath(dir, rel, p).ok, false));
      }
    });
  }
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

/**
 * Builds a throwaway tests directory (<base>/tests) and runs the runner against the fake psql.
 * `outside` files go in <base>, next to (not inside) the tests directory; `args` may be a function of
 * (testsDir, base); `setup(testsDir, base)` runs before the runner (e.g. to create a link).
 */
function runRunner(map, { args = [], url = `postgresql://tester:${SECRET}@db.example.test:5432/postgres`, extra = {}, outside = {}, setup } = {}) {
  const base = mkdtempSync(join(tmpdir(), 'sovereign-sqltest-'));
  const dir = join(base, 'tests');
  try {
    mkdirSync(join(dir, '_selftest'), { recursive: true });
    for (const [rel, body] of Object.entries(outside)) writeFileSync(join(base, rel), body);
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
    setup?.(dir, base);
    const argv = typeof args === 'function' ? args(dir, base) : args;
    const r = spawnSync(process.execPath, [RUNNER, ...argv], { env, encoding: 'utf8' });
    let calls = [];
    try {
      calls = readFileSync(log, 'utf8').trim().split('\n').filter(Boolean).map((l) => JSON.parse(l));
    } catch {
      calls = [];
    }
    return { code: r.status, out: `${r.stdout}${r.stderr}`, calls };
  } finally {
    rmSync(base, { recursive: true, force: true });
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

describe('runner --file containment (rejected paths are never read or executed)', () => {
  // The outside file is deliberately invalid (COMMIT mid-file): if the runner ever read it, the error would
  // name COMMIT instead of containment, and if it ever ran it, the fake psql would log a call.
  const OUTSIDE = SQL.replace('select ok(true);', 'commit;\nselect ok(true);');
  const refused = (r) => {
    assert.equal(r.code, 2);
    assert.equal(r.calls.length, 0);
    assert.match(r.out, /outside the tests directory/);
    assert.doesNotMatch(r.out, /COMMIT/);
  };

  it('rejects ../ traversal', () => refused(runRunner({ 'outside.test.sql': PASS }, { outside: { 'outside.test.sql': OUTSIDE }, args: ['--file', '../outside.test.sql'] })));
  it('rejects ../../ traversal', () => refused(runRunner({ 'outside.test.sql': PASS }, { args: ['--file', '../../outside.test.sql'] })));
  it('rejects nested traversal', () => {
    refused(runRunner({ 'outside.test.sql': PASS }, { outside: { 'outside.test.sql': OUTSIDE }, extra: { 'rls/x.test.sql': SQL }, args: ['--file', 'rls/../../outside.test.sql'] }));
  });
  it('rejects nested traversal written with the platform separator', () => {
    refused(runRunner({ 'outside.test.sql': PASS }, { outside: { 'outside.test.sql': OUTSIDE }, args: ['--file', ['rls', '..', '..', 'outside.test.sql'].join(sep)] }));
  });
  it('rejects an absolute path outside the tests directory', () => {
    refused(runRunner({ 'outside.test.sql': PASS }, { outside: { 'outside.test.sql': OUTSIDE }, args: (_dir, base) => ['--file', join(base, 'outside.test.sql')] }));
  });
  it('rejects a link inside the directory that leads outside it', (t) => {
    let linked = true;
    const r = runRunner(
      { 'outside.test.sql': PASS },
      {
        setup: (dir, base) => {
          mkdirSync(join(base, 'elsewhere'));
          writeFileSync(join(base, 'elsewhere', 'outside.test.sql'), OUTSIDE);
          try {
            symlinkSync(join(base, 'elsewhere'), join(dir, 'linked'), 'junction');
          } catch {
            linked = false;
          }
        },
        args: ['--file', 'linked/outside.test.sql'],
      },
    );
    if (!linked) {
      t.skip('this platform does not allow creating the link');
      return;
    }
    refused(r);
  });
  it('runs a valid file inside the directory', () => {
    const r = runRunner({ 'a.test.sql': PASS }, { args: ['--file', 'a.test.sql'] });
    assert.equal(r.code, 0);
    assert.deepEqual(r.calls.map((c) => c.file), ['a.test.sql']);
  });
  it('runs a valid nested file, also when given as an absolute path inside the directory', () => {
    const nested = runRunner({ 'x.test.sql': PASS }, { extra: { 'rls/x.test.sql': SQL }, args: ['--file', 'rls/x.test.sql'] });
    assert.equal(nested.code, 0);
    assert.equal(nested.calls.length, 1);
    const absolute = runRunner({ 'x.test.sql': PASS }, { extra: { 'rls/x.test.sql': SQL }, args: (dir) => ['--file', join(dir, 'rls', 'x.test.sql')] });
    assert.equal(absolute.code, 0);
    assert.equal(absolute.calls.length, 1);
  });
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

### `apps/web/test/integration/support/diagnostics.ts`

```ts
// Sanitised diagnostics for the integration-test helpers and their cleanup. Pure, no Supabase import, so it
// is unit-tested offline (test/cleanup-diagnostics.test.ts). Only these values ever reach a report:
//   - a user id, and only when it is a UUID (ids of throwaway test users);
//   - an HTTP status, and only when it is an integer from 100 to 599;
//   - an Auth error code, and only when it matches ^[a-z0-9_]{1,64}$;
//   - fixed text written here.
// Never an exception's message or stack, an email address, a password, a key, a token or a raw response.

const UUID = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
const CODE = /^[a-z0-9_]{1,64}$/;

/** The id when it is a UUID, otherwise a fixed placeholder. */
export const safeUserId = (id: unknown): string => (typeof id === 'string' && UUID.test(id) ? id : 'invalid-id');

/** Status and code of an Auth error, each validated; never its message. */
export function describeAuthError(error: unknown): string {
  if (error === null || error === undefined) return 'no data returned';
  if (typeof error !== 'object') return 'status=invalid, code=invalid';
  const { status, code } = error as { status?: unknown; code?: unknown };
  const s = status === undefined ? 'none' : typeof status === 'number' && Number.isInteger(status) && status >= 100 && status <= 599 ? String(status) : 'invalid';
  const c = code === undefined ? 'none' : typeof code === 'string' && CODE.test(code) ? code : 'invalid';
  return `status=${s}, code=${c}`;
}

/** An error whose message was built only from the sanitised parts above, so it may be reported as is. */
export class SafeDiagnosticError extends Error {
  override readonly name = 'SafeDiagnosticError';
}

/** The diagnostic for a failed step: a SafeDiagnosticError's message, or fixed text for anything else. */
export const diagnosticFor = (e: unknown, fallback: string): string => (e instanceof SafeDiagnosticError ? e.message : fallback);

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
import { SafeDiagnosticError, describeAuthError, safeUserId } from './diagnostics';
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
  if (error || !data.user) throw new SafeDiagnosticError(`could not create test user ${label} (${describeAuthError(error)})`);
  return { id: data.user.id, email };
}

export async function confirmTestUser(id: string): Promise<void> {
  const { error } = await adminClient().auth.admin.updateUserById(id, { email_confirm: true });
  if (error) throw new SafeDiagnosticError(`could not confirm test user ${safeUserId(id)} (${describeAuthError(error)})`);
}

/**
 * Creates a password-recovery token for a test user WITHOUT sending any email: the Auth admin API's
 * generateLink returns the token for the caller to deliver ("to be sent via a custom email provider").
 * This exercises the recovery flow; it does not exercise the app's own resetPasswordForEmail() request.
 */
export async function generateRecoveryToken(email: string, redirectTo: string): Promise<{ tokenHash: string; type: string }> {
  const { data, error } = await adminClient().auth.admin.generateLink({ type: 'recovery', email, options: { redirectTo } });
  if (error || !data.properties) throw new SafeDiagnosticError(`could not generate a recovery link (${describeAuthError(error)})`);
  return { tokenHash: data.properties.hashed_token, type: data.properties.verification_type };
}

export async function deleteTestUser(id: string): Promise<void> {
  const { error } = await adminClient().auth.admin.deleteUser(id);
  if (error) throw new SafeDiagnosticError(`could not delete test user ${safeUserId(id)} (${describeAuthError(error)})`);
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

### `apps/web/test/cleanup-diagnostics.test.ts`

```ts
// Offline tests for the integration suite's sanitised diagnostics (test/integration/support/diagnostics.ts).
// No Supabase client is involved; deletions are test doubles.
import { describe, expect, it, vi } from 'vitest';
import {
  SafeDiagnosticError,
  cleanupFailureReport,
  deleteAllTestUsers,
  describeAuthError,
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

  it('keeps the sanitised diagnostic of a SafeDiagnosticError', async () => {
    const failures = await deleteAllTestUsers([ID1], () =>
      Promise.reject(new SafeDiagnosticError(`could not delete test user ${ID1} (${describeAuthError({ status: 500, code: 'unexpected_failure' })})`)),
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
        ? Promise.reject(new SafeDiagnosticError(`could not delete test user ${ID1} (${describeAuthError({ status: 'oops', code: 'a@b' })})`))
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
   - **Static file check** (`checkSqlTestFile` in `scripts/tap.mjs`), run before any connection: a small lexer masks comments, quoted strings, dollar-quoted strings and quoted identifiers, then rejects transaction control between `BEGIN` and the final `ROLLBACK` (COMMIT, END, ABORT, ROLLBACK, BEGIN, START TRANSACTION, SAVEPOINT, RELEASE, PREPARE TRANSACTION), COPY, every psql meta-command except `\gset`, every psql variable form except the quoted literal :'name' (unquoted :name, :"name" and the :{?name} existence test are all rejected), and, anywhere including strings, dblink, pg_background, COMMIT/ROLLBACK PREPARED, PREPARE TRANSACTION and COPY. **Limitations:** it is not a SQL parser. It cannot see SQL built at run time (for example `EXECUTE format(...)` inside a function, or a function defined elsewhere that commits); it relies on PostgreSQL refusing COMMIT/ROLLBACK inside functions, DO blocks and procedures called within an explicit transaction block, and on `ON_ERROR_STOP=1` plus disconnect-rollback when a statement fails; and its string-wide bans can reject harmless text (an assertion description containing the word "copy"). Files are reviewed code, not untrusted input: the check guards against mistakes, not a malicious author. The single-file mode (--file) accepts only a *.test.sql path that resolves inside the tests directory, checked lexically and again after resolving symlinks and junctions, before the file is read.
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

**Site origin:** `SOVEREIGN_SITE_URL` (server-only, in `apps/web/.env.local`) is the canonical origin used for links in Auth emails and for the `/auth/callback` redirects; request headers (Host, X-Forwarded-*) are never used for them. It must be an origin only, https, or http for localhost / 127.0.0.1 / [::1] in local development, e.g. `SOVEREIGN_SITE_URL=http://localhost:3000`. When it is unset or invalid, sign-up and password reset report "temporarily unavailable" and the callback returns 503. It must be written exactly as the origin (lower-case host, no default port, no path; one trailing slash is allowed). `/auth/callback` refuses (400, before any code or token exchange, no cookie) a request whose own origin differs from it, so open email links on that exact origin (`localhost` and `127.0.0.1` are different origins) and make sure a reverse proxy preserves the Host the app sees. See `.env.example`.

**Allowed internal dependencies:** `@sovereign/core`, `@sovereign/db` (adapters and clients, used only in the composition root).

**Server-only convention:** `src/server/` (including the composition root) is server code. Until the `server-only` package arrives at M2, nothing enforces this at build time, so client components must not import from `src/server/`.

**Status (M0.4):** minimal App Router scaffold (`src/app/layout.tsx`, `src/app/page.tsx`) and `GET /api/health`, which checks configuration only and returns pass/fail per check, never reasons or configuration (docs/ARCHITECTURE.md §H.1). `next build` passes on Node 26.13.0 with Next.js 16.4.5. No design system or product UI yet (M1).
```
