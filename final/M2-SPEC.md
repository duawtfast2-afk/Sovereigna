# M2 Architecture Specification — Revision 6

**Accountless visitors, personal workspaces and agent Tasks for the public beta** (2026-11-13). Revisions 1–5 are superseded by this text.

Status: **specification for read-only review. Not approved for implementation or deployment.** No code, migration or test described here exists yet. No hosted verification has been performed and no AI provider request has been made. Revising this document approves nothing in it; §13 records what is confirmed by the client and what is only recommended.

Canonical file: `docs/identity/M2-SPEC.md`. The GitHub review mirror `final/M2-SPEC.md` is a synchronised copy of this file, not a separate source.

Change rule (docs/README.md): before implementation this specification needs (1) a new ADR in `docs/DECISIONS/`, (2) updates to the living documents listed in §15, and (3) a ROADMAP status update. None of those has been made; this document does not edit them.

### Changes from revision 5 (audit findings 1–17)
- **Reservations bound to lease and attempt (§6.3, §5.5, §5.7, §7.3, §7.4):** each reservation stores `lease_token` and `attempt_number`; `complete_task` and `retry_or_fail` take both, lock the Task and validate the lease **before** touching any reservation, and settle only the exactly matching open reservation. The sweep uses the same match.
- **Output allowance verifiable in the database (§5.2, §5.4, §7.3):** a new `private.agent_policies` table holds each approved agent version's `max_output_tokens` and approved provider/model; `submit_task` copies the cap onto the Task; Task fields that drive cost are immutable after submission.
- **Compromised worker stated correctly (§2, §6.2):** a worker holding `AI_API_KEY` can bypass every database control; a provider-side spend limit is a launch condition unless the client accepts that risk (D45).
- **Agent evaluation bound to provider and model (§5.2, §5.11):** each approved agent version records the approved `AI_BASE_URL` identity and model; `submit_task` and `reserve_attempt` refuse to run an agent when the active provider or model differs.
- **Medium/Low fixes:** conditional token guarantees and billable categories (§6.3); a fixed error taxonomy (§5.7); kill switch in `reserve_attempt` (§6.3); disjoint counter subjects (§7.3); storage-limit truncation (§5.7); lease-derived call deadline (§5.6); no lease fields readable by the web server (§7.4); `daily_usage` row creation and validation order (§5.4); `worker_id` removed; `attempts_exhausted` fails directly; two-key advisory locks; provider capacity mapping; UTC-day boundary exposure (§6.3).
- **Decision register (§13):** D45–D49 added; D30, D32, D38, D39, D42 updated.

---

## 1. Confirmed product scope and beta objective

### 1.1 Confirmed by the client (2026-11-13)
- Sovereign is a memecoin-related website that **displays a Contract Address as informational content**. It is **not** a crypto transaction application. (The address is rendered by the landing page from `apps/web/src/components/landing/launchConfig.ts`; it is static content.)
- Visitors open the website and **try its agent features immediately**, and must receive genuinely useful results.
- Visitors are **never** asked to sign up, log in, sign in, connect a wallet, sign a message or perform a blockchain transaction.
- Visitors **submit Tasks to agents** and expect outputs that match their Tasks.
- Each visitor has a **display username**, which is **not** a credential.
- Each visitor has **one personal workspace** that persists in the same browser profile on the same device.
- **No cross-device recovery.** Clearing browser data, deleting the profile or losing the session cookie may make the previous workspace inaccessible; this is accepted.
- **Out of the product entirely:** wallet connection, wallet ownership verification, message signing, swaps, token purchases, claims, any on-chain transaction.
- The public beta may last **about 1–2 months**. Further investment depends on visitor feedback.
- The client intends to use a custom AI API provider (proposed name **OAO**, base URL `https://oao.ai.id/v1`), subject to verification (§5.11).

### 1.2 Beta objective
Put a **usable, secure, cost-controlled** agent experience in front of visitors quickly. Priorities: reliable Task execution, visitor isolation, output quality, abuse prevention, cost control. Everything else waits for feedback.

### 1.3 Terminology
- **Task** (visitor-facing) = one visitor's instruction to one agent and the agent's output; in the blueprint's vocabulary a simplified single-agent quest/decree with one run.
- **Agent** = a platform-defined agent the visitor can choose (§5.2). Visitors do not create agents in the beta.
- **Attempt** = one execution of a Task by the worker, containing at most one model call, identified by `attempt_number`.
- **Lease** = a worker's exclusive, time-limited right to work on a running Task, identified by `lease_token`.

---

## 2. Architecture at a glance

```
Browser ──HTTPS──▶ Next.js server (apps/web)                      Worker (apps/worker, ONE long-running Node instance)
  session cookie     • validates session (hash lookup)              • claims Tasks via claim_task (global running cap)
  (HttpOnly)         • Server Actions: provision, submit, cancel    • reserves tokens via reserve_attempt
                     • reads own workspace/Tasks under RLS          • one bounded model call per attempt
                     • connects as sovereign_app                    • settles via complete_task / retry_or_fail
                              │                                     • connects as sovereign_worker
                              ▼                                               │
                       Supabase PostgreSQL (schema `sovereign` + `private`, RLS on every table)
                                                                              │
                                                    AI provider via adapter (AI_BASE_URL, AI_MODEL, AI_API_KEY)
```

- **Browser** holds only the session cookie. It never talks to Supabase or the AI provider.
- **Web server** holds the `sovereign_app` credential. It never holds `AI_API_KEY`.
- **Worker** holds the `sovereign_worker` credential and the AI configuration. It never reads visitor sessions.
- **No Realtime, no queue extension, no Supabase Auth, no Data API use** in the beta.

Security properties:
- Visitor isolation is enforced **twice**: application code derives the workspace from the validated session, and RLS filters every `sovereign_app` query by that session (§7.5–§7.6).
- `sovereign_app` and `sovereign_worker` cannot write any table directly; all writes go through SECURITY DEFINER functions with conditional updates (§7.4).
- The token hash, not a visitor id, is the database context (§7.6). This is defence in depth, **not** protection against a compromised web server or worker.

**Worker trust boundary (corrected in revision 6):**
- The worker necessarily reads every Task's input and writes every output, and it holds `AI_API_KEY`.
- A compromised worker can read all Tasks and **can call the provider directly with the key, bypassing every database control**: token reservations, daily ceilings, the global concurrency cap, the kill switch and the provider/model binding. Those database controls protect **normal application execution against bugs and races**; they do not constrain a malicious or compromised worker.
- What a compromised worker still cannot do: read `visitors` or `visitor_sessions`, create Tasks, or act as a visitor in the web application.
- The only control that limits spending by a compromised worker is a **provider-side spend or usage limit** on the account behind `AI_API_KEY` (if the provider offers one), plus rotating the key. A provider-side limit is therefore a **launch condition**, unless the client explicitly accepts the residual financial risk in a documented decision (D45).

---

## 3. Re-evaluation summary (Revision 3 → beta)

| Element | Classification | Beta decision |
|---|---|---|
| 256-bit token, hash-only storage, HttpOnly/Secure cookie, server-side validation, `SameSite=Lax`, Origin checks | **REQUIRED BEFORE PUBLIC LAUNCH** | Kept (§4). |
| RLS isolation with a token-hash context, roles without `BYPASSRLS` | **REQUIRED BEFORE PUBLIC LAUNCH** | Kept (§7). |
| Session lifetime | **SIMPLIFY FOR BETA** | One sliding idle lifetime, proposed 90 days (§4.4). |
| Rotation, lineage, grace, reuse detection, session events | **DEFER UNTIL VALIDATED** | Removed; trade-off in §4.8. |
| Provisioning nonce | **NOT REQUIRED FOR THIS PRODUCT** (beta) | Removed; replaced by the reload-and-verify flow (§4.5). |
| Scheduled retention, maintenance owner role | **SIMPLIFY FOR BETA** | Manual operator cleanup (§9.3). |
| Two owner roles | **SIMPLIFY FOR BETA** | One `NOLOGIN` owner (§7.2). |
| Per-network rate limiting and IP canonicalisation | **SIMPLIFY FOR BETA** | Available only once a trusted client-IP source is verified (§6.6). |
| PostgreSQL driver and pooler | **REQUIRED BEFORE PUBLIC LAUNCH** | Unapproved prerequisite (§11). |
| Development cleanup role | **SIMPLIFY FOR BETA** | Marker + expected-ref checks only (§12.5). |
| Data API | **NOT REQUIRED FOR THIS PRODUCT** (beta) | Unused; disabling deferred. |
| Username uniqueness and editing | **DEFER UNTIL VALIDATED** | Generated display name only. |
| pgmq, Realtime, full budget ledger, Forge, Convergence (ARCHITECTURE §F) | **DEFER UNTIL VALIDATED** | Replaced by proportionate beta mechanisms (§5, §6). |
| Wallet, signing, on-chain features | **NOT REQUIRED FOR THIS PRODUCT** | Excluded. |

---

## 4. Visitor identity (beta)

### 4.1 Concepts
| Concept | What it is | Secret? | Rule |
|---|---|---|---|
| Username | Generated display name, e.g. `quiet-heron-41` | No | Display only. Never authorises anything. |
| Visitor id | `uuid` generated by PostgreSQL | No, **not a credential** | Never accepted from the client for authorisation. |
| Workspace id | `uuid` | No, **not a credential** | Returned to the page only for consistency checks (§4.5); never trusted as input for access. |
| Session token | 32 random bytes (`node:crypto`), base64url, 43 characters | **Yes** | Lives only in the cookie. |
| Session token hash | SHA-256 of the token (32 bytes) | Sensitive | Stored in `visitor_sessions.token_hash`; never logged; not readable by `sovereign_app` or `sovereign_worker`. |
| Task id | `uuid` | No, **not a credential** | Access to a Task always requires the owning session (§5.8). |
| Lease token | `uuid` per claim | Worker-only | Never readable by `sovereign_app`, never sent to the browser (§7.4). |

Logs may contain visitor, workspace and Task ids and attempt numbers. They never contain tokens, token hashes, lease tokens, cookie values, IP addresses, AI keys, Task text, outputs or raw provider error bodies.

### 4.2 Username
Generated server-side (`adjective-noun-NN`, `node:crypto` randomness), format `^[a-z0-9][a-z0-9_-]{2,23}$` checked in the server and by a database `CHECK`. Not unique, not editable in the beta (D4, D23).

### 4.3 Cookie
| Attribute | Value |
|---|---|
| Name | `__Host-sovereign_session` in production; `sovereign_session` only on `http://localhost` |
| `HttpOnly` | yes |
| `Secure` | yes (off only on `http://localhost`) |
| `SameSite` | `Lax` (sent on top-level navigations from other sites, withheld on cross-site `POST`) |
| `Path` / `Domain` | `/` / not set (required by `__Host-`) |
| `Max-Age` | equal to the session's remaining lifetime; re-set on renewal |

### 4.4 Lifetime, renewal and states
- **Idle lifetime** (`SOVEREIGN_SESSION_IDLE_DAYS`, proposed **90 days**): `expires_at = now() + lifetime` at issue.
- **Renewal:** when `now() - last_renewed_at >= 24 hours`, a Server Action (`refreshSession`) extends `expires_at` and re-sets the cookie with a matching `Max-Age`. Page renders never renew. No absolute cap.
- **States** of a presented token at transaction time `now()` (`resolve_session`, §7.4):

| State | Condition | Result |
|---|---|---|
| `invalid_input` | cookie not 43 base64url characters (checked before any database call) or hash not 32 bytes | no visitor |
| `unknown` | no row with this hash | no visitor |
| `revoked` | `revoked_at is not null` | no visitor |
| `expired` | `now() >= expires_at` (strict) | no visitor |
| `active` | otherwise | visitor id, workspace id, `renewal_due` |

`current_visitor_id()` (§7.6) accepts exactly the `active` sessions.

