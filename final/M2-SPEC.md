# M2 — Accountless visitor identity and personal workspace: architecture specification

**M2 Architecture Specification — Revision 3** (2026-11-13). Revisions 1 and 2 are superseded by this text.

Status: **specification only. Not approved for implementation.** No code, migration or test described here exists yet. Revising this document does not approve any design decision in it.
Supersedes the account-based M2 design (Supabase Auth sign-up/sign-in, verified email, `auth.users`-keyed identity) for M2 only.
Decision register: §11. Each entry is labelled as an **accepted product requirement**, a **recommended design awaiting user approval**, an **unresolved technical prerequisite**, or **deferred to a later milestone**. Only the first label means approved.

Change rule (docs/README.md): before implementation this specification needs (1) a new ADR in `docs/DECISIONS/` superseding the affected parts of ADR-0001, ADR-0005, ADR-0006 and ADR-0011, (2) updates to the living documents listed in §12, and (3) a ROADMAP status update. None of those has been made; this document does not edit them.

### Changes from revision 2
- §5.5, §5.6, §6.3, §6.5, §6.6: **initial sessions and rotation successors are separate paths**, selected by an immutable `origin` column. Initial first use never touches another row; successor first use locks and completes its predecessor, with lineage checks that fail closed (new state `invalid_lineage`). Pending successors are not accepted by the database context until completed.
- §2.1, §5.10: the provisioning guarantee is stated **per nonce**. Different nonces may create different visitors; tab coordination is only a mitigation; cleanup accounts for it.
- §5.11, §6.3, §6.6: rate limits are enforced **inside** the database functions with a canonical subject per bucket. The generic `consume_rate_limit` function is removed; `sovereign_app` can no longer choose a subject or skip the global breaker. Username changes go through a function that derives the visitor from the transaction context.
- §5.6, §6.3, §6.6: reuse and lineage events are recorded in a restricted database table; `touch_session` returns only a session id for them; event-write failure fails closed.
- §6.2, §6.8: test cleanup uses a development-only role and target marker, with identity checks that abort before any delete.
- §2.6, §7.6: the explicit "Start a new workspace" flow clears the invalid cookie first, then obtains a nonce through the normal `GET`.
- §6.6: maintenance privileges are column-level; cleanup cannot read token or nonce hashes.

---

## 1. Goal and principles

Sovereign is **accountless**. A visitor opens Sovereign and uses it immediately. There is no sign-up, sign-in, password, email confirmation or password recovery anywhere in the product.

1. **No hidden accounts.** Supabase Auth is not the foundation of visitor identity: no `auth.users` rows, no `auth.uid()`, no Supabase Auth sessions, no anonymous Supabase sign-ins.
2. **The browser holds a credential, nothing else.** A random, opaque session token in an HttpOnly cookie is the only proof of workspace ownership. The username and the visitor id are never credentials.
3. **One backend, one door.** Browser code never talks to the database or to Supabase APIs for visitor data. Every read and write goes through the Sovereign server (Next.js server code), which establishes the visitor context.
4. **Least privilege, honestly described.** The server connects with a dedicated database role without `BYPASSRLS`. RLS is defence in depth against application bugs; it is **not** protection against a compromised server or against arbitrary SQL run as the application role (§6.5, §6.7).
5. **Fail closed.** A missing, malformed, expired, rotated-out, revoked or inconsistent session yields "no visitor", never another visitor's data. Missing security configuration disables the affected feature instead of weakening it.
6. **Accepted limitation, stated plainly.** Losing the cookie or the browser profile means losing access to the workspace. There is no recovery.

Out of scope for M2: cross-device recovery or transfer, email of any kind, operator/admin identity, public profiles, Realtime, agent runs and budgets, teams. §10 lists the consequences for later milestones.

---

## 2. Product behaviour

### 2.1 First visit
- Public pages (landing, Realm, Orders and the other `(realm)` routes) need no visitor identity and create none.
- When a browser with neither a session cookie nor a provisioning-nonce cookie enters the application area (M2: `/workspace`):
  1. The response to that `GET` sets a short-lived **provisioning nonce cookie** (§5.10). No database row is created by any `GET`.
  2. The app shell's client component sends one same-origin `POST` (Server Action) automatically, without any form or click.
  3. The server creates the visitor, the personal workspace and a session atomically (§7.1), sets the session cookie and returns the workspace view.
- **Guarantee:** all provisioning requests that carry the **same nonce** resolve to the **same visitor** (§5.10). Requests carrying **different** nonces may create different visitors — for example when several tabs load at almost the same moment and their responses interleave. One visitor per browser is the usual outcome, not a guarantee.
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
- The server never deletes or overwrites a session cookie because it failed to validate. Only the explicit visitor actions "Start a new workspace" (§7.6) and "Forget this browser" (§5.8), and successful renewal or rotation (§5.4, §5.5), change it.
- If a session cookie **was sent** but does not resolve to a visitor (any state other than `active` or `rotated_grace`, §5.6), the app area shows a notice ("This browser's previous Sovereign workspace can no longer be opened") and an explicit "Start a new workspace" action. No visitor is provisioned automatically while that cookie is present.
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
| **Session token hash** | SHA-256 of the token (32 bytes) | `sovereign.visitor_sessions.token_hash` | Sensitive (lookup key) | Session lookup and the database visitor context. Never logged; not readable by `sovereign_app` or the maintenance owner. |
| **Session id** | `uuid` of a `visitor_sessions` row | Database; returned only for reuse and lineage events | Not secret, **not a credential** | Correlating server logs with recorded events. |
| **Provisioning nonce** | 256-bit random value, ≤ 10-minute life | Browser cookie only; its SHA-256 in `sovereign.provisioning_nonces` | **Yes**, short-lived | De-duplicating provisioning requests that carry the same nonce (§5.10). |
| **Network subject** | Key-version byte + HMAC-SHA-256 of the trusted client network (33 bytes) | Rate-limit counters only | Not reversible without the server key | Per-network provisioning limits (§5.11). |

Rules:
- No request parameter, form field or header carries a visitor id or workspace id that the server trusts. The server derives both from the session.
- Visitor ids and session ids may appear in server logs. Tokens, nonces, their hashes, cookie values, IP addresses and network subjects never do.

---

## 4. Visitor identity model

### 4.1 Visitor id
Generated by `gen_random_uuid()` inside the provisioning function. The client never proposes it.

### 4.2 Initial username (D3–D5, recommended)
- Generated server-side from a fixed word list plus a random number (`adjective-noun-NN`) using `node:crypto` randomness.
- Format: 3–24 characters, lowercase `a–z`, digits, `-`, `_`, starting with a letter or digit; enforced by a database `CHECK`, by the database functions and by server-side validation before the database call.
- Uniqueness: `UNIQUE`; the lowercase-only format makes it case-insensitive. On a collision the server retries the provisioning call with a new name, at most 5 times, then returns a generic error. Each attempt counts against the rate limits (§5.11).
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
  - for a session in state `active` (§5.6) — never for a rotated predecessor or any other state;
  - when `now() - last_renewed_at >= 24 hours`;
  - in the **refresh request** (§7.2), which also re-issues the cookie with a matching `Max-Age`. Render-time validation never renews, so the database expiry and the cookie expiry cannot drift apart.
- **No absolute cap.** An absolute cap would destroy the workspaces of active visitors, because there is no recovery. Browsers may cap cookie lifetime (e.g. Chrome at 400 days); renewal keeps active visitors far inside that.
- Inactivity longer than 180 days means permanent loss of access (§2.5).

### 5.5 Session origins, first use and rotation (D10)
Every session row has an immutable `origin`:

| `origin` | Created by | `predecessor_id` at creation |
|---|---|---|
| `provisioning` (an **initial session**) | `provision_visitor` (`created` or `joined`) | always `null` |
| `rotation` (a **successor**) | `rotate_session` | always set, to a session of the **same visitor** (composite foreign key, §6.3) |

`touch_session` chooses the first-use path from `origin`, never from whether `predecessor_id` happens to be null (a predecessor row can be deleted by retention, §6.8).

**Initial-session first use** (`origin = 'provisioning'`, `first_used_at is null`, otherwise `active`):
- Locks only its own row and sets its own `first_used_at = now()`.
- Never reads, locks or updates any other session, and never sets `rotated_at`, `successor_id` or any other rotation field.
- Concurrent first uses: the second one, after taking the row lock, finds `first_used_at` already set and writes nothing further.

