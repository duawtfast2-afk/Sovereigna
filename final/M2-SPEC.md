# M2 — Accountless visitor identity and personal workspace: architecture specification

**M2 Architecture Specification — Revision 2** (2026-11-12). Revision 1 is superseded by this text.

Status: **specification only. Not approved for implementation.** No code, migration or test described here exists yet. Revising this document does not approve any design decision in it.
Supersedes the account-based M2 design (Supabase Auth sign-up/sign-in, verified email, `auth.users`-keyed identity) for M2 only.
Decision register: §11. Each entry is labelled as an **accepted product requirement**, a **recommended design awaiting user approval**, an **unresolved technical prerequisite**, or **deferred to a later milestone**. Only the first label means approved.

Change rule (docs/README.md): before implementation this specification needs (1) a new ADR in `docs/DECISIONS/` superseding the affected parts of ADR-0001, ADR-0005, ADR-0006 and ADR-0011, (2) updates to the living documents listed in §12, and (3) a ROADMAP status update. None of those has been made; this document does not edit them.

### Changes from revision 1
- §5.4–§5.6 and §7.2: one session state machine. Renewal and rotation no longer conflict; rotation completes on the successor's first use; predecessors are never renewed; reuse after grace is recorded on the retained row.
- §5.10 and §7.1: first-visit provisioning is idempotent per browser through a short-lived provisioning nonce; duplicate visitors from one browser become a narrow, documented edge case with a cleanup policy (§6.8).
- §5.11–§5.12: abuse controls are **mandatory** for public deployment; client-IP trust is a deployment prerequisite with fail-closed behaviour.
- §6.2, §6.6, §6.7: SECURITY DEFINER functions are owned by a dedicated `NOLOGIN` role with an explicit privilege model; hosted support is a prerequisite with a recorded fallback.
- §6.3: schema updated (session columns, provisioning nonces, rate-limit tables).
- §8.1: corrected description of the existing account-based work.
- §9: tests rewritten against the revised design. §11: every decision relabelled.

---

## 1. Goal and principles

Sovereign is **accountless**. A visitor opens Sovereign and uses it immediately. There is no sign-up, sign-in, password, email confirmation or password recovery anywhere in the product.

1. **No hidden accounts.** Supabase Auth is not the foundation of visitor identity: no `auth.users` rows, no `auth.uid()`, no Supabase Auth sessions, no anonymous Supabase sign-ins.
2. **The browser holds a credential, nothing else.** A random, opaque session token in an HttpOnly cookie is the only proof of workspace ownership. The username and the visitor id are never credentials.
3. **One backend, one door.** Browser code never talks to the database or to Supabase APIs for visitor data. Every read and write goes through the Sovereign server (Next.js server code), which establishes the visitor context.
4. **Least privilege, honestly described.** The server connects with a dedicated database role without `BYPASSRLS`. RLS is defence in depth against application bugs; it is **not** protection against a compromised server or against arbitrary SQL run as the application role (§6.5, §6.7).
5. **Fail closed.** A missing, malformed, expired, rotated-out or revoked session yields "no visitor", never another visitor's data. Missing security configuration disables the affected feature instead of weakening it.
6. **Accepted limitation, stated plainly.** Losing the cookie or the browser profile means losing access to the workspace. There is no recovery.

Out of scope for M2: cross-device recovery or transfer, email of any kind, operator/admin identity, public profiles, Realtime, agent runs and budgets, teams. §10 lists the consequences for later milestones.

---

## 2. Product behaviour

### 2.1 First visit
- Public pages (landing, Realm, Orders and the other `(realm)` routes) need no visitor identity and create none.
- When a browser without a session enters the application area (M2: `/workspace`):
  1. The response to that `GET` carries a short-lived **provisioning nonce cookie** (§5.10). No database row is created by any `GET`.
  2. The app shell's client component sends one same-origin `POST` (Server Action) automatically, without any form or click.
  3. The server creates the visitor, the personal workspace and a session atomically (§7.1), sets the session cookie and returns the workspace view.
- Every tab of the same browser that provisions during the nonce's lifetime joins the **same** visitor, apart from one narrow race documented in §5.10.
- Crawlers and link previews that do not run JavaScript create nothing.

### 2.2 Returning visit
- The browser sends the session cookie. The server validates it (§5.6), resolves the visitor and renders the workspace.
- Persistence across browser restarts on the same browser profile and device comes from a **persistent** cookie (`Max-Age`, §5.4).

### 2.3 Username
- Every visitor has exactly one visible username: display identity only.
- The server generates an initial username on provisioning (§4.2). The visitor can edit it later (§7.3).
- The username is **never** used to look up, authorise or prove ownership of anything.

### 2.4 Personal workspace
- Each visitor has exactly one personal workspace, created together with the visitor. Ownership is the database relationship `workspaces.visitor_id`, enforced one-to-one (§6.3).
- M2 workspace content is its name and identity only. Domain data arrives in later milestones with its own tables.

### 2.5 Browser persistence and its limits
| Event | Result |
|---|---|
| Reload, new tab, browser restart (same profile) | Same visitor and workspace. |
| Private/incognito window | A separate visitor, lost when the private session ends. |
| Another browser, profile or device | A different visitor. **Cross-device recovery is out of scope.** |
| Visitor clears cookies or site data | The old workspace becomes inaccessible from this browser. The old data is deleted later by retention cleanup (§6.8). |
| No use for longer than the idle lifetime (§5.4) | Same as clearing cookies. |
| Session revoked (§5.8) | Inaccessible from that browser. |
| Cookie stolen | The thief can act as the visitor (§5.13). |

### 2.6 Invalid, expired or revoked cookie
- The server never deletes or overwrites a session cookie because it failed to validate. Only explicit visitor actions ("Start a new workspace", "Forget this browser") and successful rotation (§5.5) change it.
- If a session cookie **was sent** but does not resolve to a visitor (unknown, expired, revoked, rotated out, malformed), the app area shows a notice ("This browser's previous Sovereign workspace can no longer be opened") and an explicit "Start a new workspace" action. Only that action provisions a new visitor and replaces the cookie.
- If **no** session cookie was sent, provisioning is automatic (§2.1).
- Reason: with `SameSite=Lax` (§5.3) the cookie is sent on same-site requests and top-level navigations, so "no cookie" means "none stored". Replacing an unreadable cookie automatically would hide a real loss and could replace a newer cookie that an older in-flight response failed to validate (§5.5).

### 2.7 Out of scope (explicit)
Cross-device recovery, account linking, identity export/import, email, passwords, magic links, OAuth, recovery codes.

---

## 3. Identity concepts

| Concept | What it is | Where it lives | Secret? | Used for |
|---|---|---|---|---|
| **Username** | Human-readable display identity, e.g. `quiet-heron-41` | `sovereign.visitors.username` | No | Display only. Never authorises anything. |
| **Visitor id** | `uuid` generated by PostgreSQL | `sovereign.visitors.id` | Not secret, **not a credential** | Internal key. Never accepted from the client for any decision. |
| **Session token** | 256-bit random bearer token | Browser cookie only; never stored in plaintext server-side | **Yes** | Proof of possession of a session. |
| **Session token hash** | SHA-256 of the token (32 bytes) | `sovereign.visitor_sessions.token_hash` | Sensitive (lookup key) | Session lookup and the database visitor context. Never logged. |
| **Provisioning nonce** | 256-bit random value, ≤ 10-minute life | Browser cookie only; its SHA-256 in `sovereign.provisioning_nonces` | **Yes**, short-lived | De-duplicating first-visit provisioning per browser (§5.10). |

Rules:
- No request parameter, form field or header carries a visitor id or workspace id that the server trusts. The server derives both from the session.
- Visitor ids and session row ids may appear in server logs. Tokens, nonces, their hashes and cookie values never do.

---

## 4. Visitor identity model

### 4.1 Visitor id
Generated by `gen_random_uuid()` inside the provisioning function. The client never proposes it.