### 4.5 First visit and concurrent provisioning
**Flow:**
1. A `GET` to the application area (`/workspace`) without a session cookie renders the app shell with no visitor. **No `GET` creates database rows.**
2. A client component sends one same-origin `POST` (`provisionVisitor`). The action runs the CSRF checks (§4.7), returns `reload` if a session cookie is present, generates the token and username, and calls `provision_visitor(token_hash, username, net_subject)`, which applies the provisioning limits (§6.5) and creates visitor, workspace and session in one transaction.
3. The response sets the session cookie and returns only `{ outcome: 'created' }` — **no workspace data**.
4. The client then performs a **full reload**. The page is rendered from whatever session cookie the browser actually holds at that moment.

**The race:** two tabs opened at the same moment can both send `provisionVisitor` before either response has stored a cookie. Two visitors V1 and V2 are created; the browser keeps whichever `Set-Cookie` it processes last (say V2).

**Why a tab can never show the wrong workspace:**
- Provisioning responses carry no workspace data, and every tab reloads after provisioning, so every rendered page comes from the final stored cookie (V2).
- Every page embeds the workspace id it was rendered for, and every later Server Action response includes the workspace id resolved from the request's cookie. If the two differ, the client discards its state and reloads. Server-side, every action already acts only on the workspace of the cookie it received.
- **Cross-tab coordination** (the Web Locks API: a lock named `sovereign-provision`, re-checking for a cookie via a lightweight action after acquiring it) makes the race rare. It is a mitigation only; correctness rests on the two points above.

**Accepted vs not accepted:**
- **Accepted:** an orphaned visitor record (V1) with an empty workspace and no Tasks. It costs one counted provisioning attempt and is removed by cleanup (§9.3).
- **Not accepted:** a page that shows, or acts on, a workspace other than the one belonging to the browser's current cookie.

### 4.6 Invalid cookie, "Start a new workspace" and "Forget this browser"
- The server **never** deletes or replaces a session cookie because it failed to validate.
- A cookie in state `unknown`, `revoked`, `expired` or `invalid_input` shows a notice and a **"Start a new workspace"** button. Its Server Action re-checks the cookie: if `active`, nothing is deleted and the page reloads; otherwise the response deletes the cookie and the client reloads, leading to §4.5. If cookies are blocked, the client reloads once and then shows "Cookies are required".
- **"Forget this browser"** (after a warning that access ends permanently) calls `revoke_session` and deletes the cookie.
- **Operator revocation:** `update sovereign.visitor_sessions set revoked_at = now(), revoke_reason = 'operator' where …` as an operator (§9.2).

### 4.7 CSRF and Origin checks (every state-changing request)
1. `Origin` is **required** and must equal the canonical origin (`SOVEREIGN_SITE_URL`, validated by `apps/web/src/lib/site-origin.ts`); a missing `Origin` is rejected. Next.js 16's own Server Action check lets a request without `Origin` through with a warning (bundled Next.js docs, `serverActions.md`), so it is not relied on.
2. `Sec-Fetch-Site`, when present, must be `same-origin`.
3. All state changes are `POST` Server Actions; the cookie is `SameSite=Lax`.
CSRF protection does not limit abuse; §6 is separate and also required.

### 4.8 Security trade-offs of the simplified identity
- **Theft:** the cookie is a bearer credential. Whoever holds it **is** that visitor until the session expires or is revoked: they can read the visitor's Tasks and outputs and submit Tasks against the visitor's limits. Without rotation (deferred), a copied cookie stays valid as long as the original. HttpOnly stops scripts reading it; XSS can still act inside the page; malware or a copied browser profile can take it. The real visitor can end the thief's access only with "Forget this browser" (which ends their own access too).
- **Expiry:** after the idle lifetime without use, access is lost permanently.
- **Loss:** clearing cookies, deleting the profile or losing the device loses access permanently. Accepted by the client.
- **What is protected:** other visitors' Task inputs and outputs. The beta holds no payment, wallet or account data, but it does hold whatever visitors type (§9.4).

---

## 5. Agent execution (primary concern)

### 5.1 Evidence: what exists, what M2 provides, what is deferred
| Capability | Already in the project | M2 beta must provide | Deferred |
|---|---|---|---|
| Agent catalogue | Demo-only data in `apps/web/src/demo/data.ts` (fictional, no instructions) | Platform-defined agents with approved acceptance packages and database policies (§5.2) | Forge, publishing, Codex |
| Task submission | Design only (`issue_decree`, ARCHITECTURE §F.8) | `submit_task` with idempotency first, then policy checks and limits (§5.4) | Priorities, deadlines, multi-agent Tasks |
| Execution state | Design only (`agent_runs` with fenced lease) | `sovereign.tasks` as the single source of truth (§5.3) | Separate quests/runs/steps |
| Queue | Design only (pgmq, `[VERIFY AT M2]`) | Claim from `tasks` with `SKIP LOCKED` and a global running cap (§5.5, §6.4) | pgmq |
| Worker | Health/config startup check only (`apps/worker/src`) | Claim → reserve → call → settle loop and a lease sweep (§5.5–§5.7) | Heartbeats for multi-step runs |
| Model access | Design: own `ModelProvider` interface (ADR-0001, §F.1, which named Anthropic) | Provider-neutral adapter configured by environment, bound to approved provider/model per agent (§5.11) | Second provider, streaming to the UI |
| Tools | Design only (§F.5, §F.10) | Decided per agent; none unless an approved agent requires it (§5.2) | Tool runtime, idempotency, SSRF guard |
| Budget | Design: reservation/settlement ledger (§F.9) | Proportionate token reservation per attempt (§6.3) | Monetary budgets per workspace |
| Status updates | Design: Realtime Broadcast | Polling (§5.9) | Realtime |
| Results | Design: `deliverables` | Output text on the Task, rendered through an allowlist (§5.7) | Versioned deliverables |

### 5.2 Agents, acceptance packages and policies (D26, D27, D42, D46)
Agents are defined in a typed, version-controlled catalogue module (`packages/core`): `agent_key`, `version`, display name, Order, public description, **server-only system instructions**, `max_output_tokens`, whether tools are used, and whether the agent is enabled.

**An agent version may be enabled for the public beta only with an approved acceptance package:**
1. **Purpose and target Task** — who it is for and what kind of Task it handles.
2. **Capability boundaries and unsupported requests** — what it must decline or redirect. For this product the client should decide at least how agents handle requests for financial or investment advice, price predictions, statements about the token beyond the website's published content, and anything involving wallets or transactions (which the product does not support).
3. **Approved system instructions** — written or approved by the client, stored server-side only.
4. **Representative successful Task examples** — inputs with descriptions of an acceptable output.
5. **Representative unsupported and failure cases** — inputs with the expected decline or limitation message.
6. **Measurable acceptance criteria** — a rubric applied to the examples (for example: addresses the Task, respects the boundaries, follows the agreed format, no fabricated facts presented as certain). The pass threshold is agreed with the client.
7. **Evaluation record** — the examples run against the verified provider and model (§5.11), scored against the rubric. The record states the **provider identity** (the normalised `AI_BASE_URL`: scheme, host and path, no credentials or query) and the **exact model identifier** used, and never contains `AI_API_KEY`. This is a live, cost-incurring run and needs explicit approval each time.
8. **Automated tests** for each agent (with a fake provider): catalogue schema valid; prompt assembly places instructions and Task text correctly; output caps applied; the browser-facing catalogue response **never** contains system instructions (also checked on the production bundle).

**Agent policy in the database (`private.agent_policies`, §7.3):** when an agent version's package and evaluation are approved, an operator records one row: `agent_key`, `agent_version`, `max_output_tokens`, `approved_provider` (normalised base URL), `approved_model`, `evaluation_ref` (identifier of the evaluation record, no secrets), `approved_at`, `enabled`. The row is the database's source of truth for what may run:
- `submit_task` accepts only an enabled policy row and copies its `max_output_tokens` onto the Task (§5.4).
- The catalogue module's values must equal the policy row (a deployment check and a test, §12); a mismatch disables submissions for that agent version.
- **Changing the provider or model** — including through deployment environment variables alone — makes every policy whose `approved_provider`/`approved_model` differs from the active configuration **unrunnable** (§5.11). Running again requires a new evaluation record, explicit approval and an updated policy row.

**Tools decision (D27):** tools are enabled for an agent only if its acceptance package shows that its target Tasks cannot be done usefully without them (for example, they need current information) **and** a tool provider is separately approved. Default for the first beta: **no tools**, as a deliberate scope decision. An agent whose purpose needs current information is not enabled without tools, or its boundaries must state plainly that it has no live data.

### 5.3 Task lifecycle
```
                submit_task (idempotent)
                       │
                       ▼
   ┌─────────────► queued ─── cancel_task ──────────────────────────────────────► cancelled
   │                   │
   │      claim_task (kill switch, global running cap, not cancel_requested)
   │                   ▼
   │                running ── reserve_attempt: paused / refused_capacity / lease_too_short ──► queued (no attempt used)
   │                   │   └── reserve_attempt: provider_mismatch / attempts_exhausted ──────► failed
   │                   │   └── reserve_attempt: cancelled ───────────────────────────────────► cancelled
   │      reserve_attempt reserved → one model call (attempt_number N, lease L)
   │                   │
   │                   ├── success ──────────── complete_task(L, N) ─────► succeeded  (or cancelled if cancel_requested)
   │                   └── error ────────────── retry_or_fail(L, N, class) ► queued / failed / cancelled
   │
   └── sweep_expired_leases: running with expired lease ─► queued (attempts < max, not cancel_requested)
                                                         ─► cancelled (cancel_requested)
                                                         ─► failed 'worker_lost' (attempts exhausted)
```

| Status | Meaning |
|---|---|
| `queued` | Durable, waiting for a worker. Never has `cancel_requested = true` (constraint). |
| `running` | Claimed by a worker holding a valid lease. May have `cancel_requested = true`. |
| `succeeded` | Output stored. |
| `failed` | Terminal; a sanitised `error_code` is stored. |
| `cancelled` | Terminal; stopped at the visitor's request. |

**Invariants (enforced by constraints, column privileges and by every transition being one conditional `UPDATE` on a locked row with the expected current status and lease in its `WHERE`):**
- Terminal states never change.
- A Task with `cancel_requested = true` never returns to `queued` and is never claimed again.
- Only the holder of the current `lease_token` can reserve, complete, retry or fail a running Task, and only for the current `attempt_number`.
- A reservation is settled or released only by the call carrying its exact `(task_id, lease_token, attempt_number)`, or by the sweep for an expired lease.
- `attempts` never exceeds `max_attempts`.
- `agent_key`, `agent_version`, `input`, `input_token_estimate` and `max_output_tokens` never change after submission.

### 5.4 Submission (`submitTask` → `submit_task`)
1. Server Action: CSRF checks (§4.7); resolve the session (`active` required); validate with Zod: `agent_key` exists and is enabled in the catalogue; Task text non-empty, within the character limit, no control characters other than newline/tab; `idempotency_key` is a UUID generated by the page per submission. Compute `input_token_estimate` over the **full assembled input** — the agent's system instructions, every server-added wrapper and the Task text (§6.3).
2. Inside `withVisitor` (§7.6), call `submit_task(agent_key, agent_version, input, input_token_estimate, idempotency_key)`. The function, in one transaction:
   1. derives the workspace from `current_visitor_id()` (none → `no_visitor`);
   2. takes a **per-workspace transaction lock** (`pg_advisory_xact_lock(1, hashtext(workspace_id::text))`, lock class 1), serialising all submissions of that workspace;
   3. **looks up `(workspace_id, idempotency_key)` first.** If a Task exists: same `agent_key`, `agent_version` and `input` → returns it with outcome `existing`, applying **no** check, limit, kill switch or counter; different payload → `idempotency_conflict`;
   4. only for a new submission, in this order:
      1. kill switch off → `paused`;
      2. a policy row for `(agent_key, agent_version)` that is enabled and whose `approved_provider`/`approved_model` equal `platform_settings.active_provider`/`active_model` → otherwise `agent_unavailable`;
      3. `input_token_estimate > max_input_tokens` → `input_too_large`;
      4. per-visitor active Tasks and Tasks today (counted under the lock) → `active_limit` / `daily_limit`;
      5. create today's `private.daily_usage` row if missing (`insert … on conflict do nothing`), then the global daily Task ceiling by an **atomic conditional increment** (`update … set tasks_submitted = tasks_submitted + 1 where day = $today and tasks_submitted < $ceiling returning …`; no row → `global_limit`);
   5. inserts the Task as `queued`, copying `max_output_tokens` **from the policy row** (the caller cannot supply it) → `queued`.