**Rotation:**
- **When:** a session in state `active` whose token was issued 30 or more days ago (`now() - created_at >= 30 days`), during a refresh request (§7.2). Initial sessions and successors qualify the same way.
- **Issue step** (`rotate_session`): locks the presented session (the predecessor) `FOR UPDATE`; it must be `active`. Inserts a successor (`origin = 'rotation'`, same visitor, `predecessor_id` = predecessor, fresh `created_at`, `last_renewed_at` and `expires_at = now() + 180 days`, `first_used_at = null`). The predecessor is **not changed**: it stays `active`. The response sets the new cookie.
- **At most one recent pending successor:** if the predecessor already has an unrevoked successor with `first_used_at is null` created less than 10 minutes ago, `rotate_session` issues nothing and returns `pending` (no cookie). An older pending successor is presumed lost: it is revoked (`superseded`) and a new one is issued.

**Successor first use** (`origin = 'rotation'`, `first_used_at is null`, not revoked, not expired):
1. Read the successor's `predecessor_id` without locking. If it is null → state `invalid_lineage`.
2. Lock the **predecessor first**, then the successor (`FOR UPDATE`), and re-read both.
3. Lineage checks, all required; any failure → state `invalid_lineage` (no visitor, no change to either session, one event recorded, §5.6):
   - the predecessor row still exists (it belongs to the same visitor by the composite key);
   - the predecessor is not revoked;
   - the predecessor's `successor_id` is null or already equals this successor;
   - the successor is still unrevoked.
4. If the successor's `first_used_at` was set by a concurrent transaction while waiting for the locks: treat it as an ordinary `active` session; no further writes.
5. Otherwise complete: successor `first_used_at = now()`; predecessor `rotated_at = now()`, `successor_id` = this successor, `expires_at = least(expires_at, now() + interval '120 seconds')`; every other unrevoked pending successor of that predecessor is revoked (`superseded`).

- **Grace period:** 120 seconds, **measured from `rotated_at`** (the completion timestamp), capped by the predecessor's previous expiry. A lost rotation response therefore does not start the clock: the browser still holds the predecessor, which stays `active`.
- **Boundary:** a rotated predecessor is valid while `now() < expires_at` (strict). `now()` is the PostgreSQL transaction timestamp of the validating transaction; all comparisons in §5.6 and §6.5 use it.
- **Determinism under concurrency:** every rotation and successor-first-use path locks the predecessor before the successor. Two concurrent first uses of one successor serialise on the predecessor lock and exactly one completes the rotation. Two different pending successors of one predecessor cannot both complete: the second finds `successor_id` pointing at the first (lineage check fails) and has already been revoked as `superseded`.

### 5.6 Session states and `touch_session()` behaviour
State of a presented token hash at transaction time `now()`, evaluated in this order:

| # | State | Condition | Returned to `sovereign_app` | Writes |
|---|---|---|---|---|
| 1 | `invalid_input` | hash is not exactly 32 bytes | state | none |
| 2 | `unknown` | no row with this hash (never issued, or deleted by retention, §6.8) | state | none |
| 3 | `revoked` | `revoked_at is not null` | state | none |
| 4 | `rotated_grace` | `rotated_at is not null` and `now() < expires_at` | state, visitor id; `renewal_due = false`, `rotation_due = false` | none |
| 5 | `rotated_reused` | `rotated_at is not null` and `now() >= expires_at` | state, **session id** | `reuse_count + 1`, `last_reuse_at = now()`, one `session_events` row (`rotated_reuse`) |
| 6 | `expired` | `now() >= expires_at` | state | none |
| 7 | `invalid_lineage` | `origin = 'rotation'`, `first_used_at is null`, and a §5.5 lineage check fails | state, **session id** | one `session_events` row (`invalid_lineage`); no change to any session |
| 8 | `active` | otherwise | state, visitor id, `expires_at`, `renewal_due`, `rotation_due` | initial or successor first use (§5.5) if `first_used_at is null`; renewal (§5.4) only with `renew = true` and due; `visitors.last_seen_at = now()` together with renewal |

- A visitor id is returned **only** for `active` and `rotated_grace`.
- `private.current_visitor_id()` (§6.5) accepts exactly: `revoked_at is null`, `now() < expires_at`, and (`origin = 'provisioning'` or `first_used_at is not null`). So it resolves `active` and `rotated_grace` sessions and initial sessions before their first use, and **never** a pending successor until `touch_session` has completed it. The request path always calls `touch_session` first (§5.7), so a valid successor is completed before its first data transaction.
- A pending successor revoked as `superseded` is `revoked`.
- `renewal_due = (now() - last_renewed_at >= 24 hours)`; `rotation_due = (now() - created_at >= 30 days)`; both only for `active`.

**Reuse and lineage events (D22):**
- Recorded **inside** `touch_session`, in the same transaction, in `private.session_events (occurred_at, event, session_id, visitor_id)` (§6.3). The table is insertable only by the function owner and readable by no application role. It never holds tokens, hashes or cookie values.
- `touch_session` returns only the **session id** for these states — never the visitor id. The server logs `{state, session_id}` and nothing else from them.
- **Failure behaviour:** if the event insert fails, the whole `touch_session` call fails (and its `reuse_count` update rolls back); the server treats any database error as `unavailable` and serves "no visitor". If the server's own log write fails, the error is swallowed after the result is decided. The result for these states is always "no visitor", so no logging failure can make a reused or inconsistent token valid.
- Reuse detection is possible only while the rotated row is retained (30 days after its validity ends, §6.8); afterwards the token is `unknown`.
- Reuse does not revoke the visitor's other sessions: without recovery, automatic visitor-wide revocation would end the legitimate visitor's access too (D10). The request gets "no visitor" and its response does not touch the cookie (§2.6).

**Rotation races (multiple requests presenting the old token):**
- Two refresh requests with the predecessor both see `rotation_due`. The first `rotate_session` issues successor S1 (`issued`, sets cookie); the second sees S1 pending and younger than 10 minutes (`pending`, no cookie). The browser ends with S1.
- Requests in flight with the predecessor after the browser received S1: the predecessor is `active` until S1's first use, then `rotated_grace` for 120 s; they succeed. Afterwards they are `rotated_reused`: recorded, "no visitor", cookie untouched, while the browser keeps S1.
- S1's response is lost: S1 is never used, the predecessor stays `active`, and the next refresh after 10 minutes supersedes S1 and issues S2.

### 5.7 Server-side validation path
1. Read the session cookie. Absent → "no visitor". Present but not 43 base64url characters → `invalid_input` without a database call.
2. Hash it and call `private.touch_session(hash, renew)` (§6.6), with `renew = false` on the render path and `true` only in the refresh request (§7.2).
3. Data access runs inside a separate transaction with the visitor context set from the same hash (§6.5). If the session became invalid between the two transactions (for example, grace ended), the context resolves to no visitor and the request fails closed.

### 5.8 Revocation
- **Per browser:** "Forget this browser" calls `revoke_session`, which revokes the presented session and every unrevoked pending successor of it (`revoke_reason = 'forgotten'`); the response then deletes the cookie. The visitor is warned first that access to the workspace ends permanently (D7).
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
**Guaranteed:** idempotency **per provisioning nonce**. Every provisioning request carrying the same nonce resolves to the same visitor, within the nonce's limits (10 minutes from its first database use, at most 5 sessions) and subject to the outcomes below. **Not guaranteed:** one visitor per browser. Requests carrying different nonces create different visitors. No client-supplied visitor id is used, and no long-lived client credential is introduced.

- **Nonce issue:** a `GET` to an app path whose request has **neither** a session cookie **nor** a nonce cookie receives `__Host-sovereign_provision` = 32 random bytes (base64url), `Max-Age=600`. Cookie write only; no database access (D19).
- **Provisioning requires the nonce.** A provisioning `POST` without a nonce cookie returns `reload`.
- **Database de-duplication** (`provision_visitor`, §6.6, one transaction, after the limits in §5.11):
  1. `insert into sovereign.provisioning_nonces (nonce_hash, created_at) values ($1, now()) on conflict do nothing`.
  2. Inserted: create visitor, workspace and initial session; set the nonce row's `visitor_id`; `mint_count = 1`; outcome `created`.
  3. Row already existed: lock it `FOR UPDATE`. If `now() >= created_at + 10 minutes` or `mint_count >= 5` → outcome `provisioning_expired`. Otherwise create a new initial session for the row's visitor, increment `mint_count`, outcome `joined`.
  A concurrent transaction with the same nonce blocks on the primary key until the first commits, then follows step 3; if the first rolls back, it performs step 2.
- **Success response** sets the session cookie and deletes the nonce cookie.

