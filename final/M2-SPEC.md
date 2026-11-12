# M2 Architecture Specification — Revision 4

**Accountless visitors, personal workspaces and agent Tasks for the public beta** (2026-11-13). Revisions 1–3 are superseded by this text.

Status: **specification only. Not approved for implementation or deployment.** No code, migration or test described here exists yet. No hosted verification has been performed. Revising this document approves nothing in it; §13 records what is confirmed by the client and what is only recommended.

Change rule (docs/README.md): before implementation this specification needs (1) a new ADR in `docs/DECISIONS/`, (2) updates to the living documents listed in §15, and (3) a ROADMAP status update. None of those has been made; this document does not edit them.

### Changes from revision 3
- **Scope (§1):** the client-confirmed product — a memecoin-related website showing a Contract Address as information only, where visitors try agents immediately. No wallet, signing or on-chain features. Beta of about 1–2 months.
- **Agent Tasks are now the primary concern (§5–§6):** a minimal, durable Task lifecycle (submit → queued → running → succeeded / failed / cancelled), a worker that claims Tasks from Postgres, one bounded model call per attempt, result persistence, polling for status, and cost controls.
- **Identity simplified (§4):** rotation, predecessor/successor lineage, grace periods, reuse detection, session events, the provisioning nonce, username editing and uniqueness, per-network HMAC key versioning, the maintenance owner role and the development cleanup role are **removed or deferred**. Kept: 256-bit tokens, hashes only, HttpOnly/Secure/`SameSite=Lax` cookie, server-side validation, expiry, revocation, CSRF/Origin checks, RLS isolation, no client-trusted ids.
- **Abuse and cost (§6):** mandatory global ceilings, per-visitor Task limits, input and output caps, timeouts, concurrency limits and a kill switch. Per-network limiting depends on a trusted client-IP source (client decision).
- **Database (§7):** one application role, one worker role, one `NOLOGIN` function owner. Tables: `visitors`, `workspaces`, `visitor_sessions`, `tasks`, plus small settings, counter and usage tables.
- **Launch boundary (§10–§11):** a prioritised checklist and a prerequisite table stating what blocks the first beta and what has a simpler alternative.
- **Decision register (§13):** D1–D24 re-statused; D25–D37 added.

---

## 1. Confirmed product scope and beta objective

### 1.1 Confirmed by the client (2026-11-13)
- Sovereign is a memecoin-related website that **displays a Contract Address as informational content**. It is **not** a crypto transaction application. (The address is already rendered by the landing page from `apps/web/src/components/landing/launchConfig.ts`; it is static content.)
- Visitors open the website and **try its agent features immediately**.
- Visitors are **never** asked to sign up, log in, sign in, connect a wallet, sign a message or perform a blockchain transaction.
- Visitors **submit Tasks to agents** and expect useful, relevant outputs that match their Tasks.
- Each visitor has a **display username**, which is **not** a credential.
- Each visitor has **one personal workspace** that persists in the same browser profile on the same device.
- **No cross-device recovery.** Clearing browser data, deleting the profile or losing the session cookie may make the previous workspace inaccessible; this is accepted.
- **Out of the product entirely:** wallet connection, wallet ownership verification, message signing, swaps, token purchases, claims, any on-chain transaction.
- The public beta may last **about 1–2 months**. Further investment depends on visitor feedback.

### 1.2 Beta objective
Put a **usable, secure, cost-controlled** agent experience in front of visitors quickly. Build only what makes Tasks work reliably and keeps each visitor's data private. Everything else waits for feedback.

### 1.3 Terminology
- **Task** (visitor-facing) = the beta's unit of work: one visitor's instruction to one agent and the agent's output. In the blueprint's vocabulary it is a simplified single-agent quest/decree with one run; §5.1 maps it.
- **Agent** = a platform-defined agent the visitor can choose (§5.2). Visitors do not create agents in the beta.

---

## 2. Architecture at a glance

```
Browser ──HTTPS──▶ Next.js server (apps/web)                      Worker (apps/worker, long-running Node)
  session cookie     • validates session (hash lookup)              • claims queued Tasks (SKIP LOCKED + lease)
  (HttpOnly)         • Server Actions: provision, submit, cancel    • one bounded model call per attempt
                     • reads own workspace/Tasks under RLS          • writes result / error, records usage
                     • connects as sovereign_app                    • connects as sovereign_worker
                              │                                               │
                              ▼                                               ▼
                       Supabase PostgreSQL (schema `sovereign` + `private`, RLS on every table)
                                                                              │
                                                              Model provider API (Anthropic, ADR-0001/§F.1)
```

- **Browser** holds only the session cookie. It never talks to Supabase or the model provider.
- **Web server** holds the `sovereign_app` database credential. It never holds the model API key.
- **Worker** holds the `sovereign_worker` database credential and the model API key. It never reads visitor sessions.
- **No Realtime, no queue extension, no Supabase Auth, no Data API use** in the beta (§3).

Security properties, in one place:
- Visitor isolation is enforced **twice**: application code always derives the workspace from the validated session, and RLS on `sovereign` tables filters every `sovereign_app` query by that session (§7.5–§7.6).
- `sovereign_app` cannot write any table directly; all writes go through small SECURITY DEFINER functions that derive the visitor from the transaction's session context (§7.4).
- The token hash, not a visitor id, is the database context, so code running SQL as `sovereign_app` can act only as visitors whose token hash it already holds. This is defence in depth, **not** protection against a compromised web server (§7.6).

---

## 3. Re-evaluation of Revision 3 for the beta

| Revision 3 element | Classification | Beta decision |
|---|---|---|
| 256-bit token, hash-only storage, HttpOnly/Secure cookie, server-side validation | **REQUIRED BEFORE PUBLIC LAUNCH** | Kept (§4). |
| `SameSite=Lax`, Origin + `Sec-Fetch-Site` checks on every `POST` | **REQUIRED BEFORE PUBLIC LAUNCH** | Kept (§4.7). |
| RLS isolation with a token-hash context, `sovereign_app` without `BYPASSRLS` | **REQUIRED BEFORE PUBLIC LAUNCH** | Kept (§7). |
| 180-day sliding lifetime | **SIMPLIFY FOR BETA** | One sliding idle lifetime, configurable; proposed 90 days (longer than the beta). Same renewal rule (§4.4). |
| 30-day rotation, predecessor/successor lineage, grace period, reuse detection, `session_events` | **DEFER UNTIL VALIDATED** | Removed. Trade-off in §4.8: a stolen cookie stays valid until expiry or revocation. |
| Provisioning nonce and per-nonce idempotency | **NOT REQUIRED FOR THIS PRODUCT** (beta) | Removed. Concurrent first loads may create an extra empty visitor; harmless and bounded by the provisioning cap (§4.5). |
| 210-day orphan retention and scheduled maintenance | **SIMPLIFY FOR BETA** | No scheduler. A documented operator cleanup query, run manually, with the same "never delete a visitor holding a valid session" rule (§9.3). |
| Two `NOLOGIN` owner roles | **SIMPLIFY FOR BETA** | One owner role, `sovereign_owner` (§7.2). Retention runs as an operator task, not through a maintenance role. |
| Database-enforced HMAC network rate limiting with key versioning | **SIMPLIFY FOR BETA** | Per-network limit kept as a capability (single HMAC key, no versioning) but **active only once a trusted client-IP source is verified** (§6.3). Global caps carry the mandatory protection. |
| Trusted proxy and IP canonicalisation | **SIMPLIFY FOR BETA** | Same rules, applied only if per-network limiting is enabled; otherwise no forwarding header is read at all (§6.3). |
| PostgreSQL driver and pooler | **REQUIRED BEFORE PUBLIC LAUNCH** | No secure alternative without `service_role`/`postgres` access; driver choice and pooler mode remain unapproved prerequisites (§11). |
| Maintenance scheduling | **DEFER UNTIL VALIDATED** | Manual operator cleanup during the beta (§9.3). |
| Development-only cleanup role, marker table, policies | **SIMPLIFY FOR BETA** | Marker table and expected-ref check kept; dedicated role and policies removed (§12.5). |
| Data API configuration | **NOT REQUIRED FOR THIS PRODUCT** (beta) | Unused; `sovereign`/`private` never exposed. Disabling it entirely is deferred (D18). |
| Username uniqueness, editing, `update_username`, username rate limit | **DEFER UNTIL VALIDATED** | Generated display username only (§4.2). |
| "Start a new workspace" and "Forget this browser" | **SIMPLIFY FOR BETA** | Kept as two small actions without the nonce flow (§4.6). |
| pgmq queue, Realtime Broadcast, budget reservation ledger, tool-calling loop, Forge, Convergence (from ARCHITECTURE §F) | **DEFER UNTIL VALIDATED** | Not needed for single-agent Tasks with one bounded model call (§5.1). |
| Wallet, signing, on-chain features | **NOT REQUIRED FOR THIS PRODUCT** | Excluded (§1.1). |