### 4.2 Initial username (D3–D5, recommended)
- Generated server-side from a fixed word list plus a random number (`adjective-noun-NN`) using `node:crypto` randomness.
- Format: 3–24 characters, lowercase `a–z`, digits, `-`, `_`, starting with a letter or digit; enforced by a database `CHECK` and by server-side validation before the database call.
- Uniqueness: `UNIQUE`; the lowercase-only format makes it case-insensitive. On a collision the server retries the provisioning call with a new name, at most 5 times, then returns a generic error.
- Reserved names: a denylist (`admin`, `sovereign`, `support`, `system`, Order and goddess names, and the test prefix `zz-test-`) checked server-side. Server Actions and the generator never produce or accept the test prefix; only test fixtures use it (§9.7).

---

## 5. Visitor session security

### 5.1 Token
- 32 bytes from `node:crypto`, encoded base64url (43 characters, no padding).
- Only `sha256(token)` is stored and compared. A fast hash is appropriate because the token is high-entropy random data, not a password.
- The raw token never reaches the database, logs, analytics or error reports. The database visitor context uses the hash (§6.5).

### 5.2 Cookies
| Attribute | Session cookie | Provisioning-nonce cookie |
|---|---|---|
| Name (production) | `__Host-sovereign_session` | `__Host-sovereign_provision` |
| Name (`http://localhost` development only) | `sovereign_session` | `sovereign_provision` |
| `HttpOnly` | yes | yes |
| `Secure` | yes (off only on `http://localhost`) | yes (same) |
| `SameSite` | `Lax` | `Lax` |
| `Path` | `/` (required by `__Host-`) | `/` |
| `Domain` | not set | not set |
| `Max-Age` | synchronised with `expires_at` on every issue or refresh (§5.4) | 600 seconds |

### 5.3 SameSite choice
- `Lax`: sent on same-site requests and on top-level cross-site `GET` navigations; withheld on cross-site `POST`, iframes and subresources.
- `Strict` was rejected: a visitor arriving from an external link would arrive without the cookie and look like a new visitor.
- `SameSite` is not the CSRF defence by itself (§5.9).

### 5.4 Lifetime and renewal (D8)
- **Idle lifetime: 180 days.** A session's `expires_at` is set to `now() + 180 days` when it is issued.
- **Renewal** extends `expires_at` to `now() + 180 days` and sets `last_renewed_at = now()`. It happens only:
  - for a session in state `active` (§5.6) — never for a rotated predecessor, an expired, revoked or unknown session;
  - when `now() - last_renewed_at >= 24 hours`;
  - in the **refresh request** (§7.2), which also re-issues the cookie with a matching `Max-Age`. Render-time validation never renews, so the database expiry and the cookie expiry cannot drift apart.
- **No absolute cap.** An absolute cap would destroy the workspaces of active visitors, because there is no recovery. Browsers may cap cookie lifetime (e.g. Chrome at 400 days); renewal keeps active visitors far inside that.
- Inactivity longer than 180 days means permanent loss of access (§2.5).

### 5.5 Rotation (D10)
Rotation replaces the token of a long-lived session without risking lock-out when a response is lost.