| Scenario | Result |
|---|---|
| Same nonce, concurrent requests | One visitor (`created` once, `joined` for the rest, up to 5 sessions, then `provisioning_expired`). |
| Same nonce, retried or duplicated Server Action | Same visitor (`joined`). |
| Refresh during provisioning | The nonce cookie survives → same nonce → same visitor. |
| Request sent before another tab's response set the session cookie | Same nonce → same visitor. |
| A tab's `POST` already carries a session cookie | `reload` (no database write); the tab then renders with that session. |
| Several tabs load at almost the same moment and their `GET` responses interleave | The tabs may hold **different** nonces when their `POST`s leave → **different visitors**. The browser keeps whichever session cookie arrives last; the other visitors become orphans (§6.8). |
| Different browser profiles or devices | Different nonces → different visitors (intended). |
| Nonce older than 10 minutes or mint limit reached | `provisioning_expired` → nonce cookie deleted, `reload` → a fresh nonce on the next `GET`. |
| Cookies blocked | No nonce is stored → `reload` once, then a "cookies are required" notice (§7.6). |

- **Mitigation, not guarantee:** the client component is single-flight per tab and may coordinate across tabs (for example with `BroadcastChannel`). This makes the interleaving case rarer; correctness does not depend on it.
- **Orphans** are ordinary visitors whose sessions are never used again; §6.8 removes them like any abandoned visitor, and cleanup can never delete a visitor with a valid session.
- **Nonce theft:** within its window and mint limit, a stolen nonce lets the thief obtain a session for that newly created visitor — the same power as stealing the session cookie at that moment.

### 5.11 Abuse controls (D13, mandatory for public deployment)
Public deployment is **not allowed** without these controls. They are enforced **inside the database functions** that perform the guarded action, backed by shared database state, so they hold across multiple application instances and cannot be skipped or redirected by the caller.

| Control | Bucket | Canonical subject (enforced by `CHECK` and by the function) | Proposed default (D13 values) | On exceed |
|---|---|---|---|---|
| Provisioning per client network | `provision_net_hour`, `provision_net_day` | **Network subject**: exactly 33 bytes = 1 key-version byte + HMAC-SHA-256 (§5.12) | 10 per hour **and** 30 per day | outcome `rate_limited_network` → "Too many new workspaces from this network. Please try again later." |
| Global provisioning circuit breaker | `provision_global_hour` | exactly the 1-byte value `\x00`, set inside the function, never passed by the caller | 1,000 per hour (operator-configurable) | outcome `rate_limited_global` → "Sovereign is temporarily unavailable. Please try again later." |
| Username changes per visitor | `username_visitor_day` | exactly the 16-byte visitor id **from `current_visitor_id()`** of the calling transaction, never passed by the caller | 10 per day | outcome `rate_limited` → "Too many changes. Please try again tomorrow." |

- **No caller-chosen subjects.** `sovereign_app` cannot write counters and has no generic rate-limit function. Limits are consumed by an internal helper (`private.consume_limit`, executable by nobody but its owner) called only from `provision_visitor` and `update_username`.
- **Provisioning** (`provision_visitor(…, net_subject)`), inside the function before anything is created:
  1. Validate `net_subject`: not null; exactly 33 bytes; version byte equal to `private.identity_settings.network_key_version`, **or** the development subject (version byte `0` followed by 32 zero bytes) **and** `identity_settings.allow_dev_network_key = true`. Anything else → outcome `config_mismatch` (generic "temporarily unavailable"); nothing consumed, nothing created.
  2. Consume `provision_net_hour` and `provision_net_day` for that subject; if either is exceeded → `rate_limited_network`.
  3. Consume `provision_global_hour` with the fixed subject; if exceeded → `rate_limited_global`.
  4. Only then run §5.10's nonce logic. Counters consumed in steps 2–3 are committed even when the outcome is a limit (attempts count).
- **Why the limits cannot be evaded by varying the subject:** the network subject is computed by the server from the trusted client address (§5.12). The database cannot verify an HMAC whose key it does not hold, so a compromised or buggy server could vary the network subject and escape the **per-network** limit. It cannot escape the **global** limit: every attempt that passes step 2 consumes the fixed global subject in step 3. And without a valid network subject there is no provisioning at all (step 1), so omitting network identity cannot bypass the global breaker either. The username limit's subject comes from the transaction context, not from any argument.
- **Username changes** (`update_username(new_username)`): validate the format and denylist first (invalid → `invalid`, nothing consumed); resolve the visitor from `current_visitor_id()` (none → `no_visitor`, nothing consumed); consume `username_visitor_day` for that visitor id (exceeded → `rate_limited`); then update. A unique violation returns `taken` and still counts.
- **Key and version handling:** the HMAC key (`SOVEREIGN_RATE_LIMIT_KEY`, ≥ 32 random bytes, server-only) and its version (`SOVEREIGN_RATE_LIMIT_KEY_VERSION`, 1–255) are configured on the server; `identity_settings.network_key_version` must hold the same version. Rotating the key means changing both together; per-network windows effectively restart after a rotation (accepted). A version mismatch fails closed (`config_mismatch`). Variable names are proposed.
- **Development subject:** accepted only where `allow_dev_network_key` is true (development databases). The column defaults to false; production rejects the subject.
- **Storage and bounds:** `private.rate_limit_counters` (fixed windows aligned to the database clock) and `private.rate_limit_settings`; atomic `insert … on conflict … do update`. A missing settings row denies. Rows older than 2 days are deleted by maintenance (§6.8). The per-network check runs before the global one so that one network cannot exhaust everyone's allowance; as a result each attempt from a new network creates counter rows, and very-many-address volumetric attacks are bounded only by request volume. Platform-level protection (WAF/edge rate limiting) is a deployment prerequisite for that class of attack.
- **Responses and logs** never reveal counts, windows, subjects, infrastructure or which internal check failed beyond the generic messages above. Logs record outcome names only.
- **Trade-off (approval needed):** visitors behind shared addresses (carrier-grade NAT, offices, schools) share the per-network limit.

### 5.12 Client IP trust (D14, unresolved deployment prerequisite)
- Next.js request handlers do not provide a trustworthy peer address on their own, and every `X-Forwarded-*`, `Forwarded` or `X-Real-IP` header can be sent by any client. **No forwarding header is trusted by default.**
- A header is trusted only when **all** of these are true and configured explicitly:
  1. The deployment's reverse proxy or platform is selected and documented.
  2. The application is reachable **only** through that proxy (no direct public path that bypasses it).
  3. The proxy **overwrites** (does not append to) one named header with the connecting client's address.
  4. The server is configured with that exact header name (proposed variable `SOVEREIGN_CLIENT_IP_HEADER`), and the value parses as exactly one IPv4 or IPv6 address. Lists, ports, empty values and anything else are "untrusted".
- **Canonical network:** an IPv4 address as its 4 bytes; an IPv6 address reduced to its /64 prefix (8 bytes); IPv4-mapped IPv6 treated as IPv4. Network subject = `version byte || HMAC-SHA-256(key, family byte || canonical bytes)`.
- The proxy/provider is **not yet selected**, so no header is named in this specification.
- **Fail-closed behaviour:** in production (canonical origin not loopback), if no trusted client address can be established, the server does not call `provision_visitor` and returns the generic "temporarily unavailable" message; a configuration error is logged without header values. Requests from existing visitors are unaffected. Username changes do not need the client address.
- **Local development** (`SOVEREIGN_SITE_URL` on a loopback host): the server sends the development subject. This branch never activates for a non-loopback origin, and the database accepts that subject only where `allow_dev_network_key` is true.

### 5.13 Theft and loss
- **Theft:** the session cookie is a bearer credential. Whoever presents it **is** the visitor: they can read and change everything the visitor can. HttpOnly prevents scripts reading it, but XSS can act within the page, and malware or a copied browser profile can take the cookie. There is no second factor and no way for the real visitor to end the thief's access from another device. Rotation (§5.5) limits the life of a copied token that is **not** used; a thief who uses the token can perform rotation themselves and the legitimate browser's token then becomes `rotated_reused` (recorded, §5.6) — the visitor loses access, which is detectable but not recoverable.
- **Loss:** clearing cookies, losing the profile or device, or inactivity beyond 180 days loses access permanently. This is accepted (product requirement).

---

## 6. Database model and Supabase security

### 6.1 Placement
- Supabase PostgreSQL remains the hosted database.
- Identity tables live in the schema `sovereign`, which is **not** exposed through the Supabase Data API (D2). Settings, rate-limit and event tables and all functions live in `private` (not exposed).
- No `profiles`, no `workspace_members`, no `workspace_data` JSONB placeholder. Domain tables arrive with their requirements, each with `workspace_id` and the same RLS pattern.