3. **Concurrent duplicates:** two requests with the same key in the same workspace are serialised by the lock; the second finds the first's Task and returns `existing`. The `UNIQUE (workspace_id, idempotency_key)` constraint is a second guard. A rolled-back insert also rolls back the counter increment.
4. Over-limit outcomes return generic messages (§6.7).

### 5.5 Claim, lease and recovery (worker)
All worker writes go through functions owned by `sovereign_owner` (§7.4). Each locks the Task row and applies one conditional update.

- **`claim_task()`** — takes the global claim lock (`pg_advisory_xact_lock(2, 0)`, lock class 2), checks the kill switch and the **global running cap** (§6.4), then claims the oldest `queued` Task with `not_before <= now()` and `cancel_requested = false` using `FOR UPDATE SKIP LOCKED`; sets `status = 'running'`, a new `lease_token`, `lease_expires_at = now() + lease`, `started_at`. Returns `task_id`, `lease_token`, `lease_expires_at`, `agent_key`, `agent_version`, `input`, `max_output_tokens`. Does **not** consume an attempt. (The unused `worker_id` argument of revision 5 is removed; worker identity is in logs only.)
- **Lease** = maximum call duration + margin (proposal: 150 s lease for a 120 s call budget).
- **`sweep_expired_leases()`** — run by the worker every poll cycle. For each `running` Task with `lease_expires_at <= now()` (`FOR UPDATE SKIP LOCKED`), in one transaction:
  1. settles the reservation matching `(task_id, the Task's current lease_token, status = 'open')` at its **full reserved amount** (a crashed or abandoned call may have been billed);
  2. moves the Task: `cancel_requested` → `cancelled`; else `attempts < max_attempts` → `queued` with `not_before = now() + backoff` and the lease cleared; else → `failed`, `error_code = 'worker_lost'`.
- **Queue timeout** (also in the sweep): `queued` Tasks older than `queue_timeout` → `failed` with `error_code = 'capacity'` if today's token or Task ceiling is reached at that moment, otherwise `'queue_timeout'`.
- **Race: cancellation vs recovery.** `cancel_task` and the sweep both lock the same row; each re-evaluates the latest version. If cancel wins, the sweep sees `cancel_requested = true` and produces `cancelled`. If the sweep wins and requeues, `cancel_task` then finds `queued` and sets `cancelled` directly. In neither order can the Task run again.
- **Stale worker (audit finding 1).** Worker A holds lease L1, attempt 1. Its call hangs past `lease_expires_at`. The sweep settles the reservation `(T, L1, 1)` in full and requeues T. Worker B claims T with lease L2 and reserves attempt 2 `(T, L2, 2)`. When A's late `complete_task(T, L1, 1, …)` or `retry_or_fail(T, L1, 1, …)` arrives, the function locks T, sees `lease_token = L2 ≠ L1`, returns `stale_lease` and changes **nothing** — neither the Task nor any reservation. A's output is discarded; A's tokens were already counted by the sweep.
- **Dependency on worker availability:** lease expiry, queue timeouts and reservation settlement after crashes all run in the worker's sweep. While no worker runs, Tasks stay `queued`/`running` and reservations stay open. Open reservations keep counting against the token ceiling, so the failure is fail-safe for **normal** spending (no new calls are authorised) but not for visitors (Tasks wait). The Task page shows "Taking longer than expected" for a `queued` Task older than the queue timeout or a `running` Task past its lease. Worker liveness is an operator check at launch and an alert before increasing traffic (§10).

### 5.6 Execution (one bounded model call per attempt)
1. Assemble the request: the agent's system instructions, any fixed server-added wrapper text, and the visitor's Task text as the user message. Recompute `input_token_estimate` over **all** of it (§6.3).
2. **`reserve_attempt(task_id, lease_token, input_token_estimate, provider_identity, model)`** (§6.3), passing the worker's active normalised `AI_BASE_URL` and `AI_MODEL`. Outcomes:
   - `reserved` → returns `attempt_number` and `call_deadline`;
   - `cancelled` → Task cancelled, no reservation;
   - `paused` (kill switch), `refused_capacity` (token ceiling), `lease_too_short` → Task back to `queued` with backoff, no attempt used, no reservation;
   - `provider_mismatch` → Task `failed` (`agent_unavailable`), no reservation, operator alerted; **no provider request is made**;
   - `attempts_exhausted` → Task `failed` (`provider_unavailable`), no reservation;
   - `input_too_large` → Task `failed` (`rejected`), no reservation;
   - `stale_lease` → the worker abandons the Task.
3. No model call happens without a `reserved` outcome.
4. Call the provider through the adapter with `max_output_tokens` **as returned by `claim_task`** (the value stored on the Task) and an **absolute abort deadline** = `call_deadline` returned by `reserve_attempt` = `min(lease_expires_at − safety margin, now() + max call duration)` (proposals: margin 15 s, max call duration 120 s). The adapter aborts the request at that instant regardless of the HTTP client's own timeout. `reserve_attempt` returns `lease_too_short` instead of reserving if less than a minimum call budget (proposal: 30 s) remains.
5. Settle (§5.7). The Task text is data, not instructions to the platform; with no tools, a prompt injection can only influence that visitor's own output.

### 5.7 Results, error taxonomy and retries (D47)
**Success:** `complete_task(task_id, lease_token, attempt_number, output, model_truncated, model, usage)` in one transaction:
1. lock the Task row; require `status = 'running'`, `lease_token` = argument, and an open reservation `(task_id, lease_token, attempt_number)`; otherwise return `stale_lease` and change nothing;
2. settle that reservation (§6.3): with usable `usage` → charge the reported billable total; without usable usage → charge the full reserved amount;
3. if `cancel_requested` → `cancelled`, output discarded;
4. otherwise store the output: if it exceeds the storage limit (40,000 characters), store the first 40,000 characters, set `output_truncated = true` and `truncation_reason = 'storage_limit'`; if the provider reported its output cap (`model_truncated`), set `output_truncated = true` and `truncation_reason = 'model_limit'` (`model_limit` wins if both); → `succeeded`. Oversized output never raises an error, never leaves the Task running and never triggers a retry.

**Errors:** `retry_or_fail(task_id, lease_token, attempt_number, error_class)` in one transaction: lock and validate exactly as step 1 above (`stale_lease` → no change); settle or release the matching reservation per the class; then `cancelled` if `cancel_requested`; otherwise the class's Task outcome.

The adapter assigns exactly one `error_class` per failed call. A class that depends on the provider's billing behaviour may be used **only** when the verified provider contract (§5.11 item 9) documents that behaviour; until then the adapter maps every failure after the request may have reached the provider to a `billing_unknown_*` class.

| `error_class` | Meaning | Reservation | Task outcome (if not `cancel_requested`) | `error_code` when failed |
|---|---|---|---|---|
| `not_sent_transient` | Request provably not sent (DNS failure, connection refused before any byte was written) | released | retry while `attempts < max_attempts`, else failed | `provider_unavailable` |
| `not_sent_config` | Request not sent because of local configuration (missing or invalid `AI_*` value) | released | failed immediately; operator alerted | `internal` |
| `unbilled_retryable` | Provider responded; contract documents the response as not billed and retryable (e.g. rate limit, if documented) | released | retry while attempts remain, else failed | `rate_limited` |
| `unbilled_terminal` | Provider responded; documented not billed; request invalid or not authorised | released | failed | `rejected` |
| `provider_capacity` | Provider refused because the account's spend, quota or usage limit is reached | released if documented unbilled, otherwise settled in full | failed, no retry; operator alerted | `capacity` |
| `billing_unknown_retryable` | Timeout, abort at the deadline, connection lost after sending, 5xx, or any response whose billing is not documented and that is plausibly transient | settled in full | retry while attempts remain, else failed | `timeout` (deadline/timeout) or `provider_unavailable` |
| `billing_unknown_terminal` | Unexpected error after sending | settled in full | failed | `internal` |

- **Retries:** a retry returns the Task to `queued` with backoff (proposal: 30 s, then 2 min) and a cleared lease; the next attempt gets a new lease and a new `attempt_number`.
- **Refusals** in the model's text are stored as normal output.
- **Partial output** from an interrupted call is discarded; only complete responses are stored.
- **Stored error data:** only `error_code` ∈ {`provider_unavailable`, `rate_limited`, `timeout`, `rejected`, `worker_lost`, `queue_timeout`, `capacity`, `agent_unavailable`, `internal`}. Raw provider messages go to server logs only, sanitised (no key, no Task text).
- **Output rendering:** markdown rendered through a **restrictive allowlist** (D34): paragraphs, line breaks, headings (h2–h4), emphasis, inline code, code blocks, ordered and unordered lists, blockquotes, horizontal rules, and links with `https:` URLs only, rendered with `rel="nofollow noopener noreferrer"`. **No** raw HTML, images, iframes, forms, scripts, styles, event attributes, tables (deferred) or non-`https` URLs. Truncated output is sanitised the same way. Output is never used in SQL, file paths, shell or `eval`.
- Failed Tasks show a plain message and a "Submit again" button that creates a **new** Task (new idempotency key).

### 5.8 Isolation between visitors
- Every Task belongs to one workspace (`tasks.workspace_id`, not null).
- `sovereign_app` reads Tasks only through RLS and a column-restricted grant, and writes only through `submit_task` and `cancel_task`, which derive the workspace from the session context. A Task id from the browser is only a lookup **within** that workspace.
- A visitor cannot read, modify or cancel another visitor's Task: RLS filters reads; `cancel_task` updates only `where id = $1 and workspace_id = <context workspace>`; another visitor's Task id returns `not_found`, indistinguishable from a non-existent id.
- The worker reads all Tasks by necessity (§2 trust boundary) and cannot read `visitors` or `visitor_sessions`.

### 5.9 Status retrieval, closing the tab, reconnecting
- The Task page polls `getTask` (proposal: every 2 s while `queued`/`running`, backing off to 10 s, stopping at a terminal status). `getTask` selects an explicit column list (§7.4) and each response includes the workspace id (§4.5). Lease fields are never selected or returned.
- Execution does not depend on the browser: closing the tab or returning later does not affect a Task. On return the workspace lists the visitor's Tasks (newest first, paginated) with status and output.

### 5.10 Cancellation
- `queued` → `cancelled` immediately (`cancel_task`, conditional on `status = 'queued'`).
- `running` → `cancel_requested = true` (conditional on `status = 'running'`); the flag is never cleared. The worker turns it into `cancelled` at reservation, completion, retry or sweep. If the model call already finished, its cost is settled and its output discarded.
- Terminal Tasks → `not_cancellable`.