---

## 4. Visitor identity (beta)

### 4.1 Concepts
| Concept | What it is | Secret? | Rule |
|---|---|---|---|
| Username | Generated display name, e.g. `quiet-heron-41` | No | Display only. Never authorises anything. |
| Visitor id | `uuid` generated by PostgreSQL | No, **not a credential** | Never accepted from the client. |
| Session token | 32 random bytes (`node:crypto`), base64url, 43 characters | **Yes** | Lives only in the cookie. |
| Session token hash | SHA-256 of the token (32 bytes) | Sensitive | Stored in `visitor_sessions.token_hash`; never logged; not readable by `sovereign_app` or `sovereign_worker`. |
| Task id | `uuid` | No, **not a credential** | Access to a Task always requires the owning session (§5.8). |

Logs may contain visitor ids, workspace ids and Task ids. They never contain tokens, token hashes, cookie values, IP addresses or raw provider errors.

### 4.2 Username
- Generated server-side (`adjective-noun-NN`, `node:crypto` randomness), format `^[a-z0-9][a-z0-9_-]{2,23}$` checked in the server and by a database `CHECK`.
- Not unique (no public profiles in the beta, so no lookup by name). Not editable in the beta (D4, D23).

### 4.3 Cookie
| Attribute | Value |
|---|---|
| Name | `__Host-sovereign_session` in production; `sovereign_session` only on `http://localhost` |
| `HttpOnly` | yes |
| `Secure` | yes (off only on `http://localhost`) |
| `SameSite` | `Lax` (sent on top-level navigations from other sites, withheld on cross-site `POST`; `Strict` would make visitors arriving from links look new) |
| `Path` / `Domain` | `/` / not set (required by `__Host-`) |
| `Max-Age` | equal to the session's remaining lifetime; re-set on renewal |

### 4.4 Lifetime, renewal and states
- **Idle lifetime** (`SOVEREIGN_SESSION_IDLE_DAYS`, proposed **90 days**): `expires_at = now() + lifetime` at issue.
- **Renewal:** when a session is used and `now() - last_renewed_at >= 24 hours`, a **Server Action** extends `expires_at` and re-sets the cookie with a matching `Max-Age`. Page renders never renew, so the database expiry and the cookie cannot drift apart. No absolute cap.
- **States** of a presented token at transaction time `now()` (returned by `resolve_session`, §7.4):

| State | Condition | Result |
|---|---|---|
| `invalid_input` | cookie value not 43 base64url characters (checked before any database call), or hash not 32 bytes | no visitor |
| `unknown` | no row with this hash | no visitor |
| `revoked` | `revoked_at is not null` | no visitor |
| `expired` | `now() >= expires_at` (strict boundary) | no visitor |
| `active` | otherwise | visitor id, workspace id, `renewal_due` |

- `current_visitor_id()` (§7.6) accepts exactly the `active` sessions, so render-time validation and the database context agree.

### 4.5 First visit (provisioning)
1. A `GET` to the application area (M2: `/workspace`) without a session cookie renders the app shell with no visitor. **No `GET` creates database rows.**
2. A client component sends one same-origin `POST` (`provisionVisitor` Server Action) automatically; no form, no click.
3. The action runs the CSRF checks (§4.7), refuses if a session cookie is present (returns `reload`), generates the token and username, and calls `provision_visitor(token_hash, username, net_subject)`. The function applies the provisioning limits (§6.2) and creates visitor, workspace and session in one transaction.
4. The response sets the session cookie and returns the workspace view.

Concurrency: two tabs loading at the same moment may create two visitors. The browser keeps the cookie that arrives last; both tabs then use it. The other visitor has no Tasks and is removed by operator cleanup (§9.3). This is accepted for the beta; it costs one counted provisioning attempt and two small rows.

### 4.6 Invalid cookie, "Start a new workspace" and "Forget this browser"
- The server **never** deletes or replaces a session cookie because it failed to validate.
- A cookie in state `unknown`, `revoked`, `expired` or `invalid_input` shows a notice ("This browser's previous Sovereign workspace can no longer be opened") with a **"Start a new workspace"** button. Its Server Action re-checks the cookie: if it is now `active`, nothing is deleted and the page reloads; otherwise the response deletes the cookie and the client reloads, which leads to normal provisioning (§4.5). If cookies are blocked, the client reloads once and then shows "Cookies are required".
- **"Forget this browser"** (in the workspace, after a warning that access ends permanently) calls `revoke_session` and deletes the cookie.
- **Operator revocation:** `update sovereign.visitor_sessions set revoked_at = now(), revoke_reason = 'operator'` by an operator in SQL (§9.2).

### 4.7 CSRF and Origin checks (every state-changing request)
1. `Origin` is **required** and must equal the canonical origin (`SOVEREIGN_SITE_URL`, validated by `apps/web/src/lib/site-origin.ts`); a missing `Origin` is rejected. Next.js 16's own Server Action check lets a request without `Origin` through with a warning (bundled Next.js docs, `serverActions.md`), so it is not relied on.
2. `Sec-Fetch-Site`, when present, must be `same-origin`.
3. All state changes are `POST` Server Actions; the cookie is `SameSite=Lax`.
CSRF protection does not limit abuse; §6 is separate and also required.

### 4.8 Security trade-offs of the simplified identity
- **Theft:** the cookie is a bearer credential. Whoever holds it **is** that visitor until the session expires or is revoked: they can read the visitor's Tasks and results and submit Tasks against the visitor's limits. Without rotation (deferred), a copied cookie stays valid as long as the original. HttpOnly stops scripts reading it; XSS can still act inside the page, and malware or a copied browser profile can take it. There is no second factor and no way for the visitor to end the thief's access from another device. The real visitor can end it by "Forget this browser" (which also ends their own access) or by asking the operator, who has no way to verify who is asking.
- **Expiry:** after 90 days without use (proposed), access is lost permanently.
- **Loss:** clearing cookies or site data, deleting the profile or losing the device loses access permanently. Accepted by the client.
- **What is protected:** Task inputs and outputs of other visitors. The beta does not hold payment data, wallets or personal account data; it does hold whatever visitors type into Tasks (§9.4 privacy notice).

---

## 5. Agent execution (primary concern)