### 6.2 Roles (D15, D16, D20)
| Role | Attributes | Purpose |
|---|---|---|
| `postgres` | Supabase-managed | Applies migrations; owns tables and schemas. Never used by the running application or by test cleanup. |
| `sovereign_identity_owner` | `NOLOGIN NOSUPERUSER NOBYPASSRLS NOCREATEDB NOCREATEROLE NOINHERIT` | Owns the request-path functions. Privileges in §6.6, owner-scoped policies in §6.7. |
| `sovereign_maintenance_owner` | same attributes | Owns retention functions (§6.8). Column-level privileges only; cannot read token or nonce hashes. |
| `sovereign_app` | `LOGIN NOSUPERUSER NOBYPASSRLS NOCREATEDB NOCREATEROLE NOREPLICATION NOINHERIT`, `CONNECTION LIMIT` (D16) | The only role the web server uses. Password set **out of band**, never in a migration or the repository. |
| `sovereign_test_cleanup` | `LOGIN NOSUPERUSER NOBYPASSRLS NOCREATEDB NOCREATEROLE NOINHERIT`; **development databases only** | Test-data cleanup (§6.8). Created by a development-only setup script, never by a migration; does not exist in production. |
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
  origin           text not null check (origin in ('provisioning', 'rotation')),
  token_hash       bytea not null unique check (octet_length(token_hash) = 32),
  created_at       timestamptz not null default now(),          -- token issue time
  first_used_at    timestamptz,                                 -- first successful validation
  last_renewed_at  timestamptz not null default now(),
  expires_at       timestamptz not null,
  predecessor_id   uuid,                                        -- set at insert for origin = 'rotation' only
  rotated_at       timestamptz,                                 -- set on a predecessor at rotation completion
  successor_id     uuid,
  revoked_at       timestamptz,
  revoke_reason    text check (revoke_reason in ('forgotten', 'operator', 'superseded')),
  reuse_count      integer not null default 0 check (reuse_count >= 0),
  last_reuse_at    timestamptz,
  unique (id, visitor_id),
  foreign key (predecessor_id, visitor_id)
    references sovereign.visitor_sessions (id, visitor_id) on delete set null (predecessor_id),
  foreign key (successor_id, visitor_id)
    references sovereign.visitor_sessions (id, visitor_id) on delete set null (successor_id),
  check (origin = 'rotation' or predecessor_id is null),       -- an initial session never has a predecessor
  check (predecessor_id is null or predecessor_id <> id),
  check (successor_id is null or successor_id <> id),
  check (expires_at > created_at),
  check ((revoked_at is null) = (revoke_reason is null)),
  check (successor_id is null or rotated_at is not null),
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

create table private.identity_settings (            -- exactly one row, seeded by migration
  singleton              boolean primary key default true check (singleton),
  network_key_version    smallint not null check (network_key_version between 1 and 255),
  allow_dev_network_key  boolean  not null default false
);

create table private.rate_limit_settings (
  bucket         text primary key check (bucket in ('provision_net_hour', 'provision_net_day',
                                                    'provision_global_hour', 'username_visitor_day')),
  window_length  interval not null check (window_length > interval '0'),
  max_count      integer  not null check (max_count > 0)
);

create table private.rate_limit_counters (
  bucket        text not null references private.rate_limit_settings (bucket),
  subject       bytea not null,
  window_start  timestamptz not null,
  hit_count     integer not null check (hit_count > 0),
  primary key (bucket, subject, window_start),
  check (
       (bucket in ('provision_net_hour', 'provision_net_day') and octet_length(subject) = 33)
    or (bucket = 'provision_global_hour' and subject = '\x00'::bytea)
    or (bucket = 'username_visitor_day'  and octet_length(subject) = 16)
  )
);
create index rate_limit_counters_window_start_idx on private.rate_limit_counters (window_start);

create table private.session_events (
  id           bigint generated always as identity primary key,
  occurred_at  timestamptz not null default now(),
  event        text not null check (event in ('rotated_reuse', 'invalid_lineage')),
  session_id   uuid not null,     -- no foreign key: the event outlives the session row
  visitor_id   uuid not null      -- no foreign key: kept until event retention (§6.8)
);
create index session_events_occurred_at_idx on private.session_events (occurred_at);
```

Notes:
- **Session lineage:** the composite foreign keys make a predecessor or successor of another visitor impossible. `on delete set null (column)` (PostgreSQL 15+) clears only the reference, never `visitor_id`. A successor whose predecessor row has been deleted keeps `origin = 'rotation'` and, if still pending, can only reach `invalid_lineage` (§5.5).
- **One-to-one visitor ↔ workspace:** `workspaces.visitor_id` is `NOT NULL UNIQUE`. "At least one" holds because the only insert path, `provision_visitor`, creates both in one transaction and `sovereign_app` has no `INSERT` on either table. A deferrable "every visitor has a workspace at commit" constraint trigger is recommended as a second guard (D17).
- **Deletion:** deleting a visitor cascades to its workspace, sessions and provisioning nonces. `session_events` rows are removed by their own retention.
- **Timestamps:** `updated_at` is maintained by a `BEFORE UPDATE` trigger using `clock_timestamp()`.
- **Session states** are derived from the columns exactly as §5.6 lists; there is no separate state column that could disagree.
- **No slug, `kind` or membership table:** they existed for teams (ADR-0006), which are out of scope.

### 6.4 Grants to `sovereign_app`

```sql
revoke all on schema sovereign, private from public, anon, authenticated, service_role;
revoke all on all tables in schema sovereign, private from public, anon, authenticated, service_role;
grant usage on schema sovereign, private to sovereign_app;

grant select on sovereign.visitors, sovereign.workspaces to sovereign_app;
-- No INSERT, UPDATE or DELETE on any table. No privileges on visitor_sessions, provisioning_nonces
-- or any private table. Username changes go through private.update_username.
```

- `EXECUTE` on every function in `private` is revoked from `public`, `anon`, `authenticated` and `service_role`; then exactly the functions marked for `sovereign_app` in §6.6 are granted to it.
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
    and pg_catalog.now() < s.expires_at
    and (s.origin = 'provisioning' or s.first_used_at is not null);
$$;
```

- It is read-only (no first use, renewal or event recording) and accepts exactly the sessions listed in §5.6.
- **Why the token hash, not a visitor id:** if the context were a visitor id, any SQL run as `sovereign_app` (for example through an SQL-injection bug) could set any id and satisfy every policy. With the hash, the context resolves only for a live session, and `sovereign_app` cannot read `visitor_sessions`. Arbitrary SQL as `sovereign_app` can therefore act only as visitors whose token hash it already knows. This narrows, but does not remove, the risk (§6.7).
- **Pooling:** `set_config(..., true)` is transaction-local; PostgreSQL discards it at commit or rollback, so it cannot reach the next transaction on a pooled connection. The data-access helper (`withVisitor`) always opens an explicit transaction and sets the context first. Session-level `SET`/`set_config(..., false)` is forbidden in application code (static test, §9.1). Whether Supabase's pooler in transaction mode preserves this as expected is a hosted prerequisite (§6.10).

### 6.6 Functions
All functions: in `private`; owned by the role shown; `set search_path = ''`; every object reference fully qualified (`sovereign.visitors`, `pg_catalog.now()`); **no dynamic SQL** (no `EXECUTE`, no `format()`-built statements, no object names from arguments); arguments typed and validated as the first statements; generic errors without internal detail.