### 5.11 AI provider configuration (D32, D38, D46)
**Configuration** (server-side environment of the worker only; never `NEXT_PUBLIC_*`; never in the web server's environment, the browser, logs or the repository):

| Variable | Meaning |
|---|---|
| `AI_API_KEY` | Provider credential. Secret. |
| `AI_BASE_URL` | Provider endpoint base, e.g. the proposed `https://oao.ai.id/v1`. |
| `AI_MODEL` | Model identifier as named by the provider. |

- The client can replace the key, base URL and model through deployment configuration alone. **Replacing the key** needs no other step. **Replacing the base URL or model** also requires (a) updating `private.platform_settings.active_provider`/`active_model` to match, and (b) for each agent version, a new evaluation and an approved policy row for the new provider/model (§5.2). Until both are done, affected agents are unrunnable (fail closed).
- **Provider identity** = the normalised `AI_BASE_URL` (lowercase scheme and host, path without trailing slash, no credentials, query or fragment). It is compared as text; it is not a secret.
- **Enforcement points:** (1) at worker start the worker compares its `AI_BASE_URL`/`AI_MODEL` with `platform_settings` and refuses to start on a mismatch (logged without the key); (2) `submit_task` refuses agents whose policy does not match `platform_settings` (`agent_unavailable`); (3) `reserve_attempt` refuses unless the worker's reported identity and model equal both `platform_settings` and the Task's policy row (`provider_mismatch`) — **before** any provider request. These checks protect against misconfiguration; a compromised worker can ignore them (§2).
- A future `.env.example` contains placeholder values only. **The key supplied in conversation is treated as exposed.** It is not recorded in this document or the repository and must be **rotated** before any real use. No request has been made with it.
- **Adapter:** provider-specific code lives behind a small `ModelProvider` port in `packages/core` (`generate(request, deadline) → { text, modelTruncated, usage | null }` or an `error_class`), with one adapter in the worker. The port carries no provider names; a different provider means a new adapter.
- **No compatibility is assumed.** A `/v1` suffix does not make the provider OpenAI-compatible. Before any real call is enabled, verify and record from the provider's documentation and a controlled test with a rotated key:
  1. API protocol and endpoint paths;
  2. authentication format (header name and scheme);
  3. supported models and the exact `AI_MODEL` value;
  4. request parameters for the system prompt, output token cap and timeout/abort behaviour;
  5. streaming behaviour (not used by the beta; confirm non-streaming works);
  6. **token usage reporting** in responses — every billable category (input, cached input reads and writes, output, reasoning or other hidden tokens) and whether it is always present;
  7. context window, maximum output tokens, and whether hidden/reasoning tokens are counted inside the output cap;
  8. rate limits and how they are signalled;
  9. error format and **which errors are billed** (drives §5.7's classes);
  10. pricing per billable token category;
  11. data handling and retention terms for submitted prompts;
  12. whether a provider-side spend or usage limit exists, and how hitting it is signalled (D45).
- If usage is not reported reliably (item 6), every call is settled at its full reservation, and the ceilings must be sized accordingly.
- If a billable category is not bounded by the request's input and output caps (item 7), the model is not approved until a verified bound for that category is added to the reservation (§6.3).
- If the input-token bound cannot be established (§6.3), the provider is not enabled.

---

## 6. Abuse prevention and cost control

### 6.1 Principles
- Anonymous execution will be abused and costs the operator money. Every limit is enforced **server-side in database functions or the worker**, never in the browser.
- Limits are configuration values in `private.platform_settings` (§7.3), editable by an operator. Numbers marked "proposal" are starting points, not evidence-based; the client sets the final values from a cost model (D30).
- When a limit is reached the system refuses new work with a generic message and never half-processes a Task.

### 6.2 Three different protections
- **Database cost controls** (§6.3, §6.4, §6.5): ceilings on tokens, Tasks, concurrency and new visitors. They bound **normal application spending** across any number of visitors and networks, under the assumptions in §6.3. They do **not** constrain a compromised worker (§2).
- **Provider-side spend limit** (D45): the only control that also bounds spending through a leaked or misused `AI_API_KEY`. Launch condition unless the client accepts the risk.
- **Per-network abuse protection** (§6.6): limits per client network. These protect **fairness** — they stop one source exhausting the global ceilings and denying service to everyone else. They do not bound cost.

### 6.3 Token reservation (atomic, per attempt) (D39, D48)
**Hard input-token limit:** `max_input_tokens` (configuration) bounds the **whole** request input: system instructions + server-added wrapper + Task text.
- **Estimate:** `input_token_estimate` = UTF-8 byte length of the full assembled input + a fixed per-request overhead (`request_overhead_tokens`, configuration). This is an upper bound **only if** the provider's tokenizer for the approved model never produces more tokens than bytes, and the overhead covers all template and role tokens. Both are **assumptions** to be checked during provider verification (§5.11) by comparing reported input tokens with the estimate on representative inputs, including non-English text. If they do not hold, a provider-supplied, verified token-counting method must be used instead; without a verified bound the provider is not enabled.
- The Server Action computes the estimate; `submit_task` and `reserve_attempt` both reject an estimate above `max_input_tokens`. The database cannot recompute the estimate (it does not hold the instructions); it trusts the web server and the worker for it, which is consistent with §2's trust boundary. The character limit (proposal 4,000) is a UX guard, not the cost bound.

**Reservation exposure per attempt** = `input_token_estimate` (recomputed by the worker) + `tasks.max_output_tokens` (copied from the approved policy at submission, §5.4) + any verified bound for billable categories outside those caps (§5.11 item 7).

**`reserve_attempt(task_id, lease_token, input_token_estimate, provider_identity, model)`** — one transaction:
1. Lock the Task row. Not `running` with this `lease_token` → `stale_lease`. `cancel_requested` → set `cancelled`, return `cancelled`.
2. Read `platform_settings` in the same transaction. `execution_enabled = false` → Task back to `queued`, return `paused`. `provider_identity`/`model` differ from `active_provider`/`active_model` or from the Task's policy row → Task `failed` (`agent_unavailable`), return `provider_mismatch`.
3. `attempts >= max_attempts` → Task `failed` (`provider_unavailable`), return `attempts_exhausted`. Estimate above `max_input_tokens` → Task `failed` (`rejected`), return `input_too_large`. Less than the minimum call budget left before `lease_expires_at − margin` → Task back to `queued`, return `lease_too_short`.
4. Create today's `private.daily_usage` row if missing, then lock it (`FOR UPDATE`). If `used_tokens + reserved_tokens + exposure > daily_token_ceiling` → Task back to `queued` with backoff, return `refused_capacity`.
5. Otherwise: `reserved_tokens += exposure`; `attempts += 1`; insert `private.attempt_reservations (task_id, attempt_number = attempts, lease_token, day, reserved_tokens = exposure, status = 'open', expires_at = lease_expires_at)`; return `reserved`, `attempt_number` and `call_deadline`.

**Kill-switch race boundary:** `reserve_attempt` reads `execution_enabled` in the transaction that creates the reservation. Once the operator's change has committed, every `reserve_attempt` whose read happens afterwards returns `paused`; a reservation whose read happened before the commit may still make its one call. At most `global_running_task_limit` provider calls can therefore start after the switch is turned off, and none after their reservations close. Calls already in flight are not interrupted.

**Settlement and release** (always in the same transaction as the Task's state change, and only for the reservation matching `(task_id, lease_token, attempt_number)` — or, in the sweep, `(task_id, current lease_token)`):
| Path | Reservation outcome |
|---|---|
| Success with usable usage | `settled`; `charged_tokens` = reported billable total (all reported categories); `used_tokens += charged`; `reserved_tokens -= reserved` |
| Success without usable usage | `settled` at the **full** reserved amount |
| Cancellation detected in `reserve_attempt` | No reservation was taken |
| Cancellation detected at completion | Settled like a success; output discarded |
| `not_sent_*`, `unbilled_*`, `provider_capacity` documented unbilled | `released`: `reserved_tokens -= reserved`; nothing added to `used` |
| `billing_unknown_*`, `provider_capacity` not documented unbilled | `settled` at the **full** reserved amount |
| Worker crash / lease expiry | The sweep settles the reservation matching the current lease at the full amount |
| Stale worker after takeover | `stale_lease`; nothing changes (§5.5) |

**What is guaranteed, and under which assumptions:**
- **Unconditional (database invariant):** at the moment any reservation is granted, `used_tokens + reserved_tokens ≤ daily_token_ceiling` for that UTC day, because grants serialise on the locked day row.
- **Conditional:** total tokens actually billed for a UTC day's reservations stay within that day's ceiling **only if** for every attempt the billed tokens ≤ the reserved exposure. That holds only if (a) the input-token bound holds for the model, (b) billed output, including any hidden/reasoning tokens, never exceeds `max_output_tokens` unless separately bounded, (c) every other billable category is covered, and (d) no provider call happens without a reservation — which holds for the application's own code but **not** for a compromised worker (§2).
- **Over-reported usage:** if reported billable usage exceeds the reservation, the reported value is charged (`used_tokens` may then exceed the ceiling), later reservations for that day are refused, and the attempt is logged for the operator as a bound violation. Repeated violations mean the assumptions are wrong and the agent or model must be disabled until fixed.
- **Billable categories:** settlement charges the sum of all reported billable categories (including cached-input and reasoning tokens if reported). Categories priced differently are still counted as tokens here; money is computed separately from verified prices (§5.11).

**UTC day boundary (D48):** reservations belong to the UTC day on which they were granted and settle against that day's row. At 00:00 UTC a new day's ceiling starts from zero while the previous day's open reservations (at most `global_running_task_limit` of them) are still outstanding. Consequences: the ceiling bounds **each UTC day's accounting**, not any rolling 24-hour window; across a boundary, spending within a few minutes can approach two daily ceilings, and outstanding exposure at the boundary can reach today's ceiling plus up to `global_running_task_limit × maximum per-attempt exposure` from the previous day. This is a **recommended beta trade-off** for simplicity; a rolling-window limit is deferred.

- A reservation never stays open indefinitely: every open reservation belongs to a running Task with a lease; the sweep closes any reservation of an expired lease together with the Task transition. While the worker is down, open reservations remain and count against the ceiling (§5.5).
- This is deliberately **not** a monetary ledger: it bounds tokens. Per-workspace monetary budgets are deferred.

### 6.4 Global concurrency (D40)
- **Database-enforced global cap:** `global_running_task_limit` (proposal: 2). `claim_task` takes the global claim lock (class 2), counts `running` Tasks and claims only if the count is below the cap. The cap holds across **any** number of worker processes running the application's code.
- **Topology for the beta: a single worker instance** with in-process concurrency equal to the cap. The database cap remains the guarantee if a second instance is started by mistake.
- **Consequences:** throughput ≤ cap ÷ average Task duration (at a cap of 2 and about 30 s per call, roughly 4 Tasks per minute); Tasks beyond that wait and may hit the queue timeout; the maximum normal spend rate is bounded by cap × per-attempt exposure per call duration. A worker outage stops all execution (§5.5). A compromised worker is not bound by the cap (§2).

### 6.5 Limits table
| Control | Where enforced | Proposal | When reached |
|---|---|---|---|
| Kill switch `execution_enabled` | `submit_task` (new submissions only), `claim_task`, `reserve_attempt` | `false` until launch | Submissions refused ("Agents are paused"); nothing claimed; no new reservation; calls in flight finish |
| Agent policy and active provider/model match | `submit_task`, `reserve_attempt`, worker start | — | `agent_unavailable` / `provider_mismatch`; no provider request |
| Global daily Task ceiling | `submit_task`, atomic conditional increment | client sets | "Agents are busy today. Please try again tomorrow." |
| Global daily token ceiling | `reserve_attempt` (§6.3) | client sets | Tasks wait, then fail `capacity` at queue timeout |
| Hard input-token limit | Server Action, `submit_task`, `reserve_attempt` | client sets from the model's context window | Validation message |
| Output token cap per agent version | `private.agent_policies`, copied to the Task | client approves per agent | Truncation flag (`model_limit`) |
| Output storage limit | `complete_task` | 40,000 characters | Truncation flag (`storage_limit`) |
| Provider-side spend/usage limit | provider account (D45) | below the operator's tolerance | `provider_capacity` → `failed` (`capacity`), no retry |
| Global provisioning ceiling (per hour) | `provision_visitor`, atomic conditional upsert | 1,000 / hour | "Sovereign is busy. Please try again shortly." |
| Per-network provisioning limit | `provision_visitor`, only with a trusted client IP (§6.6) | 10 / hour, 30 / day | "Too many new workspaces from this network." |
| Per-visitor active Tasks | `submit_task`, under the workspace lock | 1 | "Wait for your current Task to finish." |
| Per-visitor Tasks per day | `submit_task`, under the workspace lock | 20 | "You've reached today's Task limit." |
| Task text length | Server Action, `CHECK` | 4,000 characters | Validation message |
| Call deadline | `reserve_attempt` returns it; adapter aborts at it | lease − 15 s, at most 120 s | `billing_unknown_retryable` (`timeout`) |
| `max_attempts` | settings (1–3), `reserve_attempt`, `CHECK` | 2 | `failed` |
| Global running Tasks | `claim_task` | 2 | Tasks wait |
| Queue timeout | sweep | 15 min | `failed` (`queue_timeout` / `capacity`) |
| Request body size | Next.js Server Actions default 1 MB (bundled docs) | keep default | Request rejected |

**Token bounds:** per UTC day, `daily_token_ceiling` under the conditions in §6.3. Per Task ≤ `max_attempts × (max_input_tokens + max_output_tokens + any verified extra-category bound)`. Money = tokens × verified prices (§5.11); this document states no prices.

### 6.6 Per-network limiting and client IP (D14)
- Next.js handlers do not provide a trustworthy client address by themselves; `X-Forwarded-For`, `Forwarded`, `X-Real-IP` and similar headers can be sent by anyone. **No such header is read unless explicitly configured.**
- A header may be trusted only if the hosting platform is selected and verified to (1) be the only public path to the app and (2) **overwrite** that header with the connecting address (verified with a spoofed-header test). Then `SOVEREIGN_CLIENT_IP_HEADER` names it, the value must parse as exactly one IP address, IPv6 is reduced to /64, and the server passes `HMAC-SHA-256(SOVEREIGN_RATE_LIMIT_KEY, network)` — exactly 32 bytes — as `net_subject`.
- Without a verified source, `net_subject` is `null`, the per-network limit is off, and only the global protections apply. A bot can then exhaust the daily ceilings and deny service for the rest of the day, but cannot exceed them.
- Raw IP addresses are never stored or logged.

### 6.7 Responses
Refusals use the generic messages above and never reveal counts, thresholds, internal checks or infrastructure. Logs record outcome names only.

---

## 7. Database model and roles

### 7.1 Placement
- Tables in schema `sovereign`; settings, policies, counters, reservations and functions in `private`. Neither schema is exposed through the Supabase Data API; nothing Sovereign lives in `public`.
- Supabase Auth is unused; sign-ups and anonymous sign-ins are disabled in the dashboard.
- No generic JSON storage tables.

### 7.2 Roles
| Role | Attributes | Used by | Purpose |
|---|---|---|---|
| `sovereign_app` | `LOGIN NOSUPERUSER NOBYPASSRLS NOCREATEDB NOCREATEROLE NOINHERIT`, connection limit | web server | Reads own rows under RLS through column grants; calls visitor functions. No table writes. |
| `sovereign_worker` | same attributes | worker | Calls worker functions only. No table privileges. |
| `sovereign_owner` | `NOLOGIN NOSUPERUSER NOBYPASSRLS NOCREATEDB NOCREATEROLE NOINHERIT` | nobody logs in | Owns all SECURITY DEFINER functions; holds only the privileges they need, plus owner-scoped RLS policies. |
| `postgres` | Supabase-managed | operators, migrations | Owns tables; operators maintain `platform_settings` and `agent_policies`. **Never** used by the web server or worker at runtime. |
| `anon`, `authenticated`, `service_role` | Supabase-managed | — | No privileges on `sovereign`/`private`. `service_role` is not used anywhere. |

Passwords for the two login roles are set out of band by the operator and stored only in the hosting platform's secret settings. Owning functions with `postgres` instead of `sovereign_owner` would let any function bug reach everything `postgres` can; that fallback is weaker and needs explicit approval if P2 fails (D15).

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

create table private.agent_policies (                 -- maintained by operators; one row per approved agent version
  agent_key          text not null check (agent_key ~ '^[a-z0-9-]{1,40}$'),
  agent_version      text not null check (char_length(agent_version) between 1 and 40),
  max_output_tokens  integer not null check (max_output_tokens > 0),
  approved_provider  text not null check (approved_provider ~ '^https://[^\s?#@]+$'),
  approved_model     text not null check (char_length(approved_model) between 1 and 200),
  evaluation_ref     text not null check (char_length(evaluation_ref) between 1 and 200),
  approved_at        timestamptz not null,
  enabled            boolean not null default false,
  primary key (agent_key, agent_version)
);

create table sovereign.tasks (
  id                    uuid primary key default gen_random_uuid(),
  workspace_id          uuid not null references sovereign.workspaces (id) on delete cascade,
  idempotency_key       uuid not null,
  agent_key             text not null,
  agent_version         text not null,
  input                 text not null check (char_length(input) between 1 and 4000),
  input_token_estimate  integer not null check (input_token_estimate > 0),
  max_output_tokens     integer not null check (max_output_tokens > 0),       -- copied from agent_policies
  status                text not null default 'queued'
                        check (status in ('queued', 'running', 'succeeded', 'failed', 'cancelled')),
  attempts              integer not null default 0 check (attempts between 0 and 3),
  not_before            timestamptz not null default now(),
  cancel_requested      boolean not null default false,
  lease_token           uuid,
  lease_expires_at      timestamptz,
  output                text check (output is null or char_length(output) <= 40000),
  output_truncated      boolean not null default false,
  truncation_reason     text check (truncation_reason in ('model_limit', 'storage_limit')),
  error_code            text check (error_code in ('provider_unavailable', 'rate_limited', 'timeout', 'rejected',
                                                   'worker_lost', 'queue_timeout', 'capacity',
                                                   'agent_unavailable', 'internal')),
  model                 text check (model is null or char_length(model) <= 200),
  input_tokens          integer check (input_tokens >= 0),
  output_tokens         integer check (output_tokens >= 0),
  created_at            timestamptz not null default now(),
  started_at            timestamptz,
  finished_at           timestamptz,
  unique (workspace_id, idempotency_key),
  foreign key (agent_key, agent_version) references private.agent_policies (agent_key, agent_version),
  check (not (status = 'queued' and cancel_requested)),
  check ((status = 'running') = (lease_token is not null and lease_expires_at is not null)),
  check ((status = 'succeeded') = (output is not null)),
  check ((status = 'failed') = (error_code is not null)),
  check ((status in ('succeeded', 'failed', 'cancelled')) = (finished_at is not null)),
  check (output_truncated = (truncation_reason is not null))
);
create index tasks_claim_idx     on sovereign.tasks (created_at) where status = 'queued';
create index tasks_workspace_idx on sovereign.tasks (workspace_id, created_at desc);
create index tasks_lease_idx     on sovereign.tasks (lease_expires_at) where status = 'running';

create table private.platform_settings (               -- exactly one row
  singleton                       boolean primary key default true check (singleton),
  execution_enabled               boolean not null default false,
  active_provider                 text not null check (active_provider ~ '^https://[^\s?#@]+$'),
  active_model                    text not null check (char_length(active_model) between 1 and 200),
  global_daily_task_ceiling       integer not null check (global_daily_task_ceiling >= 0),
  daily_token_ceiling             bigint  not null check (daily_token_ceiling >= 0),
  max_input_tokens                integer not null check (max_input_tokens > 0),
  request_overhead_tokens         integer not null check (request_overhead_tokens >= 0),
  max_attempts                    integer not null check (max_attempts between 1 and 3),
  global_running_task_limit       integer not null check (global_running_task_limit between 1 and 20),
  lease_seconds                   integer not null check (lease_seconds between 30 and 600),
  call_margin_seconds             integer not null check (call_margin_seconds between 5 and 120),
  min_call_budget_seconds         integer not null check (min_call_budget_seconds between 5 and 300),
  max_call_seconds                integer not null check (max_call_seconds between 5 and 300),
  global_hourly_provision_ceiling integer not null check (global_hourly_provision_ceiling >= 0),
  net_hourly_provision_limit      integer not null check (net_hourly_provision_limit >= 0),
  net_daily_provision_limit       integer not null check (net_daily_provision_limit >= 0),
  visitor_active_task_limit       integer not null check (visitor_active_task_limit >= 0),
  visitor_daily_task_limit        integer not null check (visitor_daily_task_limit >= 0),
  queue_timeout                   interval not null check (queue_timeout > interval '0'),
  check (call_margin_seconds + min_call_budget_seconds < lease_seconds)
);

create table private.daily_usage (
  day              date primary key,
  tasks_submitted  integer not null default 0 check (tasks_submitted >= 0),
  reserved_tokens  bigint  not null default 0 check (reserved_tokens >= 0),
  used_tokens      bigint  not null default 0 check (used_tokens >= 0)
);

create table private.attempt_reservations (
  task_id          uuid not null references sovereign.tasks (id) on delete cascade,
  attempt_number   integer not null check (attempt_number between 1 and 3),
  lease_token      uuid not null,
  day              date not null references private.daily_usage (day),
  reserved_tokens  integer not null check (reserved_tokens > 0),
  status           text not null default 'open' check (status in ('open', 'settled', 'released')),
  charged_tokens   integer check (charged_tokens >= 0),     -- set when settled
  expires_at       timestamptz not null,
  closed_at        timestamptz,
  primary key (task_id, attempt_number),
  check ((status = 'open') = (closed_at is null)),
  check ((status = 'settled') = (charged_tokens is not null))
);
create unique index attempt_reservations_one_open on private.attempt_reservations (task_id) where status = 'open';

create table private.provision_counters (
  scope         text not null check (scope in ('global', 'network')),
  subject       bytea not null,
  window_kind   text not null check (window_kind in ('hour', 'day')),
  window_start  timestamptz not null,
  hit_count     integer not null check (hit_count > 0),
  primary key (scope, subject, window_kind, window_start),
  check ((scope = 'global'  and subject = '\x00'::bytea and window_kind = 'hour')
      or (scope = 'network' and octet_length(subject) = 32))
);
```

Notes:
- **Reservation fencing:** a reservation row carries the `lease_token` and `attempt_number` that created it; at most one is open per Task. Settlement and release always match all of `task_id`, `lease_token`, `attempt_number` and `status = 'open'` (the sweep matches `task_id`, the Task's current `lease_token` and `status = 'open'`).
- **Immutable cost fields:** `sovereign_owner`'s column-level `UPDATE` grant on `tasks` (§7.4) excludes `agent_key`, `agent_version`, `input`, `input_token_estimate`, `max_output_tokens`, `workspace_id` and `idempotency_key`, so no function can change them after insert. The foreign key to `agent_policies` ensures every Task references an approved policy row; `agent_policies` rows are never deleted while Tasks reference them.
- **Attempts:** `tasks.attempts` and `attempt_reservations.attempt_number` (≤ 3) match the `max_attempts` setting range (1–3); the configured value (proposal 2) is enforced by `reserve_attempt`.
- **Disjoint counters:** global counters use scope `global` with the single-byte subject `\x00`; network counters use scope `network` with exactly 32-byte subjects. Scope is part of the key, so the two can never share a row.
- **Atomic counters:** provisioning uses `insert … on conflict (scope, subject, window_kind, window_start) do update set hit_count = hit_count + 1 where hit_count < $limit returning`; no row means the limit is reached. Daily Tasks use the conditional update in §5.4.
- **One-to-one visitor ↔ workspace:** `workspaces.visitor_id` is `NOT NULL UNIQUE`; the only insert path creates both in one transaction.
- **Cascade:** deleting a visitor deletes its workspace, sessions, Tasks and their reservations. Deleting reservations does not alter `daily_usage`.
- **Day boundaries:** UTC (`(now() at time zone 'utc')::date`).

### 7.4 Functions (all in `private`, owned by `sovereign_owner`, `SECURITY DEFINER`, `set search_path = ''`, fully qualified names, no dynamic SQL; every state change locks the row and applies one conditional update)
| Function | `EXECUTE` to | Contract |
|---|---|---|
| `current_visitor_id() → uuid` (`STABLE`) | `sovereign_app` | §7.6. Read-only. |
| `resolve_session(token_hash, renew) → (state, visitor_id, workspace_id, renewal_due)` | `sovereign_app` | §4.4; renewal when requested and due. Rejects a hash that is not 32 bytes. |
| `provision_visitor(token_hash, username, net_subject) → (outcome, visitor_id, workspace_id)` | `sovereign_app` | Validates hash (32 bytes), username format, `net_subject` null or **exactly 32 bytes** (else error). Global and (if not null) per-network limits atomically; then visitor, workspace, session. `outcome ∈ {created, rate_limited_network, rate_limited_global}`. |
| `revoke_session(token_hash) → boolean` | `sovereign_app` | `forgotten` on an `active` session. |
| `submit_task(agent_key, agent_version, input, input_token_estimate, idempotency_key) → (outcome, task_id)` | `sovereign_app` | §5.4. `outcome ∈ {queued, existing, idempotency_conflict, paused, agent_unavailable, input_too_large, active_limit, daily_limit, global_limit, no_visitor}`. |
| `cancel_task(task_id) → text` | `sovereign_app` | §5.10. `cancelled`, `cancel_requested`, `not_cancellable`, `not_found`. |
| `claim_task() → (task_id, lease_token, lease_expires_at, agent_key, agent_version, input, max_output_tokens)` | `sovereign_worker` | §5.5. No row when paused, at the running cap, or nothing claimable. |
| `reserve_attempt(task_id, lease_token, input_token_estimate, provider_identity, model) → (outcome, attempt_number, call_deadline)` | `sovereign_worker` | §6.3. `outcome ∈ {reserved, cancelled, paused, provider_mismatch, attempts_exhausted, input_too_large, lease_too_short, refused_capacity, stale_lease}`. |
| `complete_task(task_id, lease_token, attempt_number, output, model_truncated, model, usage) → text` | `sovereign_worker` | §5.7. Validates lease and attempt before settlement. `succeeded`, `cancelled` or `stale_lease`. |
| `retry_or_fail(task_id, lease_token, attempt_number, error_class) → text` | `sovereign_worker` | §5.7 taxonomy. Validates lease and attempt before settlement. `queued`, `failed`, `cancelled` or `stale_lease`. Unknown `error_class` → error, no change. |
| `sweep_expired_leases() → integer` | `sovereign_worker` | §5.5; also queue timeouts. Settles only reservations matching the expired lease. |
| `get_worker_config() → (execution_enabled, active_provider, active_model)` | `sovereign_worker` | Read-only, for the worker's start-up check (§5.11). |

`usage` is a typed composite (`input_tokens`, `output_tokens`, `other_billable_tokens`, all non-negative, or null when unusable).

Privileges of `sovereign_owner` (nothing else; owner-scoped RLS policies for exactly these commands):
- `sovereign.visitors`: `SELECT`, `INSERT`, `UPDATE (last_seen_at)`.
- `sovereign.workspaces`: `SELECT`, `INSERT`.
- `sovereign.visitor_sessions`: `SELECT`, `INSERT`, `UPDATE (last_renewed_at, expires_at, revoked_at, revoke_reason)`.
- `sovereign.tasks`: `SELECT`, `INSERT`, `UPDATE (status, attempts, not_before, cancel_requested, lease_token, lease_expires_at, output, output_truncated, truncation_reason, error_code, model, input_tokens, output_tokens, started_at, finished_at)` — **not** the immutable cost fields.
- `private.agent_policies`, `private.platform_settings`: `SELECT`.
- `private.daily_usage`: `SELECT`, `INSERT`, `UPDATE (tasks_submitted, reserved_tokens, used_tokens)`.
- `private.attempt_reservations`: `SELECT`, `INSERT`, `UPDATE (status, charged_tokens, closed_at)`.
- `private.provision_counters`: `SELECT`, `INSERT`, `UPDATE (hit_count)`.
- `USAGE` on both schemas. **No `DELETE`.** Advisory locks use the two-key form with class 1 (workspace submission) and class 2 (global claim), so the two cannot collide.

Privileges of `sovereign_app`:
- `SELECT` on `sovereign.visitors (id, username, created_at)`, `sovereign.workspaces (id, visitor_id, created_at)` and `sovereign.tasks (id, workspace_id, agent_key, agent_version, input, status, cancel_requested, output, output_truncated, truncation_reason, error_code, created_at, started_at, finished_at)` — **no** `lease_token`, `lease_expires_at`, `not_before`, `attempts`, `idempotency_key`, `input_token_estimate`, `max_output_tokens`, `model` or token counts. RLS restricts rows to the visitor's own.
- `EXECUTE` on its six functions; `USAGE` on both schemas. Nothing else.

Privileges of `sovereign_worker`: `EXECUTE` on its six functions (`claim_task`, `reserve_attempt`, `complete_task`, `retry_or_fail`, `sweep_expired_leases`, `get_worker_config`); `USAGE` on `private`. **No table privileges.**

`EXECUTE` on every function is first revoked from `public`, `anon`, `authenticated` and `service_role`. Whether Supabase's default privileges add grants on new objects, and whether `pg_catalog.pg_advisory_xact_lock(int, int)` is executable by `sovereign_owner` on the hosted project, are checked by the catalog tests (§12.2), not assumed.

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
-- Owner policies as listed in §7.4. No policies for sovereign_worker (it has no table privileges),
-- anon, authenticated or service_role.
```

### 7.6 Visitor context per transaction
```sql
begin;
select set_config('sovereign.session_token_hash', $1, true);  -- lowercase hex of sha256(token)
-- RLS-filtered reads and function calls
commit;
```
- `current_visitor_id()` decodes the setting (must match `^[0-9a-f]{64}$`), finds a session with that hash, `revoked_at is null` and `now() < expires_at`, and returns its `visitor_id`; otherwise null.
- `set_config(…, true)` and the advisory locks are transaction-scoped and released at commit or rollback, which fits a transaction-mode pooler; whether the hosted pooler behaves accordingly is P3. The web helper `withVisitor` always opens an explicit transaction and sets the context first. Session-level `SET` is forbidden in application code (static test, §12.1).
- **Why a token hash:** code running SQL as `sovereign_app` cannot read token hashes, so it can act only as visitors whose hash it already holds.
- **Not protected:** a compromised web server (live cookies, `sovereign_app` password) or worker (all Tasks, `AI_API_KEY`). RLS is defence in depth against application bugs.

---

## 8. Application structure (mapped to the current source, read-only inventory)

### 8.1 Status of the existing account-based M2 work
Based on evidence recorded earlier in the project's working session — **not re-verified for this revision; no Git commands were run**: at that session's start `HEAD` was `10013c0` and the account-based M2 files were untracked; later measurements showed the same `HEAD` and nothing staged. On that evidence the account-based work is not contained in any commit on record, has not been merged or pushed, and has never been applied to a hosted database. If it has since been committed locally, this paragraph must be updated from verified Git evidence.

### 8.2 Existing files and their fate
| Area | Files | Fate |
|---|---|---|
| Auth pages, actions, helpers | `apps/web/src/app/auth/**`, `apps/web/src/lib/auth/**`, `apps/web/src/server/auth.ts` | Removed. `AuthShell.tsx`/`auth.css` styling may be reused for notices. |
| Supabase JS clients and proxy | `apps/web/src/lib/supabase/**`, `apps/web/src/proxy.ts` | Removed from the visitor path; no proxy logic needed. |
| Canonical origin | `apps/web/src/lib/site-origin.ts` | Kept for the Origin check. |
| Workspace page | `apps/web/src/app/(app)/workspace/page.tsx` | Rewritten: Task list, form and view. |
| Composition roots | `apps/web/src/server/composition.ts`, `apps/worker/src/composition.ts`, `apps/worker/src/main.ts` | Gain database adapters; the worker gains the loop, the start-up configuration check and the AI adapter. |
| DB configuration | `packages/db/src/config.ts` (`parseWorkerDatabaseUrl` rejects `postgres`, `supabase_admin`, `service_role`, `authenticator`) | Reused for the worker; same pattern for the web URL requiring `sovereign_app`. |
| Core | `packages/core/src` (health only) | Gains the agent catalogue and acceptance metadata, Task schemas, `ModelProvider` port, error taxonomy, token estimate, provider-identity normalisation. |
| Demo data | `apps/web/src/demo/**` | Stays for public pages until replaced; never shown as real Task results. |
| Migrations | `packages/db/supabase/migrations/20261112120000_m2_schemas_privileges.sql`, `20261112120100_m2_identity.sql` | Depend on `auth.users`/`auth.uid()`. **Replaced**, not amended (requires approval). |
| pgTAP and runner | `packages/db/supabase/tests/**`, `scripts/run-sql-tests.mjs`, `scripts/tap.mjs` | Suite files replaced (§12.2); control, manifest rules and runner kept. |
| Integration tests | `apps/web/test/integration/**` | Replaced (§12.3); `support/diagnostics.ts` reusable. |
| Guards | `apps/web/test/service-role-isolation.test.ts`, `scripts/check-client-bundle.mjs`, ESLint rules | Extended: database URLs, `AI_API_KEY`, agent instructions never in client code or bundles. |
| Environment template | `.env.example` | Later gains `AI_API_KEY`, `AI_BASE_URL`, `AI_MODEL` with placeholder values only. |
| Dependencies | `@supabase/ssr`, `@supabase/supabase-js` | Unused. New: a PostgreSQL driver, possibly a provider SDK (or plain `fetch`), and a markdown sanitiser — each needs approval (§11). |

### 8.3 New modules (proposed, not created)
- `apps/web/src/server/visitor.ts` (`server-only`): cookie, token, hashing, `requireVisitor`, `withVisitor`, CSRF checks.
- `apps/web/src/app/(app)/workspace/actions.ts`: `provisionVisitor`, `refreshSession`, `submitTask`, `getTask` (explicit column list), `listTasks`, `cancelTask`, `startNewWorkspace`, `forgetThisBrowser`.
- `packages/core`: agent catalogue and acceptance metadata, Zod schemas, `ModelProvider` port, error taxonomy, token estimate.
- `packages/db`: `app-database.ts` (`sovereign_app`) and `worker-database.ts` (`sovereign_worker`, function calls only).
- `apps/worker`: claim loop, sweep, start-up configuration check, AI adapter reading `AI_*` with a deadline-based abort, sanitised logging.

---

## 9. Operations during the beta

### 9.1 Visibility (minimum)
- Structured logs from web and worker: Task id, workspace id, agent key/version, transitions, attempt number, duration, token counts, reservation outcome, `error_class`, `error_code`. No secrets, lease tokens, Task text or outputs.
- Operator SQL (documented in the runbook): queue depth and oldest queued Task; running Tasks and lease ages; failure rate by `error_code`; today's `daily_usage` (submitted, reserved, used) against ceilings; open reservations; bound violations (charged > reserved); provisioning counts; enabled `agent_policies` versus the active provider/model.
- **Worker liveness:** at launch, an operator check of the oldest `queued` age and the latest `started_at`; an automated alert before increasing traffic (§10).

### 9.2 Operator controls
- Kill switch and ceilings via `private.platform_settings`.
- Enabling an agent version: insert or update its `private.agent_policies` row **only** after its acceptance package and evaluation are approved.
- Changing provider or model: change `AI_BASE_URL`/`AI_MODEL` in the worker's deployment configuration **and** `active_provider`/`active_model` together; agents stay unrunnable until re-evaluated and re-approved (§5.11).
- AI key rotation: replace `AI_API_KEY` in the worker's deployment configuration and restart the worker.
- Session revocation (§4.6). Operator SQL runs as `postgres` — operator access, not application runtime.

### 9.3 Cleanup (manual during the beta)
- Abandoned visitors: delete visitors older than 1 day with **no session where `least(revoked_at, expires_at) > now() - interval '30 days'`**. A visitor with a valid session cannot match, and no code path creates a session for an existing visitor. Cascades remove workspace, sessions, Tasks and reservations.
- Old sessions: delete those with `least(revoked_at, expires_at) < now() - interval '30 days'`.
- `provision_counters` older than 2 days. `daily_usage` is kept.
- Tasks: per the retention decision (D35).
- Run by an operator; automated scheduling deferred (D11).

### 9.4 Data, privacy and backups (launch blockers)
- Stored per visitor: generated username, timestamps, session hashes, Task text, outputs, token counts. No email, wallet or payment data.
- Task text is also sent to the AI provider, whose data-handling terms must be known (§5.11 item 11).
- **Before public exposure the client must decide (D35):** the notice shown at the Task form (at least: do not include personal or sensitive information; Tasks are processed by a third-party AI provider), a privacy statement, and the **Task retention period** (after which Task text and outputs are deleted). These are launch blockers, not optional.
- **Backups (D36):** what the hosted plan provides is unverified. The client must confirm the plan's backups or accept that a database loss loses visitors' workspaces during the beta. A restore rehearsal is deferred.

---

## 10. Public-beta launch checklist

None of these items has been done. The checklist describes conditions, not progress.

### A. Mandatory before public exposure
Security and isolation:
- [ ] Session design of §4 implemented, including the post-provisioning reload and workspace-id consistency check.
- [ ] CSRF/Origin checks on every state-changing request.
- [ ] Roles, grants (including column grants), RLS and functions of §7 applied; catalog tests pass against the hosted development project.
- [ ] Visitor A/B isolation tests pass for visitors, workspaces and Tasks, including cancel, spoofed ids and lease-field invisibility.
- [ ] No database credential, `AI_API_KEY` or agent system instruction in browser bundles (`check:bundle` on a fresh production build) or in the repository.
- [ ] Web server uses only `sovereign_app`; worker uses only `sovereign_worker`; neither uses `postgres` or `service_role`.
AI provider and agents:
- [ ] Supplied key rotated; new key stored only in the worker's deployment secrets.
- [ ] Provider verification of §5.11 completed and recorded, including usage categories, billed-error behaviour, hidden-token accounting and the input-token bound.
- [ ] **Provider-side spend limit configured, or the client's documented acceptance of the compromised-worker financial risk (D45).**
- [ ] Every enabled agent version has an approved acceptance package with a passing evaluation record bound to the active provider and model, and a matching enabled `agent_policies` row (§5.2).
- [ ] `platform_settings.active_provider`/`active_model` match the worker's configuration; the worker refuses to start otherwise.
Agent reliability:
- [ ] Task lifecycle works end to end against the verified provider: submit, idempotent resubmission, claim, reservation, success, every error class, deadline abort, lease recovery, stale-worker rejection, cancel (queued and running), queue timeout, capacity refusal, kill switch at reservation, provider/model mismatch.
- [ ] Output rendered through the restrictive allowlist, including storage-truncated output; error messages generic.
Cost and abuse:
- [ ] All §6.5 limits configured with client-approved values.
- [ ] Token reservation verified under concurrent claims (no reservation granted beyond the daily ceiling).
- [ ] Single worker instance deployed; global running cap set.
- [ ] Decision on per-network limiting for the first exposure (D14).
Operations and data:
- [ ] Logs reachable; operator SQL and worker liveness check available; operator procedures for kill switch, ceilings, agent policies, provider/model change and key rotation written.
- [ ] Privacy notice, privacy statement and Task retention decided and implemented (D35); backup position confirmed or accepted (D36).
- [ ] Hosted prerequisites P1–P7 verified (§11).
- [ ] Hosted verification runbook rewritten for this design and executed gate by gate with recorded evidence. **A build that compiles and local tests that pass are not sufficient.**

### B. Required before increasing traffic
- [ ] Trusted client-IP source verified and per-network limits enabled (D14).
- [ ] Automated alerts: worker not claiming, failure rate high, usage near ceilings, open reservations older than the lease, bound violations, provider/model mismatch.
- [ ] Load check at the planned global running cap and the provider's rate limits.
- [ ] Scheduled cleanup (D11).
- [ ] Cost per Task reviewed against ceilings; values adjusted.
- [ ] Automated browser tests (E2E tool needs dependency approval).

### C. Can wait until feedback validates the product
Session rotation and reuse detection; username editing and uniqueness; agent tools and their safety controls; Realtime; pgmq; monetary budgets; rolling-window token limits; visitor-created agents, Convergence, Guilds, Chronicle, public profiles; synchronizer CSRF token; disabling the Data API; restore rehearsal; operator UI; more than one worker instance.

---

## 11. External prerequisites
| # | Prerequisite | Why necessary | Blocks first beta? | Simpler supported alternative | Must verify before relying on it |
|---|---|---|---|---|---|
| P1 | Custom login roles `sovereign_app` and `sovereign_worker` can connect (pooler or direct) | Avoids `postgres`/`service_role` at runtime | **Yes** | Direct connection instead of the pooler | Test connection as each role; pooler user-name format |
| P2 | `postgres` can create `NOLOGIN` `sovereign_owner`, grant it column-level privileges and transfer function ownership | Limits what function bodies can reach | **Yes**, unless the weaker fallback is approved | `postgres`-owned functions — **needs explicit approval** (D15) | Role creation, column grants and `ALTER FUNCTION … OWNER TO` on the development project |
| P3 | Transaction-local settings and two-key advisory transaction locks behave correctly through the chosen connection mode | Visitor context must not leak; locks must serialise | **Yes** | Session-mode pooler or direct connection | Pool-isolation and concurrency tests (§12.3) |
| P4 | A PostgreSQL driver for Node (new dependency) | Web and worker talk SQL | **Yes** | — | Dependency approval; prepared-statement behaviour in the chosen mode |
| P5 | Hosting for the web app and **one** long-running worker process, with secret storage | Tasks outlive request handlers; secrets stay server-side | **Yes** | Worker as a second process on the web host | Host chosen (D37); worker runs continuously; secrets not exposed in logs or builds |
| P6 | AI provider verified (§5.11), key rotated, model chosen, provider-side spend limit (or D45 acceptance) | Agents need a working, measurable model; spending must be capped even if the worker is compromised | **Yes** | — | Every item in §5.11's list |
| P7 | Markdown sanitiser implementation (dependency or small in-house renderer) | Safe output display | **Yes** | In-house renderer limited to the allowlist | Dependency approval or review; sanitiser tests (§12.1) |
| P8 | Trusted client-IP header on the chosen host | Per-network limits | No (D14); **yes before increasing traffic** | Global protections only | Host documentation and a spoofed-header test |
| P9 | Backup capability of the hosted plan | Recovery after database loss | No, if the client accepts the risk (D36) | Accept loss for the beta | Plan documentation |
| P10 | `pg_cron` or another scheduler | Automated cleanup | No | Manual cleanup | — |

No item is assumed to be supported by Supabase, the host or the AI provider; each needs the stated evidence.

---

## 12. Testing requirements (none of these tests exists; none has been run; no counts are stated)

### 12.1 Unit (offline)
- Token generation, hashing, cookie attributes, `Max-Age`.
- CSRF checks; forwarding headers ignored unless configured.
- Task validation: empty, oversize text, estimate above `max_input_tokens`, control characters, unknown or disabled agent, missing idempotency key.
- Token estimate covers system instructions, wrapper and Task text; overhead added.
- Agent catalogue: every enabled agent has a complete acceptance package; catalogue values equal the expected policy rows; the browser-facing catalogue serialiser never includes instructions.
- Provider identity normalisation: case, trailing slash, rejection of credentials, query and fragment.
- `ModelProvider` adapter against recorded fixtures: request shape, auth header from `AI_API_KEY`, base URL from `AI_BASE_URL`, model from `AI_MODEL`; usage parsing including every reported category and missing usage; **each `error_class` produced only from its defined condition** (no `unbilled_*` class without a configured billing contract); abort exactly at the supplied deadline.
- Worker start-up: missing `AI_*` values or a mismatch with `get_worker_config()` stop the worker, with messages that never print the key.
- Allowlist sanitiser: raw HTML, images, iframes, scripts, event attributes, `javascript:`/`data:`/`http:` links all removed; allowed elements kept; links get `rel="nofollow noopener noreferrer"`; truncated output sanitised identically.
- Client consistency: a workspace-id mismatch in any action response triggers a reload; provisioning success always reloads.
- `getTask`/`listTasks` projections never include lease fields.
- Logs never contain tokens, hashes, lease tokens, cookie values, Task text, outputs, IPs or the AI key.
- Static guards: no session-level `SET`; no database URL, AI key or instructions in client code.

### 12.2 Database / RLS (pgTAP via `pnpm test:rls`, control file first, all rolled back)
- Session states and boundaries; renewal; revocation.
- Provisioning: one visitor, workspace and session; atomic global and per-network counters at their limits; `null` network subject skips only the per-network limit; a `net_subject` of 1, 31 or 33 bytes (including `\x00`) is rejected; global and network counter rows can never coincide.
- `submit_task`: idempotent lookup **before** every other check — an existing Task is returned while the kill switch is off, while its agent is unavailable, at the active limit, at the daily limit and at the global ceiling, with no counter change; different payload → `idempotency_conflict`; `daily_usage` row created when missing; the global conditional increment never exceeds the ceiling; `max_output_tokens` is copied from the policy row and cannot be supplied; no enabled matching policy → `agent_unavailable`; no context → `no_visitor`.
- Immutability: no function can change `agent_key`, `agent_version`, `input`, `input_token_estimate` or `max_output_tokens` (column privileges).
- `cancel_task`: queued → cancelled; running → `cancel_requested`; terminal → `not_cancellable`; another visitor's Task → `not_found`, unchanged.
- `claim_task`: respects the kill switch, the global running cap and `cancel_requested`; returns `max_output_tokens` and `lease_expires_at`.
- `reserve_attempt`: every outcome, including `paused` after the kill switch is turned off, `provider_mismatch` against `platform_settings` and against the policy row, `lease_too_short`, `attempts_exhausted` (→ `failed` directly), `refused_capacity` (no attempt used), and exposure computed from the stored `max_output_tokens`.
- `complete_task` / `retry_or_fail`: a wrong lease or attempt number returns `stale_lease` and changes neither the Task nor any reservation; the matching reservation is settled or released exactly once per the §5.7 table for every `error_class`; output longer than 40,000 characters is truncated with `storage_limit` and the Task succeeds; `cancel_requested` produces `cancelled`.
- `sweep_expired_leases`: settles only the reservation of the expired lease; cancelled / queued / failed transitions; queue timeouts with `capacity` vs `queue_timeout`.
- Constraints: queued with `cancel_requested` rejected; lease/output/error/finished/truncation consistency; attempts ≤ 3; one open reservation per Task; reservations carry lease and attempt.
- Isolation as `sovereign_app`: no access to other visitors' rows; spoofed context sees nothing; no direct writes; selecting `lease_token` or any non-granted `tasks` column fails with a permission error.
- As `sovereign_worker`: no table privileges at all; only its six functions.
- Catalog: role attributes; function owner, `prosecdef`, `search_path=""`, no `EXECUTE`/`format(` in bodies, exact `EXECUTE` grants; exact table **and column** privileges per role; RLS on every table; exact policy set; two-key advisory-lock usage.

### 12.3 Integration (hosted development project; fake AI provider by default)
- Pool isolation: one-connection pool, A's context then no context sees nothing; interleaved A/B transactions never cross.
- **Concurrent duplicate submissions** with one idempotency key → one Task, one counter increment.
- **Concurrent submissions** from one visitor never exceed the active or daily limit; from many visitors never exceed the global daily ceiling.
- **Concurrent provisioning** from two simulated tabs → two visitors, each with one workspace; the provisioning counter counts both.
- **Token reservation under concurrency:** several workers reserving at the edge of the ceiling never obtain a reservation that makes `used + reserved` exceed it.
- **Global running cap** with two worker instances: never more running Tasks than the cap.
- **Stale worker (audit finding 1):** worker A's call is held past its lease; the sweep requeues; worker B claims and reserves attempt 2; A then calls `complete_task` and `retry_or_fail` with its old lease and attempt — both return `stale_lease`; B's reservation stays open and B's completion settles it normally; daily totals equal A's full settlement plus B's charge.
- **Kill-switch race:** turning the switch off between claim and reservation → `paused`, no provider call; turning it off after a reservation → that one call proceeds and settles.
- **Provider/model change:** changing the worker's `AI_MODEL` (configuration only) without updating settings and policies → worker refuses to start; with settings updated but policies not re-approved → submissions `agent_unavailable` and reservations `provider_mismatch`; no provider request in either case.
- **Deadline:** a fake provider that never answers is aborted at `call_deadline`, before the lease expires; the reservation is settled in full and the Task retried.
- Lifecycle with the fake provider for every `error_class` (settlement and outcome per §5.7); oversized output; success without usage; over-reported usage logged as a bound violation and later reservations refused.
- **Cancellation races:** cancel during the call; cancel at the same moment as the sweep processes an expired lease (both orders) → `cancelled`, never claimed again.
- **UTC boundary:** a reservation granted before 00:00 UTC settles against its own day; the new day starts from zero.
- One opt-in live smoke test against the **verified** provider with a tiny prompt (separate approval; rotated key).
- Test-data cleanup per §12.5.

### 12.4 Browser (manual until an E2E tool is approved)
- Fresh profile → workspace without any form; submit a Task; close the tab; return → result shown.
- Two tabs opened at once in a fresh profile → both end on the same workspace after reload; neither keeps showing a different one.
- Reload and browser restart keep the workspace.
- Delete or tamper with the cookie → notice; "Start a new workspace" works.
- Two profiles see only their own Tasks; a copied Task URL shows nothing to the other profile; no response contains lease fields.
- A cross-site page posting to a Server Action changes nothing.
- Limits reached → generic messages; kill switch → "Agents are paused"; worker stopped → "Taking longer than expected"; truncated output shows the cut-short notice.

### 12.5 Test-data cleanup (development project only)
Integration tests record every visitor they create and use the username prefix `zz-test-`. The harness requires `SOVEREIGN_TEST_EXPECTED_PROJECT_REF` and a cleanup database URL (no defaults, no fallback). Before deleting it verifies that the URL names the expected project and that `private.test_environment` (one row, created only by a development setup script, never by a migration) holds `environment = 'development'` and the expected ref; any mismatch aborts with no `DELETE`. It deletes only recorded ids that carry the prefix, checks the count and verifies nothing related remains. No cleanup privilege exists in production.

### 12.6 Agent evaluation
Per enabled agent version: the acceptance package's examples run against the verified provider and model, scored with the agreed rubric; the record (with provider identity and model, never the key) is referenced by the agent's policy row (§5.2). Live and cost-incurring; each run needs explicit approval. Re-run whenever the instructions, provider or model change.

---

## 13. Decision register

Status: **ACCEPTED** · **REVISED** · **DEFERRED** · **REJECTED** · **NEEDS CLIENT DECISION**. Approval: **Client** = confirmed by the client; **Rec.** = recommendation awaiting client approval. Nothing marked Rec. is approved.

| # | Decision | Status | Approval | Rationale / evidence needed |
|---|---|---|---|---|
| D1 | Accountless visitors, server-generated id, HttpOnly bearer session, no Supabase Auth, display-only username, same-profile persistence, no recovery, no wallet/on-chain features | ACCEPTED | **Client** | Confirmed scope |
| D2 | Tables in non-exposed `sovereign`, support objects in `private` | ACCEPTED | Rec. | Keeps everything off the Data API |
| D3 | Generated username with format check | ACCEPTED | Rec. | Display only |
| D4 | Unique usernames | DEFERRED | Rec. | Needed only with public profiles |
| D5 | Reserved names | REVISED | Rec. | Only the test prefix matters while names are generated |
| D6 | `GET` never creates rows | ACCEPTED | Rec. | Crawlers create nothing |
| D7 | Invalid-cookie notice, "Start a new workspace", "Forget this browser" | ACCEPTED | Rec. | Never discards a valid session |
| D8 | Sliding idle lifetime, renewal in a Server Action | ACCEPTED (mechanism) / **NEEDS CLIENT DECISION** (value) | — | Proposed 90 days |
| D9 | Cookie `__Host-sovereign_session` | ACCEPTED | Rec. | |
| D10 | Rotation, lineage, grace, reuse detection | DEFERRED | Rec. | Disproportionate for the beta; trade-off §4.8 |
| D11 | Manual cleanup; scheduling deferred | REVISED | Rec. | Beta volume is small |
| D12 | Synchronizer CSRF token | DEFERRED | Rec. | Origin + `Sec-Fetch-Site` + `SameSite=Lax` suffice for the beta |
| D13 | Rate limiting mandatory before exposure | ACCEPTED | **Client** (mandatory); values Rec. | Provisioning, submission and execution |
| D14 | Per-network limits from a trusted header | **NEEDS CLIENT DECISION** | — | Host choice + spoofed-header test (P8) |
| D15 | Functions owned by `NOLOGIN` `sovereign_owner` | ACCEPTED | Rec. | `postgres` ownership only with explicit approval if P2 fails |
| D16 | PostgreSQL driver and connection mode | **NEEDS CLIENT DECISION** | — | Dependency approval + P1/P3/P4 evidence |
| D17 | Deferrable workspace trigger | DEFERRED | Rec. | Single insert path suffices |
| D18 | Disable the Data API entirely | DEFERRED | Rec. | Unused either way |
| D19 | Proxy nonce | REJECTED | Rec. | Replaced by reload-and-verify (§4.5) |
| D20 | Test-data cleanup safeguards | ACCEPTED | Rec. | §12.5 |
| D21 | Provisioning nonce | REJECTED | Rec. | Orphans accepted; inconsistent UI prevented by D41 |
| D22 | Session events | DEFERRED | Rec. | With rotation |
| D23 | Username editing | DEFERRED | Rec. | |
| D24 | Network HMAC subject | REVISED | Rec. | Exactly 32 bytes, disjoint scope from the global counter; only if D14 enables it |
| D25 | `sovereign.tasks` as single source of truth; five statuses | ACCEPTED | Rec. | |
| D26 | First-beta agent catalogue and instructions | **NEEDS CLIENT DECISION** | — | Acceptance packages (§5.2) |
| D27 | Tools decided per agent; default none | REVISED | Rec. (rule) / **NEEDS CLIENT DECISION** (per agent) | Depends on D26 |
| D28 | Worker claims via `SKIP LOCKED`; no pgmq; worker uses functions only | REVISED | Rec. | Closer to ADR-0003 |
| D29 | Polling for status | ACCEPTED | Rec. | |
| D30 | Limit values, token and Task ceilings | **NEEDS CLIENT DECISION** | — | Cost model + verified prices (P6) |
| D31 | Per-visitor limits enforced under a workspace lock | ACCEPTED | Rec. | Atomic under concurrency |
| D32 | AI provider and model | **NEEDS CLIENT DECISION** | — | OAO proposed; verification per §5.11; model name from the provider |
| D33 | One bounded call per attempt; `max_attempts` 1–3 (proposal 2); queue timeout | ACCEPTED | Rec. | Aligned with constraints |
| D34 | Restrictive markdown allowlist | ACCEPTED | Rec. | Implementation choice P7 |
| D35 | Task-input notice, privacy statement, Task retention | **NEEDS CLIENT DECISION** | — | **Launch blocker** |
| D36 | Backups | **NEEDS CLIENT DECISION** | — | Plan features or accept loss |
| D37 | Hosting; single worker instance | REVISED / **NEEDS CLIENT DECISION** (host) | Rec. (single instance) | Host choice |
| D38 | AI configuration via `AI_API_KEY`, `AI_BASE_URL`, `AI_MODEL`, worker-only; provider-neutral adapter; base URL/model changes also require settings and re-approved policies | REVISED | **Client** (configurability requested); design Rec. | Key must be rotated |
| D39 | Atomic per-attempt token reservation bound to lease and attempt; error-class settlement rules; conditional guarantees | REVISED | Rec. | §6.3; depends on verified usage reporting |
| D40 | Database-enforced global running cap; single worker instance | ACCEPTED | Rec. | Does not bind a compromised worker (§2) |
| D41 | Post-provisioning reload and workspace-id consistency check | ACCEPTED | Rec. | §4.5 |
| D42 | Agent acceptance package required for every enabled agent version, with evaluation bound to provider and model | REVISED | Rec. | §5.2 |
| D43 | Idempotency lookup before every other check | ACCEPTED | Rec. | §5.4 |
| D44 | Cancellation can never be undone by retry or recovery | ACCEPTED | Rec. | §5.3, §5.10 |
| D45 | Provider-side spend limit mandatory for launch, or documented client acceptance of the compromised-worker financial risk | **NEEDS CLIENT DECISION** (new) | — | Depends on §5.11 item 12 |
| D46 | `private.agent_policies` as the database source of truth for runnable agent versions, output caps and approved provider/model | ACCEPTED (new) | Rec. | §5.2, §7.3 |
| D47 | Fixed provider error taxonomy with deterministic settlement and retry | ACCEPTED (new) | Rec. | §5.7; billing-dependent classes only with a verified contract |
| D48 | UTC-day token accounting; boundary exposure accepted for the beta | ACCEPTED (new) | Rec. | §6.3; rolling window deferred |
| D49 | Storage-limit truncation in `complete_task` with `truncation_reason` | ACCEPTED (new) | Rec. | §5.7 |

---

## 14. Remaining client decisions and unknowns
| Item | Evidence or answer needed | From |
|---|---|---|
| First-beta agents, instructions, boundaries, examples, rubric (D26) | Approved acceptance packages | Client |
| Tools per agent (D27) | Whether any approved agent needs live data; a tool provider if so | Client |
| AI provider and model (D32, P6) | Provider documentation and a controlled test with a **rotated** key covering every §5.11 item; exact model name | Client + verification |
| Provider-side spend limit or risk acceptance (D45) | Whether the provider offers a limit; client's choice if it does not | Client + verification |
| Hard ceilings and limit values (D30) | Cost model: expected Tasks per day, tolerated daily spend, verified prices | Client |
| Task retention and privacy notice (D35) | Wording and retention period | Client |
| Hosting and worker topology (D37, P5) | Chosen host(s) | Client |
| Per-network limiting behind the real proxy (D14, P8) | Host documentation and spoofed-header test | Client + verification |
| Session lifetime value (D8) | Confirm or change 90 days | Client |
| Backups (D36, P9) | Plan features or explicit acceptance of loss | Client |
| Driver and connection mode (D16, P1–P4) | Dependency approval and hosted test results | Client / maintainer + verification |
| Sanitiser implementation (P7) | Dependency approval or in-house choice | Maintainer |
| Tokenizer bound and hidden-token accounting (§6.3) | Provider documentation and measured usage against estimates | Verification |
| Status of the old account-based work in Git (§8.1) | Read-only Git evidence | Client authorises a read-only Git check |

---

## 15. Documents that must change before implementation (not changed by this revision)
- A new ADR recording accountless visitors, the beta Task model, AI provider configuration and binding, and the simplifications above, superseding the affected parts of ADR-0001, ADR-0003, ADR-0005, ADR-0006 and ADR-0011.
- `docs/PRD.md` (accounts, verified-email gate, routes, beta scope).
- `docs/ARCHITECTURE.md` §F.1 (LLM provider), §F.2, §F.5, §F.7, §F.8, §F.9, §H.
- `docs/DATA-MODEL.md` §G (identity tables, Tasks, policies, roles). The G.8 "as implemented" line already misdescribes migration 1.
- `docs/SECURITY.md` §I (trust boundaries, privilege model, AI key handling, compromised-worker risk, verified-email gate removal).
- `docs/ROADMAP.md` (M2 scope and status, beta launch milestone).
- `.env.example` (placeholders for `AI_*` and the database URLs) — when implementation is approved.
- The M2 hosted-verification runbook (must be rewritten for this design; its previous revisions exist only in a working session, not as a file).