### 5.1 Evidence: what exists, what M2 provides, what is deferred
| Capability | Already in the project | M2 beta must provide | Deferred |
|---|---|---|---|
| Agent catalogue | Demo-only data in `apps/web/src/demo/data.ts` (5 Orders, 20 fictional agents, no instructions); design for agents/versions/Forge in ARCHITECTURE §F, DATA-MODEL §G | A small **platform-defined catalogue** of real agents with instructions (§5.2) | Visitor-created agents (Forge, M4), publishing, Codex |
| Task submission | Design only: decree → `public.issue_decree` RPC (§F.8) | `submit_task` with validation, limits and idempotency (§5.4) | Priorities, deadlines, multi-agent Tasks |
| Durable execution state | Design only: `agent_runs` with fenced lease (§F.8) | `sovereign.tasks` row as the single source of truth with a lease (§5.5) | Separate quests / runs / steps tables |
| Queue | Design only: pgmq (`[VERIFY AT M2]`) | **No queue extension**: the worker claims from `tasks` with `FOR UPDATE SKIP LOCKED` (the alternative already evaluated in §F.8) | pgmq, if volume or retry patterns require it |
| Worker | `apps/worker` has a configuration/health startup check only ("the queue loop arrives at M5", ARCHITECTURE §H.1 note) | A claim–execute–complete loop (§5.5–§5.6) | Heartbeats for long multi-step runs |
| Model provider | Design: own `ModelProvider` interface, first adapter Anthropic (ADR-0001, §F.1) | One adapter, one call per attempt, hard token and time caps (§5.6) | Second provider, streaming tokens |
| Tools (`web_search`, `web_fetch`, …) | Design only (§F.5, §F.10) | **None in the first beta** unless the client decides otherwise (D27) | Tools, tool idempotency, SSRF guard |
| Budget | Design: reservation/settlement ledger (§F.9, M4b) | Simpler bounded-cost controls (§6) | Reservation ledger, per-workspace monetary budgets |
| Status updates | Design: private Realtime Broadcast (§F.7) | **Polling** from the page (§5.9) | Realtime |
| Results | Design: `deliverables` table | Output text stored on the Task (§5.7) | Versioned deliverables, approvals |
| Approvals, Convergence, Guilds, Chronicle | Design only | — | All deferred |

### 5.2 Agents (D26)
- The beta offers a **fixed list of platform-defined agents**, defined in server-side code (a typed catalogue module in `packages/core`, version-controlled) with: `agent_key`, `version`, display name, Order, short public description, **system instructions** (server-side only, never sent to the browser), model id, `max_output_tokens`, and whether it is enabled.
- Which agents ship and their instructions are a **client decision** (D26). The demo's 20 agents are fictional placeholders and must not be shown as working agents unless they are given real instructions.
- Each Task records the `agent_key` and `agent_version` it ran with, so outputs stay traceable when instructions change.

### 5.3 Task lifecycle
```
              submit_task
                  │
                  ▼
   ┌──────────► queued ──── cancel (visitor) ───────────► cancelled
   │              │
   │   claim (worker, SKIP LOCKED, new lease)
   │              ▼
   │           running ──── model call ok ─────────────► succeeded
   │              │  └──── cancel requested, checked ──► cancelled
   │              │  └──── terminal error / attempts exhausted / queue timeout ─► failed
   │              │
   └── retryable error or lease expired (attempts < max) ── back to queued (after backoff)
```

| Status | Meaning | Who sets it |
|---|---|---|
| `queued` | Accepted and durable; waiting for a worker | `submit_task`; worker on retry |
| `running` | Claimed by a worker holding a valid lease | worker |
| `succeeded` | Output stored | worker |
| `failed` | Terminal; a sanitised `error_code` is stored | worker, or the queue-timeout sweep |
| `cancelled` | Stopped at the visitor's request | `cancel_task` (queued) or worker (running) |

Terminal states (`succeeded`, `failed`, `cancelled`) never change again.

### 5.4 Submission (`submitTask` Server Action → `submit_task` function)
1. CSRF checks (§4.7); resolve the session (`active` required).
2. Validate input with Zod in the server: `agent_key` exists and is enabled; Task text is non-empty, within `SOVEREIGN_TASK_INPUT_MAX_CHARS` (proposal: 4,000) and free of control characters other than newlines/tabs; an `idempotency_key` (UUID generated by the page per submission) is present.
3. Inside `withVisitor` (§7.6), call `submit_task(agent_key, agent_version, input, idempotency_key)`. The function:
   - derives the workspace from `current_visitor_id()` (never from an argument);
   - checks the kill switch and global submission ceilings (§6.2);
   - checks the visitor's limits: active Tasks (`queued` + `running`) and Tasks per day (§6.2);
   - inserts the Task as `queued`, or returns the existing Task if `(workspace_id, idempotency_key)` already exists — a double click or retried request never creates a second Task or a second charge against the limits.
4. Returns the Task id and status. Over-limit outcomes return generic messages (§6.4).

### 5.5 Claim and lease (worker)
- The worker polls (`SOVEREIGN_WORKER_POLL_MS`, proposal 1–2 s) and claims at most `SOVEREIGN_WORKER_CONCURRENCY` Tasks at once (proposal: 2):
  ```sql
  update sovereign.tasks t set status = 'running', attempts = attempts + 1,
         lease_token = gen_random_uuid(), lease_expires_at = now() + :lease, started_at = coalesce(started_at, now())
  where t.id = (select id from sovereign.tasks
                where status = 'queued' and not_before <= now()
                order by created_at for update skip locked limit 1)
  returning t.id, t.lease_token, …;
  ```
  Before claiming, the worker checks the kill switch and the global usage ceilings (§6.2); when blocked it claims nothing.
- **Lease** = model timeout + margin (proposal: 150 s for a 120 s call). A lease is needed because a worker can crash mid-call.
- **Recovery:** a `running` Task whose lease has expired is returned to `queued` (if `attempts < max_attempts`) or set `failed` (`error_code = 'worker_lost'`) by a sweep the worker runs each poll cycle. This can cause a second model call for the same Task (at-least-once execution; cost bounded by `max_attempts`, §6.2).
- **Fencing:** every completion update includes `where id = $1 and lease_token = $2 and status = 'running'`. A worker whose lease was taken over cannot overwrite the newer attempt.