| Function | Kind | Owner | `EXECUTE` granted to | Behaviour |
|---|---|---|---|---|
| `current_visitor_id() → uuid` | `SECURITY DEFINER`, `STABLE` | identity owner | `sovereign_app` | §6.5. Read-only. |
| `provision_visitor(nonce_hash bytea, token_hash bytea, username text, net_subject bytea) → (outcome text, visitor_id uuid, workspace_id uuid)` | `SECURITY DEFINER` | identity owner | `sovereign_app` | §5.11 then §5.10. `outcome ∈ {created, joined, provisioning_expired, rate_limited_network, rate_limited_global, config_mismatch}`; ids only for `created`/`joined`. Malformed hashes or username raise a generic error. |
| `touch_session(token_hash bytea, renew boolean) → (state text, visitor_id uuid, session_id uuid, expires_at timestamptz, renewal_due boolean, rotation_due boolean)` | `SECURITY DEFINER` | identity owner | `sovereign_app` | Exactly §5.5–§5.6. `visitor_id` only for `active`/`rotated_grace`; `session_id` only for `rotated_reused`/`invalid_lineage`; `expires_at` and the flags only for `active`. |
| `rotate_session(old_hash bytea, new_hash bytea) → text` | `SECURITY DEFINER` | identity owner | `sovereign_app` | §5.5. Returns `issued`, `pending`, `not_due` or `not_active`. Only `issued` lets the caller set a cookie. |
| `revoke_session(token_hash bytea) → boolean` | `SECURITY DEFINER` | identity owner | `sovereign_app` | §5.8. Revokes the presented session (if `active` or `rotated_grace`) and its unrevoked pending successors. |
| `update_username(new_username text) → text` | `SECURITY DEFINER` | identity owner | `sovereign_app` | §5.11. Visitor from `current_visitor_id()`; returns `updated`, `taken`, `invalid`, `rate_limited` or `no_visitor`. |
| `consume_limit(bucket text, subject bytea) → boolean` | `SECURITY INVOKER` (runs as the calling definer function's owner) | identity owner | **nobody** (revoked from `public`) | Internal helper; the bucket/subject `CHECK` and the settings row apply. |
| `set_updated_at()` (trigger) | `SECURITY INVOKER` | identity owner | nobody | Timestamps. |
| `cleanup_sessions()`, `cleanup_visitors()`, `cleanup_nonces()`, `cleanup_rate_limits()`, `cleanup_session_events()` | `SECURITY DEFINER` | maintenance owner | the maintenance executor (§6.8, D11) | Retention. Not granted to `sovereign_app`. |

**Privileges held by `sovereign_identity_owner`** (granted by `postgres`; nothing else):
- `sovereign.visitors`: `SELECT`, `INSERT`, `UPDATE (username, last_seen_at, updated_at)`.
- `sovereign.workspaces`: `INSERT`. (No `SELECT`: `provision_visitor` generates the workspace id with `gen_random_uuid()` into a variable and inserts it explicitly, without `RETURNING`.)
- `sovereign.visitor_sessions`: `SELECT`, `INSERT`, `UPDATE (first_used_at, last_renewed_at, expires_at, rotated_at, successor_id, revoked_at, revoke_reason, reuse_count, last_reuse_at)`. `origin`, `token_hash`, `visitor_id`, `predecessor_id` and `created_at` are never updated.
- `sovereign.provisioning_nonces`: `SELECT`, `INSERT`, `UPDATE (visitor_id, mint_count)`.
- `private.identity_settings`, `private.rate_limit_settings`: `SELECT`.
- `private.rate_limit_counters`: `SELECT`, `INSERT`, `UPDATE (hit_count)`.
- `private.session_events`: `INSERT` only.
- `USAGE` on schemas `sovereign` and `private`. No `DELETE` anywhere.

**Privileges held by `sovereign_maintenance_owner`** (column-level; cleanup needs no token, nonce or subject values):
- `sovereign.visitor_sessions`: `SELECT (visitor_id, revoked_at, expires_at)`, `DELETE`. **No `SELECT` on `token_hash`.**
- `sovereign.visitors`: `SELECT (id, created_at)`, `DELETE`. Cascaded deletes of workspaces, sessions and nonces are performed by PostgreSQL's referential actions and need no further privilege.
- `sovereign.provisioning_nonces`: `SELECT (created_at)`, `DELETE`. **No `SELECT` on `nonce_hash`.**
- `private.rate_limit_counters`: `SELECT (window_start)`, `DELETE`. **No `SELECT` on `subject`.**
- `private.session_events`: `SELECT (occurred_at)`, `DELETE`.
- `USAGE` on schemas `sovereign` and `private`. Cleanup functions use plain `DELETE … WHERE` on those columns only, without `RETURNING`.

**RLS and the owner roles:** the owners are not table owners and do not have `BYPASSRLS`, so RLS applies to them. They receive owner-scoped policies (§6.7) covering exactly the commands above; column privileges then limit which columns those commands can read or write. This is an explicit, testable grant, not a bypass, and not available to `sovereign_app`.

**Why a dedicated owner is safer, concretely:** a bug in a function body (for example a missing `where`) can only use the privileges listed above — it cannot drop or alter tables, read other schemas, use `DELETE` on the request path, or reach anything `postgres` can. Owning the functions with `postgres` would give every function body `postgres`'s privileges.

### 6.7 RLS policies

```sql
alter table sovereign.visitors            enable row level security;
alter table sovereign.workspaces          enable row level security;
alter table sovereign.visitor_sessions    enable row level security;
alter table sovereign.provisioning_nonces enable row level security;
alter table private.identity_settings     enable row level security;
alter table private.rate_limit_settings   enable row level security;
alter table private.rate_limit_counters   enable row level security;
alter table private.session_events        enable row level security;

-- Application role: read own rows only. No write policies.
create policy visitors_select_self on sovereign.visitors
  for select to sovereign_app using (id = (select private.current_visitor_id()));
create policy workspaces_select_own on sovereign.workspaces
  for select to sovereign_app using (visitor_id = (select private.current_visitor_id()));

-- Identity owner (using / with check true), one policy per table and command:
--   visitors: select, insert, update; workspaces: insert;
--   visitor_sessions: select, insert, update; provisioning_nonces: select, insert, update;
--   identity_settings: select; rate_limit_settings: select;
--   rate_limit_counters: select, insert, update; session_events: insert.
-- Maintenance owner: select and delete on visitors, visitor_sessions, provisioning_nonces,
--   rate_limit_counters, session_events (column privileges in §6.6 limit what it can read).
-- No policies for anon, authenticated or service_role. Test-cleanup policies exist only in
--   development databases (§6.8).
```

What RLS guarantees here:
- **Does:** an application bug that forgets a filter, or passes another visitor's id, cannot read another visitor's rows through `sovereign_app`, and `sovereign_app` cannot write any table directly.
- **Does not:** protect against a compromised web server, which receives live session cookies and holds the `sovereign_app` password. A custom PostgreSQL setting is not independently trustworthy: any code running SQL as `sovereign_app` can call `set_config`. The token-hash context limits such code to visitors whose hash it knows; it is defence in depth, not isolation from the server.
- `postgres` owns the tables and is used only for migrations and operator work.

### 6.8 Retention and cleanup (D11, D20)
| Data | Deleted when | Function |
|---|---|---|
| Session rows | `validity_end < now() - 30 days` | `cleanup_sessions()` |
| Visitors (with workspace, sessions, nonces) | **Abandoned**: `created_at < now() - 1 day` **and** no session of the visitor with `validity_end > now() - 30 days` | `cleanup_visitors()` |
| Provisioning nonces | `created_at < now() - 1 day` | `cleanup_nonces()` |
| Rate-limit counters | `window_start < now() - 2 days` | `cleanup_rate_limits()` |
| Session events | `occurred_at < now() - 90 days` | `cleanup_session_events()` |

Where `validity_end` of a session is `least(revoked_at, expires_at)` (PostgreSQL's `least` ignores a null `revoked_at`). A rotated predecessor's `expires_at` is its grace end, so the same formula covers it.

Visitor cleanup safety:
- A session that `current_visitor_id()` accepts, or that `touch_session` reports as `active` or `rotated_grace`, has `revoked_at is null` and `expires_at > now()`, so its `validity_end > now()`: the predicate can never select a visitor that holds one.
- Validity only moves forward through renewal, which applies only to `active` sessions. An expired or revoked session never becomes valid again.
- Sessions are created for an existing visitor only by rotation (requires an `active` predecessor, so the visitor is not abandoned) and by nonce joining (requires the nonce, and hence the visitor, to be younger than 10 minutes, while cleanup requires older than 1 day). So no session can be created for a visitor the predicate selects.
- These facts, not lock timing, make cleanup safe. `cleanup_visitors()` deletes one candidate at a time with `delete … where id = $1 and <predicate>`, so the predicate is evaluated again at deletion. **Any future feature that creates sessions for existing visitors must re-examine this argument.**
- **Orphans** (different-nonce provisioning, §5.10) are abandoned visitors whose only sessions were never used again; they follow the same rule (deleted about 210 days after creation). Deleting them earlier is not possible without risking a one-visit visitor who returns later.

Executor: how cleanup runs on a schedule (Supabase `pg_cron`, if available on the project, or an operator-run job) is an **unresolved prerequisite** (D11). `sovereign_app` never runs cleanup.

**Test-data cleanup (D20), development databases only:**
- **Development-only objects**, created by a separate development setup script (reviewed SQL, **not** a migration, never applied to production):
  - role `sovereign_test_cleanup` (attributes in §6.2);
  - table `private.test_environment (singleton boolean primary key check (singleton), environment text not null check (environment = 'development'), project_ref text not null)` with exactly one row: the development project's ref;
  - grants: `SELECT (id, username, created_at)` and `DELETE` on `sovereign.visitors`; `SELECT (visitor_id)` on `sovereign.workspaces`, `sovereign.visitor_sessions` and `sovereign.provisioning_nonces`; `SELECT` on `private.test_environment`; `USAGE` on both schemas; nothing else (in particular no `SELECT` on `token_hash` or `nonce_hash`);
  - policies for that role: on `visitors`, `select` and `delete` **only where `username like 'zz-test-%'`**; on the three child tables, `select` only for rows whose visitor has that prefix; on `test_environment`, `select`.
- **Configuration:** `SOVEREIGN_TEST_CLEANUP_DB_URL` and `SOVEREIGN_TEST_EXPECTED_PROJECT_REF` in the root `.env` (gitignored). **Neither has a default**, and neither is derived from any other variable.
- **Before any delete, the harness verifies, and aborts with no `DELETE` issued if anything fails:**
  1. Both variables are present and well-formed.
  2. The URL's user and host name the expected project (Supabase pooler user `sovereign_test_cleanup.<ref>`, or a direct host containing `<ref>`); the exact URL form is a hosted assumption to verify (§6.10).
  3. Connected as `sovereign_test_cleanup`: `current_user` matches; `pg_roles` shows not superuser, not `BYPASSRLS`, not `CREATEROLE`; `has_table_privilege` and `has_column_privilege` show `DELETE` on `sovereign.visitors` only, no `INSERT` or `UPDATE` anywhere, and no `SELECT` on `visitor_sessions.token_hash`.
  4. `private.test_environment` has exactly one row with `environment = 'development'` and `project_ref` equal to the expected ref. On a production database the role and the table do not exist, so this check cannot pass.
- **Deletion,** in one transaction that re-runs checks 3–4 first: `delete from sovereign.visitors where id = any($recorded_ids) and username like 'zz-test-%' and created_at >= $run_start`. The deleted row count must equal the number of recorded ids; otherwise roll back, abort and report sanitised ids (`support/diagnostics.ts` pattern).
- **Afterwards:** zero rows in `visitors`, `workspaces`, `visitor_sessions` and `provisioning_nonces` for the recorded ids, and zero `zz-test-` visitors created since `run_start`. Any remainder is reported and fails the run.
- **Independent guards:** recorded ids (primary), the `zz-test-` prefix in the statement (second), the RLS policy restricting the role to prefixed visitors (third), and the target checks above. No production role, function or privilege is involved; `sovereign_app` has no cleanup rights.
- **Why there is no silent fallback:** the harness reads only those two variables. It never falls back to `SOVEREIGN_SQL_TEST_DB_URL`, a hard-coded host or any other setting; a missing or mismatching value stops the run before any deletion is attempted.

### 6.9 Supabase settings
- **Data API:** M2 does not need it — the browser never reads visitor data from Supabase, and `sovereign`/`private` are never exposed. Whether to **disable the Data API entirely** is a separate decision (D18), because it affects later features.
- **Supabase Auth:** not used. New sign-ups and anonymous sign-ins are disabled in the dashboard so no `auth.users` rows can be created by holders of the publishable key.
- The browser bundle carries no Supabase URL or key for visitor data.

### 6.10 Hosted prerequisites and alternatives
These must be verified on the hosted development project before implementation is considered viable:
1. `postgres` can create `NOLOGIN` roles, grant them table and column privileges and `ALTER FUNCTION … OWNER TO` them (PostgreSQL 16+ requires the creator to hold `SET` membership in the new role).
2. Custom login roles (`sovereign_app`, and in development `sovereign_test_cleanup`) can connect through Supabase's pooler, and the pooler's user-name format is as assumed in §6.8 (the existing `[VERIFY AT M2]` item for custom roles in DATA-MODEL §G.6).
3. Transaction-mode pooling keeps transaction-local settings for the transaction's duration and clears them afterwards (verified by the pool-isolation tests, §9.3).
4. A scheduler for maintenance functions exists, or an operator-run job is accepted (D11).
5. The database supports `on delete set null (column_list)` (PostgreSQL 15+; the project's `config.toml` targets 17, to be confirmed on the hosted project).

Fallbacks (each needs approval; none is assumed):
- (1) fails → functions owned by `postgres`. This is explicitly **weaker**: function bodies then run with `postgres`'s privileges.
- (2) fails → direct connection or session-mode pooler with a small server-side pool (same transaction-local rule), or a dedicated pooler user mapped to `sovereign_app`; target verification for test cleanup is then re-specified for that URL form.
- (3) fails → pass the token hash as an explicit argument to SECURITY DEFINER data-access functions for every operation (no setting), at the cost of a function-only API.
- (5) fails → the composite lineage keys need a different design (open).
- **Not acceptable:** routine `service_role` or `postgres` access from the web server.

---

## 7. Application architecture

### 7.1 First-visit provisioning
1. A `GET` to an app path with neither cookie sets the nonce cookie (§5.10) and renders the app shell without a visitor.
2. The shell's client component calls the `provisionVisitor` Server Action (`POST`).
3. The action checks, in order: CSRF (§5.9); nonce cookie present (else `reload`); no session cookie (else `reload`); trusted client network established (§5.12; else generic `unavailable` in production, with no database call).
4. It generates the session token and username, computes the network subject, hashes token and nonce, and calls `provision_visitor(nonce_hash, token_hash, username, net_subject)`. The database applies the limits (§5.11) and the nonce logic (§5.10). On a username collision it retries with a new name (≤ 5).
5. `created`/`joined` → set the session cookie, delete the nonce cookie, return the workspace view. `provisioning_expired` → delete the nonce cookie and return `reload`. Limit and configuration outcomes → the generic messages in §5.11.

### 7.2 Returning-visitor resolution and refresh
- **Render path (`GET`):** `requireVisitor()` (server-only) reads the cookie and calls `touch_session(hash, renew = false)`. `active` and `rotated_grace` render the workspace; every other state renders the §2.6 notice (or provisioning when no cookie was sent). `rotated_reused` and `invalid_lineage` are logged as `{state, session_id}`.
- **Refresh path (`POST`):** when `renewal_due` or `rotation_due` is true, the shell's client component calls `refreshSession`. It runs the CSRF checks, calls `touch_session(hash, renew = true)` and, if `rotation_due`, generates a new token and calls `rotate_session(old_hash, new_hash)`. It sets the cookie only when renewal happened (same token, new `Max-Age`) or rotation returned `issued` (new token). No other outcome writes the cookie.

### 7.3 Username updates
`updateUsername` Server Action: CSRF checks, `requireVisitor()`, server-side format and denylist validation, then `update_username(new_username)` inside `withVisitor`, so the transaction context supplies the visitor. Outcomes: `updated` → success; `taken` → "That name is taken."; `invalid` → validation message; `rate_limited` → §5.11 message; `no_visitor` → §2.6 notice.

### 7.4 Workspace retrieval and data operations
- `getPersonalWorkspace()` reads `sovereign.workspaces` inside `withVisitor`; RLS returns at most the visitor's own row.
- Later domain operations use the same pattern: services in `packages/core` take an `Actor` resolved server-side; the `@sovereign/db` adapter runs them inside `withVisitor`.

### 7.5 Cookie issuance and revocation summary
| Event | Cookie written |
|---|---|
| Provisioning `created`/`joined` | Session cookie set; nonce cookie deleted |
| Provisioning `provisioning_expired` | Nonce cookie deleted |
| Refresh with renewal | Same token, new `Max-Age` |
| Rotation `issued` | New token |
| "Forget this browser" | Session cookie deleted after `revoke_session` |
| "Start a new workspace" with an invalid cookie (§7.6) | Session and nonce cookies deleted; no new cookie in that response |
| Any validation failure | **Nothing** |

### 7.6 "Start a new workspace" (explicit, after an invalid cookie)
1. The §2.6 notice offers the button. It calls the `startNewWorkspace` Server Action (`POST`), which runs the CSRF checks.
2. The action hashes the presented session cookie and calls `touch_session(hash, renew = false)`:
   - `active` or `rotated_grace` (the cookie works after all, e.g. after a transient error) → return `reload`; **nothing is deleted**. This action never discards a valid session.
   - any other state, or no session cookie → the response deletes the session cookie **and** any nonce cookie and returns `reload`. No database row changes; the old visitor, if any, is left to retention (§6.8).
3. The client performs a full navigation `GET /workspace`. That request carries neither cookie, so the response sets a fresh nonce cookie (§5.10, D19). `GET` still creates no database rows.
4. The app shell provisions as on a first visit (§7.1) using that nonce.

Failures and retries:
- **Action fails** (network error, CSRF rejection): the notice stays; nothing was changed; the visitor can press the button again.
- **Cookie deletion did not take effect** (the session cookie is still sent): the navigation shows the §2.6 notice again; pressing the button repeats steps 1–3.
- **No nonce after navigation** (cookies blocked): provisioning returns `reload`; the client reloads **once** (tracked in page state, e.g. an `?attempt=2` marker) and then shows "Cookies are required to use Sovereign". No loop.
- **Provisioning fails** with a limit or `unavailable`: the generic message and a "Try again" button that re-sends the provisioning `POST` with the same nonce while it is valid; `provisioning_expired` or a missing nonce leads to one reload for a fresh nonce.

### 7.7 Errors
Typed results `{ ok, data } | { ok: false, error }`, `error ∈ { no_visitor, invalid_session, reload, cookies_required, rate_limited, validation, conflict, unavailable }`. Client messages are generic. Logs carry no tokens, nonces, hashes, cookie values, IP addresses or network subjects; they may carry visitor ids, session ids, state names and outcome names.

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
| Env declarations | `apps/web/src/env.d.ts` | Gains server-only names for the application database URL (D16), `SOVEREIGN_RATE_LIMIT_KEY`, `SOVEREIGN_RATE_LIMIT_KEY_VERSION` and `SOVEREIGN_CLIENT_IP_HEADER`; `NEXT_PUBLIC_SUPABASE_*` not needed for M2. |
| Migrations | `packages/db/supabase/migrations/20261112120000_m2_schemas_privileges.sql`, `20261112120100_m2_identity.sql` | Depend on `authenticated`, `auth.users`, `auth.uid()`. To be **replaced**, not amended — requires approval. |
| pgTAP | `packages/db/supabase/tests/{identity_signup, identity_rls, catalog}.test.sql` | Replaced (§9.2). The control `_selftest/deliberate_failure.test.sql`, the manifest rules and the runner (`scripts/run-sql-tests.mjs`, `scripts/tap.mjs`) are kept. |
| Integration tests | `apps/web/test/integration/{auth.test.ts, support/admin.ts, support/env.ts, support/user.ts}` | Replaced (§9.3). `support/diagnostics.ts` (sanitised cleanup reports) is reusable. The service-role key is no longer needed. |
| Guards | `apps/web/test/service-role-isolation.test.ts`, `scripts/check-client-bundle.mjs`, ESLint rules | Kept; extended so the application database URL and the rate-limit key never reach client code or bundles. |
| Dependencies | `@supabase/ssr`, `@supabase/supabase-js` (web), `supabase` CLI (db) | JS clients unused for M2. A PostgreSQL driver is required — a new dependency (D16). |

### 8.3 Proposed new structure (not created)
- `apps/web/src/server/visitor.ts` (`server-only`): cookies, token and nonce generation and hashing, `requireVisitor`, `withVisitor`, CSRF checks, client-network extraction and network-subject HMAC.
- `apps/web/src/app/(app)/workspace/actions.ts`: `provisionVisitor`, `refreshSession`, `updateUsername`, `forgetThisBrowser`, `startNewWorkspace`.
- `apps/web/src/app/(app)/workspace/SessionKeeper.tsx` (client): provisioning, refresh and the single-reload rule.
- `packages/db/src/app-database.ts`: driver wrapper exposing only `withVisitor` and the §6.6 functions granted to `sovereign_app`.
- Development-only: the test setup SQL (§6.8) and the cleanup harness with its target checks.

---

## 9. Testing requirements
None of these tests exists. Counts are intentionally not stated.

### 9.1 Unit (offline, Vitest)
- Token and nonce generation: 32 bytes, base64url, 43 characters; hashes are SHA-256 of the raw bytes.
- Cookie attributes for both cookies, production and `http://localhost` variants (§5.2); session `Max-Age` equals remaining lifetime.
- CSRF checks: missing `Origin` rejected; foreign origin rejected; canonical accepted; `Sec-Fetch-Site: cross-site` rejected; `Host`/`X-Forwarded-Host` ignored.
- Client network (§5.12): with no configured header, any `X-Forwarded-For`, `Forwarded` or `X-Real-IP` is ignored and the result is "untrusted"; with a configured header, a single valid address is accepted and lists, ports and garbage are untrusted; IPv6 reduced to /64; IPv4-mapped IPv6 treated as IPv4; production without a trusted address never calls `provision_visitor`; the loopback development branch never activates for a non-loopback origin.
- Network subject: always 33 bytes; version byte equals the configured version; the same network gives the same subject; the subject never appears in logs.
- Provisioning action order (§7.1): each failing precondition stops before the database call.
- `requireVisitor`: every `touch_session` state maps to the right render outcome; malformed cookie makes no database call; `rotated_reused` and `invalid_lineage` log exactly `{state, session_id}`; a throwing logger does not change the result.
- Cookie writes: no validation failure produces `Set-Cookie` for the session cookie; refresh writes only on renewal or `issued`.
- `startNewWorkspace`: a valid session is never deleted; an invalid one causes deletion of both cookies and `reload`; the client reloads at most once without a nonce.
- Cleanup harness target checks (with a fake connection): missing `SOVEREIGN_TEST_CLEANUP_DB_URL` or expected ref → abort; URL naming another project → abort; unexpected `current_user`, role attributes or extra privileges → abort; missing, duplicated or mismatching `test_environment` row → abort; deleted-count mismatch → rollback and abort. In every abort case no `DELETE` statement is issued.
- Username validation, generator validity, reserved names including the test prefix.
- Logs: sentinel tokens, nonces, hashes, IPs and network subjects never appear in any log line.
- Static guards: no session-level `SET` or `set_config(..., false)` in application SQL; the application database URL and rate-limit key never appear in client bundles.

### 9.2 Database / RLS (pgTAP via `pnpm test:rls`, control first; everything rolled back)
- **Session states:** for each of `invalid_input`, `unknown`, `revoked`, `rotated_grace`, `rotated_reused`, `expired`, `invalid_lineage`, `active`, `touch_session` returns exactly the §5.6 columns and performs exactly the listed writes; `current_visitor_id()` agrees with §5.6.
- **Boundaries:** `now()` is fixed within the test transaction, so tests arrange `expires_at` relative to it: `now() + interval '1 microsecond'` → `rotated_grace`; exactly `now()` → `rotated_reused`; the same pattern for `active` versus `expired`.
- **Initial-session first use:** sets only its own `first_used_at`; no other row changes (snapshot before and after); no rotation field is set; a second first use changes nothing.
- **Successor first use:** completes the predecessor exactly once (`rotated_at`, `successor_id`, grace `expires_at`) and supersedes other pending successors.
- **Lineage failures, each → `invalid_lineage`, no session changed, one event row:** predecessor deleted (`predecessor_id` null on an `origin = 'rotation'` row); predecessor revoked; predecessor's `successor_id` points at another successor. A superseded successor → `revoked`.
- **Lineage constraints:** an `origin = 'provisioning'` row with a predecessor is rejected; a predecessor or successor belonging to another visitor is rejected by the composite key; self-references are rejected; deleting a predecessor sets only `predecessor_id` to null.
- **Pending successor and context:** before completion, `current_visitor_id()` does not resolve a pending successor; after completion it does.
- **Renewal:** `active` renews only with `renew = true` after 24 hours; a rotated predecessor never renews.
- **Reuse:** each presentation after grace increments `reuse_count`, sets `last_reuse_at` and adds one `rotated_reuse` event with the right session and visitor ids; presentation during grace records nothing; a forced event-insert failure makes the call fail, with `reuse_count` unchanged and no visitor returned.
- **Rotation:** `issued` creates one successor (`origin = 'rotation'`) and leaves the predecessor `active`; a recent pending successor → `pending`; an old one → `superseded`; `not_due` and `not_active` change nothing.
- **Provisioning:** `created` creates exactly one visitor, workspace, initial session and nonce row; `joined` adds an initial session to the same visitor; expired window and `mint_count = 5` → `provisioning_expired`; invalid input creates nothing; a forced failure leaves no partial rows.
- **Rate limits through the functions only:**
  - spoofed or malformed network subjects (32 or 34 bytes, wrong version byte, null, the development subject with `allow_dev_network_key = false`) → `config_mismatch`, nothing consumed or created;
  - every attempt that passes the network check consumes the global bucket with exactly `\x00`, even when each attempt uses a different network subject, so varying subjects stops at the global limit;
  - `update_username` consumes `username_visitor_day` with the context's visitor id; with no context → `no_visitor`, nothing consumed; the limit is reached after the configured count;
  - inserts into `rate_limit_counters` with a wrong-length subject, a non-`\x00` global subject or an unknown bucket fail the constraints (run as the owner role to test the constraint itself);
  - `sovereign_app` cannot execute `consume_limit`, cannot write any counter and cannot read any `private` table.
- **One-to-one:** a second workspace for a visitor fails; a workspace without a visitor fails; the deferrable "visitor has a workspace" check, if approved (D17).
- **Isolation as `sovereign_app`:** A's context cannot select B's visitor or workspace row; `sovereign_app` cannot update any table directly.
- **Spoofing:** a fabricated hash, B's visitor id or malformed text in the context resolves to no visitor; `sovereign_app` cannot select `visitor_sessions`, `provisioning_nonces` or `session_events`, and cannot execute maintenance functions.
- **Transaction-local context:** after commit and after rollback, the setting is empty in the same connection.
- **Cleanup:** never deletes a visitor with a session `current_visitor_id()` accepts; deletes an abandoned visitor with cascade; leaves visitors younger than 1 day; session, nonce, counter and event cleanup follow §6.8.
- **Catalog:**
  - role attributes for `sovereign_app` and both owners (no `SUPERUSER`, no `BYPASSRLS`, owners `NOLOGIN`);
  - every function in `private`: expected `prosecdef`, `proconfig = {search_path=""}`, expected `proowner`, no `EXECUTE` or `format(` in `prosrc`, execute ACL exactly as §6.6 (`consume_limit` and the trigger function executable by no other role);
  - exact table **and column** privileges of every role, including that `sovereign_maintenance_owner` has no `SELECT` on `visitor_sessions.token_hash`, `provisioning_nonces.nonce_hash` or `rate_limit_counters.subject`, and `sovereign_app` has no write privilege;
  - RLS enabled on all eight tables; the exact policy set (§6.7);
  - `sovereign_test_cleanup` and `private.test_environment` are **absent** from the migration-built schema (they exist only after the development setup script).

### 9.3 Visitor-session integration (Vitest against the hosted development database)
- **Transaction-pool isolation:** with the client pool forced to one connection, a transaction with visitor A's context followed by one without context sees nothing in the second; many interleaved concurrent A/B transactions never see each other's rows.
- **Same-nonce concurrency:** N parallel `provision_visitor` calls with one nonce → one visitor, `created` once and `joined` for the rest up to the mint limit, then `provisioning_expired`.
- **Different-nonce interleaving:** parallel calls with different nonces → different visitors, each with exactly one workspace.
- **Retries and refreshes:** repeating a call with the same nonce after success → `joined`, same visitor.
- **Concurrent first use:** many parallel `touch_session` calls with one new initial session leave exactly one `first_used_at` and change nothing else; with one new successor, the predecessor is completed exactly once.
- **Old token during and after grace:** repeated use within grace succeeds; after grace each use is `rotated_reused` with one event each; the successor keeps working throughout.
- **Rotation races:** parallel refreshes with one predecessor → exactly one `issued`.
- **Lost rotation response:** an issued successor never used → predecessor stays `active`; a refresh after 10 minutes supersedes it and issues a new one.
- **Rate limits across instances:** two independent connection pools (two simulated application instances) provisioning from one network share one count; varying network subjects across both pools still stop at the global limit.
- **Cleanup safety:** cleanup run concurrently with session activity and with orphan visitors from different-nonce provisioning never deletes a visitor holding a valid session, and does delete eligible orphans.
- **Test-data cleanup:** every created visitor is recorded immediately and removed through the D20 path; the target checks pass only against the development project; a final check finds no remaining test visitors, workspaces, sessions or nonces; failures are reported with sanitised ids only.

### 9.4 Server Action / API integration (Vitest, server code against the development database)
- `provisionVisitor`: duplicated and retried requests with one nonce → one visitor; missing nonce → `reload`; session cookie present → `reload` without any database write; spoofed forwarding headers ignored; missing trusted address in production configuration → generic `unavailable` without a database call; limits enforced across two server instances sharing the database.
- `refreshSession`: renewal re-issues the cookie with matching `Max-Age`; rotation `issued` sets a new token; `pending`, `not_due`, `not_active` set no cookie.
- `updateUsername`: taken → conflict; invalid → validation; limit enforced per visitor regardless of request input; another visitor unchanged.
- `startNewWorkspace`: with a valid cookie → `reload`, nothing deleted; with an invalid cookie → both cookies deleted and `reload`; the following `GET` sets a nonce; provisioning then succeeds with it.
- `forgetThisBrowser`: the session and its pending successors are revoked; the cookie is deleted.
- Spoofed ids in form data or headers are ignored everywhere.

### 9.5 Browser (manual at G7 until an E2E tool is approved; Playwright is listed in ADR-0001 but not installed)
- Fresh profile → `/workspace` → workspace appears without any form; both cookies' attributes correct; nonce cookie gone after success.
- Several tabs opened at once in a fresh profile → usually one shared visitor; if two visitors appear, the browser ends with one valid session and the other visitor is an orphan (expected per §5.10).
- Reload during provisioning → same visitor.
- Reload and full browser restart → same username and workspace.
- Username edit persists across restart.
- Delete the session cookie → automatic provisioning of a new visitor. With a tampered or revoked cookie → the §2.6 notice; the cookie is not replaced until "Start a new workspace", which then provisions through a fresh nonce.
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
- **Operators, admin and moderation:** need an operator identity separate from visitor identity; reading `session_events` is an operator task until then.
- **Public profiles, publishing, reports:** usernames are not proof of a person; impersonation and squatting policies are needed if usernames become public.
- **Teams (ADR-0006):** dropped from the schema; re-adding them is a schema migration.
- **Storage:** any use must go through the server.
- **Privacy:** retention values (D11) and the lawful basis for storing network subjects (D13).

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
| D7 | Invalid cookie → notice; "Start a new workspace" clears the cookies, then provisions through a fresh nonce (§7.6); "Forget this browser" with warning | RD | |
| D8 | 180-day sliding idle lifetime; renewal ≥ 24 h apart, only for `active`, only in the refresh `POST`, cookie re-issued with matching `Max-Age`; no absolute cap | RD | Inactivity > 180 days loses the workspace |
| D9 | Cookie names `__Host-sovereign_session` / `__Host-sovereign_provision`; unprefixed names only on `http://localhost` | RD | |
| D10 | Session origins (`provisioning`/`rotation`) with separate first-use paths; rotation every 30 days; completion on successor first use with lineage checks (`invalid_lineage` fails closed); 120 s grace from completion, strict boundary; ≤ 1 recent pending successor (10 min); reuse recorded, no visitor-wide revocation | RD | |
| D11 | Retention: sessions 30 days after validity ends; abandoned visitors (older than 1 day, no session valid within 30 days) deleted with predicate re-check; nonces 1 day; counters 2 days; session events 90 days | RD (values and rules) · **UP** (scheduler: `pg_cron` availability or operator job) | |
| D12 | Synchronizer CSRF token | **DL** | Origin + `Sec-Fetch-Site` + `SameSite=Lax` required in M2 |
| D13 | Rate limits and global circuit breaker, enforced inside `provision_visitor` and `update_username` with canonical subjects | **AR** that they are mandatory before public deployment (user directive, 2026-11-12) · RD for the values and the database-enforced design | Shared-address trade-off needs approval |
| D14 | Trusted client-IP header | **UP** | Provider not selected; fail closed until configured (§5.12) |
| D15 | SECURITY DEFINER functions owned by `NOLOGIN` roles `sovereign_identity_owner` and `sovereign_maintenance_owner` with explicit, column-level privileges and owner-scoped RLS | RD · **UP** (hosted role-creation and ownership support, §6.10) | Fallback to `postgres` ownership is weaker and needs separate approval |
| D16 | PostgreSQL driver (new dependency), pooler mode, connection limit, application database URL variable name | **UP** — not approved | Needs compatibility review and dependency approval |
| D17 | Deferrable "visitor has a workspace" constraint trigger | RD | |
| D18 | Data API: not needed by M2 (design fact); disabling it entirely | Disabling: **DL** | Decide when Realtime/Storage plans are set |
| D19 | Proxy sets only the provisioning-nonce cookie, when neither cookie is present (no database access); session cookies written only by Server Actions | RD · **UP** (verify Next.js 16 proxy behaviour for cookie writes on prefetch and RSC requests before relying on it) | Proxy database access not recommended |
| D20 | Test cleanup through a development-only role, marker table and policies; harness verifies target, role and privileges and aborts before any delete; deletes only recorded ids with the prefix as second guard | RD · **UP** (pooler user-name format for target verification, §6.10) | No production role, function or privilege |
| D21 | Provisioning nonce: idempotency per nonce, 10-minute life, ≤ 5 mints; different nonces may create different visitors | RD | Tab coordination is a mitigation only |
| D22 | Reuse and lineage events recorded inside `touch_session` in `private.session_events`; app receives and logs only `{state, session_id}`; event-write failure fails the call | RD | |
| D23 | Username changes only through `update_username` (visitor from transaction context); `sovereign_app` has no table write privileges | RD | |
| D24 | Network subject format (version byte + HMAC-SHA-256), key version held in `private.identity_settings`, development subject only where explicitly allowed | RD | Variable names proposed |

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
The hosted-verification runbook must follow this specification once approved, with the owner-defined gates G1–G9. Its test expectations can be stated only after the tests in §9 exist. §6.10 lists the hosted facts it must verify first; the development setup script for test cleanup (§6.8) is an operator step it must include.