- **When:** a session in state `active` whose token was issued 30 or more days ago (`now() - created_at >= 30 days`), during a refresh request (§7.2).
- **Issue step** (`private.rotate_session`): inserts a **successor** session for the same visitor (new token hash, `predecessor_id` = the old session, fresh `created_at`, `last_renewed_at` and `expires_at = now() + 180 days`, `first_used_at = null`). The predecessor is **not changed**: it stays `active`. The response sets the new cookie.
- **Completion step** (inside `private.touch_session`, on the successor's **first** successful validation): sets `successor.first_used_at = now()`; on the predecessor sets `rotated_at = now()`, `successor_id`, and `expires_at = least(expires_at, now() + interval '120 seconds')`; revokes every other pending successor of that predecessor (`revoke_reason = 'superseded'`).
- **Grace period:** 120 seconds, **measured from `rotated_at`, the completion timestamp** (the successor's first use), capped by the predecessor's previous expiry. Measuring from completion rather than from issue means a lost rotation response does not start the clock: the browser still holds the predecessor, which stays `active`, and the next refresh simply issues another successor.
- **Boundary:** a rotated predecessor is valid while `now() < expires_at` (strict). At `now() = expires_at` and later it is rotated-out. `now()` is the PostgreSQL transaction timestamp of the validating transaction; all comparisons in §5.6 and §6.5 use it.
- **At most one recent pending successor:** if the predecessor already has a pending successor (`first_used_at is null`, not revoked) created less than 10 minutes ago, `rotate_session` issues nothing and returns `pending` (the response sets no cookie). An older pending successor is presumed lost: it is revoked (`superseded`) and a new one is issued.

### 5.6 Session states and `touch_session()` behaviour
State of a presented token hash at transaction time `now()`, evaluated in this order:

| # | State | Condition | `touch_session` returns | Writes |
|---|---|---|---|---|
| 1 | `invalid_input` | hash is not exactly 32 bytes | state only | none |
| 2 | `unknown` | no row with this hash (never issued, or deleted by retention, §6.8) | state only | none |
| 3 | `revoked` | `revoked_at is not null` | state, session id | none |
| 4 | `rotated_grace` | `rotated_at is not null` and `now() < expires_at` | state, session id, visitor id; `renewal_due = false`, `rotation_due = false` | none |
| 5 | `rotated_reused` | `rotated_at is not null` and `now() >= expires_at` | state, session id; **no** visitor id | `reuse_count = reuse_count + 1`, `last_reuse_at = now()` |
| 6 | `expired` | `rotated_at is null` and `now() >= expires_at` | state, session id; no visitor id | none |
| 7 | `active` | otherwise | state, session id, visitor id, `expires_at`, `renewal_due`, `rotation_due` | first-use completion (§5.5) if `first_used_at is null`; renewal (§5.4) only when called with `renew = true` and due; `visitors.last_seen_at = now()` together with renewal |

- A visitor id is returned **only** for `active` and `rotated_grace`. These are exactly the sessions that `private.current_visitor_id()` accepts (`revoked_at is null and now() < expires_at`), so validation and the database context agree.
- A pending successor that was `superseded` is in state `revoked`.
- `renewal_due = (now() - last_renewed_at >= 24 hours)`; `rotation_due = (now() - created_at >= 30 days)`; both only for `active`.
- **Reuse detection is possible only while the predecessor row is retained.** Rotated rows are kept for 30 days after their validity ends (§6.8); a reused token presented after deletion is `unknown`. Reuse is recorded on the row (`reuse_count`, `last_reuse_at`) and logged by the server as an event with the session row id and visitor id only — never the token, hash or cookie.
- Reuse does not revoke the visitor's other sessions: without recovery, automatic visitor-wide revocation would end the legitimate visitor's access too (D10). The reusing request is treated as "no visitor" and its response does not touch the cookie (§2.6).

**Rotation races (multiple requests presenting the old token):**
- Two refresh requests with the predecessor both see `rotation_due`. `rotate_session` locks the predecessor row `FOR UPDATE`; the first issues successor S1 and returns `issued` (sets cookie); the second sees S1 pending and younger than 10 minutes and returns `pending` (sets no cookie). The browser ends with S1.
- Ordinary requests in flight with the predecessor after the browser received S1: the predecessor is still `active` until S1's first use, then `rotated_grace` for 120 s. They succeed. After the grace period they are `rotated_reused` and get the "no visitor" result without touching the cookie, while the browser keeps S1.
- S1's response is lost: S1 is never used, the predecessor stays `active`, and the next refresh after 10 minutes supersedes S1 and issues S2.
- Lock order is fixed to prevent deadlocks: any function that touches a predecessor and its successor locks the **predecessor first**.

### 5.7 Server-side validation path
1. Read the session cookie. Absent → "no visitor". Present but not 43 base64url characters → `invalid_input` without a database call.
2. Hash it and call `private.touch_session(hash, renew)` (§6.6), with `renew = false` on the render path and `true` only in the refresh request (§7.2).
3. Data access runs inside a separate transaction with the visitor context set from the same hash (§6.5). If the session became invalid between the two transactions (for example, grace ended), the context resolves to no visitor and the request fails closed.

### 5.8 Revocation
- **Per browser:** "Forget this browser" revokes the current session (`revoke_reason = 'forgotten'`) and deletes the cookie, after a warning that access to the workspace ends permanently (D7).
- **Operator:** revocation with `revoke_reason = 'operator'` by an operator in SQL; no admin UI in M2.
- **Superseded:** pending successors replaced during rotation (§5.5).

### 5.9 CSRF and Origin validation
Every state-changing request (Server Actions and Route Handlers, `POST` only) passes all of these before any work:
1. `Origin` is **required** and must equal the canonical origin (`SOVEREIGN_SITE_URL`, validated by `apps/web/src/lib/site-origin.ts`). A missing `Origin` is rejected. Next.js 16's built-in Server Action check compares `Origin` with `Host`/`X-Forwarded-Host` and lets a request without `Origin` through with a warning (bundled Next.js docs, `serverActions.md`); it is kept but not relied on.
2. `Sec-Fetch-Site`, when present, must be `same-origin`.
3. Session and nonce cookies are `SameSite=Lax`.
A synchronizer CSRF token is deferred (D12).

**CSRF protection is not abuse protection.** Origin checks stop other sites from making a visitor's browser act; they do nothing against a script that sends same-origin-looking requests directly. Abuse controls (§5.11) are separate and both are required.

### 5.10 Provisioning idempotency (D21)
Goal: every concurrent or repeated provisioning attempt from **one browser** yields **one** visitor, without a long-lived client credential and without trusting a client-supplied visitor id.

- **Nonce issue:** when a request to an app path has neither a session cookie nor a provisioning-nonce cookie, the response sets `__Host-sovereign_provision` = 32 random bytes (base64url), `Max-Age=600`. This is a cookie write only; no database access (D19).
- **Provisioning requires the nonce.** A provisioning `POST` without a valid nonce cookie is refused with "reload"; the client reloads once, which obtains a nonce.
- **Database de-duplication** (`private.provision_visitor`, §6.6, one transaction):
  1. `insert into sovereign.provisioning_nonces (nonce_hash, created_at) values ($1, now()) on conflict do nothing`.
  2. If the insert succeeded: create visitor, workspace and session; set the nonce row's `visitor_id`; `mint_count = 1`; return `created`.
  3. If the row already existed: lock it `FOR UPDATE`. If `now() >= created_at + 10 minutes` or `mint_count >= 5`, raise `provisioning_expired`. Otherwise create a **new session** for the row's visitor, increment `mint_count`, return `joined`.
  A concurrent second transaction with the same nonce blocks on the primary key until the first commits, then follows step 3. If the first rolls back (for example a username collision), the second performs step 2 itself.
- **Success response** sets the session cookie and deletes the nonce cookie.
- **Scenarios:**

| Scenario | Result |
|---|---|
| Several tabs opened at the same time | The tabs' `GET`s may set different nonces; the cookie jar keeps one. The provisioning `POST`s run after page load and carry that one nonce → one visitor; each tab receives a session for it. A tab whose `POST` already carries the session cookie set by another tab's response gets `reload` (no database write) and then renders with that session. |
| Duplicated or retried Server Action | Same nonce → `joined` → another session for the same visitor. |
| Requests arriving before the first response set the session cookie | Same nonce → same visitor. |
| Refresh during provisioning | The nonce cookie survives the reload → same visitor. |
| Different browser profiles | Different cookie jars, different nonces → different visitors (correct). |
| Cookies blocked entirely | No nonce is stored → provisioning refused; the app shows a "cookies are required" notice. |

- **Remaining edge case:** a `POST` that leaves the browser carrying nonce N1 while another tab's `GET` replaces it with N2 before a second `POST` can still create two visitors. This needs two near-simultaneous first loads plus interleaved responses. The extra visitor is never referenced by the browser's final cookie and is removed by §6.8. Client-side single-flight reduces this further but is **not** relied on for uniqueness.
- **Nonce theft:** within its 10-minute window and 5 mints, a stolen nonce lets the thief obtain a session for the newly created visitor — the same power as stealing the session cookie at that moment. After the window it is useless.

### 5.11 Abuse controls (D13, mandatory for public deployment)
Public deployment is **not allowed** without these controls. They are enforced in the server and backed by shared database state, so they hold across multiple application instances.

| Control | Key | Proposed default (D13 values) | On exceed |
|---|---|---|---|
| Provisioning per client network | HMAC of client IP (IPv4 address, IPv6 /64 prefix), §5.12 | 10 per hour **and** 30 per day | generic "Too many new workspaces from this network. Please try again later." |
| Global provisioning circuit breaker | fixed global key | 1,000 per hour (operator-configurable) | generic "Sovereign is temporarily unavailable. Please try again later." until the window ends |
| Username changes per visitor | visitor id | 10 per day | generic "Too many changes. Please try again tomorrow." |

- **Order for provisioning:** CSRF checks (§5.9) → nonce present (§5.10) → no session cookie (otherwise `reload`, §5.10) → trusted client IP established (§5.12) → per-network limits → global breaker → `provision_visitor`. A `joined` provisioning (same nonce) also counts.
- **Storage:** `private.rate_limit_counters` (fixed windows) and `private.rate_limit_settings` (§6.3), changed only through `private.consume_rate_limit(bucket, subject)` (§6.6), an atomic `insert … on conflict … do update … returning`. A missing settings row denies (fail closed).
- **Bounds:** keys are fixed-size (16-byte visitor id, 32-byte HMAC, or the global key); windows are aligned to the database clock; rows older than 2 days are deleted by maintenance (§6.8). Because the per-network check runs before the global breaker (so one network cannot exhaust everyone's allowance), every attempt from a new address creates one counter row per window even after the global breaker has tripped. Volumetric attacks from very many distinct addresses are therefore bounded only by request volume; platform-level protection (WAF/edge rate limiting) is a deployment prerequisite for that class of attack.
- **Secrets:** the HMAC key `SOVEREIGN_RATE_LIMIT_KEY` (server-only, ≥ 32 random bytes; name proposed) keeps raw addresses out of the database. Raw IPs are never stored or logged.
- **Responses** never reveal counts, windows, keys, infrastructure or which check failed beyond the generic messages above. Logs record bucket and outcome only.
- **Trade-off (approval needed):** visitors behind shared addresses (carrier-grade NAT, offices, schools) share the per-network limit.

### 5.12 Client IP trust (D14, unresolved deployment prerequisite)
- Next.js request handlers do not provide a trustworthy peer address on their own, and every `X-Forwarded-*`/`Forwarded` header can be sent by any client. **No forwarding header is trusted by default.**
- A header is trusted only when **all** of these are true and configured explicitly:
  1. The deployment's reverse proxy or platform is selected and documented.
  2. The application is reachable **only** through that proxy (no direct public path that bypasses it).
  3. The proxy **overwrites** (does not append to) one named header with the connecting client's address.
  4. The server is configured with that exact header name (proposed variable `SOVEREIGN_CLIENT_IP_HEADER`), and the value parses as exactly one IPv4 or IPv6 address. Lists, ports, empty values and anything else are "untrusted".
- The proxy/provider is **not yet selected**, so no header is named in this specification.
- **Fail-closed behaviour:** in production (canonical origin not loopback), if no trusted client IP can be established for a request, provisioning is refused with the generic "temporarily unavailable" message and a configuration error is logged (without header values). Requests from existing visitors are unaffected. Username changes are limited per visitor and do not need the client IP.
- **Local development** (`SOVEREIGN_SITE_URL` on a loopback host): a fixed development key is used instead of an IP. This branch never activates for a non-loopback origin.

### 5.13 Theft and loss
- **Theft:** the session cookie is a bearer credential. Whoever presents it **is** the visitor: they can read and change everything the visitor can. HttpOnly prevents scripts reading it, but XSS can act within the page, and malware or a copied browser profile can take the cookie. There is no second factor and no way for the real visitor to end the thief's access from another device. Rotation (§5.5) limits the life of a copied token that is **not** used; a thief who uses the token can perform rotation themselves and the legitimate browser's token then becomes `rotated_reused` (recorded, §5.6) — the visitor loses access, which is detectable but not recoverable.
- **Loss:** clearing cookies, losing the profile or device, or inactivity beyond 180 days loses access permanently. This is accepted (product requirement).

---

## 6. Database model and Supabase security

### 6.1 Placement
- Supabase PostgreSQL remains the hosted database.
- Identity tables live in the schema `sovereign`, which is **not** exposed through the Supabase Data API (D2). Maintenance and rate-limit tables and all functions live in `private` (not exposed).
- No `profiles`, no `workspace_members`, no `workspace_data` JSONB placeholder. Domain tables arrive with their requirements, each with `workspace_id` and the same RLS pattern.

### 6.2 Roles (D15, D16)
| Role | Attributes | Purpose |
|---|---|---|
| `postgres` | Supabase-managed | Applies migrations; owns tables and schemas. Never used by the running application. |
| `sovereign_identity_owner` | `NOLOGIN NOSUPERUSER NOBYPASSRLS NOCREATEDB NOCREATEROLE NOINHERIT` | Owns the identity SECURITY DEFINER functions. Holds only the table and column privileges listed in §6.6, plus owner-scoped RLS policies (§6.7). |
| `sovereign_maintenance_owner` | same attributes | Owns retention and cleanup functions (§6.8), which need `DELETE`. Kept apart so the request-path functions never hold `DELETE` on visitors. |
| `sovereign_app` | `LOGIN NOSUPERUSER NOBYPASSRLS NOCREATEDB NOCREATEROLE NOREPLICATION NOINHERIT`, `CONNECTION LIMIT` (D16) | The only role the web server uses. Password set **out of band** by the operator, never in a migration or the repository. |
| `anon`, `authenticated` | Supabase-managed | No privileges on `sovereign` or `private` objects. |
| `service_role` | Supabase-managed; bypasses RLS | **Not used** by application runtime or by tests. |

### 6.3 Tables

```sql
-- Illustrative DDL for review. Not a migration.
create schema sovereign;

create table sovereign.visitors (
  id            uuid primary key default gen_random_uuid(),
  username      text not null unique
                check (username ~ '^[a-z0-9][a-z0-9_-]{2,23}$'),
  created_at    timestamptz not null default now(),
  updated_at    timestamptz not null default now(),
  last_seen_at  timestamptz not null default now()
);

create table sovereign.workspaces (
  id          uuid primary key default gen_random_uuid(),
  visitor_id  uuid not null unique references sovereign.visitors (id) on delete cascade,
  name        text not null default 'Personal workspace' check (char_length(name) between 1 and 80),
  created_at  timestamptz not null default now(),
  updated_at  timestamptz not null default now()
);

create table sovereign.visitor_sessions (
  id               uuid primary key default gen_random_uuid(),
  visitor_id       uuid not null references sovereign.visitors (id) on delete cascade,
  token_hash       bytea not null unique check (octet_length(token_hash) = 32),
  created_at       timestamptz not null default now(),          -- token issue time
  first_used_at    timestamptz,                                 -- first successful validation
  last_renewed_at  timestamptz not null default now(),
  expires_at       timestamptz not null,
  predecessor_id   uuid references sovereign.visitor_sessions (id) on delete set null,
  rotated_at       timestamptz,                                 -- set on a predecessor at rotation completion
  successor_id     uuid references sovereign.visitor_sessions (id) on delete set null,
  revoked_at       timestamptz,
  revoke_reason    text check (revoke_reason in ('forgotten', 'operator', 'superseded')),
  reuse_count      integer not null default 0 check (reuse_count >= 0),
  last_reuse_at    timestamptz,
  check (expires_at > created_at),
  check ((revoked_at is null) = (revoke_reason is null)),
  check (successor_id is null or rotated_at is not null),   -- successor_id may later become null when the successor row is deleted
  check ((reuse_count = 0) = (last_reuse_at is null)),
  check (reuse_count = 0 or rotated_at is not null)
);
create index visitor_sessions_visitor_id_idx     on sovereign.visitor_sessions (visitor_id);
create index visitor_sessions_predecessor_id_idx on sovereign.visitor_sessions (predecessor_id);
create index visitor_sessions_expires_at_idx     on sovereign.visitor_sessions (expires_at);

create table sovereign.provisioning_nonces (
  nonce_hash  bytea primary key check (octet_length(nonce_hash) = 32),
  visitor_id  uuid references sovereign.visitors (id) on delete cascade,  -- null only inside the creating transaction
  created_at  timestamptz not null default now(),
  mint_count  integer not null default 0 check (mint_count between 0 and 5)
);
create index provisioning_nonces_created_at_idx on sovereign.provisioning_nonces (created_at);

create table private.rate_limit_settings (
  bucket         text primary key check (bucket in ('provision_net_hour', 'provision_net_day',
                                                    'provision_global_hour', 'username_visitor_day')),
  window_length  interval not null check (window_length > interval '0'),
  max_count      integer  not null check (max_count > 0)
);

create table private.rate_limit_counters (
  bucket        text not null references private.rate_limit_settings (bucket),
  subject       bytea not null check (octet_length(subject) in (1, 16, 32)),  -- global key, visitor id, HMAC
  window_start  timestamptz not null,
  hit_count     integer not null check (hit_count > 0),
  primary key (bucket, subject, window_start)
);
create index rate_limit_counters_window_start_idx on private.rate_limit_counters (window_start);
```

Notes:
- **One-to-one visitor ↔ workspace:** `workspaces.visitor_id` is `NOT NULL UNIQUE` (at most one workspace per visitor; never an unowned workspace). "At least one" is guaranteed because the only insert path, `private.provision_visitor`, creates both in one transaction and `sovereign_app` has no `INSERT` on either table. A deferrable constraint trigger asserting "every visitor has a workspace at commit" is recommended as a second guard (D17).
- **Deletion:** deleting a visitor cascades to its workspace, sessions and provisioning nonces. Later domain tables reference `workspaces(id) on delete cascade`.
- **Timestamps:** `updated_at` is maintained by a `BEFORE UPDATE` trigger using `clock_timestamp()`.
- **Session states** are derived from the columns exactly as §5.6 lists; there is no separate state column that could disagree.
- **No slug, `kind` or membership table:** they existed for teams (ADR-0006), which are out of scope. Re-adding teams later is a schema migration.

### 6.4 Grants to `sovereign_app`

```sql
revoke all on schema sovereign, private from public, anon, authenticated, service_role;
revoke all on all tables in schema sovereign, private from public, anon, authenticated, service_role;
grant usage on schema sovereign, private to sovereign_app;

grant select on sovereign.visitors, sovereign.workspaces to sovereign_app;
grant update (username) on sovereign.visitors to sovereign_app;
-- No privileges for sovereign_app on visitor_sessions, provisioning_nonces or any private table.
```

- `EXECUTE` on every function in `private` is revoked from `public`, `anon`, `authenticated` and `service_role`; then each request-path function in §6.6 is granted to `sovereign_app` individually. Maintenance functions are **not** granted to `sovereign_app`.
- Whether Supabase default privileges add grants on new objects must be checked, not assumed: the catalog tests (§9.2) assert the exact grant set.

### 6.5 Visitor context per transaction

```sql
begin;
select set_config('sovereign.session_token_hash', $1, true);  -- $1 = lowercase hex of sha256(token)
-- queries under RLS
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
          when pg_catalog.current_setting('sovereign.session_token_hash', true) ~ '^[0-9a-f]{64}$'
          then pg_catalog.decode(pg_catalog.current_setting('sovereign.session_token_hash', true), 'hex')
        end
    and s.revoked_at is null
    and pg_catalog.now() < s.expires_at;
$$;
```

- It resolves exactly the sessions `touch_session` reports as `active` or `rotated_grace`, and is read-only (no first-use completion, no renewal, no reuse recording).
- **Why the token hash, not a visitor id:** if the context were a visitor id, any SQL run as `sovereign_app` (for example through an SQL-injection bug) could set any id and satisfy every policy. With the hash, the context resolves only for a live session, and `sovereign_app` cannot read `visitor_sessions`. Arbitrary SQL as `sovereign_app` can therefore act only as visitors whose token hash it already knows. This narrows, but does not remove, the risk (§6.7).
- **Pooling:** `set_config(..., true)` is transaction-local; PostgreSQL discards it at commit or rollback, so it cannot reach the next transaction on a pooled connection. The data-access helper (`withVisitor`) always opens an explicit transaction and sets the context first. Session-level `SET`/`set_config(..., false)` is forbidden in application code (static test, §9.1). Whether Supabase's pooler in transaction mode preserves this as expected is a hosted prerequisite (§6.10).

### 6.6 Functions
All functions: in `private`; `SECURITY DEFINER`; `set search_path = ''`; every object reference fully qualified (`sovereign.visitors`, `pg_catalog.now()`); **no dynamic SQL** (no `EXECUTE`, no `format()`-built statements, no object names from arguments); arguments typed and validated as the first statements (hash and nonce lengths, username format, bucket membership); generic errors without internal detail.

| Function | Owner | `EXECUTE` granted to | Behaviour |
|---|---|---|---|
| `current_visitor_id() → uuid` | `sovereign_identity_owner` | `sovereign_app` | §6.5. `STABLE`, read-only. |
| `provision_visitor(nonce_hash bytea, token_hash bytea, username text) → (visitor_id uuid, workspace_id uuid, outcome text)` | `sovereign_identity_owner` | `sovereign_app` | §5.10. `outcome ∈ {created, joined}`; raises `provisioning_expired` or a generic validation error. |
| `touch_session(token_hash bytea, renew boolean) → (state text, session_id uuid, visitor_id uuid, expires_at timestamptz, renewal_due boolean, rotation_due boolean)` | `sovereign_identity_owner` | `sovereign_app` | Exactly §5.6. Locks predecessor before successor. |
| `rotate_session(old_hash bytea, new_hash bytea) → text` | `sovereign_identity_owner` | `sovereign_app` | §5.5. Returns `issued`, `pending`, `not_due` or `not_active`. Only `issued` lets the caller set a cookie. |
| `revoke_session(token_hash bytea) → boolean` | `sovereign_identity_owner` | `sovereign_app` | Sets `revoked_at = now()`, `revoke_reason = 'forgotten'` on an `active` or `rotated_grace` session. |
| `consume_rate_limit(bucket text, subject bytea) → boolean` | `sovereign_identity_owner` | `sovereign_app` | §5.11. Reads `rate_limit_settings`; missing bucket → `false`. |
| `set_updated_at()` (trigger) | `sovereign_identity_owner` | nobody | Timestamps. |
| `cleanup_sessions()`, `cleanup_visitors()`, `cleanup_nonces()`, `cleanup_rate_limits()` | `sovereign_maintenance_owner` | the maintenance executor (§6.8, D11) | Retention. Not granted to `sovereign_app`. |

**Privileges held by `sovereign_identity_owner`** (granted by `postgres`; nothing else):
- `sovereign.visitors`: `SELECT`, `INSERT`, `UPDATE (last_seen_at, updated_at)`.
- `sovereign.workspaces`: `INSERT`.
- `sovereign.visitor_sessions`: `SELECT`, `INSERT`, `UPDATE (first_used_at, last_renewed_at, expires_at, rotated_at, successor_id, revoked_at, revoke_reason, reuse_count, last_reuse_at)`.
- `sovereign.provisioning_nonces`: `SELECT`, `INSERT`, `UPDATE (visitor_id, mint_count)`.
- `private.rate_limit_settings`: `SELECT`. `private.rate_limit_counters`: `SELECT`, `INSERT`, `UPDATE (hit_count)`.
- `USAGE` on schemas `sovereign` and `private`. No `DELETE` anywhere.
- Because the owner has no `SELECT` on `sovereign.workspaces`, `provision_visitor` generates the workspace id with `gen_random_uuid()` into a variable and inserts it explicitly (no `RETURNING` on that table).

**Privileges held by `sovereign_maintenance_owner`:** `SELECT` and `DELETE` on `sovereign.visitors`, `sovereign.visitor_sessions`, `sovereign.provisioning_nonces`, `private.rate_limit_counters`; `USAGE` on both schemas.

**RLS and the owner roles:** the owners are not table owners and do not have `BYPASSRLS`, so RLS applies to them. They receive owner-scoped policies (§6.7) covering exactly the commands above. This is an explicit, testable grant of cross-visitor access to the functions only — not a bypass, and not available to `sovereign_app`.

**Why a dedicated owner is safer, concretely:** a bug in a function body (for example a missing `where`) can only use the privileges listed above — it cannot drop or alter tables, read other schemas, use `DELETE` (request path), or reach anything `postgres` can. Owning the functions with `postgres` would give every function body `postgres`'s privileges.

### 6.7 RLS policies

```sql
alter table sovereign.visitors            enable row level security;
alter table sovereign.workspaces          enable row level security;
alter table sovereign.visitor_sessions    enable row level security;
alter table sovereign.provisioning_nonces enable row level security;
alter table private.rate_limit_settings   enable row level security;
alter table private.rate_limit_counters   enable row level security;

-- Application role: own rows only.
create policy visitors_select_self on sovereign.visitors
  for select to sovereign_app using (id = (select private.current_visitor_id()));
create policy visitors_update_self on sovereign.visitors
  for update to sovereign_app
  using (id = (select private.current_visitor_id()))
  with check (id = (select private.current_visitor_id()));
create policy workspaces_select_own on sovereign.workspaces
  for select to sovereign_app using (visitor_id = (select private.current_visitor_id()));

-- Function owner: one policy per table and command it needs (using/with check true).
-- visitors: select, insert, update; workspaces: insert; visitor_sessions: select, insert, update;
-- provisioning_nonces: select, insert, update; rate_limit_settings: select;
-- rate_limit_counters: select, insert, update.
-- Maintenance owner: select and delete on visitors, visitor_sessions, provisioning_nonces,
-- rate_limit_counters.
-- No policies for anon, authenticated or service_role.
```

What RLS guarantees here:
- **Does:** an application bug that forgets a filter, or passes another visitor's id, cannot read or change another visitor's rows through `sovereign_app`, because every row is filtered by the live session behind the transaction's context.
- **Does not:** protect against a compromised web server, which receives live session cookies and holds the `sovereign_app` password. A custom PostgreSQL setting is not independently trustworthy: any code running SQL as `sovereign_app` can call `set_config`. The token-hash context limits such code to visitors whose hash it knows; it is defence in depth, not isolation from the server.
- `postgres` owns the tables and is used only for migrations and operator work.

### 6.8 Retention and cleanup (D11, D20)
| Data | Deleted when | Function |
|---|---|---|
| Session rows | `validity_end < now() - 30 days` | `cleanup_sessions()` |
| Visitors (with workspace, sessions, nonces) | **Abandoned**: `created_at < now() - 1 day` **and** no session of the visitor with `validity_end > now() - 30 days` | `cleanup_visitors()` |
| Provisioning nonces | `created_at < now() - 1 day` | `cleanup_nonces()` |
| Rate-limit counters | `window_start < now() - 2 days` | `cleanup_rate_limits()` |

Where `validity_end` of a session is `least(revoked_at, expires_at)` (PostgreSQL's `least` ignores a null `revoked_at`, so it is `expires_at` for unrevoked sessions). A rotated predecessor's `expires_at` is its grace end, so the same formula covers it.

Visitor cleanup safety:
- A session in state `active` or `rotated_grace` has `revoked_at is null` and `expires_at > now()`, so its `validity_end > now()`: the predicate can never select a visitor that holds one.
- Validity only moves forward through renewal, and renewal applies only to `active` sessions. An expired or revoked session can never become valid again.
- Sessions are created for an existing visitor only by rotation (requires an `active` predecessor, so the visitor is not abandoned) and by nonce joining (requires the visitor to be younger than 10 minutes, while cleanup requires older than 1 day). So no session can be created for a visitor the predicate selects.
- These three facts are what make cleanup safe; they do not depend on lock timing. `cleanup_visitors()` deletes candidates one at a time with `delete … where id = $1 and <predicate>` so the predicate is evaluated again at deletion. **Any future feature that creates sessions for existing visitors must re-examine this argument.**
- Orphans from §5.10's edge case are abandoned visitors whose only session was never used; they follow the same rule (deleted about 210 days after creation). Deleting them earlier is not possible without risking a one-visit visitor who returns later.

Executor: how cleanup runs on a schedule (Supabase `pg_cron`, if available on the project, or an operator-run job) is an **unresolved prerequisite** (D11). `sovereign_app` never runs cleanup.

**Test-data cleanup (D20):** integration tests record every visitor they create. After the run, the test harness deletes exactly those visitor ids through the existing development-only SQL test connection (`SOVEREIGN_SQL_TEST_DB_URL`, root `.env`, already used by `pnpm test:rls`), with the test username prefix as a second guard, and then asserts that no test visitors, workspaces, sessions or nonces remain. No cleanup privilege is granted to `sovereign_app`, and no test-only function exists in production migrations.

### 6.9 Supabase settings
- **Data API:** M2 does not need it — the browser never reads visitor data from Supabase, and `sovereign`/`private` are never exposed. Whether to **disable the Data API entirely** is a separate decision (D18), because it affects later features.
- **Supabase Auth:** not used. New sign-ups and anonymous sign-ins are disabled in the dashboard so no `auth.users` rows can be created by holders of the publishable key.
- The browser bundle carries no Supabase URL or key for visitor data.

### 6.10 Hosted prerequisites and alternatives
These must be verified on the hosted development project before implementation is considered viable:
1. `postgres` can create `NOLOGIN` roles, grant them table privileges and `ALTER FUNCTION … OWNER TO` them (PostgreSQL 16+ requires the creator to hold `SET` membership in the new role).
2. A custom login role (`sovereign_app`) can connect through Supabase's pooler (the existing `[VERIFY AT M2]` item for custom roles in DATA-MODEL §G.6).
3. Transaction-mode pooling keeps transaction-local settings for the transaction's duration and clears them afterwards (verified by the pool-isolation tests, §9.3).
4. A scheduler for maintenance functions exists, or an operator-run job is accepted (D11).

Fallbacks (each needs approval; none is assumed):
- (1) fails → functions owned by `postgres`. This is explicitly **weaker**: function bodies then run with `postgres`'s privileges.
- (2) fails → direct connection or session-mode pooler with a small server-side pool (same transaction-local rule), or a dedicated pooler user mapped to `sovereign_app`.
- (3) fails → pass the token hash as an explicit argument to SECURITY DEFINER data-access functions for every operation (no setting), at the cost of a function-only API.
- **Not acceptable:** routine `service_role` or `postgres` access from the web server.

---

## 7. Application architecture

### 7.1 First-visit provisioning
1. A `GET` to an app path without session or nonce cookie sets the nonce cookie (§5.10) and renders the app shell without a visitor.
2. The shell's client component calls the `provisionVisitor` Server Action (`POST`).
3. The action runs §5.11's order: CSRF checks, nonce present, no session cookie, trusted client IP, per-network limits, global breaker.
4. It generates the session token and username, hashes token and nonce, and calls `private.provision_visitor(nonce_hash, token_hash, username)`. On a username collision it retries with a new name (≤ 5).
5. On `created` or `joined` it sets the session cookie, deletes the nonce cookie and returns the workspace view model. On `provisioning_expired` it returns "reload".

### 7.2 Returning-visitor resolution and refresh
- **Render path (`GET`):** `requireVisitor()` (server-only) reads the cookie and calls `touch_session(hash, renew = false)`. `active` and `rotated_grace` render the workspace; other states render the §2.6 notice (or provisioning when no cookie was sent). The result includes `renewal_due` and `rotation_due`.
- **Refresh path (`POST`):** when either flag is true, the shell's client component calls a `refreshSession` Server Action. It runs the CSRF checks, calls `touch_session(hash, renew = true)` and, if `rotation_due`, generates a new token and calls `rotate_session(old_hash, new_hash)`. It sets the cookie only when renewal happened (same token, new `Max-Age`) or when rotation returned `issued` (new token). No other outcome writes the cookie.

### 7.3 Username updates
`updateUsername` Server Action: CSRF checks, `requireVisitor()`, server-side validation, reserved-name check, `consume_rate_limit('username_visitor_day', visitor_id)`, then `update sovereign.visitors set username = $1` inside `withVisitor` (RLS restricts it to the visitor's own row). Unique violation → "That name is taken."

### 7.4 Workspace retrieval and data operations
- `getPersonalWorkspace()` reads `sovereign.workspaces` inside `withVisitor`; RLS returns at most the visitor's own row.
- Later domain operations use the same pattern: services in `packages/core` take an `Actor` resolved server-side; the `@sovereign/db` adapter runs them inside `withVisitor`.

### 7.5 Cookie issuance and revocation summary
| Event | Cookie written |
|---|---|
| Provisioning `created`/`joined` | Session cookie set; nonce cookie deleted |
| Refresh with renewal | Same token, new `Max-Age` |
| Rotation `issued` | New token |
| "Forget this browser" | Session cookie deleted after `revoke_session` |
| "Start a new workspace" (explicit, §2.6) | Old session cookie removed; normal provisioning follows |
| Any validation failure | **Nothing** |

### 7.6 Errors
Typed results `{ ok, data } | { ok: false, error }`, `error ∈ { no_visitor, invalid_session, reload, cookies_required, rate_limited, validation, conflict, unavailable }`. Client messages are generic. Logs carry no tokens, nonces, hashes, cookie values or IP addresses; they may carry visitor ids, session row ids, state names and bucket names.

---

## 8. Mapping to the current source (read-only inventory)

### 8.1 Status of the existing account-based M2 work
Based on the evidence previously recorded in this project's working session — **not re-verified for this revision, and no Git commands were run**:
- At the start of that session, `HEAD` was `10013c0` ("feat: checkpoint Sovereign landing page and realm work"), and the account-based M2 files (for example `apps/web/src/app/auth/`, `apps/web/src/lib/`, `packages/db/supabase/`) were listed by `git status` as **untracked**.
- Later recorded measurements showed the same `HEAD`, zero staged files and an unchanged working-tree status hash.
- On that evidence the account-based M2 work is **not contained in any commit on record**. It has not been merged or pushed. It has **never been applied to a hosted database**.
- If the work has since been committed locally (for example in a checkpoint made outside that session), this paragraph must be updated from verified Git evidence; it should then read "committed locally, not necessarily merged or pushed, never applied to a hosted database".

### 8.2 Existing files and their fate under this specification
| Area | Files | Fate |
|---|---|---|
| Auth pages and actions | `apps/web/src/app/auth/{actions.ts, AuthShell.tsx, auth.css, callback/route.ts, sign-in, sign-up, verify-email, reset-password, update-password}` | Removed. `AuthShell.tsx`/`auth.css` styling may be reused for the §2.6 notice. |
| Auth helpers | `apps/web/src/lib/auth/{gate.ts, auth-outcome.ts, next-path.ts}`, `apps/web/src/server/auth.ts` | Replaced by `server/visitor.ts`. |
| Supabase clients | `apps/web/src/lib/supabase/{browser.ts, server.ts, session.ts, config.ts}` | Removed from the visitor path. |
| Proxy | `apps/web/src/proxy.ts` | Repurposed to set the provisioning-nonce cookie only (D19). |
| Canonical origin | `apps/web/src/lib/site-origin.ts` | Kept; source of the Origin check. |
| Workspace page | `apps/web/src/app/(app)/workspace/page.tsx` | Rewritten against `requireVisitor`/`getPersonalWorkspace`. |
| Composition root | `apps/web/src/server/composition.ts` | Gains the database adapter wiring. |
| DB configuration | `packages/db/src/config.ts` (`parseWorkerDatabaseUrl` rejects over-privileged roles) | Pattern reused for an application database URL that must use `sovereign_app`. |
| Env declarations | `apps/web/src/env.d.ts` | Gains server-only names for the application database URL (D16), `SOVEREIGN_RATE_LIMIT_KEY` and `SOVEREIGN_CLIENT_IP_HEADER`; `NEXT_PUBLIC_SUPABASE_*` not needed for M2. |
| Migrations | `packages/db/supabase/migrations/20261112120000_m2_schemas_privileges.sql`, `20261112120100_m2_identity.sql` | Depend on `authenticated`, `auth.users`, `auth.uid()`. To be **replaced**, not amended — requires approval. |
| pgTAP | `packages/db/supabase/tests/{identity_signup, identity_rls, catalog}.test.sql` | Replaced (§9.2). The control `_selftest/deliberate_failure.test.sql`, the manifest rules and the runner (`scripts/run-sql-tests.mjs`, `scripts/tap.mjs`) are kept. |
| Integration tests | `apps/web/test/integration/{auth.test.ts, support/admin.ts, support/env.ts, support/user.ts}` | Replaced (§9.3). `support/diagnostics.ts` (sanitised cleanup reports) is reusable. The service-role key is no longer needed. |
| Guards | `apps/web/test/service-role-isolation.test.ts`, `scripts/check-client-bundle.mjs`, ESLint rules | Kept; extended so the application database URL and the rate-limit key never reach client code or bundles. |
| Dependencies | `@supabase/ssr`, `@supabase/supabase-js` (web), `supabase` CLI (db) | JS clients unused for M2. A PostgreSQL driver is required — a new dependency (D16). |

### 8.3 Proposed new structure (not created)
- `apps/web/src/server/visitor.ts` (`server-only`): cookies, token and nonce generation and hashing, `requireVisitor`, `withVisitor`, CSRF checks, client-IP extraction.
- `apps/web/src/app/(app)/workspace/actions.ts`: `provisionVisitor`, `refreshSession`, `updateUsername`, `forgetThisBrowser`, `startNewWorkspace`.
- `apps/web/src/app/(app)/workspace/SessionKeeper.tsx` (client): provisioning and refresh calls.
- `packages/db/src/app-database.ts`: driver wrapper exposing only `withVisitor` and the §6.6 request-path functions.

---

## 9. Testing requirements
None of these tests exists. Counts are intentionally not stated.

### 9.1 Unit (offline, Vitest)
- Token and nonce generation: 32 bytes, base64url, 43 characters; hashes are SHA-256 of the raw bytes.
- Cookie attributes for both cookies, production and `http://localhost` variants (§5.2); session `Max-Age` equals remaining lifetime.
- CSRF checks: missing `Origin` rejected; foreign origin rejected; canonical accepted; `Sec-Fetch-Site: cross-site` rejected; `Host`/`X-Forwarded-Host` ignored.
- Client IP (§5.12): with no configured header, any `X-Forwarded-For`, `Forwarded` or `X-Real-IP` is ignored and the result is "untrusted"; with a configured header, a single valid address is accepted, lists/ports/garbage are untrusted; IPv6 reduced to /64; production without a trusted IP refuses provisioning; the loopback development branch never activates for a non-loopback origin.
- Provisioning order (§5.11): each failing precondition stops before the database call with the generic response.
- `requireVisitor`: every `touch_session` state maps to the right render outcome; malformed cookie makes no database call.
- Cookie writes: no validation failure ever produces `Set-Cookie` for the session cookie; refresh writes only on renewal or `issued`.
- Username validation, generator validity, reserved names including the test prefix.
- Logs: sentinel tokens, nonces, hashes and IPs never appear in any log line.
- Static guards: no session-level `SET` or `set_config(..., false)` in application SQL; the application database URL and rate-limit key never appear in client bundles.

### 9.2 Database / RLS (pgTAP via `pnpm test:rls`, control first; everything rolled back)
- **Session states:** for each of `invalid_input`, `unknown`, `revoked`, `rotated_grace`, `rotated_reused`, `expired`, `active`, `touch_session` returns the §5.6 result and performs exactly the listed writes; `current_visitor_id()` resolves only `active` and `rotated_grace`.
- **Boundaries:** `now()` is fixed within the test transaction, so the test arranges a rotated predecessor's `expires_at` relative to it: `now() + interval '1 microsecond'` → `rotated_grace`; exactly `now()` → `rotated_reused`. The same pattern checks `active` versus `expired`.
- **Renewal:** `active` renews only with `renew = true` and when 24 hours have passed; a rotated predecessor is never renewed, even with `renew = true`.
- **Reuse:** repeated presentation after grace increments `reuse_count` each time and sets `last_reuse_at`; presentation during grace records nothing.
- **Rotation:** `issued` creates one successor and leaves the predecessor `active`; first use of the successor completes rotation exactly once; a second `rotate_session` while a recent pending successor exists returns `pending`; an old pending successor is `superseded`; superseded successors are `revoked`.
- **Provisioning:** `created` creates exactly one visitor, workspace, session and nonce row; `joined` adds a session to the same visitor and increments `mint_count`; expired window and `mint_count = 5` raise; invalid input creates nothing; a forced failure leaves no partial rows.
- **One-to-one:** a second workspace for a visitor fails; a workspace without a visitor fails; the deferrable "visitor has a workspace" check, if approved (D17).
- **Isolation as `sovereign_app`:** A's context cannot select or update B's visitor or workspace rows (zero rows, B unchanged).
- **Spoofing:** setting the context to a fabricated hash, to B's visitor id, or to malformed text resolves to no visitor; `sovereign_app` cannot select `visitor_sessions`, `provisioning_nonces` or any `private` table, cannot insert into `visitors`/`workspaces`, and cannot execute maintenance functions.
- **Transaction-local context:** after commit and after rollback, the setting is empty in the same connection.
- **Cleanup:** `cleanup_visitors` never deletes a visitor with an `active` or `rotated_grace` session (including a session renewed moments before), deletes an abandoned visitor with cascade, and leaves a visitor younger than 1 day untouched; session, nonce and counter cleanup follow §6.8.
- **Rate limits:** `consume_rate_limit` counts atomically, denies above `max_count`, starts a new window, denies a bucket missing from settings.
- **Catalog:**
  - `sovereign_app`, `sovereign_identity_owner`, `sovereign_maintenance_owner`: exact attributes (no `SUPERUSER`, no `BYPASSRLS`, owners `NOLOGIN`).
  - Every function in `private`: `prosecdef`, `proconfig = {search_path=""}`, expected `proowner`, no `EXECUTE` or `format(` in `prosrc`, execute ACL exactly as §6.6.
  - Exact table and column privileges for the three roles; none for `anon`, `authenticated`, `service_role`.
  - RLS enabled on all six tables; the exact policy set (§6.7), including owner-scoped policies and their commands.

### 9.3 Visitor-session integration (Vitest against the hosted development database as `sovereign_app`)
- **Transaction-pool isolation:** with the client pool forced to one connection, a transaction with visitor A's context followed by one without context sees nothing in the second; many interleaved concurrent A/B transactions never see each other's rows.
- **Concurrent provisioning, one nonce:** N parallel `provision_visitor` calls with the same nonce produce one visitor and N sessions (`created` once, `joined` otherwise).
- **Concurrent provisioning, different nonces:** produce different visitors, each with exactly one workspace.
- **Old token during and after grace:** repeated use within grace succeeds; after grace each use is `rotated_reused` and is recorded; the successor keeps working throughout.
- **Rotation races:** parallel refreshes with one predecessor produce exactly one `issued`; parallel first uses of the successor complete rotation exactly once.
- **Lost rotation response:** an issued successor never used → predecessor stays `active`; a refresh after 10 minutes supersedes and issues a new successor.
- **Rate limits across instances:** two independent connection pools (simulating two application instances) consuming the same bucket and subject share one count.
- **Cleanup safety:** with cleanup run concurrently with session activity, no visitor holding a valid session is deleted.
- **Test-data cleanup:** every visitor created is recorded immediately and removed through the D20 path; a final check finds no remaining test visitors, workspaces, sessions or nonces; failures are reported with sanitised ids only.

### 9.4 Server Action / API integration (Vitest, server code against the development database)
- `provisionVisitor`: duplicated and retried requests with one nonce → one visitor; missing nonce → `reload`; session cookie present → `reload` without any database write; spoofed forwarding headers ignored; missing trusted IP in production configuration → generic `unavailable`; per-network and global limits enforced across two server instances sharing the database.
- `refreshSession`: renewal re-issues the cookie with matching `Max-Age`; rotation `issued` sets a new token; `pending`, `not_due`, `not_active` set no cookie.
- `updateUsername`: taken → conflict; invalid → validation; limit enforced; another visitor unchanged.
- `forgetThisBrowser`: session revoked, cookie deleted.
- Spoofed ids in form data or headers are ignored everywhere.

### 9.5 Browser (manual at G7 until an E2E tool is approved; Playwright is listed in ADR-0001 but not installed)
- Fresh profile → `/workspace` → workspace appears without any form; both cookies' attributes correct; nonce cookie gone after success.
- Several tabs opened at once in a fresh profile → one username shared by all tabs.
- Reload during provisioning → same visitor.
- Reload and full browser restart → same username and workspace.
- Username edit persists across restart.
- Delete the session cookie → automatic provisioning of a new visitor; with a tampered or revoked cookie → the §2.6 notice; the cookie is not replaced until "Start a new workspace".
- Two profiles A and B → each sees only its own workspace.
- A page on another origin auto-submitting a `POST` to a Server Action is rejected without state change.
- "Forget this browser" → warning, then access ends.

### 9.6 Rotation in the browser
Rotation needs a 30-day-old token; it is verified in §9.2–§9.4 with arranged timestamps rather than manually.

### 9.7 Test-data rules
Test visitors use the reserved prefix `zz-test-` (database-level tests pass it directly; server-level tests inject a test username generator that produces it), are created only by the test harness, and are deleted per D20 by recorded id.

---

## 10. Consequences for the rest of the blueprint (deferred; not part of M2)
- **Cost and abuse for agent runs (M4b/M5):** the PRD's verified-email gate disappears. A replacement is required before any run can spend money (per-visitor and per-workspace budgets, provisioning controls, possibly a human-verification challenge).
- **Realtime (ADR-0005):** private channels authorised by Supabase Auth JWTs no longer apply. Options: server-sent events from the Sovereign server, or server-minted short-lived tokens for Realtime (a new signing credential).
- **Operators, admin and moderation:** need an operator identity separate from visitor identity.
- **Public profiles, publishing, reports:** usernames are not proof of a person; impersonation and squatting policies are needed if usernames become public.
- **Teams (ADR-0006):** dropped from the schema; re-adding them is a schema migration.
- **Storage:** any use must go through the server.
- **Privacy:** retention values (D11) and the lawful basis for storing client-network HMACs (D13).

---

## 11. Decision register

Labels: **AR** = accepted product requirement · **RD** = recommended design awaiting user approval · **UP** = unresolved technical prerequisite · **DL** = deferred to a later milestone.

| # | Decision | Label | Notes |
|---|---|---|---|
| D1 | Accountless identity: server-generated visitor id, HttpOnly bearer session, no Supabase Auth for visitors; username display-only; same-profile persistence only; no cross-device recovery; cookie loss may lose the workspace | **AR** (2026-11-12) | |
| D2 | Identity tables in non-exposed schema `sovereign`; support tables and functions in `private` | RD | |
| D3 | Username format `^[a-z0-9][a-z0-9_-]{2,23}$`, generator `adjective-noun-NN` | RD | |
| D4 | Unique usernames (case-insensitive by lowercase-only format) | RD | |
| D5 | Reserved-name denylist including the test prefix `zz-test-` | RD | |
| D6 | No visitor rows from `GET`; provisioning only by same-origin `POST` | RD | |
| D7 | Invalid cookie → notice + explicit "Start a new workspace"; never auto-replace; "Forget this browser" with warning | RD | |
| D8 | 180-day sliding idle lifetime; renewal ≥ 24 h apart, only for `active`, only in the refresh `POST`, cookie re-issued with matching `Max-Age`; no absolute cap | RD | Inactivity > 180 days loses the workspace |
| D9 | Cookie names `__Host-sovereign_session` / `__Host-sovereign_provision`; unprefixed names only on `http://localhost` | RD | |
| D10 | Rotation every 30 days; completion on successor's first use; 120 s grace from completion, strict boundary; ≤ 1 recent pending successor (10 min); reuse recorded, no visitor-wide revocation | RD | |
| D11 | Retention: sessions 30 days after validity ends; abandoned visitors (older than 1 day, no session valid within 30 days) deleted with locking re-check; nonces 1 day; counters 2 days | RD (values and rules) · **UP** (scheduler: `pg_cron` availability or operator job) | |
| D12 | Synchronizer CSRF token | **DL** | Origin + `Sec-Fetch-Site` + `SameSite=Lax` required in M2 |
| D13 | Rate limits and global circuit breaker | **AR** that they are mandatory before public deployment (user directive, 2026-11-12) · RD for the values in §5.11 and the Postgres-backed design | Shared-address trade-off needs approval |
| D14 | Trusted client-IP header | **UP** | Provider not selected; fail closed until configured (§5.12) |
| D15 | SECURITY DEFINER functions owned by `NOLOGIN` roles `sovereign_identity_owner` and `sovereign_maintenance_owner` with explicit privileges and owner-scoped RLS | RD · **UP** (hosted role-creation and ownership support, §6.10) | Fallback to `postgres` ownership is weaker and needs separate approval |
| D16 | PostgreSQL driver (new dependency), pooler mode, connection limit, application database URL variable name | **UP** — not approved | Needs compatibility review and dependency approval |
| D17 | Deferrable "visitor has a workspace" constraint trigger | RD | |
| D18 | Data API: not needed by M2 (design fact); disabling it entirely | Disabling: **DL** | Decide when Realtime/Storage plans are set |
| D19 | Proxy sets only the provisioning-nonce cookie (no database access); session renewal and rotation cookies written by the refresh Server Action | RD · **UP** (verify the Next.js 16 proxy behaviour for cookie writes on prefetch and RSC requests before relying on it) | Proxy database access not recommended |
| D20 | Test-data cleanup through the development-only SQL test connection, deleting recorded ids with the test prefix as a second guard; no cleanup rights for `sovereign_app`; no test functions in production migrations | RD | |
| D21 | Provisioning nonce: 10-minute life, ≤ 5 mints, per-browser de-duplication | RD | |

---

## 12. Documents that must change before implementation (not changed by this revision)
- A new ADR superseding the affected parts of ADR-0001, ADR-0005, ADR-0006 and ADR-0011.
- `docs/PRD.md` (accounts, verified email, `/login`, `/signup`, `/reset-password`, `/auth/callback`).
- `docs/ARCHITECTURE.md` §F.1, §F.2, §F.4, §F.7, §H.1, §H.2.
- `docs/DATA-MODEL.md` §G.1, §G.2, §G.3, §G.5, §G.7, §G.8. The G.8 "as implemented" line already says migration 1 created "fail-closed default privileges", which the current migration file does not do.
- `docs/SECURITY.md` §I.1, §I.2, §I.3, §I.5, §I.7, §I.8, §I.9.
- `docs/ROADMAP.md` M2 exit criteria, status row and the `pnpm test:integration` description.
- The M2 hosted-verification runbook (revision 6 blocked: its revision 5 source exists only in a working session, not as a file).

## 13. Hosted verification
The hosted-verification runbook must follow this specification once approved, with the owner-defined gates G1–G9. Its test expectations can be stated only after the tests in §9 exist. §6.10 lists the hosted facts it must verify first.