### 5.6 Execution (one bounded model call per attempt)
- Build the request from the agent's system instructions and the visitor's Task text as the user message. The Task text is **data, not instructions to the platform**; agent instructions say so, but the system prompt is not treated as a security boundary (SECURITY §I.4). With no tools, a prompt injection can only influence the text of that visitor's own output.
- Call the provider with `max_tokens = agent.max_output_tokens` (proposal: ≤ 2,048) and a request timeout (proposal: 120 s, matching §F.5).
- Before the call and after it returns, check `cancel_requested`. If set after return, the output is discarded and the Task becomes `cancelled` (the call's cost has already been incurred).
- Record `input_tokens`, `output_tokens` and the model id from the provider's usage data on the Task and in the daily usage counters (§6.2).

### 5.7 Results, partial outputs and errors
- **Success:** output text stored in `tasks.output` (capped at `SOVEREIGN_TASK_OUTPUT_MAX_CHARS`, proposal 40,000), `status = 'succeeded'`. If the provider stopped because of the token cap, `output_truncated = true` and the page says the answer was cut short.
- **Refusals** by the model are stored as normal output; the page shows them as the agent's answer.
- **Retryable errors** (provider rate limit, 5xx, timeout, connection loss): back to `queued` with `not_before = now() + backoff` (proposal: 30 s, then 2 min) while `attempts < max_attempts` (proposal: 2); then `failed`.
- **Terminal errors** (request rejected as invalid, content filtered, configuration error): `failed` immediately.
- **Queue timeout:** a Task still `queued` after `SOVEREIGN_TASK_QUEUE_TIMEOUT_MIN` (proposal: 15 min) becomes `failed` (`error_code = 'queue_timeout'`), so visitors do not receive answers long after leaving.
- **Stored error data:** only an `error_code` from a fixed list (`provider_unavailable`, `rate_limited`, `timeout`, `rejected`, `worker_lost`, `queue_timeout`, `capacity`, `internal`). Raw provider messages go to server logs only (sanitised), never to the visitor or the database.
- **Display:** output is rendered as markdown through an allowlist sanitiser (no raw HTML, no `javascript:` links; SECURITY §I.4, LLM05). Failed Tasks show a plain message and a "Submit again" button that creates a **new** Task.

### 5.8 Isolation between visitors
- Every Task belongs to exactly one workspace (`tasks.workspace_id`, foreign key, not null).
- `sovereign_app` can read Tasks only through RLS (`workspace of current_visitor_id()`), and cannot insert, update or delete Tasks directly. `submit_task` and `cancel_task` derive the workspace from the session context; a Task id or workspace id sent by the browser is used only as a lookup **within** that workspace.
- A visitor cannot read, modify or cancel another visitor's Task: reads are filtered by RLS, `cancel_task(task_id)` updates only `where id = $1 and workspace_id = <context workspace>`, and a Task id from another workspace returns `not_found`, indistinguishable from a non-existent id.
- The worker sees all Tasks (it must), but cannot read `visitors` or `visitor_sessions`.

### 5.9 Status retrieval, closing the tab, reconnecting
- The Task page polls a Server Action (`getTask`) every few seconds (proposal: 2 s while `queued`/`running`, backing off to 10 s; stops at a terminal status). No Realtime in the beta.
- Execution does not depend on the browser: closing the tab, reloading or returning later does not affect a queued or running Task. On return the workspace lists the visitor's Tasks (most recent first, paginated) with status and output.
- A visitor who has lost their session cannot see their old Tasks (§4.8).

### 5.10 Cancellation
- `queued` → `cancelled` immediately by `cancel_task`.
- `running` → `cancel_task` sets `cancel_requested = true`; the worker finishes or abandons the current call and sets `cancelled` (§5.6).
- Terminal Tasks cannot be cancelled (`not_cancellable`).

---

## 6. Abuse prevention and cost control

### 6.1 Principles
- Anonymous execution will be abused and costs the operator money. Every limit below is enforced **server-side in the database functions or the worker**, never in the browser.
- All limits are **configuration values** (rows in `private.platform_settings`, §7.3, editable by an operator in SQL). The numbers marked "proposal" are starting points for the client to confirm after cost modelling; they are not evidence-based.
- When a limit is reached, the system **refuses new work with a generic message** and never half-processes a Task.

### 6.2 Limits
| Control | Where enforced | Proposal | When reached |
|---|---|---|---|
| **Kill switch** `execution_enabled` | `submit_task`, worker claim | `false` until launch | Submissions refused ("Agents are paused"); worker claims nothing; running Tasks finish |
| **Global daily Task ceiling** | `submit_task` | client to set from cost model | "Agents are busy today. Please try again tomorrow." |
| **Global daily token ceiling** (input + output, from provider usage) | worker before claim | client to set from cost model | Worker stops claiming; queued Tasks wait, then hit queue timeout |
| **Provider-side spend limit** | model provider account, if the provider offers one (to verify, §11) | set below the operator's tolerance | Provider rejects calls → Tasks fail `capacity` |
| **Global provisioning ceiling** (new visitors per hour) | `provision_visitor` | 1,000 / hour | "Sovereign is busy. Please try again shortly." |
| **Per-network provisioning limit** | `provision_visitor`, only when a trusted client IP is configured (§6.3) | 10 / hour and 30 / day | "Too many new workspaces from this network." |
| **Per-visitor active Tasks** (`queued` + `running`) | `submit_task` | 1 | "Wait for your current Task to finish." |
| **Per-visitor Tasks per day** | `submit_task` | 20 | "You've reached today's Task limit." |
| **Input size** | Server Action (Zod) and `CHECK` | 4,000 characters | Validation message |
| **Output size** | worker (`max_tokens`) and `CHECK` | 2,048 tokens / 40,000 characters | Truncation flag (§5.7) |
| **Per-attempt timeout** | worker | 120 s | Retryable error |
| **Attempts per Task** | worker | 2 | `failed` |
| **Worker concurrency** | worker | 2 per process | Tasks wait in queue |
| **Queue timeout** | worker sweep | 15 min | `failed` (`queue_timeout`) |
| **Request body size** | Next.js Server Actions default 1 MB (bundled docs) | keep default | Request rejected |

**Upper bound on cost:** per Task ≤ `max_attempts × (max input tokens × input price + max_output_tokens × output price)`; per day ≤ that × the global daily Task ceiling, and also ≤ the global token ceiling. Prices come from the provider's published pricing at the time of configuration; this document does not state them.

**Why these are enough for the first exposure:** a bot can create many visitors, but every visitor shares the global Task and token ceilings, so total cost is bounded. What a bot can still do is **exhaust the daily ceilings** and deny service to real visitors for the rest of the day. Per-network limiting reduces that; it depends on §6.3.

### 6.3 Client network identity (per-network limiting)
- Next.js handlers do not provide a trustworthy client address by themselves, and `X-Forwarded-For`, `Forwarded`, `X-Real-IP` and similar headers can be sent by anyone. **No such header is read unless explicitly configured.**
- A header may be trusted only if the hosting platform is selected and verified to (1) be the only public path to the app and (2) **overwrite** that header with the connecting address. Then `SOVEREIGN_CLIENT_IP_HEADER` names it, the value must parse as exactly one IP address, IPv6 is reduced to /64, and the server passes `HMAC-SHA-256(SOVEREIGN_RATE_LIMIT_KEY, network)` (32 bytes) to `provision_visitor`.
- If no trusted source is configured, `net_subject` is `null`, the per-network limit is **off**, and only the global ceilings apply. This is a **client decision** for the first beta (D14); it must be resolved before increasing traffic (§10 B).
- Raw IP addresses are never stored or logged.

### 6.4 Responses
Refusals use generic messages (above) and never reveal counts, thresholds, internal checks or infrastructure. Logs record the outcome name only.

---

## 7. Database model and roles

### 7.1 Placement
- Tables in schema `sovereign`; settings, counters and functions in `private`. Neither schema is exposed through the Supabase Data API; nothing Sovereign lives in `public`.
- Supabase Auth is unused; sign-ups and anonymous sign-ins are disabled in the dashboard so no `auth.users` rows can be created with the publishable key.
- No generic JSON storage tables; each table below serves a stated requirement.

### 7.2 Roles
| Role | Attributes | Used by | Purpose |
|---|---|---|---|
| `sovereign_app` | `LOGIN NOSUPERUSER NOBYPASSRLS NOCREATEDB NOCREATEROLE NOINHERIT`, connection limit | web server | Reads own rows under RLS; calls the visitor functions. No table writes. |
| `sovereign_worker` | same attributes | worker | Claims and completes Tasks; updates usage counters; reads settings. No access to `visitors` or `visitor_sessions`. |
| `sovereign_owner` | `NOLOGIN NOSUPERUSER NOBYPASSRLS NOCREATEDB NOCREATEROLE NOINHERIT` | nobody logs in | Owns the SECURITY DEFINER functions (§7.4); holds only the privileges they need, plus owner-scoped RLS policies. |
| `postgres` | Supabase-managed | operators, migrations | Owns tables. **Never** used by the web server or worker at runtime. |
| `anon`, `authenticated`, `service_role` | Supabase-managed | — | No privileges on `sovereign`/`private`. `service_role` is not used anywhere. |

Passwords for the two login roles are set out of band by the operator and stored only in the hosting platform's secret settings.

**Why one owner role instead of `postgres`:** a SECURITY DEFINER function runs with its owner's privileges. Owned by `postgres`, a bug in any function body could reach every object `postgres` can. Owned by `sovereign_owner`, it can reach only the table privileges listed in §7.4. Whether the hosted project lets `postgres` create this role and transfer function ownership is unverified (§11, P2); falling back to `postgres` ownership is weaker and needs explicit approval (D15).

### 7.3 Tables (illustrative DDL for review, not a migration)

```sql
create schema sovereign;

create table sovereign.visitors (
  id            uuid primary key default gen_random_uuid(),
  username      text not null check (username ~ '^[a-z0-9][a-z0-9_-]{2,23}$'),
  created_at    timestamptz not null default now(),
  last_seen_at  timestamptz not null default now()
);

create table sovereign.workspaces (
  id          uuid primary key default gen_random_uuid(),
  visitor_id  uuid not null unique references sovereign.visitors (id) on delete cascade,
  created_at  timestamptz not null default now()
);

create table sovereign.visitor_sessions (
  id               uuid primary key default gen_random_uuid(),
  visitor_id       uuid not null references sovereign.visitors (id) on delete cascade,
  token_hash       bytea not null unique check (octet_length(token_hash) = 32),
  created_at       timestamptz not null default now(),
  last_renewed_at  timestamptz not null default now(),
  expires_at       timestamptz not null check (expires_at > created_at),
  revoked_at       timestamptz,
  revoke_reason    text check (revoke_reason in ('forgotten', 'operator')),
  check ((revoked_at is null) = (revoke_reason is null))
);
create index on sovereign.visitor_sessions (visitor_id);

create table sovereign.tasks (
  id                uuid primary key default gen_random_uuid(),
  workspace_id      uuid not null references sovereign.workspaces (id) on delete cascade,
  idempotency_key   uuid not null,
  agent_key         text not null check (agent_key ~ '^[a-z0-9-]{1,40}$'),
  agent_version     text not null check (char_length(agent_version) between 1 and 40),
  input             text not null check (char_length(input) between 1 and 4000),
  status            text not null default 'queued'
                    check (status in ('queued', 'running', 'succeeded', 'failed', 'cancelled')),
  attempts          integer not null default 0 check (attempts between 0 and 5),
  not_before        timestamptz not null default now(),
  cancel_requested  boolean not null default false,
  lease_token       uuid,
  lease_expires_at  timestamptz,
  output            text check (output is null or char_length(output) <= 40000),
  output_truncated  boolean not null default false,
  error_code        text check (error_code in ('provider_unavailable', 'rate_limited', 'timeout', 'rejected',
                                               'worker_lost', 'queue_timeout', 'capacity', 'internal')),
  model             text,
  input_tokens      integer check (input_tokens >= 0),
  output_tokens     integer check (output_tokens >= 0),
  created_at        timestamptz not null default now(),
  started_at        timestamptz,
  finished_at       timestamptz,
  unique (workspace_id, idempotency_key),
  check ((status = 'running') = (lease_token is not null and lease_expires_at is not null)),
  check ((status = 'succeeded') = (output is not null)),
  check ((status = 'failed') = (error_code is not null)),
  check ((status in ('succeeded', 'failed', 'cancelled')) = (finished_at is not null))
);
create index tasks_claim_idx     on sovereign.tasks (created_at) where status = 'queued';
create index tasks_workspace_idx on sovereign.tasks (workspace_id, created_at desc);
create index tasks_lease_idx     on sovereign.tasks (lease_expires_at) where status = 'running';

create table private.platform_settings (               -- exactly one row
  singleton                       boolean primary key default true check (singleton),
  execution_enabled               boolean not null default false,
  global_daily_task_ceiling       integer not null check (global_daily_task_ceiling >= 0),
  global_daily_token_ceiling      bigint  not null check (global_daily_token_ceiling >= 0),
  global_hourly_provision_ceiling integer not null check (global_hourly_provision_ceiling >= 0),
  net_hourly_provision_limit      integer not null check (net_hourly_provision_limit >= 0),
  net_daily_provision_limit       integer not null check (net_daily_provision_limit >= 0),
  visitor_active_task_limit       integer not null check (visitor_active_task_limit >= 0),
  visitor_daily_task_limit        integer not null check (visitor_daily_task_limit >= 0)
);

create table private.daily_usage (
  day              date primary key,
  tasks_submitted  integer not null default 0 check (tasks_submitted >= 0),
  input_tokens     bigint  not null default 0 check (input_tokens >= 0),
  output_tokens    bigint  not null default 0 check (output_tokens >= 0)
);

create table private.provision_counters (
  subject       bytea not null check (octet_length(subject) in (1, 32)),   -- '\x00' = global, 32 = network HMAC
  window_start  timestamptz not null,
  window_kind   text not null check (window_kind in ('hour', 'day')),
  hit_count     integer not null check (hit_count > 0),
  primary key (subject, window_kind, window_start)
);
```

Notes:
- **One-to-one visitor ↔ workspace:** `workspaces.visitor_id` is `NOT NULL UNIQUE`, and the only insert path (`provision_visitor`) creates both in one transaction.
- **Cascade:** deleting a visitor deletes its workspace, sessions and Tasks.
- **Per-visitor daily Task limit** is counted from `tasks` (`created_at >= date_trunc('day', now())` for the workspace), so it needs no extra table.
- The input `CHECK` (4,000) mirrors the proposal in §6.2; if the client changes the limit, the constraint changes with it in a migration.
- `day` boundaries use the database's UTC date.

### 7.4 Functions (all in `private`, owned by `sovereign_owner`, `SECURITY DEFINER`, `set search_path = ''`, fully qualified names, no dynamic SQL)
| Function | `EXECUTE` granted to | Behaviour |
|---|---|---|
| `current_visitor_id() → uuid` (`STABLE`) | `sovereign_app` | Visitor of the `active` session whose hash is in the transaction setting (§7.6); else null. Read-only. |
| `resolve_session(token_hash bytea, renew boolean) → (state, visitor_id, workspace_id, renewal_due)` | `sovereign_app` | §4.4. With `renew = true` and renewal due, extends `expires_at`, sets `last_renewed_at` and `visitors.last_seen_at`. |
| `provision_visitor(token_hash bytea, username text, net_subject bytea) → (outcome, visitor_id, workspace_id)` | `sovereign_app` | Applies the global and (if `net_subject` is not null) per-network provisioning limits, then creates visitor, workspace and session. `outcome ∈ {created, rate_limited_network, rate_limited_global}`. |
| `revoke_session(token_hash bytea) → boolean` | `sovereign_app` | Sets `revoked_at`, `revoke_reason = 'forgotten'` on an `active` session. |
| `submit_task(agent_key text, agent_version text, input text, idempotency_key uuid) → (outcome, task_id)` | `sovereign_app` | §5.4. `outcome ∈ {queued, existing, paused, global_limit, active_limit, daily_limit, no_visitor}`. Increments `daily_usage.tasks_submitted` only for a new Task. |
| `cancel_task(task_id uuid) → text` | `sovereign_app` | §5.10. `cancelled`, `cancel_requested`, `not_cancellable` or `not_found` (also for other visitors' Tasks). |

Privileges of `sovereign_owner` (nothing else):
- `sovereign.visitors`: `SELECT`, `INSERT`, `UPDATE (last_seen_at)`.
- `sovereign.workspaces`: `SELECT`, `INSERT`.
- `sovereign.visitor_sessions`: `SELECT`, `INSERT`, `UPDATE (last_renewed_at, expires_at, revoked_at, revoke_reason)`.
- `sovereign.tasks`: `SELECT`, `INSERT`, `UPDATE (status, cancel_requested, finished_at)`.
- `private.platform_settings`: `SELECT`. `private.daily_usage`: `SELECT`, `INSERT`, `UPDATE (tasks_submitted)`. `private.provision_counters`: `SELECT`, `INSERT`, `UPDATE (hit_count)`.
- `USAGE` on both schemas. **No `DELETE`.** Owner-scoped RLS policies (`using/with check true`) exist for exactly these commands.

Privileges of `sovereign_worker`:
- `sovereign.tasks`: `SELECT`, `UPDATE (status, attempts, not_before, lease_token, lease_expires_at, output, output_truncated, error_code, model, input_tokens, output_tokens, started_at, finished_at)`; RLS policies for `select` and `update` (`using true`). No `INSERT` or `DELETE`.
- `private.platform_settings`: `SELECT`. `private.daily_usage`: `SELECT`, `INSERT`, `UPDATE (input_tokens, output_tokens)`.
- `USAGE` on both schemas. Nothing on `visitors`, `workspaces`, `visitor_sessions` or `provision_counters`.

Privileges of `sovereign_app`:
- `SELECT` on `sovereign.visitors`, `sovereign.workspaces`, `sovereign.tasks` (RLS: own rows only); `EXECUTE` on the six functions above; `USAGE` on both schemas. **No `INSERT`, `UPDATE` or `DELETE` on any table**, nothing on `visitor_sessions` or any `private` table.

`EXECUTE` on every function is first revoked from `public`, `anon`, `authenticated` and `service_role`. Whether Supabase's default privileges add grants on new objects must be checked by the catalog tests (§12.2), not assumed.

### 7.5 RLS
```sql
-- RLS enabled on every table in sovereign and private.
create policy visitors_own   on sovereign.visitors   for select to sovereign_app
  using (id = (select private.current_visitor_id()));
create policy workspaces_own on sovereign.workspaces for select to sovereign_app
  using (visitor_id = (select private.current_visitor_id()));
create policy tasks_own      on sovereign.tasks      for select to sovereign_app
  using (workspace_id = (select w.id from sovereign.workspaces w
                         where w.visitor_id = (select private.current_visitor_id())));
-- Owner and worker policies as listed in §7.4. No policies for anon, authenticated or service_role.
```

### 7.6 Visitor context per transaction
```sql
begin;
select set_config('sovereign.session_token_hash', $1, true);  -- lowercase hex of sha256(token)
-- RLS-filtered reads and function calls
commit;
```
- `current_visitor_id()` decodes the setting (must match `^[0-9a-f]{64}$`), finds a session with that hash, `revoked_at is null` and `now() < expires_at`, and returns its `visitor_id`; otherwise null, so every policy returns no rows.
- `set_config(…, true)` is transaction-local; PostgreSQL discards it at commit or rollback, so it cannot leak to the next transaction on a pooled connection. The web server's data helper (`withVisitor`) always opens an explicit transaction and sets the context first. Session-level `SET` is forbidden in application code (static test, §12.1).
- **Why a token hash and not a visitor id:** code running SQL as `sovereign_app` (for example via an injection bug) could set any visitor id; it cannot obtain token hashes, because `sovereign_app` cannot read `visitor_sessions`. It can therefore act only as visitors whose hash it already holds.
- **What this does not protect against:** a compromised web server sees live cookies and holds the `sovereign_app` password; a compromised worker can read every Task. RLS is defence in depth against application bugs, not isolation from a compromised server.

---

## 8. Application structure (mapped to the current source, read-only inventory)

### 8.1 Status of the existing account-based M2 work
Based on evidence recorded earlier in the project's working session — **not re-verified for this revision; no Git commands were run**: at that session's start `HEAD` was `10013c0` and the account-based M2 files were untracked; later measurements showed the same `HEAD` and nothing staged. On that evidence the account-based work is not contained in any commit on record, has not been merged or pushed, and has never been applied to a hosted database. If it has since been committed locally, this paragraph must be updated from verified Git evidence.

### 8.2 Existing files and their fate
| Area | Files | Fate |
|---|---|---|
| Auth pages, actions, helpers | `apps/web/src/app/auth/**`, `apps/web/src/lib/auth/**`, `apps/web/src/server/auth.ts` | Removed (no accounts). `AuthShell.tsx`/`auth.css` styling may be reused for notices. |
| Supabase JS clients and proxy | `apps/web/src/lib/supabase/**`, `apps/web/src/proxy.ts` | Removed from the visitor path; the beta needs no proxy logic. |
| Canonical origin | `apps/web/src/lib/site-origin.ts` | Kept for the Origin check. |
| Workspace page | `apps/web/src/app/(app)/workspace/page.tsx` | Rewritten: Task list, Task form, Task view. |
| Composition roots | `apps/web/src/server/composition.ts`, `apps/worker/src/composition.ts`, `apps/worker/src/main.ts` | Gain database adapters; the worker gains the claim loop and model adapter. |
| DB configuration | `packages/db/src/config.ts` (`parseWorkerDatabaseUrl` already rejects `postgres`, `supabase_admin`, `service_role`, `authenticator`) | Reused for the worker; same pattern for the web URL requiring `sovereign_app`. |
| Core | `packages/core/src` (health only) | Gains the agent catalogue, Task validation schemas and the `ModelProvider` port. |
| Demo data | `apps/web/src/demo/**` | Stays for public pages until replaced; never shown as real Task results. |
| Migrations | `packages/db/supabase/migrations/20261112120000_m2_schemas_privileges.sql`, `20261112120100_m2_identity.sql` | Depend on `auth.users`/`auth.uid()`. **Replaced**, not amended (requires approval). |
| pgTAP and runner | `packages/db/supabase/tests/**`, `scripts/run-sql-tests.mjs`, `scripts/tap.mjs` | Suite files replaced (§12.2); control file, manifest rules and runner kept. |
| Integration tests | `apps/web/test/integration/**` | Replaced (§12.3); `support/diagnostics.ts` reusable. Service-role key no longer needed. |
| Guards | `apps/web/test/service-role-isolation.test.ts`, `scripts/check-client-bundle.mjs`, ESLint rules | Kept; extended so the database URLs and the model API key never reach client code or bundles. |
| Dependencies | `@supabase/ssr`, `@supabase/supabase-js` | Unused by the beta. New: a PostgreSQL driver and a model-provider client (both need approval, §11). |

### 8.3 New modules (proposed, not created)
- `apps/web/src/server/visitor.ts` (`server-only`): cookie, token, hashing, `requireVisitor`, `withVisitor`, CSRF checks.
- `apps/web/src/app/(app)/workspace/actions.ts`: `provisionVisitor`, `refreshSession`, `submitTask`, `getTask`, `listTasks`, `cancelTask`, `startNewWorkspace`, `forgetThisBrowser`.
- `packages/core`: agent catalogue, Zod schemas, `ModelProvider` port, retry classification.
- `packages/db`: `app-database.ts` (web, `sovereign_app`) and `worker-database.ts` (worker, `sovereign_worker`).
- `apps/worker`: claim loop, lease sweep, model adapter, sanitised logging.

---

## 9. Operations during the beta

### 9.1 Visibility (minimum)
- Structured logs from web and worker: Task id, workspace id, agent key, status transitions, attempt, duration, token counts, `error_code`. No secrets, no Task text, no outputs in logs.
- Operator SQL (documented in the runbook): queue depth and oldest queued Task; Tasks per status in the last hour; failure rate by `error_code`; today's `daily_usage` against ceilings; provisioning counts.
- Worker liveness: the operator can see whether Tasks are being claimed (oldest `queued` age). An automated alert is required before increasing traffic (§10 B).

### 9.2 Operator controls
- Kill switch: `update private.platform_settings set execution_enabled = false`.
- Ceiling changes: update `platform_settings`.
- Session revocation: §4.6. Operator actions run as `postgres` in the SQL editor — operator access, not application runtime.

### 9.3 Cleanup (manual during the beta)
- Abandoned visitors: delete visitors older than 1 day that have **no session with `least(revoked_at, expires_at) > now() - interval '30 days'`**. A visitor holding a valid session can never match (its `expires_at > now()`), and no code path creates a session for an existing visitor, so the rule cannot delete a usable workspace. Cascades remove workspace, sessions and Tasks.
- Old sessions: delete sessions with `least(revoked_at, expires_at) < now() - interval '30 days'`.
- Counters: delete `provision_counters` rows older than 2 days; keep `daily_usage`.
- Run by an operator as `postgres` (weekly is sufficient at beta volume). Automated scheduling is deferred (D11).

### 9.4 Data, privacy and backups
- Stored per visitor: generated username, timestamps, session hashes, Task text, outputs, token counts. No email, wallet or payment data.
- Task text may contain anything a visitor types. A short notice near the Task form ("Don't include personal or sensitive information") and a privacy statement are a **client decision** (D35) and a launch item (§10 A).
- Task retention period: **client decision** (D35); until decided, Tasks live as long as their visitor.
- Backups: what the hosted project's plan provides (frequency, point-in-time recovery, retention) is **unverified**. The client must either confirm the plan's backups or accept that a database loss during the beta loses visitors' workspaces (D36). A restore rehearsal is deferred (§10 C).

---

## 10. Public-beta launch checklist

### A. Mandatory before public exposure
Security and isolation:
- [ ] Session design of §4 implemented: 256-bit tokens, hash-only storage, cookie attributes, server-side validation, expiry, revocation, invalid-cookie handling.
- [ ] CSRF/Origin checks on every state-changing request (§4.7).
- [ ] Roles, grants, RLS and functions of §7 applied; catalog tests pass against the hosted development project (no `BYPASSRLS`, no table writes for `sovereign_app`, exact grants and policies).
- [ ] Visitor A/B isolation tests pass for visitors, workspaces and Tasks, including cancel and spoofed ids (§12).
- [ ] No database credential, model API key or service-role key in browser bundles (`check:bundle` on a fresh production build) or in the repository.
- [ ] Web server uses only `sovereign_app`; worker uses only `sovereign_worker`; neither uses `postgres` or `service_role`.
Agent reliability:
- [ ] Agent catalogue with real instructions approved by the client (D26); model chosen (D32).
- [ ] Task lifecycle of §5 works end to end against the real provider: submit, claim, success, retryable failure, terminal failure, timeout, lease recovery, cancel, queue timeout, idempotent resubmission.
- [ ] Outputs rendered through the markdown sanitiser; error messages generic.
Cost and abuse:
- [ ] Kill switch, global daily Task ceiling, global daily token ceiling, global provisioning ceiling, per-visitor active and daily limits, input/output caps, timeouts, attempt cap, worker concurrency — all configured with client-approved values (D30).
- [ ] Provider-side spend limit set, if the provider offers one; otherwise the client accepts that the in-app token ceiling is the only spend stop (D30).
- [ ] Decision on per-network limiting for the first exposure (D14).
Operations:
- [ ] Logs from web and worker reachable by the operator; operator SQL for queue, failures and usage available (§9.1).
- [ ] Operator procedure for kill switch, ceilings and revocation written down (§9.2).
- [ ] Backup position confirmed or accepted (D36); Task-input notice and privacy statement decided (D35).
- [ ] Hosted prerequisites P1–P6 verified (§11).
- [ ] Hosted verification runbook revised for this design and executed gate by gate with recorded evidence. **A build that compiles and local tests that pass are not sufficient.**

### B. Required before increasing traffic
- [ ] Trusted client-IP source verified and per-network provisioning limits enabled (D14).
- [ ] Automated alerting: worker not claiming, failure rate high, usage near ceilings.
- [ ] Load check of the claim loop and provider rate limits at the planned concurrency.
- [ ] Scheduled cleanup instead of manual (D11).
- [ ] Review of real costs per Task against the ceilings; adjust values.
- [ ] Browser end-to-end tests automated (Playwright needs dependency approval).

### C. Can wait until feedback validates the product
- Session rotation and reuse detection (D10, D22); username editing and uniqueness (D4, D23).
- Tools for agents (`web_search`, `web_fetch`), tool idempotency and SSRF guard (D27).
- Realtime status (D29), pgmq (D28), budget reservation ledger (D30 follow-up), visitor-created agents (Forge), Convergence, Guilds, Chronicle, public profiles.
- Synchronizer CSRF token (D12), disabling the Data API entirely (D18).
- Restore rehearsal, retention automation, operator UI.

---

## 11. External prerequisites
| # | Prerequisite | Why necessary | Blocks first beta? | Simpler supported alternative | Must verify before relying on it |
|---|---|---|---|---|---|
| P1 | Custom login roles `sovereign_app` and `sovereign_worker` can connect to the hosted database (pooler or direct) | The only way to avoid `postgres`/`service_role` at runtime | **Yes** | Direct connection instead of the pooler | A test connection as each role on the development project; the exact user-name format for the pooler |
| P2 | `postgres` can create the `NOLOGIN` role `sovereign_owner` and transfer function ownership to it | Limits what function bodies can reach (§7.2) | **Yes**, unless the weaker fallback is approved | Functions owned by `postgres` — weaker, **needs explicit approval** (D15) | Run the role creation and `ALTER FUNCTION … OWNER TO` on the development project |
| P3 | Transaction-local `set_config` behaves correctly through the chosen connection mode | Visitor context must not leak between requests (§7.6) | **Yes** | Session-mode pooler or direct connection with a small server-side pool | Pool-isolation tests (§12.3) |
| P4 | A PostgreSQL driver for Node (new dependency) compatible with the chosen pooler mode | The web server and worker must talk SQL | **Yes** | — (Supabase JS clients would require `service_role` or Supabase Auth, both excluded) | Dependency approval under the repository's supply-chain policy; prepared-statement behaviour in transaction mode |
| P5 | Hosting for the web app **and** a long-running worker process, with secret storage and outbound access to the model provider | Tasks outlive request handlers; secrets must stay server-side | **Yes** | Run the worker as a second process on the same host as the web app | Host chosen by the client (D37); worker can run continuously; secrets not exposed in logs or build output |
| P6 | Model provider account, API key, chosen model, and (if offered) a provider-side spend limit | Agents need a model; spend must be capped | **Yes** for the key and model; spend limit is a strong backstop | In-app token ceiling only (D30) | Key works from the worker host; the provider console's spend-limit feature exists and is set |
| P7 | Trusted client-IP header on the chosen host | Per-network provisioning limits (§6.3) | No (client decision D14); **yes before increasing traffic** | Global ceilings only | Host documentation **and** a test that a client-supplied header is overwritten |
| P8 | Backup capability of the hosted plan | Recovering visitors' workspaces after a database loss | No, if the client accepts the risk (D36) | Accept loss for the beta | The plan's documented backup features |
| P9 | `pg_cron` or another scheduler | Automated cleanup | No | Manual operator cleanup (§9.3) | — |

No item above is assumed to be supported by Supabase or the host; each needs the stated evidence.

---

## 12. Testing requirements (none of these tests exists; no counts are stated)

### 12.1 Unit (offline)
- Token generation, hashing, cookie attributes (production and localhost), `Max-Age` equal to remaining lifetime.
- CSRF checks: missing or foreign `Origin` rejected; `Sec-Fetch-Site: cross-site` rejected; forwarding headers ignored unless configured.
- Task input validation: empty, oversize, control characters, unknown or disabled agent, missing idempotency key.
- Agent catalogue: every enabled agent has instructions, a model, an output cap; instructions never reach client bundles.
- Retry classification of provider errors; backoff; attempt cap; truncation flag on token-cap stop.
- Markdown sanitiser: no raw HTML, no `javascript:` links.
- Logs never contain tokens, hashes, cookie values, Task text, outputs or IPs.
- Static guards: no session-level `SET` in application SQL; no database URL or model key in client code.

### 12.2 Database / RLS (pgTAP via `pnpm test:rls`, control file first, all rolled back)
- Session states and boundaries (`expires_at` exactly `now()` → `expired`); renewal only when due; revoked sessions resolve to nothing.
- Provisioning creates exactly one visitor, workspace and session; global and per-network limits; `null` network subject skips only the per-network limit.
- `submit_task`: workspace from context only; kill switch; global and per-visitor limits; idempotent resubmission returns the same Task and does not count twice; no context → `no_visitor`.
- `cancel_task`: queued → cancelled; running → `cancel_requested`; terminal → `not_cancellable`; another visitor's Task → `not_found` and unchanged.
- Isolation as `sovereign_app`: visitor A sees none of B's visitors, workspaces or Tasks; spoofed context (fabricated hash, B's visitor id) sees nothing; no direct `INSERT`/`UPDATE`/`DELETE` succeeds anywhere.
- As `sovereign_worker`: can claim and complete Tasks; cannot read `visitors` or `visitor_sessions`; cannot insert or delete Tasks; a completion with a stale lease token changes nothing.
- Constraints: status/lease/output/error/finished consistency checks; one workspace per visitor.
- Catalog: role attributes; function owner, `prosecdef`, `search_path=""`, no `EXECUTE`/`format(` in bodies, exact `EXECUTE` grants; exact table and column privileges per role; RLS enabled on every table; exact policy set.

### 12.3 Integration (against the hosted development project, with a fake model provider by default)
- Pool isolation: one-connection pool, A's context then no context sees nothing; interleaved A/B transactions never cross.
- Worker lifecycle with the fake provider: success; retryable error then success; retryable errors until `failed`; terminal error; timeout; worker killed mid-call → lease expiry → re-claim → single final result; cancel while queued and while running; queue timeout; kill switch stops claiming; token ceiling stops claiming.
- Concurrency: several workers never claim the same Task; duplicate submissions with one idempotency key create one Task.
- One opt-in live smoke test against the real provider with a tiny prompt and output cap.
- Test-data cleanup per §12.5.

### 12.4 Browser (manual until an E2E tool is approved)
- Fresh profile → workspace without any form; submit a Task; close the tab; return → result shown.
- Reload and browser restart keep the same workspace and Tasks.
- Delete or tamper with the cookie → notice; "Start a new workspace" works.
- Two profiles see only their own Tasks; a copied Task URL shows nothing to the other profile.
- A cross-site page posting to a Server Action changes nothing.
- Limits reached → generic messages; kill switch → "Agents are paused".

### 12.5 Test-data cleanup (development project only)
- Integration tests record every visitor they create and use the reserved username prefix `zz-test-`.
- The harness requires `SOVEREIGN_TEST_EXPECTED_PROJECT_REF` and a cleanup database URL (no defaults, no fallback to other variables). Before deleting it verifies that the URL names the expected project and that `private.test_environment` (a one-row marker created only by a development setup script, never by a migration) holds `environment = 'development'` and the expected ref. Any mismatch aborts with no `DELETE`.
- It deletes only recorded visitor ids that also carry the prefix, checks the deleted count, and verifies nothing related remains.
- No cleanup privilege exists in production; `sovereign_app` and `sovereign_worker` have no `DELETE`.

---

## 13. Decision register

Status: **ACCEPTED** (kept in this revision) · **REVISED** · **DEFERRED** · **REJECTED** · **NEEDS CLIENT DECISION**. Approval: **Client** = confirmed by the client; **Rec.** = recommendation awaiting client approval. Nothing marked Rec. is approved.

| # | Decision | Status | Approval | Notes / evidence needed |
|---|---|---|---|---|
| D1 | Accountless visitors; server-generated id; HttpOnly bearer session; no Supabase Auth; username display-only; same-profile persistence; no cross-device recovery; no wallet or on-chain features | ACCEPTED | **Client** (2026-11-12, scope confirmed 2026-11-13) | |
| D2 | Tables in non-exposed `sovereign`, support objects in `private` | ACCEPTED | Rec. | |
| D3 | Generated username `adjective-noun-NN`, format check | REVISED | Rec. | Uniqueness removed |
| D4 | Unique usernames | DEFERRED | Rec. | Needed only with public profiles |
| D5 | Reserved-name denylist | REVISED | Rec. | Only the test prefix `zz-test-` matters while names are generated |
| D6 | `GET` never creates rows; provisioning by same-origin `POST` | ACCEPTED | Rec. | |
| D7 | Invalid-cookie notice; "Start a new workspace"; "Forget this browser" | REVISED | Rec. | Simplified, no nonce (§4.6) |
| D8 | Sliding idle lifetime, renewal ≥ 24 h apart in a Server Action | REVISED | **NEEDS CLIENT DECISION** on the value | Proposed 90 days |
| D9 | Cookie name `__Host-sovereign_session` | REVISED | Rec. | Nonce cookie removed |
| D10 | Rotation, lineage, grace, reuse detection | DEFERRED | Rec. | Trade-off §4.8 |
| D11 | Retention and scheduled cleanup | REVISED | Rec. | Manual operator cleanup (§9.3); scheduling deferred |
| D12 | Synchronizer CSRF token | DEFERRED | Rec. | |
| D13 | Rate limiting mandatory before public exposure | REVISED | **Client** (mandatory); values Rec. | Now covers provisioning and Task execution (§6) |
| D14 | Per-network limits from a trusted client-IP header | **NEEDS CLIENT DECISION** | — | Launch with global ceilings only, or wait for a verified header (P7) |
| D15 | Functions owned by `NOLOGIN` `sovereign_owner` | REVISED | Rec. | One owner role; `postgres` ownership only with explicit approval if P2 fails |
| D16 | PostgreSQL driver and connection mode | **NEEDS CLIENT DECISION** | — | Dependency approval + P1/P3/P4 evidence |
| D17 | Deferrable "visitor has a workspace" trigger | DEFERRED | Rec. | Single insert path suffices for the beta |
| D18 | Disable the Data API entirely | DEFERRED | Rec. | Unused either way |
| D19 | Proxy sets a provisioning nonce | REJECTED | Rec. | Nonce removed; no proxy logic needed |
| D20 | Test-data cleanup safeguards | REVISED | Rec. | Marker + expected ref kept; dedicated role removed (§12.5) |
| D21 | Provisioning nonce idempotency | REJECTED | Rec. | Duplicate empty visitors accepted (§4.5) |
| D22 | Session events table | DEFERRED | Rec. | With rotation |
| D23 | Username editing via function | DEFERRED | Rec. | |
| D24 | Network subject with key versioning | REVISED | Rec. | Single HMAC key, no versioning; only if D14 enables it |
| D25 | Beta Task model: one table `sovereign.tasks` as the single source of truth; statuses queued/running/succeeded/failed/cancelled | ACCEPTED (new) | Rec. | Maps to quests/agent_runs later |
| D26 | Platform-defined agent catalogue in code; which agents and their instructions | **NEEDS CLIENT DECISION** | — | List of agents and approved instructions |
| D27 | No agent tools in the first beta | **NEEDS CLIENT DECISION** | — | Recommended: none; tools add a search provider, SSRF and cost risk |
| D28 | Worker claims with `SKIP LOCKED` + lease; no pgmq; `sovereign_worker` has narrow direct table grants on `tasks` | ACCEPTED (new) | Rec. | Option already evaluated in ARCHITECTURE §F.8. Deviates from ADR-0003's function-only worker API, which is deferred with the full runtime |
| D29 | Polling for status; no Realtime | ACCEPTED (new) | Rec. | |
| D30 | Cost controls of §6.2 and their values; provider-side spend limit | **NEEDS CLIENT DECISION** | — | Values from a cost model; whether the provider's spend limit is set |
| D31 | Per-visitor limits (active, daily) | ACCEPTED (new) | Rec. | Values part of D30 |
| D32 | Model provider and model | **NEEDS CLIENT DECISION** | — | Anthropic per ADR-0001; specific model and price |
| D33 | One bounded model call per attempt; at most 2 attempts; queue timeout | ACCEPTED (new) | Rec. | |
| D34 | Output rendered through a markdown allowlist sanitiser | ACCEPTED (new) | Rec. | Sanitiser library may need dependency approval |
| D35 | Task-input notice, privacy statement, Task retention | **NEEDS CLIENT DECISION** | — | Wording and retention period |
| D36 | Backups for the beta | **NEEDS CLIENT DECISION** | — | Plan's backup features, or accept loss |
| D37 | Hosting for web and worker | **NEEDS CLIENT DECISION** | — | Host(s) and whether the worker shares the web host |

---

## 14. Remaining unknowns and the evidence needed
| Unknown | Evidence or answer needed | From |
|---|---|---|
| Custom roles via pooler (P1), owner role (P2), context isolation (P3) | Test connection, role creation and pool-isolation test results on the development project | Hosted verification |
| Driver choice (P4) | Dependency review and approval | Client / maintainer |
| Hosting and worker placement (P5, D37) | Chosen host(s) and their process and secret model | Client |
| Trusted client IP (P7, D14) | Host documentation plus a spoofed-header test | Client + hosted verification |
| Model, prices and spend-limit feature (P6, D32, D30) | Chosen model, current published prices, provider console capability | Client |
| Agent list and instructions (D26), tools (D27) | Approved agent definitions | Client |
| Limit values (D30) | Cost model: expected visitors and Tasks per day, tolerated daily spend | Client |
| Backups (D36), privacy and retention (D35) | Plan features; client's position | Client |
| Status of the old account-based work in Git (§8.1) | Read-only Git evidence | Client authorises a read-only Git check |

---

## 15. Documents that must change before implementation (not changed by this revision)
- A new ADR recording accountless visitors, the beta Task model and the simplifications above, superseding the affected parts of ADR-0001, ADR-0003, ADR-0005, ADR-0006 and ADR-0011.
- `docs/PRD.md` (accounts, verified-email gate, routes, MVP scope for the beta).
- `docs/ARCHITECTURE.md` §F.1, §F.2, §F.5, §F.7, §F.8, §F.9, §H (beta deviations).
- `docs/DATA-MODEL.md` §G (identity tables, Tasks, roles). The G.8 "as implemented" line already misdescribes migration 1.
- `docs/SECURITY.md` §I (trust boundaries, privilege model, verified-email gate removal).
- `docs/ROADMAP.md` (M2 scope and status, beta launch milestone).
- The M2 hosted-verification runbook (must be rewritten for this design; its previous revisions exist only in a working session, not as a file).
