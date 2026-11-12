# M2 — Accountless visitor identity and personal workspace: architecture specification

Status: **specification only. Not approved for implementation.** No code, migration or test described here exists yet.
Revision: **1** (2026-11-12). Supersedes the account-based M2 design (Supabase Auth sign-up/sign-in, verified email, `auth.users`-keyed identity) for M2 only.
Decision register: §11. Everything marked **Proposed** there is **not** approved.

Change rule (docs/README.md): before implementation, this specification needs (1) a new ADR in `docs/DECISIONS/` that supersedes the affected parts of ADR-0001, ADR-0005, ADR-0006 and ADR-0011, (2) updates to the living documents listed in §12, and (3) a ROADMAP status update. None of those has been made yet; this document does not edit them.

---

## 1. Goal and principles

Sovereign is **accountless**. A visitor opens Sovereign and uses it immediately. There is no sign-up, sign-in, password, email confirmation or password recovery anywhere in the product.

Principles:

1. **No hidden accounts.** The design must not be an account system behind a login-free UI. Supabase Auth is **not** the foundation of visitor identity: no `auth.users` rows, no `auth.uid()`, no Supabase Auth sessions, no anonymous Supabase sign-ins.
2. **The browser holds a credential, nothing else.** A random, opaque session token in an HttpOnly cookie is the only proof of workspace ownership. The visible username is never a credential.
3. **One backend, one door.** Browser code never talks to the database or to Supabase APIs for visitor data. Every read and write goes through the Sovereign server (Next.js server code), which establishes the visitor context.
4. **Least privilege, honestly described.** The server connects with a dedicated database role without `BYPASSRLS`. RLS is defence in depth against application bugs; it is **not** described as protection against a compromised server (§6.7).
5. **Fail closed.** A missing, malformed, expired or revoked session yields "no visitor", never another visitor's data.
6. **Accepted limitation, stated plainly.** Losing the cookie means losing access to the workspace. There is no recovery.

Out of scope for M2: cross-device recovery or transfer, email of any kind, operator/admin identity, public profiles, Realtime, agent runs and budgets, teams. §10 lists the consequences for later milestones.

---

## 2. Product behaviour

### 2.1 First visit
- Public pages (landing, Realm, Orders and the other `(realm)` routes) need no visitor identity and create none.
- When the visitor enters the application area (M2: `/workspace`), the app shell detects that there is no valid session and provisions a visitor **automatically**, without any form or click:
  - A small client component on the app shell sends one same-origin `POST` (Server Action) to provision.
  - The server creates the visitor, the personal workspace and the session atomically (§7.1), sets the session cookie, and returns the workspace view.
- **GET requests never create visitors.** Crawlers and link previews that do not run JavaScript create nothing (§5.10).
- The visitor sees their generated username and their personal workspace immediately.

### 2.2 Returning visit
- The browser sends the session cookie. The server validates it (§5.6), resolves the visitor and renders the workspace.
- Persistence across browser restarts on the same browser profile and device is provided by a **persistent** cookie (`Max-Age`, §5.4), not a session-only cookie.

### 2.3 Username
- Every visitor has exactly one visible username. It is their personal identity in the UI, nothing more.
- On provisioning, the server generates an initial username (§4.2). The visitor can edit it later (§7.3).
- The username is **never** used to look up, authorise or prove ownership of anything. Knowing someone's username grants nothing.
- Exact format, uniqueness and editing UX are **proposed decisions** D3–D5 (§11).

### 2.4 Personal workspace
- Each visitor has exactly one personal workspace, created together with the visitor. Ownership is the database relationship `workspaces.visitor_id`, enforced one-to-one (§6.3).
- M2 workspace content: the workspace's name and identity only. Domain data (agents, quests, deliverables) arrives in later milestones with its own tables (§6.1).

### 2.5 Browser persistence and its limits
| Event | Result |
|---|---|
| Reload, new tab, browser restart (same profile) | Same visitor and workspace. |
| Private/incognito window | A separate visitor; it is lost when the private session ends. |
| Another browser, profile or device | A different visitor. **Cross-device recovery is out of scope.** |
| Visitor clears cookies or site data | The old workspace becomes inaccessible. On the next visit to the app area a new visitor is offered (§2.6). The old data stays in the database until retention cleanup (D11). |
| Session expires after inactivity (§5.4) | Same as clearing cookies: the old workspace becomes inaccessible. |
| Session revoked (§5.7) | Same: inaccessible from that browser. |
| Cookie stolen | The thief can use the workspace as the visitor until the session is rotated away, revoked or expires (§5.9). |

### 2.6 Invalid, expired or revoked cookie
- The server never silently replaces a cookie it received but could not validate.
- If a cookie **was sent** but is invalid (unknown, expired, revoked, malformed), the app area shows a clear notice ("This browser's previous Sovereign workspace can no longer be opened") and an explicit "Start a new workspace" action. Only that action provisions a new visitor and overwrites the cookie. (D7.)
- If **no** cookie was sent, provisioning is automatic (§2.1).
- Reason: with `SameSite=Lax` (§5.3) cookies are sent on same-site requests and top-level navigations, so "no cookie" really means "none stored". Treating an invalid cookie as "start fresh" automatically would hide a real loss from the visitor.

### 2.7 Out of scope (explicit)
Cross-device recovery, account linking, export/import of an identity, email, passwords, magic links, OAuth, recovery codes. Any of these would be a new product decision.

---

## 3. Identity concepts

| Concept | What it is | Where it lives | Secret? | Used for |
|---|---|---|---|---|
| **Username** | Human-readable display identity, e.g. `quiet-heron-41` | `sovereign.visitors.username` | No (visible) | Display only |
| **Visitor id** | `uuid`, server-generated (`gen_random_uuid()`) | `sovereign.visitors.id` | Not a credential; never trusted from the client | Internal foreign key; never accepted as input for authorisation |
| **Session token** | 256-bit random bearer token | Browser cookie only (plaintext); **never stored in plaintext server-side** | **Yes** | Proof of possession of the visitor's session |
| **Session token hash** | SHA-256 of the token (32 bytes) | `sovereign.visitor_sessions.token_hash` | Sensitive (lookup key) | Server-side session lookup and the database visitor context |

Rules:
- No request parameter, form field or header carries a visitor id or workspace id that the server trusts for authorisation. The server derives both from the session.
- Visitor ids may appear in server logs for operations; they must not appear in URLs or client bundles in M2.

---

## 4. Visitor identity model

### 4.1 Visitor id
Generated by PostgreSQL (`gen_random_uuid()`, v4, cryptographically random) inside the provisioning function. The client never proposes it.

### 4.2 Initial username (Proposed, D3)
- Generated server-side from a fixed word list plus a random number, e.g. `adjective-noun-NN`, using `node:crypto` randomness.
- Proposed format: 3–24 characters, lowercase `a–z`, digits, `-` and `_`, starting with a letter or digit; enforced by a database `CHECK` and validated with Zod in the server before the database call.
- Proposed uniqueness (D4): case-insensitive unique (lowercase-only format plus `UNIQUE`), so a later public profile (`/u/[username]`, PRD) is unambiguous. On a uniqueness collision during provisioning the server retries with a new generated name, at most 5 times, then fails with a generic error.
- Reserved names (D5): a small denylist (e.g. `admin`, `sovereign`, `support`, `system`, Order and goddess names) checked in the server.

### 4.3 What the visitor id is not
- It is not derived from the IP, user agent, fingerprint or username.
- It is not exposed to the browser in M2.

---

## 5. Visitor session security

### 5.1 Token
- 32 bytes from `node:crypto` (`randomBytes(32)` or `crypto.getRandomValues`), encoded base64url (43 characters, no padding). 256 bits of entropy.
- Server-side, only `sha256(token)` is stored and compared. A fast hash is appropriate because the token is high-entropy random data, not a password.
- The raw token never reaches the database, logs, analytics or error reports. The database visitor context uses the hash (§6.5).

### 5.2 Cookie
| Attribute | Value | Why |
|---|---|---|
| Name | Production: `__Host-sovereign_session`. Local development over `http://localhost`: `sovereign_session` (D9) | The `__Host-` prefix forces `Secure`, `Path=/` and no `Domain`, so subdomains cannot set or shadow it. |
| `HttpOnly` | Always | Not readable by page scripts (limits XSS token theft; does not stop XSS acting as the visitor). |
| `Secure` | Always in production; off only for `http://localhost` development | Never sent over plain HTTP in production. |
| `SameSite` | `Lax` | See §5.3. |
| `Path` | `/` | Required by `__Host-`; the cookie is only meaningful to the app area but the prefix forbids narrower paths. |
| `Domain` | Not set | Host-only cookie. |
| `Max-Age` | Sliding, = remaining session lifetime (§5.4) | Persistent across browser restarts. |

### 5.3 SameSite choice
- `Lax`: the cookie is sent on same-site requests and on top-level cross-site **GET** navigations, and withheld on cross-site POST, iframe and subresource requests.
- `Strict` was rejected: a visitor arriving from an external link would arrive without the cookie, see "no visitor", and could be pushed into a new visitor, which looks like data loss.
- `SameSite` is **not** the CSRF defence by itself (§5.8).

### 5.4 Expiration and renewal (Proposed, D8)
- **Idle lifetime: 180 days**, sliding. Each validated use may extend `expires_at` to `now() + 180 days`.
- Renewal is throttled: the database row and the cookie are only extended when the last extension is older than 24 hours, so not every request writes.
- No absolute maximum lifetime in M2: an absolute cap would destroy workspaces of active visitors, because there is no recovery. (Browsers may cap cookie lifetime, e.g. Chrome at 400 days; renewal keeps active visitors well inside that.)
- Inactivity beyond the idle lifetime means permanent loss of access (§2.5). This is a product decision (D8).

### 5.5 Rotation (Proposed, D10)
- The token is rotated when the session is older than **30 days** since its last rotation, on the next validated request that can set cookies (Server Action, Route Handler or proxy).
- Rotation is atomic in the database (§7.5): the new session row is inserted and the old one is marked rotated with a **2-minute grace period** (`rotated_at = now()`, `replaced_by = new id`, `expires_at = least(expires_at, now() + 2 minutes)`), so parallel requests still carrying the old cookie do not fail.
- A concurrent second rotation of the same old session finds `replaced_by` already set and does not rotate again; that request keeps using the old token within the grace period.
- Rotation limits how long a **copied** token stays valid if the copy is not used. It does not stop an attacker who uses the stolen token first (§5.9).
- Reuse detection (an old, rotated token presented after the grace period): logged, request treated as "no valid session". Automatically revoking the whole visitor on reuse is **not** proposed, because without recovery it would destroy the legitimate visitor's access too (D10).

### 5.6 Server-side validation
On every request that needs a visitor:
1. Read the cookie. Absent → "no visitor". Present but not 43 base64url characters → "invalid session" (no database call).
2. Hash it and call `private.touch_session(hash)` (§7.2). The function returns the visitor id only for a session that exists, is not revoked and has not expired; otherwise null.
3. Run the request's data access inside a transaction with the visitor context set from the hash (§6.5).
4. Server Components never trust a cached result across requests.
Validation lives in one `server-only` module (§8.2). Pages and actions call it; the proxy may call it for renewal but is not the authorisation boundary.

### 5.7 Revocation
- **Per browser:** a "Forget this browser" action in the app revokes the current session (`revoked_at = now()`) and deletes the cookie. The visitor is warned first that this ends access to the workspace permanently. (Proposed; exact UX in D7.)
- **Operator:** revoking a session or a visitor is an operator/database task in M2; there is no admin UI.
- **Expiry cleanup:** expired and revoked session rows are deleted by a retention job (D11). Their workspaces are not deleted by that job.

### 5.8 CSRF and Origin validation
State-changing requests are Server Actions and Route Handlers using `POST` only. Each one passes **all** of these checks before any work:
1. **Origin is required and must equal the canonical origin** (`SOVEREIGN_SITE_URL`, already validated by `apps/web/src/lib/site-origin.ts`). A **missing** `Origin` header is rejected. (Next.js 16's built-in Server Action check compares `Origin` with `Host`/`X-Forwarded-Host` and lets a request **without** `Origin` through with only a warning; it is useful but not sufficient, and it trusts forwarded headers.)
2. **`Sec-Fetch-Site`**, when present, must be `same-origin`.
3. The session cookie is `SameSite=Lax` (cross-site POSTs carry no cookie).
4. **Provisioning** additionally refuses to run if a session cookie is present (it is not a way to replace a cookie; §2.6).
A synchronizer CSRF token (per-session HMAC rendered into forms) is **Proposed** as an additional layer (D12); M2 does not depend on it.

### 5.9 Theft and loss: what they mean
- **Theft:** the cookie is a bearer credential. Whoever presents it **is** the visitor for that workspace: they can read and change everything the visitor can. HttpOnly prevents script reading it, but XSS can still act within the page, and malware or a copied browser profile can take the cookie. There is no second factor and no way for the real visitor to "log the thief out" from another device. Defences are limited to: rotation (§5.5), `Secure`, HttpOnly, `__Host-`, a strict CSP (SECURITY §I.7) and short-lived grace windows.
- **Loss:** clearing cookies, losing the profile or device, or long inactivity loses access permanently. This is accepted (product requirement).

### 5.10 Rate limits and abuse controls (Proposed, D13)
Anonymous provisioning is a resource-exhaustion target. Proposed controls:
- Provisioning only via same-origin `POST` from the app shell (no GET side effects; non-JS crawlers create nothing).
- Per-client rate limit on provisioning: e.g. 10 per hour and 50 per day per client key, where the client key is an HMAC (server secret) of the client IP as seen behind a **trusted** proxy configuration (D14). Exceeding the limit returns a generic "try again later".
- Global provisioning circuit breaker: e.g. a configurable maximum of new visitors per hour, stopping provisioning when exceeded.
- Per-visitor write limits on username changes (e.g. 10 per day).
- Storage: a small Postgres table in a non-exposed schema (fixed-window counters keyed by the HMAC; no raw IPs stored). Its schema is defined only when D13 is approved; it is not part of the minimum identity schema in §6.
- Cleanup of abandoned visitors (D11) bounds long-term growth.
These do not replace the cost controls needed before agent runs exist (§10).

---

## 6. Database model and Supabase security

### 6.1 Placement
- Supabase PostgreSQL remains the hosted database.
- Sovereign identity tables live in a **non-exposed schema `sovereign`** (D2). It is never added to the Data API's exposed schemas, and `anon`/`authenticated` get no privileges on it. Reason: the browser no longer uses the Data API for visitor data, so nothing should be reachable that way even if a publishable key leaks.
- Helper functions live in the non-exposed schema `private` (as today).
- No `profiles`, no `workspace_members`, no `workspace_data` JSONB placeholder. Domain tables are added when their requirements exist, each with `workspace_id` and the same RLS pattern.

### 6.2 Roles
| Role | Attributes | Purpose |
|---|---|---|
| `postgres` | Supabase-managed owner | Runs migrations; owns tables and SECURITY DEFINER functions (D15). |
| `sovereign_app` | `LOGIN NOSUPERUSER NOBYPASSRLS NOCREATEDB NOCREATEROLE NOREPLICATION NOINHERIT`, `CONNECTION LIMIT` (D16) | The only role the web server uses. Password set **out of band** by the operator (never in a migration or in the repository). |
| `anon`, `authenticated` | Supabase-managed | No privileges on `sovereign` or on the identity functions. |
| `service_role` | Supabase-managed, bypasses RLS | **Not used** by application runtime. No routine use. |

The existing integration-test use of the service-role key (ADR-0011) becomes unnecessary: tests create visitors through `private.provision_visitor` as `sovereign_app` and clean up with a dedicated test-cleanup path (§9.6).

### 6.3 Tables (minimum viable schema)

```sql
-- Illustrative DDL for review. Not a migration.
create schema sovereign;

create table sovereign.visitors (
  id             uuid primary key default gen_random_uuid(),
  username       text not null unique                                     -- D4
                 check (username ~ '^[a-z0-9][a-z0-9_-]{2,23}$'),          -- D3
  created_at     timestamptz not null default now(),
  updated_at     timestamptz not null default now(),
  last_seen_at   timestamptz not null default now()
);

create table sovereign.workspaces (
  id          uuid primary key default gen_random_uuid(),
  visitor_id  uuid not null unique                                         -- one workspace per visitor
              references sovereign.visitors (id) on delete cascade,
  name        text not null default 'Personal workspace'
              check (char_length(name) between 1 and 80),
  created_at  timestamptz not null default now(),
  updated_at  timestamptz not null default now()
);

create table sovereign.visitor_sessions (
  id             uuid primary key default gen_random_uuid(),
  visitor_id     uuid not null references sovereign.visitors (id) on delete cascade,
  token_hash     bytea not null unique check (octet_length(token_hash) = 32),
  created_at     timestamptz not null default now(),
  last_used_at   timestamptz not null default now(),
  expires_at     timestamptz not null,
  revoked_at     timestamptz,
  revoke_reason  text check (revoke_reason in ('forgotten', 'operator')),
  rotated_at     timestamptz,
  replaced_by    uuid references sovereign.visitor_sessions (id) on delete set null,
  check (expires_at > created_at),
  check ((revoked_at is null) = (revoke_reason is null)),
  check ((rotated_at is null) = (replaced_by is null))
);
create index visitor_sessions_visitor_id_idx on sovereign.visitor_sessions (visitor_id);
create index visitor_sessions_expires_at_idx on sovereign.visitor_sessions (expires_at);
```

Notes:
- **One-to-one visitor ↔ workspace:** `workspaces.visitor_id` is `NOT NULL UNIQUE` (at most one workspace per visitor, never an unowned workspace). "At least one" is guaranteed by the only insert path, `private.provision_visitor`, which creates both in one transaction; `sovereign_app` has no `INSERT` on either table. A deferrable constraint trigger enforcing "every visitor has a workspace at commit" is a possible addition (D17).
- **Deletion:** deleting a visitor cascades to its workspace and sessions. Domain tables added later reference `workspaces(id) on delete cascade`.
- **Timestamps:** `updated_at` maintained by a `BEFORE UPDATE` trigger (`clock_timestamp()`), as in the current `private.set_profile_timestamps()` pattern.
- **Rotated vs revoked sessions** are separate states. Rotation sets `rotated_at`, `replaced_by` and shortens `expires_at` to the grace end; the row stays valid until then. Revocation sets `revoked_at` and `revoke_reason` together and ends validity immediately. The two `CHECK`s keep each pair consistent. A rotated token presented after its grace period is recognisable (`rotated_at` set, expired) and is logged as reuse (§5.5).
- **Username uniqueness:** the format `CHECK` allows only lowercase, so a plain `UNIQUE` is case-insensitive by construction.
- No slug, no `kind`, no membership table: they existed for teams (ADR-0006), which are out of scope. Re-adding teams later is a schema migration, not an RLS-only change (consequence of superseding ADR-0006).

### 6.4 Grants

```sql
revoke all on schema sovereign from public, anon, authenticated, service_role;
revoke all on all tables in schema sovereign from public, anon, authenticated, service_role;
grant usage on schema sovereign to sovereign_app;
grant usage on schema private   to sovereign_app;

grant select on sovereign.visitors   to sovereign_app;
grant update (username) on sovereign.visitors to sovereign_app;
grant select on sovereign.workspaces to sovereign_app;
-- visitor_sessions: no table privileges for sovereign_app at all.
```

- Function `EXECUTE` is revoked from `public`, `anon`, `authenticated` and `service_role` on every function in `private`, then granted individually to `sovereign_app` (§6.6).
- Whether Supabase default privileges grant anything automatically on a new schema must be checked, not assumed: a pgTAP catalog test asserts the exact grant set on every `sovereign` table and every `private` function (§9.2).

### 6.5 Visitor context per transaction
The server sets the context **inside an explicit transaction**, transaction-locally, then runs its queries:

```sql
begin;
select set_config('sovereign.session_token_hash', $1, true);   -- $1 = hex(sha256(token)), 64 chars
-- ... queries under RLS ...
commit;
```

```sql
create function private.current_visitor_id()
returns uuid
language sql stable security definer set search_path = ''
as $$
  select s.visitor_id
  from sovereign.visitor_sessions s
  where s.token_hash = case
          when current_setting('sovereign.session_token_hash', true) ~ '^[0-9a-f]{64}$'
          then decode(current_setting('sovereign.session_token_hash', true), 'hex')
        end
    and s.revoked_at is null
    and s.expires_at > now();
$$;
```

Why the **token hash**, not a visitor id, is the context:
- If the context were a visitor id, anything able to run SQL as `sovereign_app` (an SQL-injection bug, for example) could set any visitor id and pass every policy.
- With the hash, the context only resolves to a visitor if it matches a live session row, and `sovereign_app` cannot read `visitor_sessions`. An attacker with arbitrary SQL as `sovereign_app` can only act as visitors whose token hash they already know.
- The raw token never reaches the database (no risk of it appearing in database logs).

Pooling:
- `set_config(..., true)` is **transaction-local**: PostgreSQL discards it at commit or rollback, so it cannot survive into the next transaction on a pooled connection.
- The server's data-access helper always opens an explicit transaction (`withVisitor(hash, fn)`), sets the context first, and never issues visitor queries outside it.
- With Supabase's pooler in **transaction mode** (port 6543), each transaction gets a server connection for its duration only; transaction-local settings fit this model. Session-level `SET` (without `LOCAL`) is forbidden in application code (lint/test, §9.2).
- Driver requirements for transaction mode (e.g. disabling named prepared statements) depend on the chosen driver (D16).

### 6.6 Functions (all in `private`, `SECURITY DEFINER`, `set search_path = ''`, fully qualified names, no dynamic SQL)

| Function | Granted to | Purpose |
|---|---|---|
| `current_visitor_id() → uuid` | `sovereign_app` | Policy helper (§6.5). Returns null on any mismatch. |
| `provision_visitor(token_hash bytea, username text) → (visitor_id uuid, workspace_id uuid)` | `sovereign_app` | Atomic creation of visitor, workspace and session (§7.1). Validates hash length (32) and username format; raises a generic error otherwise. |
| `touch_session(token_hash bytea) → (visitor_id uuid, expires_at timestamptz, needs_rotation boolean)` | `sovereign_app` | Validation and throttled sliding renewal (§5.4, §7.2). Returns no row for invalid sessions. |
| `rotate_session(old_hash bytea, new_hash bytea) → boolean` | `sovereign_app` | Atomic rotation with grace (§5.5, §7.5). |
| `revoke_session(token_hash bytea, reason text) → boolean` | `sovereign_app` | "Forget this browser" (§5.7). Reason restricted to `forgotten`. |
| `set_updated_at()` (trigger) | nobody | Timestamps. |

### 6.7 RLS

```sql
alter table sovereign.visitors         enable row level security;
alter table sovereign.workspaces       enable row level security;
alter table sovereign.visitor_sessions enable row level security;   -- no policies: deny-all

create policy visitors_select_self on sovereign.visitors
  for select to sovereign_app using (id = (select private.current_visitor_id()));
create policy visitors_update_self on sovereign.visitors
  for update to sovereign_app
  using (id = (select private.current_visitor_id()))
  with check (id = (select private.current_visitor_id()));
create policy workspaces_select_own on sovereign.workspaces
  for select to sovereign_app using (visitor_id = (select private.current_visitor_id()));
```

What RLS does and does not guarantee here:
- **Does:** an application bug that forgets a `where visitor_id = …`, or passes another visitor's id, cannot read or change another visitor's rows, because every row is filtered by the session behind the transaction's context.
- **Does not:** protect against a compromised web server. The server holds live session tokens in incoming requests and the `sovereign_app` password; whoever controls it can act as any visitor whose cookie passes through it. A custom PostgreSQL setting is **not independently trustworthy**: any code running SQL as `sovereign_app` can call `set_config`. The token-hash context (§6.5) narrows this to "visitors whose token hash is known", but it is defence in depth, not an isolation guarantee against the server itself.
- `postgres` (table owner) bypasses RLS; it is used only for migrations and operator tasks.

### 6.8 Exposed schemas and Supabase settings
- Data API exposed schemas: `public` only (as today), and `public` contains no Sovereign tables. Optionally disable the Data API entirely (D18).
- Supabase Auth is not used. Sign-ups and anonymous sign-ins are disabled in the dashboard so no `auth.users` rows can be created by anyone holding the publishable key.
- The browser bundle contains no Supabase URL or key for visitor data. Whether `NEXT_PUBLIC_SUPABASE_*` variables remain at all depends on later features (Realtime, Storage; §10).

### 6.9 If the context model cannot be supported
The model depends on two hosted facts that must be verified at the hosted gates, not assumed:
1. A custom login role (`sovereign_app`) can connect through Supabase's pooler (the existing ROADMAP/DATA-MODEL `[VERIFY AT M2]` item for custom roles).
2. Transaction-mode pooling preserves transaction-local settings for the transaction's duration (expected from PostgreSQL semantics; verify with the pool-isolation test, §9.2).

Alternatives if (1) fails: (a) direct connection or session-mode pooler with a small server-side pool (same transaction-local rule); (b) a dedicated pooler user mapped to `sovereign_app`. If transaction-local settings proved unreliable (2): (c) pass the token hash as an explicit argument to SECURITY DEFINER data-access functions for every operation (no setting at all), at the cost of a function-only API like the worker's (§G.6). **Not acceptable:** routine `service_role` or `postgres` access from the web server.

---

## 7. Application architecture

### 7.1 First-visit provisioning
1. App shell renders without a visitor and mounts a provisioning client component.
2. The component calls the `provisionVisitor` Server Action (`POST`).
3. The action checks Origin (§5.8), refuses if a session cookie was sent, applies rate limits (§5.10), generates the token and username, hashes the token and calls `private.provision_visitor(hash, username)` in one transaction. On a username collision it retries (≤ 5).
4. On success it sets the cookie (§5.2) and returns the workspace view model (no ids beyond what the UI needs).
5. Concurrency: two parallel provisioning requests from one browser without a cookie create **two** visitors (the server cannot tell they belong to one browser); the browser keeps whichever cookie arrives last. The other visitor is an orphan with no data, removed by retention cleanup (D11). The client component is single-flight to make this rare. Database integrity is unaffected: each visitor has exactly one workspace and its own session.

### 7.2 Returning-visitor resolution
`requireVisitor()` (server-only) reads the cookie, validates it via `private.touch_session` (§5.6) and returns `{ visitorId, sessionHashHex, needsRotation }` or a typed "no visitor" / "invalid session" result. Pages render either the workspace or the §2.6 notice. Renewal and rotation (which need `Set-Cookie`) happen in the proxy for app paths or in the next Server Action (D19).

### 7.3 Username updates
`updateUsername` Server Action: Origin check, `requireVisitor()`, Zod validation, reserved-name check, per-visitor rate limit, then `update sovereign.visitors set username = $1` inside `withVisitor` (RLS restricts it to the visitor's own row). A unique violation returns "That name is taken." Generic errors otherwise.

### 7.4 Workspace retrieval and data operations
- `getPersonalWorkspace()` reads `sovereign.workspaces` inside `withVisitor`; RLS returns at most the visitor's own row.
- All future domain operations follow the same pattern: services in `packages/core` take an explicit `Actor` (visitor id + workspace id resolved server-side), and the `@sovereign/db` adapter executes them inside `withVisitor`.

### 7.5 Cookie issuance, rotation and revocation
- Issue: provisioning and rotation only.
- Rotate: generate new token → `private.rotate_session(oldHash, newHash)` → on `true`, set the new cookie; on `false` (already rotated), keep the current cookie.
- Revoke: `forgetThisBrowser` Server Action → `private.revoke_session(hash, 'forgotten')` → delete the cookie (`Max-Age=0`).

### 7.6 Errors
Typed results `{ ok, data } | { ok: false, error }` with `error ∈ { no_visitor, invalid_session, rate_limited, validation, conflict, unavailable }`. Messages to the client are generic. Logs carry no tokens, hashes or cookie values; they may carry visitor ids and status codes.

---

## 8. Mapping to the current Next.js source (read-only inventory, 2026-11-12)

### 8.1 What exists today (account-based M2, uncommitted, never applied to a hosted database)
| Area | Files | Fate under this spec |
|---|---|---|
| Auth pages and actions | `apps/web/src/app/auth/{actions.ts, AuthShell.tsx, auth.css, callback/route.ts, sign-in, sign-up, verify-email, reset-password, update-password}` | Removed (no accounts). `AuthShell.tsx`/`auth.css` styling may be reused for the §2.6 notice. |
| Auth helpers | `apps/web/src/lib/auth/{gate.ts, auth-outcome.ts, next-path.ts}`, `apps/web/src/server/auth.ts` | Replaced by `server/visitor.ts` (`requireVisitor`, `withVisitor`). `next-path.ts` is no longer needed (no post-login redirect). |
| Supabase clients | `apps/web/src/lib/supabase/{browser.ts, server.ts, session.ts, config.ts}`, `apps/web/src/proxy.ts` | Removed from the visitor path. `proxy.ts` is repurposed for renewal/rotation on app paths (D19). |
| Canonical origin | `apps/web/src/lib/site-origin.ts` | **Kept**; becomes the source for the Origin check (§5.8). |
| Workspace page | `apps/web/src/app/(app)/workspace/page.tsx` | Rewritten against `requireVisitor` / `getPersonalWorkspace`; sign-out replaced by "Forget this browser". |
| Composition root | `apps/web/src/server/composition.ts` | Gains the database adapter wiring. |
| DB package | `packages/db/src/config.ts` (`parseWorkerDatabaseUrl` rejects over-privileged roles) | Pattern reused for a `parseAppDatabaseUrl` that rejects `postgres`, `supabase_admin`, `service_role`, `authenticator` and requires `sovereign_app`. |
| Env declarations | `apps/web/src/env.d.ts` (`NEXT_PUBLIC_SUPABASE_*`, `SOVEREIGN_SITE_URL`) | Gains a server-only database URL for `sovereign_app` (name to be decided with D16); `NEXT_PUBLIC_SUPABASE_*` not needed for M2. |
| Migrations | `packages/db/supabase/migrations/20261112120000_m2_schemas_privileges.sql`, `20261112120100_m2_identity.sql` | Both depend on `authenticated`, `auth.users` and `auth.uid()`. Never applied; to be **replaced** (not amended) by new migrations — requires approval. |
| pgTAP | `packages/db/supabase/tests/{identity_signup, identity_rls, catalog}.test.sql` | Replaced by the suites in §9.2. The control `_selftest/deliberate_failure.test.sql`, `manifest.json` mechanics and the runner `scripts/run-sql-tests.mjs` / `scripts/tap.mjs` are **kept**. |
| Integration tests | `apps/web/test/integration/{auth.test.ts, support/admin.ts, support/env.ts, support/user.ts}` | Replaced by visitor-session integration tests (§9.3). `support/diagnostics.ts` (sanitised cleanup) is reusable. The service-role key is no longer needed. |
| Isolation/bundle guards | `apps/web/test/service-role-isolation.test.ts`, `scripts/check-client-bundle.mjs`, ESLint rules | Kept; extended to the `sovereign_app` database URL (must never reach the client bundle). |
| Dependencies | `@supabase/ssr`, `@supabase/supabase-js` (web), `supabase` CLI (db) | Supabase JS clients unused for M2. **A PostgreSQL driver is required** and is a new dependency needing approval (D16). |

### 8.2 Proposed new structure (not created)
- `apps/web/src/server/visitor.ts` (`server-only`): cookie read/write, token generation and hashing, `requireVisitor`, `withVisitor`, Origin check.
- `apps/web/src/app/(app)/workspace/actions.ts`: `provisionVisitor`, `updateUsername`, `forgetThisBrowser`.
- `apps/web/src/app/(app)/workspace/Provision.tsx` (client): single-flight provisioning call.
- `packages/db/src/app-database.ts`: driver wrapper that only exposes `withVisitor(hashHex, fn)` and the provisioning/session function calls.

---

## 9. Testing requirements (none of these tests exists yet)

### 9.1 Unit (offline, Vitest)
- Token generation: 32 bytes, base64url, 43 characters; uniqueness across many draws; hash is SHA-256 hex of the raw bytes.
- Cookie attributes: production `__Host-` name, `HttpOnly`, `Secure`, `SameSite=Lax`, `Path=/`, no `Domain`, `Max-Age` from expiry; dev variant only for `http://localhost`.
- Origin check: missing `Origin` rejected; foreign origin rejected; canonical accepted; `Sec-Fetch-Site: cross-site` rejected; forwarded-host headers ignored.
- Username validation: format, length, reserved names, normalisation; generator always produces valid names.
- `requireVisitor`: absent cookie → `no_visitor`; malformed → `invalid_session` without a database call; database returns nothing → `invalid_session`.
- Provisioning refuses when a cookie is present; retries on username collision ≤ 5 times.
- Logs contain no token, hash or cookie value (spy on the logger with sentinel tokens).
- Static guards: no session-level `SET`/`set_config(..., false)` in application SQL; the app database URL never appears in client bundles.

### 9.2 Database / RLS (pgTAP via `pnpm test:rls`, control first)
- Provisioning: one call creates exactly one visitor, one workspace (owned by it) and one session; invalid hash length or username raises and creates nothing.
- Atomicity: a failure inside provisioning (forced, e.g. duplicate token hash) leaves no partial rows.
- One-to-one: inserting a second workspace for a visitor fails (unique); a workspace without a visitor fails (not null / FK).
- Context: with no setting, a malformed setting, an unknown hash, an expired or a revoked session, `current_visitor_id()` is null and every policy returns zero rows.
- Isolation as `sovereign_app`: visitor A's context cannot select, update or otherwise touch visitor B's visitor or workspace row (zero rows affected); B's rows unchanged.
- Spoofing: setting `sovereign.session_token_hash` to a fabricated value, or to B's visitor id, yields nothing; `sovereign_app` cannot select `visitor_sessions` (permission denied); it cannot insert into `visitors` or `workspaces` directly.
- Session functions: `touch_session` throttled renewal; `rotate_session` grace behaviour and single rotation under a repeated call; `revoke_session` makes the context resolve to null.
- Transaction-local context: after `commit`/`rollback`, `current_setting('sovereign.session_token_hash', true)` is empty in the same connection.
- Catalog: RLS enabled on all three tables; exact policy set; exact table and column grants (none for `anon`, `authenticated`, `service_role`); every `private` function `prosecdef` with `search_path=""`, no `EXECUTE` in bodies, execute grants exactly `{sovereign_app}` where listed; `sovereign_app` has no `BYPASSRLS`; `sovereign` not in exposed schemas (checked at G6, not by pgTAP).
- Cascade: deleting a visitor removes its workspace and sessions.

### 9.3 Visitor-session integration (Vitest against the hosted development database as `sovereign_app`)
- End-to-end provisioning through the real driver and pooler; returned cookie validates; a second resolution returns the same visitor.
- **Transaction-pool isolation:** with the pool forced to one connection, request A (visitor A context) followed by request B with **no** context returns nothing for B; interleaved concurrent A/B transactions never see each other's rows (many iterations).
- Concurrent first requests: N parallel provisionings create N visitors, each with exactly one workspace.
- Rotation race: parallel requests with the old token during the grace period succeed; after grace, the old token fails.
- Expiry: a session with `expires_at` in the past (set by a test-only path) is rejected.
- Cleanup: every visitor created is recorded immediately and deleted afterwards; failures reported with sanitised ids only (reuse `support/diagnostics.ts` pattern). Test cleanup needs a deletion path; proposed: a separate test-only role or the operator running a documented cleanup script, **not** a privilege granted to `sovereign_app` in production (D20).

### 9.4 API / Server Action integration (Vitest, server code against the development database)
- `provisionVisitor`, `updateUsername`, `forgetThisBrowser`: Origin enforcement (missing/foreign rejected), cookie set/cleared with correct attributes, rate limits enforced, generic errors.
- Username: taken name → conflict; invalid → validation; another visitor's username unchanged.
- Spoofed ids: any visitor/workspace id supplied in form data or headers is ignored.

### 9.5 Browser (manual at G7 until an E2E tool is approved; Playwright is listed in ADR-0001 but not installed)
- New visitor: open `/workspace` in a fresh profile → workspace appears without any form; cookie present with correct attributes.
- Reload and full browser restart → same username and workspace.
- Username edit persists across restart.
- Delete the cookie → §2.6 behaviour: automatic provisioning only when no cookie is sent; with an invalid cookie, the notice and explicit "Start a new workspace".
- Tamper with the cookie value → invalid-session notice; no other visitor's data.
- Two profiles (A and B) → each sees only its own workspace; B pasting A's URL sees only B's data.
- CSRF: a page on another origin auto-submitting a POST to a Server Action endpoint is rejected (no state change).
- "Forget this browser" → warning, then access ends.

### 9.6 Cleanup of test data
All visitors created by tests use a recognisable username prefix (e.g. `it-`, reserved for tests, D5) and are deleted at the end; a final check asserts zero remaining test visitors, workspaces and sessions.

---

## 10. Consequences for the rest of the blueprint (not part of M2; to be decided)
- **Cost and abuse for agent runs (M4b/M5):** the PRD's verified-email gate disappears. A replacement is required before any run can spend money (per-visitor and per-workspace budgets already planned, plus provisioning controls; possibly a human-verification challenge). Open.
- **Realtime (ADR-0005):** private channels authorised by Supabase Auth JWTs no longer apply. Options: server-sent events from the Sovereign server, or server-minted short-lived JWTs for Realtime (which introduces a signing credential). Open.
- **Operators, admin and moderation (PRD platform roles, §I.9 `service_role` admin module):** need an operator identity that is **separate** from visitor identity. Open.
- **Public profiles, publishing, reports:** a username is not unique proof of a person; impersonation and squatting policies are needed if usernames become public (D4). Open.
- **Teams (ADR-0006):** dropped from the schema; re-adding them is a schema migration.
- **Storage:** deliverables in DB columns in MVP; any Storage use must go through the server.
- **Privacy/retention:** retention for abandoned visitors (D11) and the lawful basis for storing client-IP HMACs (D13/D14).

---

## 11. Decision register

| # | Decision | Status |
|---|---|---|
| D1 | Accountless visitor identity with a server-generated id and an HttpOnly bearer session; no Supabase Auth for visitors | **Approved (product requirement, 2026-11-12)** |
| D2 | Identity tables in non-exposed schema `sovereign`; nothing Sovereign in `public` | Proposed |
| D3 | Username format `^[a-z0-9][a-z0-9_-]{2,23}$`; generator `adjective-noun-NN` | Proposed |
| D4 | Case-insensitive unique usernames | Proposed |
| D5 | Reserved-name denylist (incl. a test prefix) | Proposed |
| D6 | Provisioning only via same-origin POST from the app shell; GET never provisions | Proposed |
| D7 | Invalid cookie → explicit notice + "Start a new workspace"; "Forget this browser" UX with warning | Proposed |
| D8 | Sliding idle lifetime 180 days, renewal throttled to 24 h, no absolute cap | Proposed |
| D9 | Cookie name `__Host-sovereign_session` (production), `sovereign_session` on `http://localhost` | Proposed |
| D10 | Rotation every 30 days with a 2-minute grace; reuse logged, no automatic visitor-wide revocation | Proposed |
| D11 | Retention cleanup of expired/revoked sessions and of visitors with no live session for N days (N open) | Open |
| D12 | Additional synchronizer CSRF token | Proposed (optional layer) |
| D13 | Rate-limit values and Postgres-backed counters (HMAC client key) | Proposed |
| D14 | Trusted-proxy configuration for client IP | Open (depends on hosting) |
| D15 | SECURITY DEFINER functions owned by `postgres` vs a dedicated NOLOGIN owner role (as `worker_api_owner`, §G.6) | Open |
| D16 | PostgreSQL driver (new dependency), pooler mode and connection limit; env var name for the app database URL | Open — needs dependency approval |
| D17 | Deferrable "visitor must have a workspace" constraint trigger | Proposed (optional) |
| D18 | Disable the Supabase Data API entirely vs keep `public` exposed and empty | Open |
| D19 | Where renewal/rotation set the cookie: proxy on app paths vs next Server Action | Proposed: proxy |
| D20 | Test-data deletion path without granting delete to `sovereign_app` | Open |

---

## 12. Documents that must change before implementation (not changed by this revision)
- New ADR superseding: ADR-0001 (Supabase Auth as identity), ADR-0005 (JWT-based Realtime authorisation), ADR-0006 (membership/team-ready schema), ADR-0011 (Auth flows, service-role test helper, Auth integration layer).
- `docs/PRD.md` §C.1–C.4 and journeys (accounts, verified email, `/login`, `/signup`, `/reset-password`, `/auth/callback`).
- `docs/ARCHITECTURE.md` §F.1, §F.2, §F.4, §F.7, §H.1, §H.2 (`/auth/callback`, workspace "auto-created on signup").
- `docs/DATA-MODEL.md` §G.1, §G.2, §G.3, §G.5, §G.7, §G.8. Note: the G.8 "as implemented" line says migration 1 created "fail-closed default privileges"; the current migration file does not change default privileges (it was narrowed in a later review round), so that line is already inaccurate.
- `docs/SECURITY.md` §I.1, §I.2, §I.3, §I.5, §I.7, §I.8, §I.9.
- `docs/ROADMAP.md` M2 exit criteria and status row; `pnpm test:integration` description.
- The M2 hosted-verification runbook (revision 6 is blocked: §13).

## 13. Hosted verification
The hosted-verification runbook must be revised (revision 6) to follow this specification. Its gate sequence is fixed by the owner (G1–G9). Test counts for G4/G5 can only be stated once the new test files exist; they do not exist yet.
