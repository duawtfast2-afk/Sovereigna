# M2 Architecture Specification — Revision 8

**Accountless visitors, personal workspaces and agent Tasks for the public beta** (2026-11-13). Revisions 1–7 are superseded by this text.

Status: **specification for read-only review. Not approved for implementation or deployment.** No code, migration or test described here exists yet. No hosted verification has been performed and no AI provider request has been made. Revising this document approves nothing in it; §13 records what is confirmed by the client and what is only recommended.

Canonical file: `docs/identity/M2-SPEC.md`. The GitHub review mirror `final/M2-SPEC.md` is a synchronised copy of this file, not a separate source.

Change rule (docs/README.md): before implementation this specification needs (1) a new ADR in `docs/DECISIONS/`, (2) updates to the living documents listed in §15, and (3) a ROADMAP status update. None of those has been made; this document does not edit them.

### Changes from revision 7 (R7 audit and client-confirmed requirements)
Revision 8 is the bounded revision intended to produce the initial baseline candidate. Mechanisms from revisions 6 and 7 are kept unless listed here.
- **Model routing (new §5.13; §5.2, §5.6, §6.3, §7.3, §7.4):** client-confirmed primary model `gpt-6-astra` with round-robin fallback between `auto-vip` and `auto-regular`; the route is part of the approved policy; one model per attempt; an atomic database counter selects fallbacks; per-model cooldown; per-attempt model, route and usage records.
- **Timeout profiles (new §5.14; §5.5, §5.6, §6.4, §7.3):** client-confirmed request timeouts of 300 s (standard) and 1,200 s (slow), bound to the policy; lease derived per Task from its timeout; queue timeout, overall Task deadline and backoff defined separately; HTTP client configuration and verification gates for long requests; a slow-Task concurrency cap.
- **Pause and emergency controls (§5.6, §6.2, §6.3, §9.2):** one meaning only — `execution_enabled = false` means **execution is paused**; provider-side limits and key revocation are described as emergency mitigations with unverified delay and in-flight behaviour, not a guaranteed hard stop.
- **Prompt integrity (§2, §5.12, §6.3):** the prompt hash is described as protection against honest drift, not proof of what was sent; deterministic request construction is required; `authorize_send` re-checks that the policy is enabled.
- **Attempt accounting (§5.6, §5.7, §6.3):** only attempts that may have reached the provider count toward `max_attempts` and determine the final error code; unsent reservations are recorded as released and unsent.
- **Cost bound (§6.3):** the rolling-window statement now depends explicitly on an unverified assumption about billing after lease end or abort.
- **Key rotation (§5.11):** spend-limit confirmation on **every** key rotation, tied to a non-secret key reference with a mandatory evidence reference.
- **Model records (§5.11, §7.3):** alias acceptance is recorded per model; drift alerts only for reported models outside an approved set.
- **CSP (§8.2, §8.3, §10):** one design — per-request nonce issued by `proxy.ts`, with dynamic rendering.
- **Decision register (§13):** D32, D47, D48, D51, D55, D57, D58 revised; D61–D69 added.

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
- **Model routing requirement:** primary model `gpt-6-astra`; fallback models `auto-vip` and `auto-regular`, selected by automatic **round-robin** between the two (not a fixed sequential chain) (§5.13).
- **Request timeout requirement:** 300 seconds for normal requests and 1,200 seconds for slow requests (§5.14).
- These are client requirements. **Whether OAO offers these model identifiers, this behaviour and these request durations is unverified** (§5.11 verification gates).

### 1.2 Beta objective
Put a **usable, secure, cost-controlled** agent experience in front of visitors quickly. Priorities: reliable Task execution, visitor isolation, output quality, abuse prevention, cost control. Everything else waits for feedback.

### 1.3 Terminology
- **Task** (visitor-facing) = one visitor's instruction to one agent and the agent's output; in the blueprint's vocabulary a simplified single-agent quest/decree with one run.
- **Agent** = a platform-defined agent the visitor can choose (§5.2). Visitors do not create agents in the beta.
- **Policy** = one approved, immutable combination of agent version, prompt content, output cap, request timeout, provider, provider account and model route (`private.agent_policies` with `private.policy_models`).
- **Model route** = the policy's primary model plus its fallback models (§5.13).
- **Attempt** = one execution of a Task by the worker, containing at most one call to exactly one model, identified by `attempt_number`. A **sent attempt** is one whose request may have reached the provider (§5.6).
- **Paused** = `platform_settings.execution_enabled = false`. This document uses "paused" / "resumed" only; "kill switch" means this setting.
- **Lease** = a worker's exclusive, time-limited right to work on a running Task, identified by `lease_token`.

---

## 2. Architecture at a glance

```
Browser ──HTTPS──▶ Next.js server (apps/web)                      Worker (apps/worker, ONE long-running Node instance)
  session cookie     • validates session (hash lookup)              • claim_task (global and slow running caps)
  (HttpOnly)         • Server Actions: provision, submit, cancel    • reserve_attempt (tokens, policy, hash, config, model)
                     • reads own workspace/Tasks under RLS          • authorize_send (paused, policy, lease, remaining time)
                     • connects as sovereign_app                    • one bounded model call per attempt
                              │                                     • complete_task / retry_or_fail
                              ▼                                     • connects as sovereign_worker
                       Supabase PostgreSQL (schema `sovereign` + `private`, RLS on every table)
                                                                              │
              AI provider via adapter (AI_BASE_URL, AI_MODEL, AI_FALLBACK_MODELS, AI_ACCOUNT_REF, AI_KEY_REF, AI_API_KEY)
```

- **Browser** holds only the session cookie. It never talks to Supabase or the AI provider.
- **Web server** holds the `sovereign_app` credential. It never holds `AI_API_KEY`.
- **Worker** holds the `sovereign_worker` credential and the AI configuration. It never reads visitor sessions.
- **No Realtime, no queue extension, no Supabase Auth, no Data API use** in the beta.

Security properties:
- Visitor isolation is enforced **twice**: application code derives the workspace from the validated session, and RLS filters every `sovereign_app` query by that session (§7.5–§7.6).
- `sovereign_app` and `sovereign_worker` cannot write any table directly; all writes go through SECURITY DEFINER functions with conditional updates (§7.4).
- The token hash, not a visitor id, is the database context (§7.6). This is defence in depth, **not** protection against a compromised web server or worker.

**Worker trust boundary:**
- The worker necessarily reads every Task's input and writes every output, and it holds `AI_API_KEY`.
- A compromised worker can read all Tasks and **can call the provider directly with the key, bypassing every database control**: token reservations, daily ceilings, the concurrency caps, pausing, `authorize_send`, the policy, model-route and prompt-hash checks. It can **report the approved prompt hash and model while sending different content or a different model**. Those database controls protect **normal application execution against bugs, races and honest misconfiguration**; they do not constrain a malicious or compromised worker.
- What a compromised worker still cannot do: read `visitors` or `visitor_sessions`, create Tasks, or act as a visitor in the web application.
- The only controls that limit spending through the key are a **provider-side spend or usage limit** on the account behind `AI_API_KEY` (if the provider offers one) and **revoking the key**. Both are emergency mitigations whose effect delay and effect on requests already accepted by the provider are unverified (§5.11 item 16). A recorded spend guard (§5.11) is a **launch condition** (D45).

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

Logs may contain visitor, workspace, Task and policy ids and attempt numbers. They never contain tokens, token hashes, lease tokens, cookie values, IP addresses, AI keys, Task text, outputs or raw provider error bodies.

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
- **Theft:** the cookie is a bearer credential. Whoever holds it **is** that visitor until the session expires or is revoked: they can read the visitor's Tasks and outputs and submit Tasks against the visitor's limits. Without rotation (deferred), a copied cookie stays valid as long as the original. HttpOnly stops scripts reading it, but injected script running in the page (XSS) can still send requests that carry the cookie and act as the visitor; malware or a copied browser profile can take it. Security headers and a CSP are therefore launch items (§10 A). The real visitor can end the thief's access only with "Forget this browser" (which ends their own access too).
- **Expiry:** after the idle lifetime without use, access is lost permanently.
- **Loss:** clearing cookies, deleting the profile or losing the device loses access permanently. Accepted by the client.
- **What is protected:** other visitors' Task inputs and outputs. The beta holds no payment, wallet or account data, but it does hold whatever visitors type (§9.4).

---

## 5. Agent execution (primary concern)

### 5.1 Evidence: what exists, what M2 provides, what is deferred
| Capability | Already in the project | M2 beta must provide | Deferred |
|---|---|---|---|
| Agent catalogue | Demo-only data in `apps/web/src/demo/data.ts` (fictional, no instructions) | Platform-defined agents with approved acceptance packages and immutable database policies (§5.2) | Forge, publishing, Codex |
| Task submission | Design only (`issue_decree`, ARCHITECTURE §F.8) | `submit_task` with idempotency first, then policy checks and limits (§5.4) | Priorities, deadlines, multi-agent Tasks |
| Execution state | Design only (`agent_runs` with fenced lease) | `sovereign.tasks` as the single source of truth (§5.3) | Separate quests/runs/steps |
| Queue | Design only (pgmq, `[VERIFY AT M2]`) | Claim from `tasks` with `SKIP LOCKED` and a global running cap (§5.5, §6.4) | pgmq |
| Worker | Health/config startup check only (`apps/worker/src`) | Claim → reserve → authorise → call → settle loop and a lease sweep (§5.5–§5.7) | Heartbeats for multi-step runs |
| Model access | Design: own `ModelProvider` interface (ADR-0001, §F.1, which named Anthropic) | Provider-neutral adapter configured by environment, bound to an approved policy (§5.11) | Second provider, streaming to the UI |
| Tools | Design only (§F.5, §F.10) | Decided per agent; none unless an approved agent requires it (§5.2) | Tool runtime, idempotency, SSRF guard |
| Budget | Design: reservation/settlement ledger (§F.9) | Proportionate token reservation per attempt (§6.3) | Monetary budgets per workspace |
| Status updates | Design: Realtime Broadcast | Polling with limits (§5.9) | Realtime |
| Results | Design: `deliverables` | Output text on the Task, rendered through an allowlist (§5.7) | Versioned deliverables |

### 5.2 Agents, acceptance packages and policies (D26, D27, D42, D46, D50)
Agents are defined in a typed, version-controlled catalogue module (`packages/core`): `agent_key`, `version`, display name, Order, public description, **server-only system instructions**, the server-added wrapper template, fixed request parameters, `max_output_tokens`, the **timeout profile** (`standard` = 300 s or `slow` = 1,200 s, §5.14), whether tools are used, and whether the agent is enabled.

**An agent version may be enabled for the public beta only with an approved acceptance package:**
1. **Purpose and target Task** — who it is for and what kind of Task it handles.
2. **Capability boundaries and unsupported requests** — what it must decline or redirect. For this product the client should decide at least how agents handle requests for financial or investment advice, price predictions, statements about the token beyond the website's published content, and anything involving wallets or transactions (which the product does not support).
3. **Approved system instructions** — written or approved by the client, stored server-side only.
4. **Representative successful Task examples** — inputs with descriptions of an acceptable output.
5. **Representative unsupported and failure cases** — inputs with the expected decline or limitation message.
6. **Measurable acceptance criteria** — a rubric applied to the examples (for example: addresses the Task, respects the boundaries, follows the agreed format, no fabricated facts presented as certain). The pass threshold is agreed with the client.
7. **Evaluation record** — the examples run against the verified provider and **every model in the route** (primary and each fallback, §5.13), scored against the rubric **per model**; each model must pass on its own. The record states the **`prompt_hash`** (§5.12), the **provider identity** (normalised `AI_BASE_URL`), the **provider account reference** (`AI_ACCOUNT_REF`), each **requested model identifier**, the **model identifiers reported in the provider's responses** for each, the output cap and the timeout profile. It never contains `AI_API_KEY`. This is a live, cost-incurring run and needs explicit approval each time.
8. **Automated tests** for each agent (with a fake provider): catalogue schema valid; prompt assembly places instructions, wrapper and Task text correctly; the computed `prompt_hash` equals the approved value; output caps applied; the browser-facing catalogue response **never** contains system instructions or wrapper text (also checked on the production bundle).

**Agent policies (`private.agent_policies`, §7.3):** when an agent version's package and evaluation are approved, an operator inserts one policy row — `policy_id`, `agent_key`, `agent_version`, `prompt_hash`, `max_output_tokens`, `request_timeout_seconds`, `approved_provider`, `approved_account_ref`, `primary_model`, `fallback_models`, `evaluation_ref` (unique), `approved_at`, `enabled` — and one `private.policy_models` row **per model in the route**: `model`, `route_role`, `model_pinning`, `alias_acceptance_ref` (required for an alias), `allowed_reported_models`, `model_evaluation_ref`.
- **Approval is the policy row, not the version label.** A row exists only for an evaluated combination of prompt content, cap, timeout, provider, account and complete model route. A version label with no matching row, or with a different prompt hash, is not runnable. A policy can be enabled only if every model in its route has a `policy_models` row.
- **Policies are immutable** (D50): a trigger rejects any `UPDATE` that changes an approved field of `agent_policies` (everything except `enabled` and `enabled_changed_at`) and any `UPDATE` or `DELETE` on `policy_models`, and rejects `DELETE` on `agent_policies` (Tasks also reference rows by foreign key). Changing anything that was evaluated — instructions, wrapper, parameters, cap, timeout, provider, account, **any model or the route** — means **inserting a new policy** with new evaluation references. (An operator with table-owner rights could still disable the trigger; operator actions are trusted and logged by procedure, §9.2.)
- **At most one enabled policy** per `(agent_key, agent_version, approved_provider, approved_account_ref, primary_model, fallback_models)` (partial unique index), so `submit_task` can select one unambiguously.
- **Tasks bind to the exact `policy_id`** chosen at submission (§5.4). `max_output_tokens` and `request_timeout_seconds` are copied from it.
- **Disabling a policy** stops every Task bound to it, including Tasks queued before it was disabled: `reserve_attempt` fails them, and `authorize_send` refuses to authorise a request for a policy disabled after reservation (§6.3).
- **Prompt changes in code** produce a different `prompt_hash`. The worker computes the hash for the Task's agent version from its own catalogue and sends it; `reserve_attempt` compares it with the policy before any provider call (§6.3). A code deploy that **honestly** changes instructions or wrapper without a newly approved policy therefore cannot run that agent. This does not protect against a compromised worker (§2, §5.12).

**Tools decision (D27):** tools are enabled for an agent only if its acceptance package shows that its target Tasks cannot be done usefully without them (for example, they need current information) **and** a tool provider is separately approved. Default for the first beta: **no tools**, as a deliberate scope decision. An agent whose purpose needs current information is not enabled without tools, or its boundaries must state plainly that it has no live data. Tool definitions, if ever added, are part of the hashed prompt content (§5.12).

### 5.3 Task lifecycle
```
                submit_task (idempotent; binds policy_id)
                       │
                       ▼
   ┌─────────────► queued ─── cancel_task ──────────────────────────────────────► cancelled
   │                   │
   │      claim_task (not paused, spend guard, global and slow running caps, not cancel_requested)
   │                   ▼
   │                running ── reserve_attempt: paused / refused_capacity / lease_too_short / config_mismatch / models_unavailable ──► queued (no reservation)
   │                   │   └── reserve_attempt: policy_unavailable / attempts_exhausted / input_too_large / deadline_exceeded ─► failed
   │                   │   └── reserve_attempt: cancelled ─────────────────────────────────────────────────────► cancelled
   │      reserve_attempt reserved (attempt_number N, lease L, selected model M)
   │                   │
   │      authorize_send: paused / policy_unavailable / lease_too_short ──► queued / failed (reservation released, unsent)
   │                   └── cancelled ───────────────► cancelled (reservation released, unsent)
   │      authorize_send send → one call to model M (deadline from remaining_ms)
   │                   │
   │                   ├── success ──────────── complete_task(L, N) ─────► succeeded  (or cancelled if cancel_requested)
   │                   └── error ────────────── retry_or_fail(L, N, class) ► queued / failed / cancelled
   │
   └── sweep_expired_leases: running with expired lease ─► queued (attempts < max, not cancel_requested)
                                                         ─► cancelled (cancel_requested)
                                                         ─► failed 'worker_lost' (attempts exhausted)
       sweep: queued past queue timeout or past deadline_at ─► failed
```

| Status | Meaning |
|---|---|
| `queued` | Durable, waiting for a worker. Never has `cancel_requested = true` (constraint). |
| `running` | Claimed by a worker holding a valid lease. May have `cancel_requested = true`. |
| `succeeded` | Output stored. |
| `failed` | Terminal; a sanitised `error_code` is stored. |
| `cancelled` | Terminal; stopped at the visitor's request. |

**Invariants (enforced by constraints, column privileges, triggers and by every transition being one conditional `UPDATE` on a locked row with the expected current status and lease in its `WHERE`):**
- Terminal states never change.
- A Task with `cancel_requested = true` never returns to `queued` and is never claimed again.
- Only the holder of the current `lease_token` can reserve, authorise, complete, retry or fail a running Task, and only for the current `attempt_number`.
- A reservation is settled or released only by the call carrying its exact `(task_id, lease_token, attempt_number)`, or by the sweep for an expired lease.
- `attempts` (sent attempts) never exceeds `max_attempts`.
- `agent_key`, `agent_version`, `policy_id`, `input`, `input_token_estimate`, `max_output_tokens`, `request_timeout_seconds` and `deadline_at` never change after submission; the bound policy row never changes except `enabled`, and its `policy_models` rows never change.
- No provider request is sent unless `authorize_send` returned `send` for the current lease and attempt (in the application's own code; §2), and each request goes to exactly the model selected for that attempt.

### 5.4 Submission (`submitTask` → `submit_task`)
1. Server Action: CSRF checks (§4.7); resolve the session (`active` required); validate with Zod: `agent_key` exists and is enabled in the catalogue; Task text non-empty, within the character limit, no control characters other than newline/tab; `idempotency_key` is a UUID generated by the page per submission. Compute `input_token_estimate` over the **full assembled input** — the agent's system instructions, every server-added wrapper and the Task text (§6.3).
2. Inside `withVisitor` (§7.6), call `submit_task(agent_key, agent_version, input, input_token_estimate, idempotency_key)`. The function, in one transaction:
   1. derives the workspace from `current_visitor_id()` (none → `no_visitor`);
   2. takes a **per-workspace transaction lock** (`pg_advisory_xact_lock(1, hashtext(workspace_id::text))`, lock class 1), serialising all submissions of that workspace;
   3. **looks up `(workspace_id, idempotency_key)` first.** If a Task exists: same `agent_key`, `agent_version` and `input` → returns it with outcome `existing`, applying **no** check, limit, pause state or counter; different payload → `idempotency_conflict`;
   4. only for a new submission, in this order:
      1. execution paused (`execution_enabled = false`), or no valid spend guard for the active account and key (§5.11) → `paused`;
      2. exactly one **enabled** policy for `(agent_key, agent_version)` whose `approved_provider`, `approved_account_ref`, `primary_model` and `fallback_models` equal `platform_settings.active_provider`, `active_account_ref`, `active_primary_model` and `active_fallback_models` → otherwise `agent_unavailable`;
      3. `input_token_estimate > max_input_tokens` → `input_too_large`;
      4. per-visitor active Tasks (`queued` + `running`) and **Tasks created today in any status** (counted under the lock) → `active_limit` / `daily_limit`;
      5. create today's `private.daily_usage` row if missing (`insert … on conflict do nothing`), then the global daily Task ceiling by an **atomic conditional increment** (`update … set tasks_submitted = tasks_submitted + 1 where day = $today and tasks_submitted < $ceiling returning …`; no row → `global_limit`);
   5. inserts the Task as `queued` with that `policy_id`, copying `max_output_tokens` and `request_timeout_seconds` **from the policy row** (the caller cannot supply them), `queued_at = now()` and `deadline_at = now() + task_max_age` for the policy's timeout profile (§5.14) → `queued`.
3. **Daily-limit counting rule (D59):** the per-visitor daily limit counts every Task created on the current UTC day, whatever its final status (`queued`, `running`, `succeeded`, `failed`, `cancelled`). Cancelling or failing a Task does not give the quota back, so cancel-and-resubmit cannot exceed the limit. An idempotent `existing` result is not a new Task and does not count again. The global daily Task ceiling counts the same way (new Tasks only).
4. **Concurrent duplicates:** two requests with the same key in the same workspace are serialised by the lock; the second finds the first's Task and returns `existing`. The `UNIQUE (workspace_id, idempotency_key)` constraint is a second guard. A rolled-back insert also rolls back the counter increment.
5. Over-limit outcomes return generic messages (§6.7).

### 5.5 Claim, lease and recovery (worker)
All worker writes go through functions owned by `sovereign_owner` (§7.4). Each locks the Task row and applies one conditional update.

- **Worker configuration gate:** at start and before every claim cycle, the worker calls `get_worker_config()` and compares `execution_enabled`, `active_provider`, `active_account_ref`, `active_primary_model`, `active_fallback_models` and the spend guard (account **and** `spend_guard_key_ref`) with its own `AI_*` configuration. If anything differs, or the spend guard does not match its account and `AI_KEY_REF`, it **does not claim**, logs a configuration alert (never the key) and retries after a backoff. This prevents a misconfigured worker from churning Tasks during a rollout (M3).
- **`claim_task()`** — takes the global claim lock (`pg_advisory_xact_lock(2, 0)`, lock class 2), checks that execution is not paused, the spend guard and the **global running cap**, and for slow Tasks also the **slow running cap** (§6.4), then claims the oldest `queued` Task with `not_before <= now()` and `cancel_requested = false` (skipping slow Tasks while the slow cap is reached) using `FOR UPDATE SKIP LOCKED`; sets `status = 'running'`, a new `lease_token`, `lease_expires_at = now() + request_timeout_seconds + lease_overhead_seconds` (§5.14), `started_at`. Returns `task_id`, `lease_token`, `agent_key`, `agent_version`, `policy_id`, `input`, `max_output_tokens`, `request_timeout_seconds`. Does **not** consume an attempt. (Worker identity is in logs only.)
- **Lease** = the Task's request timeout + `lease_overhead_seconds` (proposal 60 s): 360 s for standard Tasks, 1,260 s for slow Tasks (§5.14).
- **`sweep_expired_leases()`** — run by the worker every poll cycle. For each `running` Task with `lease_expires_at <= now()` (`FOR UPDATE SKIP LOCKED`), in one transaction:
  1. settles the reservation matching `(task_id, the Task's current lease_token, status = 'open')` at its **full reserved amount**, recording it as sent with outcome `expired` and counting it as a sent attempt (a crashed or abandoned call may have been billed);
  2. moves the Task: `cancel_requested` → `cancelled`; else `attempts < max_attempts` and before `deadline_at` → `queued` with `queued_at = now()`, `not_before = now() + max(backoff, call_margin_seconds)` and the lease cleared; else → `failed`, `error_code = 'worker_lost'`.
- **Queue timeout and deadline** (also in the sweep): a `queued` Task → `failed` when `now() >= deadline_at` (`error_code = 'deadline_exceeded'`), or when it has waited longer than `queue_timeout` since `queued_at` (`'capacity'` if today's token or Task ceiling is reached at that moment, `'agent_unavailable'` if its last refusal was `config_mismatch`, `'provider_unavailable'` if its last refusal was `models_unavailable`, otherwise `'queue_timeout'`). `queued_at` is set at submission and reset only when the Task re-enters the queue after a **sent** attempt or a lease expiry; refusals (`paused`, `refused_capacity`, `config_mismatch`, `models_unavailable`, `lease_too_short`) and unsent releases do **not** reset it, so refusal cycles are bounded by the queue timeout and every Task is bounded by `deadline_at`.
- **Race: cancellation vs recovery.** `cancel_task` and the sweep both lock the same row; each re-evaluates the latest version. If cancel wins, the sweep sees `cancel_requested = true` and produces `cancelled`. If the sweep wins and requeues, `cancel_task` then finds `queued` and sets `cancelled` directly. In neither order can the Task run again.
- **Stale worker.** Worker A holds lease L1, attempt 1. Its call hangs past `lease_expires_at` (possible only if A failed to abort, e.g. after a host suspend; §5.6). The sweep settles the reservation `(T, L1, 1)` in full and requeues T. Worker B claims T with lease L2 and reserves attempt 2 `(T, L2, 2)`. When A's late `authorize_send`, `complete_task` or `retry_or_fail` with `(T, L1, 1)` arrives, the function locks T, sees `lease_token = L2 ≠ L1`, returns `stale_lease` and changes **nothing** — neither the Task nor any reservation. A's output is discarded; A's tokens were already counted by the sweep.
- **Dependency on worker availability:** lease expiry, queue timeouts and reservation settlement after crashes all run in the worker's sweep. While no worker runs, Tasks stay `queued`/`running` and reservations stay open. Open reservations keep counting against the token ceiling, so the failure is fail-safe for **normal** spending (no new calls are authorised) but not for visitors (Tasks wait). The Task page shows "Taking longer than expected" for a `queued` Task older than the queue timeout or a `running` Task past its lease. Worker liveness is an operator check at launch and an alert before increasing traffic (§10).

### 5.6 Execution (one bounded model call per attempt)
1. Assemble the request inputs for the Task's agent version from the worker's catalogue: system instructions, wrapper template, fixed parameters and the visitor's Task text. Compute `prompt_hash` (§5.12) and recompute `input_token_estimate` over the full assembled input (§6.3).
2. **`reserve_attempt(task_id, lease_token, input_token_estimate, provider_identity, account_ref, key_ref, primary_model, fallback_models, prompt_hash)`** (§6.3). The database selects the model for this attempt (§5.13). Outcomes:
   - `reserved` → returns `attempt_number`, `selected_model` and `route_role`;
   - `cancelled` → Task cancelled, no reservation;
   - `paused` (execution paused or no valid spend guard), `refused_capacity` (token ceiling), `lease_too_short`, `config_mismatch` (worker configuration or prompt hash differs from the active settings or the bound policy), `models_unavailable` (no eligible model in the route, §5.13) → Task back to `queued` with backoff, no reservation; `config_mismatch` and `models_unavailable` also raise an operator alert;
   - `policy_unavailable` (bound policy disabled) → Task `failed` (`agent_unavailable`), no reservation;
   - `attempts_exhausted` → Task `failed` with the error code of its last sent attempt (`last_error_code`), no reservation;
   - `deadline_exceeded` (a full request timeout no longer fits before `deadline_at`) → Task `failed` (`deadline_exceeded`), no reservation;
   - `input_too_large` → Task `failed` (`rejected`), no reservation;
   - `stale_lease` → the worker abandons the Task.
3. **`authorize_send(task_id, lease_token, attempt_number)`** — called **immediately before** the request is sent (§6.3). Outcomes:
   - `send` with `remaining_ms` → proceed to step 4;
   - `paused` (execution paused since the reservation) or `lease_too_short` → reservation **released and recorded as unsent**, Task back to `queued`; **no request is sent**;
   - `policy_unavailable` (policy disabled since the reservation) → released, unsent, Task `failed` (`agent_unavailable`);
   - `cancelled` → released, unsent, Task `cancelled`;
   - `stale_lease` → abandon, no request.
4. **Deadline (M2):** the worker records its **monotonic** time `t0` (Node `performance.now()`) and its wall-clock time `w0` **immediately before sending** the `authorize_send` call, and sets `send_deadline = t0 + remaining_ms`. Because `remaining_ms` was computed by the database *after* `t0`, any RPC latency only shortens the usable window; it can never extend it. The worker clock is never compared with the database clock; the wall clock is used only as an elapsed-time cross-check.
   - `remaining_ms` = `lease_expires_at − call_margin_seconds − now()` (database time), capped at the Task's `request_timeout_seconds`; `authorize_send` returns `lease_too_short` instead of `send` if it is below `min_call_budget_seconds`.
   - The adapter **refuses to start** the request if the remaining budget, measured by **either** clock (`send_deadline − monotonic_now`, or `remaining_ms − (wall_now − w0)`), is below `min_call_budget_seconds` (for example after a GC pause, process stall or host suspend between authorisation and sending); it then calls `retry_or_fail(…, 'not_sent_transient')`, which records the attempt as unsent.
   - The adapter **aborts** the request when the monotonic deadline passes (`AbortController`), and a watchdog checking every few seconds also aborts it if wall-clock elapsed time since `w0` exceeds `remaining_ms` (covers a host suspend during which the monotonic clock may not advance). Both are classified `billing_unknown_retryable` with `timed_out = true`.
   - **Host suspend (accepted residual risk):** a request already in flight during a suspend cannot be stopped until the worker resumes; on resume the watchdog aborts it. If the lease expired meanwhile, the sweep has already settled the reservation in full and requeued the Task, so a new attempt **may overlap** the provider's processing of the old request. Its tokens are counted (full settlement), and late `complete_task`/`retry_or_fail` calls are rejected by lease fencing. The beta accepts this residual risk (D52); the hosting choice (P5) should avoid suspendable hosts.
   - **No overlapping paid attempts from a healthy worker:** a new attempt can be claimed only after the old lease has expired and the sweep has requeued the Task with `not_before ≥ now() + call_margin_seconds`, or after the old attempt was settled by `complete_task`/`retry_or_fail`. A healthy worker has already aborted at `send_deadline ≤ lease_expires_at − call_margin_seconds`, so its request has ended on the client side before the next claim. **Residual risk:** aborting the HTTP request does **not** guarantee the provider stops processing or billing it (verification item 13, §5.11). Any such tokens are covered because the old reservation was settled in full.
5. Call the provider through the adapter with the **selected model**, `max_output_tokens` **as returned by `claim_task`**, and the abort deadline above, using the HTTP client configuration of §5.14.
6. Settle (§5.7). The Task text is data, not instructions to the platform; with no tools, a prompt injection can only influence that visitor's own output.

**Sent and unsent attempts (attempt accounting):** every reservation creates an attempt record (`private.attempt_reservations`). An attempt is **sent** if the request may have reached the provider: every attempt closed by `complete_task`, by `retry_or_fail` with any class other than `not_sent_*`, or by the sweep. Attempts released by `authorize_send` or closed with `not_sent_*` are **unsent** (`sent = false`). Only sent attempts increment `tasks.attempts`, count toward `max_attempts`, and set `tasks.last_error_code`; a Task failed by attempt exhaustion takes `last_error_code`, never a code derived from an unsent attempt. Unsent attempts are bounded by the queue timeout and `deadline_at` (§5.5) and visible to operators (§9.1).

### 5.7 Results, error taxonomy, retries and alerts (D47, D57)
**Success:** `complete_task(task_id, lease_token, attempt_number, output, model_truncated, reported_model, usage)` in one transaction:
1. lock the Task row; require `status = 'running'`, `lease_token` = argument, and an open reservation `(task_id, lease_token, attempt_number)`; otherwise return `stale_lease` and change nothing;
2. settle that reservation (§6.3) and record it as sent with outcome `succeeded`: with usable `usage` → charge the mapped billable total for the selected model (§5.11 usage mapping) and store the three usage fields on the attempt record; without usable usage → charge the full reserved amount;
3. record `reported_model` on the attempt record and on the Task; if it is not in the selected model's `allowed_reported_models` (§5.11), return `model_drift = true` so the worker emits a `model_drift` alert (the completed result is kept);
4. if `cancel_requested` → `cancelled`, output discarded;
5. otherwise store the output: if it exceeds the storage limit (40,000 characters), store the first 40,000 characters, set `output_truncated = true` and `truncation_reason = 'storage_limit'`; if the provider reported its output cap (`model_truncated`), set `output_truncated = true` and `truncation_reason = 'model_limit'` (`model_limit` wins if both); → `succeeded`. Oversized output never raises an error, never leaves the Task running and never triggers a retry.

**Errors:** `retry_or_fail(task_id, lease_token, attempt_number, error_class, timed_out, reported_model)` in one transaction: lock and validate exactly as step 1 above (`stale_lease` → no change); settle or release the matching reservation per the class and record `error_class`, `sent` and outcome on the attempt record; for a sent attempt increment `attempts` and set `last_error_code`; for `model_unavailable` set the model's cooldown (§5.13); then `cancelled` if `cancel_requested`; otherwise the class's Task outcome. Unknown `error_class` → error, no change. A retry of a sent attempt resets `queued_at`; a retry of an unsent attempt does not (§5.5).

The adapter assigns exactly one `error_class` per failed attempt. A class that depends on the provider's billing behaviour may be used **only** when the verified provider contract (§5.11 item 9) documents that behaviour; until then the adapter maps every failure after the request may have reached the provider to a `billing_unknown_*` class.

| `error_class` | Meaning | Reservation | Task outcome (if not `cancel_requested`) | `error_code` when failed | Operator alert |
|---|---|---|---|---|---|
| `not_sent_transient` | Request provably not sent (DNS failure, connect timeout or connection refused before any byte was written, or refused by the adapter's start-deadline check) | released; attempt **unsent** | retry with the **same** model selection rule (not counted), bounded by queue timeout and `deadline_at` | (never final; see §5.5) | **Threshold** |
| `not_sent_config` | Request not sent because of local configuration (missing or invalid `AI_*` value) | released; attempt **unsent** | failed immediately | `internal` | **Immediate** |
| `unbilled_retryable` | Provider responded; contract documents the response as not billed and retryable (e.g. rate limit, if documented) | released | retry while attempts remain (**fallback-eligible**, §5.13), else failed | `rate_limited` | **Threshold** |
| `model_unavailable` | Provider responded that the **requested model** is unavailable, overloaded or not currently served, by a signal the verified contract documents (§5.11 item 17) | released if documented unbilled, otherwise settled in full | retry while attempts remain (**fallback-eligible**; sets the model's cooldown), else failed | `provider_unavailable` | **Threshold**; **immediate** when the primary enters cooldown |
| `unbilled_terminal` | Provider responded; documented not billed; request invalid or not authorised | released | failed | `rejected` | **Immediate** (indicates a bug, an expired key or a contract change) |
| `provider_capacity` | Provider refused because the account's spend, quota or usage limit is reached | released if documented unbilled, otherwise settled in full | failed, no retry | `capacity` | **Immediate**; the operator should also consider pausing execution |
| `billing_unknown_retryable` | Abort at the deadline, transport or provider timeout after the request was written, connection lost after sending, 5xx, or any undocumented response that is plausibly transient | settled in full | retry while attempts remain (**fallback-eligible**, §5.13), else failed | **`timeout` if `timed_out = true`** (adapter abort at `send_deadline`, or a timeout reported by the transport or provider); **`provider_unavailable` in every other case** | **Threshold** |
| `billing_unknown_terminal` | Unexpected error after sending | settled in full | failed | `internal` | **Immediate** |

- **Alert levels (L3):** *Immediate* = one structured `alert` log event per occurrence. *Threshold* = an `alert` event when the count of that class in a sliding window exceeds a configured threshold (proposal: 5 in 15 minutes, `SOVEREIGN_ALERT_THRESHOLD_*`). At launch these are log events the operator checks (§9.1); automated notification is required before increasing traffic (§10 B). Alerts never contain Task text, output, keys or lease tokens.
- **Other alerts:** `config_mismatch` from `reserve_attempt` or the worker gate (immediate, deduplicated per cycle); `models_unavailable` (immediate); every fallback selection (threshold, so sustained primary failure is visible); `model_drift` (immediate); bound violation (reported usage above the reservation, immediate); open reservations older than the lease (threshold).
- **Retries:** a retry returns the Task to `queued` with backoff (proposal: 30 s, then 2 min) and a cleared lease; the next attempt gets a new lease, a new `attempt_number` and a fresh model selection (§5.13). The final `error_code` of a Task failed after sent attempts is `last_error_code`, the code of its **last sent** attempt; unsent attempts never determine it.
- The eight `error_class` values above are the complete set; the fallback-eligible classes are exactly `unbilled_retryable`, `model_unavailable` and `billing_unknown_retryable`.
- **Refusals** in the model's text are stored as normal output.
- **Partial output** from an interrupted call is discarded; only complete responses are stored.
- **Stored error data:** only `error_code` ∈ {`provider_unavailable`, `rate_limited`, `timeout`, `rejected`, `worker_lost`, `queue_timeout`, `deadline_exceeded`, `capacity`, `agent_unavailable`, `internal`}. Raw provider messages go to server logs only, sanitised (no key, no Task text).
- **Output rendering:** markdown rendered through a **restrictive allowlist** (D34): paragraphs, line breaks, headings (h2–h4), emphasis, inline code, code blocks, ordered and unordered lists, blockquotes, horizontal rules, and links with `https:` URLs only, rendered with `rel="nofollow noopener noreferrer"`. **No** raw HTML, images, iframes, forms, scripts, styles, event attributes, tables (deferred) or non-`https` URLs. Truncated output is sanitised the same way. Output is never used in SQL, file paths, shell or `eval`.
- Failed Tasks show a plain message and a "Submit again" button that creates a **new** Task (new idempotency key).

### 5.8 Isolation between visitors
- Every Task belongs to one workspace (`tasks.workspace_id`, not null).
- `sovereign_app` reads Tasks only through RLS and a column-restricted grant, and writes only through `submit_task` and `cancel_task`, which derive the workspace from the session context. A Task id from the browser is only a lookup **within** that workspace.
- A visitor cannot read, modify or cancel another visitor's Task: RLS filters reads; `cancel_task` updates only `where id = $1 and workspace_id = <context workspace>`; another visitor's Task id returns `not_found`, indistinguishable from a non-existent id.
- The worker reads all Tasks by necessity (§2 trust boundary) and cannot read `visitors` or `visitor_sessions`.

### 5.9 Status retrieval, polling limits, closing the tab (D60)
- The Task page polls `getTask` (every 2 s while `queued`/`running`, backing off to 10 s after 1 minute, stopping at a terminal status, and pausing while the tab is hidden). `listTasks` is called on page load and after submissions, not polled.
- **Read limits (L7):** `getTask` and `listTasks` are limited per session token hash with an in-memory token bucket in each web server instance (proposal: 2 requests/second sustained, burst 10, combined across both actions). Exceeding it returns a generic `slow_down` result; the client doubles its polling interval (up to 30 s) and retries. `listTasks` returns at most 20 Tasks per page. Because the buckets are per instance, the effective limit scales with the number of web instances; reads cost no AI tokens, so this is an availability control only and needs no shared database state.
- **Interaction with execution limits:** read limits are entirely separate from Task limits. Polling never consumes the per-visitor or global Task limits, never affects a Task's execution, and a `slow_down` result never cancels or delays a Task.
- `getTask` selects an explicit column list (§7.4) and each response includes the workspace id (§4.5). Lease fields are never selected or returned.
- Execution does not depend on the browser: closing the tab or returning later does not affect a Task. On return the workspace lists the visitor's Tasks (newest first, paginated) with status and output.

### 5.10 Cancellation
- `queued` → `cancelled` immediately (`cancel_task`, conditional on `status = 'queued'`).
- `running` → `cancel_requested = true` (conditional on `status = 'running'`); the flag is never cleared. The worker turns it into `cancelled` at reservation, authorisation, completion, retry or sweep. If the model call already finished, its cost is settled and its output discarded.
- Terminal Tasks → `not_cancellable`.

### 5.11 AI provider configuration (D32, D38, D45, D54, D55, D56, D68)
**Configuration** (server-side environment of the worker only; never `NEXT_PUBLIC_*`; never in the web server's environment, the browser, logs or the repository):

| Variable | Meaning | Secret? |
|---|---|---|
| `AI_API_KEY` | Provider credential | **Yes** |
| `AI_BASE_URL` | Provider endpoint base, e.g. the proposed `https://oao.ai.id/v1` | No |
| `AI_MODEL` | **Primary** model identifier as named by the provider (client requirement: `gpt-6-astra`) | No |
| `AI_FALLBACK_MODELS` | Comma-separated fallback model identifiers (client requirement: `auto-vip,auto-regular`); order is not significant and is normalised to sorted order | No |
| `AI_ACCOUNT_REF` | Non-secret reference to the provider account/project behind the key (the provider's own account or project id where one exists; otherwise an operator-assigned label recorded with the provider-console evidence) | No |
| `AI_KEY_REF` | Non-secret identifier of the **current key**: the provider's key id or name where one exists, otherwise an operator-assigned rotation reference (e.g. a date-stamped label). Never derived from the key value | No |

**Provider identity (M4)** is the tuple (normalised `AI_BASE_URL`, `AI_ACCOUNT_REF`). Normalisation: lowercase scheme and host, default port removed, path without trailing slash, no credentials, query or fragment. It is compared as text and is not a secret.

**Spend guard (D45, D68):** `private.platform_settings` records `spend_guard` ∈ {`provider_limit_confirmed`, `risk_accepted`}, `spend_guard_account_ref`, `spend_guard_key_ref`, `spend_guard_confirmed_at` and a **mandatory** `spend_guard_evidence_ref` (a console screenshot reference, ticket or decision record; never a secret).
- `provider_limit_confirmed` = an operator has confirmed in the provider's console that a spend or usage limit applies to **the key identified by `spend_guard_key_ref`** on that account (at whatever level the provider applies it: account, project or key), within the client's tolerance.
- `risk_accepted` = the client has explicitly accepted, in a documented decision referenced by the evidence field, running that key without a provider-side limit.
- A spend guard is **valid** only if `spend_guard_account_ref = active_account_ref` and `spend_guard_key_ref = active_key_ref`. `submit_task`, the worker gate, `claim_task` and `reserve_attempt` treat "no valid spend guard" as `paused`; `reserve_attempt` also requires the worker's `key_ref` to equal `active_key_ref`.

**Key rotation procedure (D68) — every rotation, even if the account is believed unchanged:**
1. Create the replacement key in the provider console and note its non-secret identifier (or assign a rotation reference).
2. Determine which account/project the new key belongs to. **Confirm the spend limit that applies to the new key** (or obtain a new `risk_accepted` decision) — never assume the new key inherits the old key's account or limit. If the account behind the key cannot be determined, **do not activate it** without an explicit client decision.
3. Pause execution. Record the new `active_key_ref` (and `active_account_ref` if changed) and the new spend guard with its evidence reference and confirmation time.
4. Replace `AI_API_KEY`, `AI_KEY_REF` (and `AI_ACCOUNT_REF` if changed) in the worker's deployment configuration; restart the worker; resume execution.
5. Revoke the old key in the provider console.
Because the worker gate and `reserve_attempt` compare `AI_KEY_REF` with the recorded spend guard, a rotated key with an unconfirmed limit cannot run. Whether the system itself can verify that a key belongs to `AI_ACCOUNT_REF` or carries `AI_KEY_REF` depends on the provider (verification item 14); if it cannot, both bindings are **operator-attested** and this procedure is the control.

**Model identity (M5, D55):**
- Each model in a route should be a **pinned, versioned model identifier** wherever the provider offers one (verification item 3). Each `policy_models` row records `model_pinning` ∈ {`pinned`, `alias_accepted`}.
- A model that is a moving alias or a provider-side router (the names `auto-vip` and `auto-regular` suggest this; unverified) may be used only with `model_pinning = 'alias_accepted'` **and** an `alias_acceptance_ref` pointing to the client's recorded acceptance of drift risk (the model behind the alias can change without any configuration change, which silently invalidates the evaluation).
- Each `policy_models` row lists `allowed_reported_models`: for a pinned model, the identifiers the provider reports for it (normally just itself); for an alias, the identifiers observed during evaluation plus any the client explicitly approves. The model reported in each response is stored on the attempt record and on the Task; **a `model_drift` alert is raised only when the reported model is outside that set** (§5.7). For aliases this alert is the only drift signal.

**Enforcement points (fail closed before any provider request):**
1. **Worker gate** (§5.5): configuration ≠ settings, or no valid spend guard → no claiming.
2. **`submit_task`**: no enabled policy matching the active provider, account and model route → `agent_unavailable`; no valid spend guard → `paused`.
3. **`reserve_attempt`**: bound policy disabled → `policy_unavailable` (permanent); worker-reported provider identity, account, key reference, model route or `prompt_hash` differing from the active settings or the bound policy → `config_mismatch` (transient, requeue with backoff, alert). The database, not the worker, selects the model for the attempt (§5.13).
4. **`authorize_send`**: paused state, policy `enabled` and lease re-checked immediately before sending.
These checks protect against bugs and honest misconfiguration; a compromised worker can ignore them (§2).

**Changing provider, account, key or model route (rollout procedure, M3):**
1. Evaluate and insert new policies for the new combination (§5.2); enable them. Old policies stay enabled.
2. Pause execution and let running attempts finish (or wait for queued Tasks to drain if preferred).
3. Change the worker's `AI_*` configuration and `platform_settings.active_*` (and the spend guard for a new account or key); restart the worker.
4. Resume execution.
5. Disable the old policies once no queued Task references them, or accept that Tasks still bound to them fail at queue timeout.
During any window where the worker and the settings disagree, the worker does not claim, and any Task already claimed is requeued (`config_mismatch`) rather than failed. A Task bound to a policy whose combination no longer matches the active configuration waits with backoff and fails with `agent_unavailable` at queue timeout (bounded; no infinite loop). A **disabled** policy fails its Tasks immediately.

**Other rules:**
- A future `.env.example` contains placeholder values only. **The key supplied in conversation is treated as exposed.** It is not recorded in this document or the repository and must be **rotated** before any real use. No request has been made with it.
- **Adapter:** provider-specific code lives behind a small `ModelProvider` port in `packages/core` (`generate(request, model, sendDeadline) → { text, modelTruncated, reportedModel, usage | null }` or `{ errorClass, timedOut, reportedModel | null }`), with one adapter in the worker. The port carries no provider names; a different provider means a new adapter. The adapter builds the wire request deterministically (§5.12).

**Usage mapping (L1, D56).** Each adapter defines, from the verified provider contract, a mapping **per model** (one mapping may be shared only if verification shows the models report usage identically) from the provider's reported usage fields into exactly three internal fields, each a non-negative integer:
- `input_tokens` — every billable input unit, including cached-input reads and writes **if** the provider reports them separately from input; if the provider reports cached tokens as a subset already included in its input count, they are **not** added again.
- `output_tokens` — every billable output unit, including reasoning or other hidden output **if** reported separately from output; if reported as a subset of output, not added again.
- `other_billable_tokens` — billable units that belong to neither (for example tool or system surcharges expressed in tokens), if any.
Rules: every billable unit the provider reports is counted **exactly once**; a mapping is defined only from documentation plus recorded fixture responses (tests, §12.1); a response containing a usage field the mapping does not know, or missing a field the mapping requires, is treated as **unusable usage** (full reservation charged) and raises an alert. Usage categories priced differently are still counted as tokens here; money is computed separately from verified prices **per model**, using the selected model and usage stored on each attempt record. Because the daily ceiling counts tokens across all three models, the cost model (D30) must size it with the most expensive model's prices.

**No compatibility is assumed.** A `/v1` suffix does not make the provider OpenAI-compatible. Before any real call is enabled, verify and record from the provider's documentation and a controlled test with a rotated key:
1. API protocol and endpoint paths;
2. authentication format (header name and scheme);
3. supported models, the exact `AI_MODEL` value, and **whether it is a pinned version or a moving alias**;
4. request parameters for the system prompt, output token cap and timeout/abort behaviour;
5. streaming behaviour (not used by the beta; confirm non-streaming works);
6. **token usage reporting** — every billable category (input, cached input reads and writes, output, reasoning or other hidden tokens), whether categories are reported as subsets or separately, and whether usage is always present;
7. context window, maximum output tokens, and whether hidden/reasoning tokens are counted inside the output cap;
8. rate limits and how they are signalled;
9. error format and **which errors are billed** (drives §5.7's classes);
10. pricing per billable token category;
11. data handling and retention terms for submitted prompts;
12. whether a provider-side spend or usage limit exists, at which level (account, project, key), and how hitting it is signalled (D45);
13. whether the provider keeps processing and billing a request after the client disconnects or aborts;
14. whether the account or project behind a key, and the key's own identifier, can be determined (console or API), so `AI_ACCOUNT_REF` and `AI_KEY_REF` can be verified rather than only attested;
15. which model identifier the provider reports in responses, **for each of the three models**;
16. **emergency controls:** how long a spend-limit change and a key revocation take to take effect, and whether either stops requests the provider has already accepted;
17. **model routing:** whether `gpt-6-astra`, `auto-vip` and `auto-regular` all exist and are selectable for this account; whether `auto-*` are fixed models, moving aliases or provider-side routers; how the provider signals that a requested model is unavailable or overloaded, and whether that response is billed; whether items 6, 7 and 10 differ per model; and whether the input-token bound (§6.3) holds for every model's tokenizer;
18. **long requests:** the provider's own maximum request duration and idle-connection behaviour for non-streaming requests of up to 300 s and 1,200 s, and whether any intermediary (CDN, load balancer) in front of the provider closes idle connections earlier (§5.14).
- Items 1–18 are **pre-activation verification gates**: no model and no timeout profile is enabled for real traffic until the items that concern it are recorded as verified.
- If usage is not reported reliably (item 6), every call is settled at its full reservation, and the ceilings must be sized accordingly.
- If a billable category is not bounded by the request's input and output caps (item 7), the model is not approved until a verified bound for that category is added to the reservation (§6.3).
- If the input-token bound cannot be established (§6.3), the provider is not enabled.

### 5.12 Prompt hash (H1, D50)
**Purpose:** detect **honest drift** — a code or configuration change that alters the evaluated prompt content without a newly approved policy. The hash is computed by the worker from its own catalogue; it does **not** prove what bytes were actually sent to the provider. A compromised worker can report the approved hash and send different content (§2), and the provider may add its own hidden instructions (especially for router models), which no client-side hash can see.

**Deterministic request construction (required):** the adapter builds the wire request as a pure function of exactly these inputs: the canonical prompt document below, the Task text, the selected model, `max_output_tokens` and the abort deadline. It adds no other text, message or parameter. Every request parameter that can change model behaviour and that the provider supports is set explicitly from the document's `parameters`, so a change in provider defaults cannot silently alter an evaluated parameter (defaults the provider applies to parameters it does not expose remain outside this control).

**Canonical prompt document** (built by `packages/core`, identically in the evaluation tool and the worker):
```json
{
  "schema": "sovereign.prompt.v1",
  "agent_key": "<agent_key>",
  "agent_version": "<agent_version>",
  "system_instructions": "<exact system instruction text>",
  "wrapper": { "before_task": "<exact server text placed before the Task text>",
               "after_task": "<exact server text placed after the Task text>" },
  "message_layout": ["system", "user"],
  "parameters": { "<fixed sampling and request parameters that affect behaviour, e.g. temperature, top_p, stop sequences>": "..." },
  "tools": []
}
```
- **Included:** everything the server adds to the request that can change the model's behaviour — system instructions, wrapper text before and after the Task text, the message layout (roles and order), fixed sampling/request parameters, and tool definitions (empty in the beta).
- **Excluded, and why:** the visitor's Task text (varies per Task; it is data); `max_output_tokens` (stored and checked separately in the policy); provider identity, account and model (stored and checked separately); `AI_API_KEY` and all credentials; per-request values (request ids, timestamps, deadlines); transport headers.
- **Canonicalisation:** strings are taken exactly as written, after converting line endings to `\n` when the catalogue is loaded; no other whitespace, case or Unicode normalisation (so any visible or invisible change to the text changes the hash). The JSON document is serialised with the JSON Canonicalization Scheme (RFC 8785): object keys sorted, no insignificant whitespace, standard escaping, UTF-8 output.
- **Hash:** `prompt_hash = SHA-256(UTF-8 bytes of the canonical JSON)`, 32 bytes, shown as lowercase hex in records.
- **Use:** the evaluation record and the policy row store the hash; the worker computes it for each attempt from its catalogue and passes it to `reserve_attempt`, which requires equality with the bound policy's `prompt_hash` before reserving (§6.3). The same prompt document is used for every model in the route. The `schema` field changes only if the canonical form itself changes, which requires re-approving every policy.

### 5.13 Model routing: primary with round-robin fallback (D61–D63)
**Client requirement:** primary `gpt-6-astra`; fallbacks `auto-vip` and `auto-regular`, alternated by round robin. Each attempt calls **exactly one** model; a Task never calls both fallbacks within one attempt.

**Route binding:** the route is part of the approved policy (`primary_model`, `fallback_models`, one `policy_models` row per model, §5.2). Every model is evaluated; changing any model or the route means a new policy. `platform_settings.active_primary_model` and `active_fallback_models` must match the policy and the worker's `AI_MODEL` / `AI_FALLBACK_MODELS` (§5.11).

**State (two small tables, written only by `sovereign_owner` functions):**
- `private.route_counters (route_key, next_index)` — one row per fallback set; `route_key` = provider identity, account reference and the sorted fallback models.
- `private.model_health (provider, account_ref, model, unavailable_until, last_signal_at)` — per-model cooldown.

**Selection, inside `reserve_attempt`** (same transaction as the reservation, after all refusal checks):
1. **Fallback needed?** Yes if this Task's last **sent** attempt failed with a fallback-eligible class (`unbilled_retryable`, `model_unavailable`, `billing_unknown_retryable`) — reason `primary_failed` or `fallback_failed`; or if the primary is in cooldown (`unavailable_until > now()`) — reason `primary_cooldown`. Otherwise the **primary** is selected (`route_role = 'primary'`).
2. **Round robin:** `update private.route_counters set next_index = next_index + 1 where route_key = $k returning next_index - 1` → `rr_index`. Candidates in order: `fallback_models[rr_index mod n]`, then the other one(s). The first candidate not in cooldown is selected (`route_role = 'fallback'`). The counter advances exactly once per fallback selection, whether or not a candidate was skipped.
3. **One fallback in cooldown:** the other is selected (recorded with its `rr_index`).
4. **Both fallbacks in cooldown:** if the primary is not in cooldown, it is selected again (reason `fallbacks_unavailable`); otherwise no model is eligible → `models_unavailable`: no reservation, Task back to `queued` with `config_wait_backoff_seconds`, alert; bounded by the queue timeout (→ `provider_unavailable`) and `deadline_at`.
5. The selected model, `route_role`, `fallback_reason` and `rr_index` are stored on the attempt record and returned to the worker, which calls only that model.

**Concurrency:** the `UPDATE … RETURNING` takes a row lock on the counter row, so concurrent `reserve_attempt` calls from any number of workers receive distinct, consecutive indexes; no worker writes the counter or chooses a fallback itself. If the reserving transaction rolls back, its increment rolls back too. The counter row is locked only for the remainder of that short transaction. Alternation is **across all Tasks** using the route, in commit order: over any run of fallback selections without cooldown skips, the two fallbacks differ by at most one selection. Neither fallback is preferred.

**Cooldown:** `retry_or_fail` with `model_unavailable` sets `unavailable_until = now() + model_cooldown_seconds` (proposal 120 s) for that model (upsert). Other fallback-eligible classes redirect only that Task's next attempt and do not change cooldown (a single timeout does not take a model out of rotation).

**Not fallback-eligible** (fail, retry the same selection rule, or wait — never switch model): `not_sent_transient` (the request never reached the model; retried unsent), `not_sent_config`, `unbilled_terminal` (invalid request, authentication or authorisation failure), `provider_capacity` (an account-level limit applies to every model), `billing_unknown_terminal`, model refusals (stored as normal output), `config_mismatch`, `policy_unavailable`, `input_too_large`, cancellation.

**Bounds:** every model call is one sent attempt; sent attempts per Task ≤ `max_attempts` (proposal 2: the primary plus at most one fallback; with 3, a second fallback attempt alternates via the counter). There is no cycling within an attempt, and the overlap rules of §5.6 apply unchanged because each attempt holds its own lease.

**Usage and cost:** each attempt record stores the selected model, the provider-reported model and the mapped usage for that model (§5.11); reservation exposure uses the policy's `max_output_tokens`, which applies to whichever model is selected, and the input bound must hold for every model (item 17).

### 5.14 Timeout profiles, leases and deadlines (D64–D66)
**Client requirement:** request timeout **300 s** (standard) and **1,200 s** (slow).

**Classification:** the timeout profile is a property of the **agent version**, decided in its acceptance package (an agent whose representative Tasks need more than 300 s in evaluation is `slow`) and stored in the policy as `request_timeout_seconds ∈ {300, 1200}`. It is never chosen by the visitor or derived from Task text. Which agents are slow is a client decision per agent (D64).

**Separate durations (never interchangeable):**
| Duration | Definition | Proposal |
|---|---|---|
| Request timeout | Maximum duration of one provider call, from the policy, copied to the Task | 300 s / 1,200 s (client) |
| Lease | `request_timeout_seconds + lease_overhead_seconds`, set at each claim | 360 s / 1,260 s (overhead 60 s) |
| Queue timeout | Maximum time a Task waits `queued` since `queued_at` (§5.5) | 15 min |
| Task deadline | `deadline_at = created_at + task_max_age` of the profile; no attempt is reserved unless a full request timeout plus `call_margin_seconds` fits before it | 30 min standard / 75 min slow |
| Retry backoff | Delay before a retry (`not_before`) | 30 s, then 2 min; at least `call_margin_seconds` after a lease expiry |

**Why a legitimate 1,200 s call is not reclaimed:** the lease is set from the Task's own timeout at claim, so it exceeds the call deadline by `lease_overhead_seconds`. `authorize_send` grants at most `request_timeout_seconds` and never beyond `lease_expires_at − call_margin_seconds` (proposal 20 s); the adapter aborts at that deadline, before the lease ends. Constraints (DDL, §7.3): `call_margin_seconds < lease_overhead_seconds` and `min_call_budget_seconds < 300`. No lease heartbeat is used (deferred).

**Trade-offs (accepted for the beta):**
- **Recovery delay:** after a worker crash, a slow Task is recovered only when its lease expires — up to about 21 minutes.
- **Throughput:** a slow Task can hold a running slot for 21 minutes. `slow_running_task_limit` (proposal 1, ≤ `global_running_task_limit`) keeps at least one slot for standard Tasks; with the proposals, slow throughput is about 3 Tasks per hour at worst.
- **Pausing:** a slow request already sent continues for up to 20 minutes after execution is paused (§6.3).
- **Cost window:** the rolling-window term of §6.3 grows with the lease.

**HTTP client configuration (required, D65):** Node's built-in `fetch` uses undici, whose default header and body timeouts are 300 s (per undici documentation; to be confirmed for the pinned Node version). That would cut off slow requests and races the 300 s profile. The adapter must therefore use a dedicated dispatcher (`undici` `Agent`, a dependency needing approval, §11 P4) with `headersTimeout` and `bodyTimeout` set above the largest request timeout (e.g. request timeout + 30 s) so that the adapter's own abort deadline is always the controlling limit, and a short connect timeout (proposal 15 s, classified `not_sent_transient`). The beta does not stream; a non-streaming request may send no bytes for up to 1,200 s.

**Verification gates before enabling a profile:** the provider's maximum request duration and idle behaviour (§5.11 item 18); the worker host's outbound connection and idle limits (NAT/firewall idle timeouts, platform request limits, P5); and a controlled test at the full duration. **Until verified, no policy with `request_timeout_seconds = 1200` is enabled** (and the 300 s profile is likewise gated). If long non-streaming requests are not supported, the slow profile stays disabled; adopting streaming would be a separate approved change.

---

## 6. Abuse prevention and cost control

### 6.1 Principles
- Anonymous execution will be abused and costs the operator money. Every limit is enforced **server-side in database functions or the worker**, never in the browser.
- Limits are configuration values in `private.platform_settings` (§7.3), editable by an operator. Numbers marked "proposal" are starting points, not evidence-based; the client sets the final values from a cost model (D30).
- When a limit is reached the system refuses new work with a generic message and never half-processes a Task.

### 6.2 Three different protections
- **Database cost controls** (§6.3, §6.4, §6.5): ceilings on tokens, Tasks, concurrency and new visitors. They bound **normal application spending** across any number of visitors and networks, under the assumptions in §6.3. They do **not** constrain a compromised worker (§2).
- **Provider-side spend limit and key revocation** (D45, D51): the only controls that also bound spending through a leaked or misused `AI_API_KEY`. They are **emergency mitigations**, not a guaranteed hard stop: how quickly they take effect, and whether they stop requests the provider has already accepted, are unverified (§5.11 item 16). A recorded spend guard is a launch condition.
- **Per-network abuse protection** (§6.6): limits per client network. These protect **fairness** — they stop one source exhausting the global ceilings and denying service to everyone else. They do not bound cost.

### 6.3 Token reservation and send authorisation (D39, D48, D51)
**Hard input-token limit:** `max_input_tokens` (configuration) bounds the **whole** request input: system instructions + server-added wrapper + Task text.
- **Estimate:** `input_token_estimate` = UTF-8 byte length of the full assembled input + a fixed per-request overhead (`request_overhead_tokens`, configuration). This is an upper bound **only if** the provider's tokenizer for the approved model never produces more tokens than bytes, and the overhead covers all template and role tokens. Both are **assumptions** to be checked during provider verification (§5.11) by comparing reported input tokens with the estimate on representative inputs, including non-English text. If they do not hold, a provider-supplied, verified token-counting method must be used instead; without a verified bound the provider is not enabled.
- The Server Action computes the estimate; `submit_task` and `reserve_attempt` both reject an estimate above `max_input_tokens`. The database cannot recompute the estimate (it does not hold the instructions); it trusts the web server and the worker for it, which is consistent with §2's trust boundary. The character limit (proposal 4,000) is a UX guard, not the cost bound.

**Reservation exposure per attempt** = `input_token_estimate` (recomputed by the worker) + `tasks.max_output_tokens` (copied from the bound policy) + any verified bound for billable categories outside those caps (§5.11 item 7).

**`reserve_attempt(task_id, lease_token, input_token_estimate, provider_identity, account_ref, key_ref, primary_model, fallback_models, prompt_hash)`** — one transaction:
1. Lock the Task row. Not `running` with this `lease_token` → `stale_lease`. `cancel_requested` → set `cancelled`, return `cancelled`.
2. Read `platform_settings` and the bound policy in the same transaction.
   - Policy `enabled = false` → Task `failed` (`agent_unavailable`), return `policy_unavailable`.
   - Execution paused (`execution_enabled = false`), or no valid spend guard (§5.11) → Task back to `queued` with backoff, return `paused`.
   - `provider_identity`, `account_ref`, `key_ref`, `primary_model` or `fallback_models` differ from the active settings **or** from the policy's approved values, or `prompt_hash` differs from the policy's `prompt_hash` → Task back to `queued` with `not_before = now() + config_wait_backoff_seconds`, `last_refusal = 'config_mismatch'`, return `config_mismatch` (the worker raises an alert).
3. `attempts` (sent attempts) `>= max_attempts` → Task `failed` with `last_error_code`, return `attempts_exhausted`. Estimate above `max_input_tokens` → Task `failed` (`rejected`), return `input_too_large`. `now() + request_timeout_seconds + call_margin_seconds > deadline_at` → Task `failed` (`deadline_exceeded`), return `deadline_exceeded`. Less than `min_call_budget_seconds` left before `lease_expires_at − call_margin_seconds` → Task back to `queued`, `last_refusal = 'lease_too_short'`, return `lease_too_short`.
4. Create today's `private.daily_usage` row if missing, then lock it (`FOR UPDATE`). If `used_tokens + reserved_tokens + exposure > daily_token_ceiling` → Task back to `queued` with backoff, `last_refusal = 'capacity'`, return `refused_capacity`. (Exposure does not depend on the model: the output cap and input bound apply to every model in the route.)
5. Select the model (§5.13); the round-robin counter is advanced only here, so refusals never consume a rotation slot. No eligible model → Task back to `queued` with backoff, `last_refusal = 'models_unavailable'`, return `models_unavailable`.
6. Otherwise: `reserved_tokens += exposure`; insert `private.attempt_reservations (task_id, attempt_number = (highest attempt_number for the Task) + 1, lease_token, day, reserved_tokens = exposure, status = 'open', expires_at = lease_expires_at, requested_model, route_role, fallback_reason, rr_index)`; return `reserved`, `attempt_number`, `selected_model`, `route_role`. `tasks.attempts` is **not** incremented here (§5.6 attempt accounting).

**`authorize_send(task_id, lease_token, attempt_number)`** — one transaction, called immediately before the request (M1, M2):
1. Lock the Task row; require `running`, the lease and an open reservation `(task_id, lease_token, attempt_number)`; otherwise `stale_lease`, no change.
2. `cancel_requested` → release the reservation (unsent), Task `cancelled`, return `cancelled`.
3. Bound policy `enabled = false` → release (unsent), Task `failed` (`agent_unavailable`), return `policy_unavailable`.
4. Execution paused → release (unsent), Task back to `queued`, return `paused`.
5. `remaining = lease_expires_at − call_margin_seconds − now()`, capped at the Task's `request_timeout_seconds`; below `min_call_budget_seconds` → release (unsent), Task back to `queued`, return `lease_too_short`.
6. Otherwise set `send_authorized_at = now()` on the attempt record and return `send` and `remaining_ms`.
An attempt released here was never sent: it does not increment `attempts`, does not set `last_error_code` and does not reset `queued_at` (§5.6).

**Pause boundary (M1):**
- After the operator's change to `execution_enabled = false` commits, no **new reservation** is granted and every later `authorize_send` returns `paused`.
- A request can still start after pausing **only** if its `authorize_send` returned `send` before the change committed; the remaining window is the time between that call and the request leaving the worker (normally milliseconds). At most `global_running_task_limit` such requests can exist at once (plus any request from a suspended worker, §5.6).
- Requests already in flight are not interrupted; a slow request may continue for up to 20 minutes.
- **Pausing therefore does not guarantee zero provider requests or charges after activation.** The emergency provider controls — lowering or exhausting the provider-side spend limit, and **revoking `AI_API_KEY`** — are mitigations whose effect delay and effect on already-accepted requests are unverified (§5.11 item 16). Both are in the operator procedure (§9.2); neither is described as a guaranteed hard stop.

**Settlement and release** (always in the same transaction as the Task's state change, and only for the reservation matching `(task_id, lease_token, attempt_number)` — or, in the sweep, `(task_id, current lease_token)`):
| Path | Reservation outcome |
|---|---|
| Success with usable usage | `settled`, sent; `charged_tokens` = mapped billable total for the selected model (`input + output + other`, §5.11); `used_tokens += charged`; `reserved_tokens -= reserved` |
| Success without usable usage | `settled`, sent, at the **full** reserved amount |
| Cancellation detected in `reserve_attempt` | No reservation was taken |
| Cancellation, pause, disabled policy or short lease detected in `authorize_send` | `released`, **unsent** (nothing was sent) |
| Cancellation detected at completion | Settled like a success; output discarded |
| `not_sent_*` | `released`, **unsent**: `reserved_tokens -= reserved`; nothing added to `used` |
| `unbilled_*`, `model_unavailable` or `provider_capacity` documented unbilled | `released`, sent (the provider responded): `reserved_tokens -= reserved`; nothing added to `used` |
| `billing_unknown_*`, `model_unavailable` or `provider_capacity` not documented unbilled | `settled`, sent, at the **full** reserved amount |
| Worker crash / lease expiry | The sweep settles the reservation matching the current lease at the full amount, sent, outcome `expired` |
| Stale worker after takeover | `stale_lease`; nothing changes (§5.5) |

**What is guaranteed, and under which assumptions:**
- **Unconditional (database invariant):** at the moment any reservation is granted, `used_tokens + reserved_tokens ≤ daily_token_ceiling` for that UTC day, because grants serialise on the locked day row.
- **Conditional:** total tokens actually billed for one UTC day's reservations stay within that day's ceiling **only if** for every attempt the billed tokens ≤ the reserved exposure. That holds only if (a) the input-token bound holds for the model, (b) billed output, including any hidden/reasoning tokens, never exceeds `max_output_tokens` unless separately bounded, (c) every other billable category is covered and mapped exactly once (§5.11) for every model in the route, and (d) no provider request happens without a reservation and a `send` authorisation — which holds for the application's own code but **not** for a compromised worker (§2). Billing **after** the lease or an abort (provider continuing after disconnect, or a request from a suspended worker) does not break this per-day bound, because such an attempt's reservation is already settled at its full exposure on its grant day; it only moves *when* the tokens are billed (next paragraph).
- **Over-reported usage:** if mapped billable usage exceeds the reservation, the reported value is charged (`used_tokens` may then exceed the ceiling), later reservations for that day are refused, and the attempt is logged for the operator as a bound violation. Repeated violations mean the assumptions are wrong and the policy must be disabled until fixed.

**UTC day boundary and rolling-window bound (L2, D48):**
- Reservations are charged to the UTC day on which they were **granted**, and settle against that day's row, even if the call finishes after midnight. At 00:00 UTC a new day's ceiling starts from zero.
- **Rolling 24-hour exposure (an estimate, not a guarantee).** Let `C` = `daily_token_ceiling`, `E` = the maximum exposure of one attempt (`max_input_tokens + max_output_tokens` + any verified extra-category bound), `K` = `global_running_task_limit`. A 24-hour window W overlaps at most two UTC days, and reservations granted on those days contribute at most `2C`. Attempts granted on an **earlier** day can also bill inside W:
  - **if** every attempt stops billing by the end of its lease — an **unverified assumption** (§5.11 item 13; host suspend, §5.6) — such attempts must have been running when W started, which is possible only if W starts less than one lease (up to about 21 minutes for slow Tasks) after 00:00 UTC; at most `K` run at once, so W sees at most **`2C + K·E`**;
  - **if** the provider continues billing after an abort or lease end, late billing from earlier days' attempts can fall into any window. Each such attempt is still bounded by `E` and already counted at full exposure in its own day's `C`, but **no time-window bound is claimed** until verification item 13 is recorded.
  - Therefore "at most twice the daily ceiling" is not a guarantee. The per-UTC-day accounting bound above is the control the system enforces; the rolling-window figure is an operator estimate. Over-reported usage (bound violations, above) is outside both.
- The ceiling bounds **each UTC day's accounting**, not a rolling window. This is a **recommended beta trade-off** for simplicity; a rolling-window limit is deferred.
- A reservation never stays open indefinitely: every open reservation belongs to a running Task with a lease; the sweep closes any reservation of an expired lease together with the Task transition. While the worker is down, open reservations remain and count against the ceiling (§5.5).
- This is deliberately **not** a monetary ledger: it bounds tokens. Per-workspace monetary budgets are deferred.

### 6.4 Global concurrency (D40)
- **Database-enforced global cap:** `global_running_task_limit` (proposal: 2). `claim_task` takes the global claim lock (class 2), counts `running` Tasks and claims only if the count is below the cap. The cap holds across **any** number of worker processes running the application's code.
- **Slow cap:** `slow_running_task_limit` (proposal: 1, never above the global cap). Under the same lock, a slow Task (`request_timeout_seconds = 1200`) is claimed only while fewer slow Tasks are running than this cap; standard Tasks behind it can still be claimed (§5.14).
- **Topology for the beta: a single worker instance** with in-process concurrency equal to the cap. The database cap remains the guarantee if a second instance is started by mistake.
- **Consequences:** throughput ≤ cap ÷ average Task duration (at a cap of 2 and about 30 s per call, roughly 4 Tasks per minute; far less when slow Tasks run, §5.14); Tasks beyond that wait and may hit the queue timeout; the maximum normal spend rate is bounded by cap × per-attempt exposure per call duration. A worker outage stops all execution (§5.5). A compromised worker is not bound by the cap (§2).

### 6.5 Limits table
| Control | Where enforced | Proposal | When reached |
|---|---|---|---|
| Pause (`execution_enabled = false` = paused) | `submit_task` (new submissions only), worker gate, `claim_task`, `reserve_attempt`, `authorize_send` | paused until launch | Submissions refused ("Agents are paused"); nothing claimed; no new reservation or send authorisation; already-authorised requests may still start (§6.3); calls in flight finish |
| Valid spend guard for the active account and key | `submit_task`, worker gate, `claim_task`, `reserve_attempt` | must be set (D45, D68) | `paused` |
| Policy, prompt hash and active provider/account/key/model route match | `submit_task`, worker gate, `reserve_attempt`; policy `enabled` also in `authorize_send` | — | `agent_unavailable` / `policy_unavailable` / `config_mismatch`; no provider request |
| Model cooldown | `retry_or_fail` (`model_unavailable`), `reserve_attempt` | 120 s | Fallback or `models_unavailable` (§5.13) |
| Global daily Task ceiling | `submit_task`, atomic conditional increment | client sets | "Agents are busy today. Please try again tomorrow." |
| Global daily token ceiling | `reserve_attempt` (§6.3) | client sets | Tasks wait, then fail `capacity` at queue timeout |
| Hard input-token limit | Server Action, `submit_task`, `reserve_attempt` | client sets from the model's context window | Validation message |
| Output token cap per policy | `private.agent_policies`, copied to the Task | client approves per agent | Truncation flag (`model_limit`) |
| Output storage limit | `complete_task` | 40,000 characters | Truncation flag (`storage_limit`) |
| Provider-side spend/usage limit | provider account (D45) | below the operator's tolerance | `provider_capacity` → `failed` (`capacity`), no retry |
| Global provisioning ceiling (per hour) | `provision_visitor`, atomic conditional upsert | 1,000 / hour | "Sovereign is busy. Please try again shortly." |
| Per-network provisioning limit | `provision_visitor`, only with a trusted client IP (§6.6) | 10 / hour, 30 / day | "Too many new workspaces from this network." |
| Per-visitor active Tasks | `submit_task`, under the workspace lock | 1 | "Wait for your current Task to finish." |
| Per-visitor Tasks per day (all statuses count, §5.4) | `submit_task`, under the workspace lock | 20 | "You've reached today's Task limit." |
| Task text length | Server Action, `CHECK` | 4,000 characters | Validation message |
| Read rate (`getTask` + `listTasks`) | web server, per session, per instance (§5.9) | 2/s sustained, burst 10 | `slow_down`; client backs off |
| Request timeout (call deadline) | policy → Task; `authorize_send` returns `remaining_ms`; adapter aborts on its monotonic clock and wall-clock watchdog | 300 s / 1,200 s (client); margin 20 s; minimum budget 30 s | `billing_unknown_retryable` (`timeout`) |
| Lease | `claim_task` | request timeout + 60 s | Sweep recovery (§5.5) |
| `max_attempts` (sent attempts) | settings (1–3), `reserve_attempt`, `CHECK` | 2 | `failed` with `last_error_code` |
| Global running Tasks | `claim_task` | 2 | Tasks wait |
| Slow running Tasks | `claim_task` | 1 | Slow Tasks wait |
| Queue timeout (since `queued_at`) | sweep | 15 min | `failed` (`queue_timeout` / `capacity` / `agent_unavailable` / `provider_unavailable`) |
| Task deadline (`deadline_at`) | `reserve_attempt`, sweep | 30 min standard / 75 min slow | `failed` (`deadline_exceeded`) |
| Request body size | Next.js Server Actions default 1 MB (bundled docs) | keep default | Request rejected |

**Token bounds:** per UTC day, `daily_token_ceiling` under the conditions in §6.3; any rolling 24 hours, an estimate of `2C + K·E` only under the additional unverified assumption stated there. Per Task ≤ `max_attempts × E`. Money = tokens × verified prices (§5.11); this document states no prices.

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
| `postgres` | Supabase-managed | operators, migrations | Owns tables; operators maintain `platform_settings` and insert or enable/disable `agent_policies`. **Never** used by the web server or worker at runtime. |
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

create table private.agent_policies (                 -- inserted by operators after approval; immutable except `enabled`
  policy_id             uuid primary key default gen_random_uuid(),
  agent_key             text not null check (agent_key ~ '^[a-z0-9-]{1,40}$'),
  agent_version         text not null check (char_length(agent_version) between 1 and 40),
  prompt_hash           bytea not null check (octet_length(prompt_hash) = 32),
  max_output_tokens     integer not null check (max_output_tokens > 0),
  request_timeout_seconds integer not null check (request_timeout_seconds in (300, 1200)),  -- client profiles
  approved_provider     text not null check (approved_provider ~ '^https://[^\s?#@]+$'),
  approved_account_ref  text not null check (char_length(approved_account_ref) between 1 and 200),
  primary_model         text not null check (char_length(primary_model) between 1 and 200),
  fallback_models       text[] not null check (cardinality(fallback_models) between 0 and 2),  -- sorted, distinct
  evaluation_ref        text not null unique check (char_length(evaluation_ref) between 1 and 200),
  approved_at           timestamptz not null,
  enabled               boolean not null default false,
  enabled_changed_at    timestamptz,
  check (not (primary_model = any (fallback_models)))
);
create unique index agent_policies_one_enabled
  on private.agent_policies (agent_key, agent_version, approved_provider, approved_account_ref, primary_model, fallback_models)
  where enabled;

create table private.policy_models (                  -- one row per model in the route; immutable
  policy_id               uuid not null references private.agent_policies (policy_id),
  model                   text not null check (char_length(model) between 1 and 200),
  route_role              text not null check (route_role in ('primary', 'fallback')),
  model_pinning           text not null check (model_pinning in ('pinned', 'alias_accepted')),
  alias_acceptance_ref    text,                       -- client's recorded acceptance of alias drift
  allowed_reported_models text[] not null check (cardinality(allowed_reported_models) >= 1),
  model_evaluation_ref    text not null check (char_length(model_evaluation_ref) between 1 and 200),
  primary key (policy_id, model),
  check ((model_pinning = 'alias_accepted') = (alias_acceptance_ref is not null))
);
-- Trigger private.agent_policies_immutable(): on agent_policies, BEFORE UPDATE raises unless only
-- `enabled`/`enabled_changed_at` change, and setting enabled = true raises unless policy_models has exactly
-- the primary and every fallback model with matching route_role; BEFORE DELETE always raises.
-- On policy_models, BEFORE UPDATE or DELETE always raises, and INSERT raises once the policy has ever been enabled.

create table sovereign.tasks (
  id                    uuid primary key default gen_random_uuid(),
  workspace_id          uuid not null references sovereign.workspaces (id) on delete cascade,
  idempotency_key       uuid not null,
  agent_key             text not null,
  agent_version         text not null,
  policy_id             uuid not null references private.agent_policies (policy_id),
  input                 text not null check (char_length(input) between 1 and 4000),
  input_token_estimate  integer not null check (input_token_estimate > 0),
  max_output_tokens     integer not null check (max_output_tokens > 0),       -- copied from the bound policy
  request_timeout_seconds integer not null check (request_timeout_seconds in (300, 1200)),  -- copied from the policy
  deadline_at           timestamptz not null,                                  -- created_at + task_max_age of the profile
  status                text not null default 'queued'
                        check (status in ('queued', 'running', 'succeeded', 'failed', 'cancelled')),
  attempts              integer not null default 0 check (attempts between 0 and 3),   -- sent attempts only
  last_error_code       text,                                                  -- error_code of the last sent attempt
  queued_at             timestamptz not null default now(),
  not_before            timestamptz not null default now(),
  last_refusal          text check (last_refusal in ('capacity', 'config_mismatch', 'paused', 'lease_too_short',
                                                     'models_unavailable')),
  cancel_requested      boolean not null default false,
  lease_token           uuid,
  lease_expires_at      timestamptz,
  output                text check (output is null or char_length(output) <= 40000),
  output_truncated      boolean not null default false,
  truncation_reason     text check (truncation_reason in ('model_limit', 'storage_limit')),
  error_code            text check (error_code in ('provider_unavailable', 'rate_limited', 'timeout', 'rejected',
                                                   'worker_lost', 'queue_timeout', 'deadline_exceeded', 'capacity',
                                                   'agent_unavailable', 'internal')),
  model                 text check (model is null or char_length(model) <= 200),   -- last model reported by the provider
  input_tokens          integer check (input_tokens >= 0),
  output_tokens         integer check (output_tokens >= 0),
  other_billable_tokens integer check (other_billable_tokens >= 0),
  created_at            timestamptz not null default now(),
  started_at            timestamptz,
  finished_at           timestamptz,
  unique (workspace_id, idempotency_key),
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
  active_account_ref              text not null check (char_length(active_account_ref) between 1 and 200),
  active_key_ref                  text not null check (char_length(active_key_ref) between 1 and 200),
  active_primary_model            text not null check (char_length(active_primary_model) between 1 and 200),
  active_fallback_models          text[] not null check (cardinality(active_fallback_models) between 0 and 2),
  spend_guard                     text check (spend_guard in ('provider_limit_confirmed', 'risk_accepted')),
  spend_guard_account_ref         text,
  spend_guard_key_ref             text,
  spend_guard_evidence_ref        text,
  spend_guard_confirmed_at        timestamptz,
  global_daily_task_ceiling       integer not null check (global_daily_task_ceiling >= 0),
  daily_token_ceiling             bigint  not null check (daily_token_ceiling >= 0),
  max_input_tokens                integer not null check (max_input_tokens > 0),
  request_overhead_tokens         integer not null check (request_overhead_tokens >= 0),
  max_attempts                    integer not null check (max_attempts between 1 and 3),
  global_running_task_limit       integer not null check (global_running_task_limit between 1 and 20),
  slow_running_task_limit         integer not null check (slow_running_task_limit >= 0),
  lease_overhead_seconds          integer not null check (lease_overhead_seconds between 30 and 300),
  call_margin_seconds             integer not null check (call_margin_seconds between 5 and 120),
  min_call_budget_seconds         integer not null check (min_call_budget_seconds between 5 and 299),
  config_wait_backoff_seconds     integer not null check (config_wait_backoff_seconds between 10 and 3600),
  model_cooldown_seconds          integer not null check (model_cooldown_seconds between 10 and 3600),
  task_max_age_standard           interval not null check (task_max_age_standard >= interval '15 minutes'),
  task_max_age_slow               interval not null check (task_max_age_slow >= interval '25 minutes'),
  global_hourly_provision_ceiling integer not null check (global_hourly_provision_ceiling >= 0),
  net_hourly_provision_limit      integer not null check (net_hourly_provision_limit >= 0),
  net_daily_provision_limit       integer not null check (net_daily_provision_limit >= 0),
  visitor_active_task_limit       integer not null check (visitor_active_task_limit >= 0),
  visitor_daily_task_limit        integer not null check (visitor_daily_task_limit >= 0),
  queue_timeout                   interval not null check (queue_timeout > interval '0'),
  check (call_margin_seconds < lease_overhead_seconds),
  check (slow_running_task_limit <= global_running_task_limit),
  check ((spend_guard is null) = (spend_guard_account_ref is null)),
  check ((spend_guard is null) = (spend_guard_key_ref is null)),
  check ((spend_guard is null) = (spend_guard_evidence_ref is null)),
  check ((spend_guard is null) = (spend_guard_confirmed_at is null))
);
-- A spend guard is valid only when spend_guard_account_ref = active_account_ref
-- and spend_guard_key_ref = active_key_ref.

create table private.daily_usage (
  day              date primary key,
  tasks_submitted  integer not null default 0 check (tasks_submitted >= 0),
  reserved_tokens  bigint  not null default 0 check (reserved_tokens >= 0),
  used_tokens      bigint  not null default 0 check (used_tokens >= 0)
);

create table private.attempt_reservations (            -- one row per attempt: reservation + model record
  task_id          uuid not null references sovereign.tasks (id) on delete cascade,
  attempt_number   integer not null check (attempt_number >= 1),  -- includes unsent attempts
  lease_token      uuid not null,
  day              date not null references private.daily_usage (day),
  reserved_tokens  integer not null check (reserved_tokens > 0),
  status           text not null default 'open' check (status in ('open', 'settled', 'released')),
  charged_tokens   integer check (charged_tokens >= 0),     -- set when settled
  requested_model  text not null,                           -- model selected for this attempt
  route_role       text not null check (route_role in ('primary', 'fallback')),
  fallback_reason  text check (fallback_reason in ('primary_failed', 'fallback_failed', 'primary_cooldown',
                                                   'fallbacks_unavailable')),
  rr_index         bigint,                                  -- set when a fallback was selected
  send_authorized_at timestamptz,
  sent             boolean,                                 -- null while open; false = released before sending
  outcome          text check (outcome in ('succeeded', 'failed', 'released_unsent', 'expired')),
  error_class      text,
  reported_model   text,
  input_tokens     integer check (input_tokens >= 0),
  output_tokens    integer check (output_tokens >= 0),
  other_billable_tokens integer check (other_billable_tokens >= 0),
  expires_at       timestamptz not null,
  closed_at        timestamptz,
  primary key (task_id, attempt_number),
  check ((status = 'open') = (closed_at is null)),
  check ((status = 'open') = (sent is null)),
  check ((status = 'settled') = (charged_tokens is not null)),
  check (not (status = 'settled' and sent is false)),
  check ((route_role = 'fallback') = (rr_index is not null))
);
create unique index attempt_reservations_one_open on private.attempt_reservations (task_id) where status = 'open';

create table private.route_counters (                   -- round-robin state, one row per fallback set
  route_key   text primary key,                         -- provider identity | account ref | sorted fallback models
  next_index  bigint not null default 0 check (next_index >= 0)
);

create table private.model_health (                     -- per-model cooldown
  provider           text not null,
  account_ref        text not null,
  model              text not null,
  unavailable_until  timestamptz,
  last_signal_at     timestamptz,
  primary key (provider, account_ref, model)
);

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
- **Immutable cost and approval fields:** `sovereign_owner`'s column-level `UPDATE` grant on `tasks` (§7.4) excludes `agent_key`, `agent_version`, `policy_id`, `input`, `input_token_estimate`, `max_output_tokens`, `request_timeout_seconds`, `deadline_at`, `workspace_id` and `idempotency_key`. `sovereign_owner` has no `UPDATE` on `agent_policies` or `policy_models` at all, and the immutability trigger also stops operators from silently changing approved fields or routes. The foreign keys keep referenced policy rows from being deleted.
- **Attempts:** `tasks.attempts` counts **sent** attempts only (≤ 3, matching the `max_attempts` range 1–3; proposal 2, enforced by `reserve_attempt`). `attempt_reservations.attempt_number` numbers every attempt record, including unsent ones, which are bounded by the queue timeout and `deadline_at`.
- **Round robin and cooldown:** `route_counters` and `model_health` rows are created by an operator setup step (counter) or by upsert (health); only `reserve_attempt` advances a counter and only `retry_or_fail` sets a cooldown.
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
| `get_worker_config() → (execution_enabled, active_provider, active_account_ref, active_key_ref, active_primary_model, active_fallback_models, spend_guard_ok)` | `sovereign_worker` | Read-only, for the worker gate (§5.5). `spend_guard_ok` = the spend guard is valid for the active account and key. |
| `claim_task() → (task_id, lease_token, agent_key, agent_version, policy_id, input, max_output_tokens, request_timeout_seconds)` | `sovereign_worker` | §5.5. No row when paused, without a valid spend guard, at the running caps, or nothing claimable. |
| `reserve_attempt(task_id, lease_token, input_token_estimate, provider_identity, account_ref, key_ref, primary_model, fallback_models, prompt_hash) → (outcome, attempt_number, selected_model, route_role)` | `sovereign_worker` | §6.3, §5.13. `outcome ∈ {reserved, cancelled, paused, policy_unavailable, config_mismatch, models_unavailable, attempts_exhausted, deadline_exceeded, input_too_large, lease_too_short, refused_capacity, stale_lease}`. |
| `authorize_send(task_id, lease_token, attempt_number) → (outcome, remaining_ms)` | `sovereign_worker` | §6.3. `outcome ∈ {send, paused, policy_unavailable, cancelled, lease_too_short, stale_lease}`. |
| `complete_task(task_id, lease_token, attempt_number, output, model_truncated, reported_model, usage) → (outcome, model_drift)` | `sovereign_worker` | §5.7. Validates lease and attempt before settlement. `outcome` ∈ {`succeeded`, `cancelled`, `stale_lease`}; `model_drift` true if the reported model is outside the allowed set. |
| `retry_or_fail(task_id, lease_token, attempt_number, error_class, timed_out, reported_model) → text` | `sovereign_worker` | §5.7 taxonomy (eight classes). Validates lease and attempt before settlement. `queued`, `failed`, `cancelled` or `stale_lease`. Unknown `error_class` → error, no change. |
| `sweep_expired_leases() → integer` | `sovereign_worker` | §5.5; also queue timeouts and Task deadlines. Settles only reservations matching the expired lease. |

`usage` is a typed composite (`input_tokens`, `output_tokens`, `other_billable_tokens`, all non-negative, or null when unusable). `prompt_hash` is 32 bytes; any other length is an error. `fallback_models` is compared as a sorted array.

Privileges of `sovereign_owner` (nothing else; owner-scoped RLS policies for exactly these commands):
- `sovereign.visitors`: `SELECT`, `INSERT`, `UPDATE (last_seen_at)`.
- `sovereign.workspaces`: `SELECT`, `INSERT`.
- `sovereign.visitor_sessions`: `SELECT`, `INSERT`, `UPDATE (last_renewed_at, expires_at, revoked_at, revoke_reason)`.
- `sovereign.tasks`: `SELECT`, `INSERT`, `UPDATE (status, attempts, last_error_code, queued_at, not_before, last_refusal, cancel_requested, lease_token, lease_expires_at, output, output_truncated, truncation_reason, error_code, model, input_tokens, output_tokens, other_billable_tokens, started_at, finished_at)` — **not** the immutable fields.
- `private.agent_policies`, `private.policy_models`, `private.platform_settings`: `SELECT` only.
- `private.daily_usage`: `SELECT`, `INSERT`, `UPDATE (tasks_submitted, reserved_tokens, used_tokens)`.
- `private.attempt_reservations`: `SELECT`, `INSERT`, `UPDATE (status, charged_tokens, send_authorized_at, sent, outcome, error_class, reported_model, input_tokens, output_tokens, other_billable_tokens, closed_at)`.
- `private.route_counters`: `SELECT`, `UPDATE (next_index)`.
- `private.model_health`: `SELECT`, `INSERT`, `UPDATE (unavailable_until, last_signal_at)`.
- `private.provision_counters`: `SELECT`, `INSERT`, `UPDATE (hit_count)`.
- `USAGE` on both schemas. **No `DELETE`.** Advisory locks use the two-key form with class 1 (workspace submission) and class 2 (global claim), so the two cannot collide.

Privileges of `sovereign_app`:
- `SELECT` on `sovereign.visitors (id, username, created_at)`, `sovereign.workspaces (id, visitor_id, created_at)` and `sovereign.tasks (id, workspace_id, agent_key, agent_version, input, status, cancel_requested, output, output_truncated, truncation_reason, error_code, created_at, started_at, finished_at)` — **no** `policy_id`, `lease_token`, `lease_expires_at`, `not_before`, `last_refusal`, `attempts`, `idempotency_key`, `input_token_estimate`, `max_output_tokens`, `model` or token counts. RLS restricts rows to the visitor's own.
- `EXECUTE` on its six functions; `USAGE` on both schemas. Nothing else.

Privileges of `sovereign_worker`: `EXECUTE` on its seven functions (`get_worker_config`, `claim_task`, `reserve_attempt`, `authorize_send`, `complete_task`, `retry_or_fail`, `sweep_expired_leases`); `USAGE` on `private`. **No table privileges.**

`EXECUTE` on every function is first revoked from `public`, `anon`, `authenticated` and `service_role`. The immutability trigger function (on `agent_policies` and `policy_models`) is owned by `postgres` (it must act on operator writes) and is executable by nobody else. Whether Supabase's default privileges add grants on new objects, and whether `pg_catalog.pg_advisory_xact_lock(int, int)` is executable by `sovereign_owner` on the hosted project, are checked by the catalog tests (§12.2), not assumed.

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
| Supabase JS clients | `apps/web/src/lib/supabase/**` | Removed from the visitor path. |
| Proxy | `apps/web/src/proxy.ts` | **Rewritten** for one purpose only: generate a per-request CSP nonce and set the security headers (§10 A, D69). No session, auth or Supabase logic. |
| Canonical origin | `apps/web/src/lib/site-origin.ts` | Kept for the Origin check. |
| Workspace page | `apps/web/src/app/(app)/workspace/page.tsx` | Rewritten: Task list, form and view. |
| Composition roots | `apps/web/src/server/composition.ts`, `apps/worker/src/composition.ts`, `apps/worker/src/main.ts` | Gain database adapters; the worker gains the loop, the configuration gate and the AI adapter. |
| DB configuration | `packages/db/src/config.ts` (`parseWorkerDatabaseUrl` rejects `postgres`, `supabase_admin`, `service_role`, `authenticator`) | Reused for the worker; same pattern for the web URL requiring `sovereign_app`. |
| Core | `packages/core/src` (health only) | Gains the agent catalogue and acceptance metadata, the canonical prompt document and hash, Task schemas, `ModelProvider` port, error taxonomy, usage mapping, token estimate, provider-identity normalisation. |
| Demo data | `apps/web/src/demo/**` | Stays for public pages until replaced; never shown as real Task results. |
| Migrations | `packages/db/supabase/migrations/20261112120000_m2_schemas_privileges.sql`, `20261112120100_m2_identity.sql` | Depend on `auth.users`/`auth.uid()`. **Replaced**, not amended (requires approval). |
| pgTAP and runner | `packages/db/supabase/tests/**`, `scripts/run-sql-tests.mjs`, `scripts/tap.mjs` | Suite files replaced (§12.2); control, manifest rules and runner kept. |
| Integration tests | `apps/web/test/integration/**` | Replaced (§12.3); `support/diagnostics.ts` reusable. |
| Guards | `apps/web/test/service-role-isolation.test.ts`, `scripts/check-client-bundle.mjs`, ESLint rules | Extended: database URLs, `AI_API_KEY`, agent instructions and wrapper text never in client code or bundles. |
| Environment template | `.env.example` | Later gains `AI_API_KEY`, `AI_BASE_URL`, `AI_MODEL`, `AI_FALLBACK_MODELS`, `AI_ACCOUNT_REF`, `AI_KEY_REF` with placeholder values only. |
| Dependencies | `@supabase/ssr`, `@supabase/supabase-js` | Unused. New: a PostgreSQL driver, `undici` (for an explicit HTTP dispatcher, §5.14), possibly a provider SDK (or plain `fetch`), a markdown sanitiser, and an RFC 8785 canonical-JSON implementation (or a small reviewed in-house one) — each needs approval (§11). |

### 8.3 New modules (proposed, not created)
- `apps/web/src/server/visitor.ts` (`server-only`): cookie, token, hashing, `requireVisitor`, `withVisitor`, CSRF checks, per-session read limiter.
- `apps/web/src/app/(app)/workspace/actions.ts`: `provisionVisitor`, `refreshSession`, `submitTask`, `getTask` (explicit column list), `listTasks`, `cancelTask`, `startNewWorkspace`, `forgetThisBrowser`.
- `apps/web/src/proxy.ts`: per-request CSP nonce and security headers (D69). Pages that need the nonce render dynamically; the landing page's rendering cost and its WebGL/animation scripts are checked under the CSP before launch (§10 A). The exact Next.js 16 mechanism is taken from the bundled Next.js documentation on Content Security Policy at implementation time.
- `packages/core`: agent catalogue and acceptance metadata (including timeout profile and model route), canonical prompt document and `prompt_hash`, deterministic request builder, Zod schemas, `ModelProvider` port, error taxonomy, per-model usage mapping, token estimate.
- `packages/db`: `app-database.ts` (`sovereign_app`) and `worker-database.ts` (`sovereign_worker`, function calls only).
- `apps/worker`: configuration gate, claim loop, sweep, AI adapter reading `AI_*` with an explicit `undici` dispatcher, monotonic send deadline and wall-clock watchdog, alert events, sanitised logging.

---

## 9. Operations during the beta

### 9.1 Visibility (minimum)
- Structured logs from web and worker: Task id, workspace id, agent key/version, policy id, transitions, attempt number, selected and reported model, route role, fallback reason, sent/unsent, duration, token counts, reservation outcome, `error_class`, `error_code`, and `alert` events (§5.7). No secrets, lease tokens, Task text or outputs.
- Operator SQL (documented in the runbook): queue depth and oldest queued Task; running Tasks (standard and slow) and lease ages; failure rate by `error_code`; attempts by selected model, route role and fallback reason; fallback balance between the two fallback models; models in cooldown; unsent attempts per Task; tokens and estimated cost per model; Tasks waiting with `last_refusal` = `config_mismatch` or `models_unavailable`; today's `daily_usage` (submitted, reserved, used) against ceilings; open reservations; bound violations (charged > reserved); provisioning counts; enabled `agent_policies` versus the active provider/account/model; recorded spend guard.
- **Worker liveness:** at launch, an operator check of the oldest `queued` age and the latest `started_at`; an automated alert before increasing traffic (§10).

### 9.2 Operator controls and procedures
- **Pause execution:** `update private.platform_settings set execution_enabled = false` (paused). **Resume:** set it to `true`. Pausing stops new submissions, claims, reservations and send authorisations; it does not stop requests already authorised or in flight (up to 20 minutes for slow requests) and does not guarantee zero further charges (§6.3).
- **Emergency provider controls** (when spending must stop as fast as possible or the worker may be compromised): pause execution first; then, in the provider console, lower or exhaust the provider-side spend limit **and** revoke `AI_API_KEY`. These are mitigations, not a guaranteed stop: their effect delay and whether they end requests the provider has already accepted are verification item 16 (§5.11). After revocation, a new key requires the full key rotation procedure.
- **Ceilings:** update `private.platform_settings`.
- **Enabling an agent version:** insert a policy row and its `policy_models` rows **only** after its acceptance package and per-model evaluations (with `prompt_hash`, timeout profile, provider, account and route) are approved, with alias acceptance references where needed; then set `enabled = true`. Never edit approved fields; insert a new policy instead (the trigger enforces this).
- **Disabling an agent version:** set `enabled = false`; Tasks bound to it fail at their next reservation or send authorisation.
- **Changing provider, account, key or model route:** follow the rollout procedure in §5.11.
- **Clearing a model cooldown early:** set `unavailable_until = now()` on its `model_health` row (logged).
- **Key rotation:** follow the key rotation procedure in §5.11 on **every** rotation, including confirming the spend limit for the new key before activating it.
- **Session revocation:** §4.6.
- Operator SQL runs as `postgres` — operator access, not application runtime. Every policy insert, enable/disable, pause/resume, spend-guard change, cooldown change and key rotation is recorded in the operator log with time, actor and evidence reference.

### 9.3 Cleanup (manual during the beta)
- Abandoned visitors: delete visitors older than 1 day with **no session where `least(revoked_at, expires_at) > now() - interval '30 days'`**. A visitor with a valid session cannot match, and no code path creates a session for an existing visitor. Cascades remove workspace, sessions, Tasks and reservations.
- Old sessions: delete those with `least(revoked_at, expires_at) < now() - interval '30 days'`.
- `provision_counters` older than 2 days. `daily_usage` and `agent_policies` are kept.
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
- [ ] **Security headers and Content Security Policy** (consistent with SECURITY §I.7, `default-src 'self'`; D69): a nonce-based CSP issued per request by `proxy.ts` — `script-src 'self' 'nonce-…'` with no `'unsafe-inline'` or third-party script sources unless the application demonstrably needs one, `object-src 'none'`, `base-uri 'none'`, `frame-ancestors 'none'`, `form-action 'self'`; `Strict-Transport-Security` in production; `X-Content-Type-Options: nosniff`; a restrictive `Referrer-Policy`; a minimal `Permissions-Policy`. Verified on the deployed site, including the landing page's WebGL and animation code. Reason: the session cookie is a bearer credential; HttpOnly stops scripts reading it, but injected script running in the page can still act as the visitor (§4.8), so preventing script injection is the remaining defence.
- [ ] Roles, grants (including column grants), RLS, the policy immutability trigger and functions of §7 applied; catalog tests pass against the hosted development project.
- [ ] Visitor A/B isolation tests pass for visitors, workspaces and Tasks, including cancel, spoofed ids and lease-field invisibility.
- [ ] No database credential, `AI_API_KEY`, agent system instruction or wrapper text in browser bundles (`check:bundle` on a fresh production build) or in the repository.
- [ ] Web server uses only `sovereign_app`; worker uses only `sovereign_worker`; neither uses `postgres` or `service_role`.
AI provider and agents:
- [ ] Supplied key rotated; new key stored only in the worker's deployment secrets.
- [ ] Provider verification of §5.11 completed and recorded (items 1–18), including usage mapping per model, billed-error behaviour, the model-unavailable signal, hidden-token accounting, the input-token bound for every model, model pinning or alias status of all three models, post-disconnect billing, emergency-control delay, and long-request support.
- [ ] **Valid spend guard recorded for the active account and key** (`provider_limit_confirmed`, or `risk_accepted` with the client's documented decision, D45, D68), with evidence reference.
- [ ] Every enabled agent version has an approved acceptance package with a passing evaluation record **for every model in its route** (prompt hash, provider, account, model, timeout profile) and a matching enabled, immutable policy with its `policy_models` rows (§5.2); alias acceptance recorded for every `alias_accepted` model.
- [ ] Round-robin counter row created for the active fallback set; routing verified with the fake provider (primary, fallback alternation, cooldown, both fallbacks unavailable).
- [ ] Timeout profiles: the explicit `undici` dispatcher configured (§5.14); each enabled profile verified end to end on the chosen host and provider at its full duration. **No slow (1,200 s) policy is enabled until this passes.**
- [ ] `platform_settings.active_*` match the worker's configuration; the worker refuses to claim otherwise.
Agent reliability:
- [ ] Task lifecycle works end to end against the verified provider: submit, idempotent resubmission, claim, reservation, send authorisation, success, every error class, fallback selection, deadline abort and refusal to start late, lease recovery, stale-worker rejection, cancel (queued and running), queue timeout, Task deadline, capacity refusal, pausing at reservation and at send, prompt-hash mismatch, disabled policy (at reservation and at send), configuration rollout.
- [ ] Output rendered through the restrictive allowlist, including storage-truncated output; error messages generic.
Cost and abuse:
- [ ] All §6.5 limits configured with client-approved values, including read limits.
- [ ] Token reservation verified under concurrent claims (no reservation granted beyond the daily ceiling).
- [ ] Single worker instance deployed; global running cap set.
- [ ] Decision on per-network limiting for the first exposure (D14).
Operations and data:
- [ ] Logs and alert events reachable; operator SQL and worker liveness check available; operator procedures (pause/resume, emergency provider controls, ceilings, policies, cooldowns, rollout, key rotation, session revocation) written.
- [ ] Privacy notice, privacy statement and Task retention decided and implemented (D35); backup position confirmed or accepted (D36).
- [ ] Hosted prerequisites P1–P7 verified (§11).
- [ ] Hosted verification runbook rewritten for this design and executed gate by gate with recorded evidence. **A build that compiles and local tests that pass are not sufficient.**

### B. Required before increasing traffic
- [ ] Trusted client-IP source verified and per-network limits enabled (D14).
- [ ] Automated notification for alert events: worker not claiming, configuration mismatch, models unavailable, sustained fallback use, threshold classes, immediate classes, usage near ceilings, open reservations older than the lease, bound violations, model drift.
- [ ] Load check at the planned global running cap and the provider's rate limits.
- [ ] Scheduled cleanup (D11).
- [ ] Cost per Task reviewed against ceilings; values adjusted.
- [ ] Automated browser tests (E2E tool needs dependency approval).

### C. Can wait until feedback validates the product
Session rotation and reuse detection; username editing and uniqueness; agent tools and their safety controls; Realtime; pgmq; monetary budgets; rolling-window token limits; visitor-created agents, Convergence, Guilds, Chronicle, public profiles; synchronizer CSRF token; disabling the Data API; restore rehearsal; operator UI; more than one worker instance; shared (cross-instance) read limits; lease heartbeats for faster slow-Task recovery; streaming; per-model output caps or prices in the reservation; automatic cooldown tuning.

---

## 11. External prerequisites
| # | Prerequisite | Why necessary | Blocks first beta? | Simpler supported alternative | Must verify before relying on it |
|---|---|---|---|---|---|
| P1 | Custom login roles `sovereign_app` and `sovereign_worker` can connect (pooler or direct) | Avoids `postgres`/`service_role` at runtime | **Yes** | Direct connection instead of the pooler | Test connection as each role; pooler user-name format |
| P2 | `postgres` can create `NOLOGIN` `sovereign_owner`, grant it column-level privileges, transfer function ownership, and create the immutability trigger | Limits what function bodies can reach; protects approved policies | **Yes**, unless the weaker fallback is approved | `postgres`-owned functions — **needs explicit approval** (D15) | Role creation, column grants, `ALTER FUNCTION … OWNER TO` and trigger behaviour on the development project |
| P3 | Transaction-local settings and two-key advisory transaction locks behave correctly through the chosen connection mode | Visitor context must not leak; locks must serialise | **Yes** | Session-mode pooler or direct connection | Pool-isolation and concurrency tests (§12.3) |
| P4 | A PostgreSQL driver for Node and `undici` (new dependencies) | Web and worker talk SQL; long provider requests need an explicit HTTP dispatcher (§5.14) | **Yes** | — | Dependency approval; prepared-statement behaviour in the chosen mode; undici timeout defaults for the pinned Node version |
| P5 | Hosting for the web app and **one** long-running, non-suspending worker process, with secret storage and outbound connections that can stay open for the longest enabled request timeout | Tasks outlive request handlers; secrets stay server-side; deadlines; 300 s / 1,200 s requests | **Yes** | Worker as a second process on the web host | Host chosen (D37); worker runs continuously; outbound idle/NAT and request-duration limits; secrets not exposed in logs or builds |
| P6 | AI provider verified (§5.11 items 1–18), key rotated, all three models confirmed, spend guard recorded | Agents need working, measurable models; spending must be capped even if the worker is compromised | **Yes** | — | Every item in §5.11's list |
| P7 | Markdown sanitiser and canonical-JSON implementation (dependencies or small reviewed in-house code) | Safe output display; deterministic prompt hash | **Yes** | In-house code limited to the allowlist / RFC 8785 subset needed | Dependency approval or review; tests (§12.1) |
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
- **Prompt hash:** the canonical document and hash are deterministic across runs and platforms (fixed fixtures with known hashes); changing one character of the instructions, the wrapper before or after the Task text, the message layout, a fixed parameter or the tool list changes the hash; changing the Task text, `max_output_tokens`, provider, account, model or key does not; CRLF and LF sources hash identically; no other normalisation occurs.
- Agent catalogue: every enabled agent has a complete acceptance package; the computed hash equals the approved fixture; the browser-facing catalogue serialiser never includes instructions or wrapper text.
- Provider identity normalisation: case, default port, trailing slash, rejection of credentials, query and fragment.
- `ModelProvider` adapter against recorded fixtures: request shape, auth header from `AI_API_KEY`, base URL from `AI_BASE_URL`, model from the attempt's `selected_model` (never chosen by the adapter); **usage mapping** per model fixture (subset-reported cached and reasoning tokens not double-counted; separately reported ones counted once; unknown or missing fields → unusable usage); `reportedModel` extracted; each `error_class` produced only from its defined condition (no `unbilled_*` class without a configured billing contract); `timedOut` set exactly for deadline aborts and transport/provider timeouts.
- **Deadline:** with fake clocks, `send_deadline = t0 + remaining_ms` where `t0` is taken before the `authorize_send` call; simulated RPC delay shortens, never extends, the window; a simulated pause after authorisation (monotonic advance, or wall-clock-only advance simulating suspend) makes the adapter refuse to send (`not_sent_transient`, unsent); abort happens at the monotonic deadline, and the watchdog aborts when only wall-clock time has advanced past `remaining_ms`; the worker never compares its clock with the database clock.
- **Deterministic request construction:** for fixed inputs the adapter's serialised wire request is byte-identical across runs; it contains exactly the prompt document's content, the Task text, the selected model, `max_output_tokens` and the explicit parameters — no extra messages, text or parameters; changing the prompt document changes the request only through the hashed fields.
- **HTTP dispatcher:** the adapter uses the configured dispatcher whose header and body timeouts exceed the largest request timeout; a fake server that responds after 1,200 s minus a margin is not cut off by the client before the adapter's own deadline (run with a scaled clock or as an integration test).
- **Routing helper:** fallback reason and route role derived correctly from the previous sent attempt's class and cooldown state; non-eligible classes never select a fallback.
- Worker configuration gate: mismatch or missing spend guard → no claim, alert event, key never printed.
- Alert events: each `error_class` emits its defined alert level; thresholds counted in a sliding window; alerts contain no Task text, output, keys or lease tokens.
- Read limiter: burst and sustained limits per session; `slow_down` doubles the client interval; polling never calls Task-limit paths.
- Allowlist sanitiser: raw HTML, images, iframes, scripts, event attributes, `javascript:`/`data:`/`http:` links all removed; allowed elements kept; links get `rel="nofollow noopener noreferrer"`; truncated output sanitised identically.
- Client consistency: a workspace-id mismatch in any action response triggers a reload; provisioning success always reloads.
- `getTask`/`listTasks` projections never include lease or policy fields.
- Logs never contain tokens, hashes, lease tokens, cookie values, Task text, outputs, IPs or the AI key.
- Static guards: no session-level `SET`; no database URL, AI key, instructions or wrapper text in client code.

### 12.2 Database / RLS (pgTAP via `pnpm test:rls`, control file first, all rolled back)
- Session states and boundaries; renewal; revocation.
- Provisioning: one visitor, workspace and session; atomic global and per-network counters at their limits; `null` network subject skips only the per-network limit; a `net_subject` of 1, 31 or 33 bytes (including `\x00`) is rejected; global and network counter rows can never coincide.
- `submit_task`: idempotent lookup **before** every other check — an existing Task is returned while execution is paused, while the spend guard is missing, while its agent is unavailable, at the active limit, at the daily limit and at the global ceiling, with no counter change; different payload → `idempotency_conflict`; `daily_usage` row created when missing; the global conditional increment never exceeds the ceiling; the Task binds the single enabled matching `policy_id` and copies its `max_output_tokens`; no enabled matching policy → `agent_unavailable`; missing spend guard → `paused`; no context → `no_visitor`.
- **Daily-limit rule:** cancelled and failed Tasks created today count toward the per-visitor daily limit; `existing` results do not.
- **Policy immutability:** updating any approved field of `agent_policies` (as `postgres`) raises, including `primary_model`, `fallback_models` and `request_timeout_seconds`; updating or deleting a `policy_models` row raises; deleting a policy raises; enabling a policy without a `policy_models` row for every route model raises; an `alias_accepted` row without `alias_acceptance_ref` is rejected; toggling `enabled` succeeds; a second enabled row for the same agent version, provider, account and route is rejected by the partial unique index; a new row with a new `evaluation_ref` can be inserted for changed content.
- Task immutability: no function can change `agent_key`, `agent_version`, `policy_id`, `input`, `input_token_estimate`, `max_output_tokens`, `request_timeout_seconds` or `deadline_at` (column privileges).
- `cancel_task`: queued → cancelled; running → `cancel_requested`; terminal → `not_cancellable`; another visitor's Task → `not_found`, unchanged.
- `claim_task`: claims nothing while paused; respects the spend guard, the global and slow running caps (a slow Task waits while a standard Task behind it is claimed) and `cancel_requested`; sets the lease to `request_timeout_seconds + lease_overhead_seconds`; returns `policy_id`, `max_output_tokens` and `request_timeout_seconds`.
- `reserve_attempt`: every outcome — `paused` (execution paused; missing spend guard; spend guard for a different account; spend guard for a different key reference), `policy_unavailable` for a Task queued **before** its policy was disabled, `config_mismatch` for a differing provider, account, key reference, model route or `prompt_hash` (prompt drift and wrapper drift fixtures) with requeue, backoff and `last_refusal`, `models_unavailable`, `lease_too_short`, `deadline_exceeded`, `attempts_exhausted` (→ `failed` with `last_error_code`), `refused_capacity` (no attempt counted, counter not advanced), and exposure computed from the stored `max_output_tokens`.
- **Round robin:** with the primary in cooldown, consecutive reservations alternate between the two fallbacks starting from the counter's current index; concurrent reservations from several sessions receive distinct consecutive `rr_index` values and the counts per fallback differ by at most one; a rolled-back reservation does not advance the counter; one fallback in cooldown → the other is selected; both in cooldown with the primary eligible → primary with `fallbacks_unavailable`; all three in cooldown → `models_unavailable`; a Task whose last sent attempt failed with `unbilled_retryable`, `model_unavailable` or `billing_unknown_retryable` gets a fallback, and one that failed with any other class does not; `model_unavailable` sets the cooldown.
- `authorize_send`: wrong lease or attempt → `stale_lease`, no change; execution paused after reservation → reservation released (unsent), Task queued, `paused`; **policy disabled after reservation → released (unsent), Task `failed` (`agent_unavailable`)**; cancel requested → released, `cancelled`; remaining time below the minimum budget → released, queued; otherwise `send` with `remaining_ms` ≤ `request_timeout_seconds` × 1000 and ≤ time to `lease_expires_at − call_margin_seconds`.
- **Attempt accounting:** releases in `authorize_send` and `not_sent_*` results do not increment `attempts`, do not set `last_error_code` and do not reset `queued_at`; a Task whose last sent attempt failed with `rate_limited` and whose later attempt was released unsent fails at exhaustion with `rate_limited`.
- `complete_task` / `retry_or_fail`: a wrong lease or attempt number returns `stale_lease` and changes neither the Task nor any reservation; the matching reservation is settled or released exactly once per the §5.7 table for every `error_class` (eight classes); per-attempt `sent`, outcome, `error_class`, reported model and usage recorded; `timed_out` selects `timeout` versus `provider_unavailable` for `billing_unknown_retryable`; output longer than 40,000 characters is truncated with `storage_limit` and the Task succeeds; `model_drift` is true only for a reported model outside `allowed_reported_models`; `cancel_requested` produces `cancelled`.
- `sweep_expired_leases`: settles only the reservation of the expired lease (sent, `expired`); cancelled / queued / failed transitions; requeue `not_before` at least `call_margin_seconds` ahead; queue timeouts measured from `queued_at` with `capacity`, `agent_unavailable` (after `config_mismatch`), `provider_unavailable` (after `models_unavailable`) and `queue_timeout`; `deadline_exceeded` at `deadline_at`; a Task cycling through refusals fails within the queue timeout.
- Constraints: queued with `cancel_requested` rejected; lease/output/error/finished/truncation consistency; attempts ≤ 3; one open reservation per Task; reservations carry lease, attempt and selected model; a fallback row has `rr_index`; settled rows are never unsent; spend-guard columns (including key reference and evidence) all set or all null; `slow_running_task_limit ≤ global_running_task_limit`; `request_timeout_seconds ∈ {300, 1200}`.
- Isolation as `sovereign_app`: no access to other visitors' rows; spoofed context sees nothing; no direct writes; selecting `lease_token`, `policy_id` or any non-granted `tasks` column fails with a permission error.
- As `sovereign_worker`: no table privileges at all; only its seven functions.
- Catalog: role attributes; function owner, `prosecdef`, `search_path=""`, no `EXECUTE`/`format(` in bodies, exact `EXECUTE` grants; exact table **and column** privileges per role; RLS on every table; exact policy set; the immutability trigger present on `agent_policies`; two-key advisory-lock usage.

### 12.3 Integration (hosted development project; fake AI provider by default)
- Pool isolation: one-connection pool, A's context then no context sees nothing; interleaved A/B transactions never cross.
- **Concurrent duplicate submissions** with one idempotency key → one Task, one counter increment.
- **Concurrent submissions** from one visitor never exceed the active or daily limit; from many visitors never exceed the global daily ceiling.
- **Concurrent provisioning** from two simulated tabs → two visitors, each with one workspace; the provisioning counter counts both.
- **Token reservation under concurrency:** several workers reserving at the edge of the ceiling never obtain a reservation that makes `used + reserved` exceed it.
- **Global running cap** with two worker instances: never more running Tasks than the cap.
- **Stale worker:** worker A's call is held past its lease; the sweep requeues; worker B claims and reserves attempt 2; A then calls `authorize_send`, `complete_task` and `retry_or_fail` with its old lease and attempt — all return `stale_lease`; B's reservation stays open and B's completion settles it normally; daily totals equal A's full settlement plus B's charge.
- **Pause race:** execution paused between claim and reservation → `paused`, no provider call; between reservation and `authorize_send` → `paused`, reservation released unsent, no provider call; after `authorize_send` returned `send` → that one request proceeds and settles (documented residual window).
- **Routing under concurrency:** two worker instances, primary forced into cooldown by the fake provider, many Tasks → each attempt calls exactly one model, fallbacks alternate across Tasks with balanced counts, the per-attempt records show requested, selected and reported models, route role, reason and `rr_index`, and no Task exceeds `max_attempts` sent attempts.
- **Timeout profiles:** a standard Task and a slow Task against a fake provider that answers just inside each timeout → both succeed and neither lease is reclaimed; a fake provider that never answers → aborted at the deadline, before the lease expires; a slow Task waits while the slow cap is reached; a Task whose retries cannot fit before `deadline_at` fails `deadline_exceeded`.
- **Deadline, clock skew and pauses:** worker wall clock set minutes ahead of and behind the database → deadlines unaffected; artificial RPC latency on `authorize_send` → usable window shrinks accordingly; worker paused between `authorize_send` and sending past the deadline → request not sent, `not_sent_transient`; a fake provider that never answers is aborted at the deadline, before the lease expires; the next attempt is not claimable until the old lease has expired and `not_before` has passed (no overlap from this application).
- **Prompt and wrapper drift:** a worker whose catalogue instructions or wrapper differ from the approved policy → `config_mismatch`, requeue, alert, no provider request; after the queue timeout → `failed` (`agent_unavailable`).
- **Disabled policy:** Tasks queued before disabling fail at their next reservation with `agent_unavailable`; no provider request.
- **Rollout order:** (a) settings changed before the worker restarts and (b) worker restarted before settings change → in both orders the worker stops claiming, no Task fails during the window, already-claimed Tasks are requeued with bounded backoff; after alignment Tasks run; Tasks bound to a superseded policy fail at queue timeout, not in a loop.
- **Spend guard and key rotation:** changing `AI_KEY_REF` (same account) or `AI_ACCOUNT_REF` without recording a spend guard for the new key → worker does not claim and submissions return `paused`; recording it resumes execution.
- **Model drift:** fake provider reporting a model id outside the allowed set → Task completes, `model_drift` alert raised, reported model stored; a reported id inside the allowed set of an alias → no alert.
- **Disabled policy at send:** policy disabled between reservation and `authorize_send` → no provider request, reservation released unsent, Task `failed` (`agent_unavailable`).
- Lifecycle with the fake provider for every `error_class` (settlement, outcome, error code and alert per §5.7); oversized output; success without usage; over-reported usage logged as a bound violation and later reservations refused.
- **Cancellation races:** cancel during the call; cancel at the same moment as the sweep processes an expired lease (both orders) → `cancelled`, never claimed again.
- **UTC boundary:** a reservation granted before 00:00 UTC settles against its own day; the new day starts from zero.
- **Read limits:** sustained polling above the limit receives `slow_down`; Task execution and Task limits unaffected.
- One opt-in live smoke test against the **verified** provider with a tiny prompt (separate approval; rotated key).
- Test-data cleanup per §12.5.

### 12.4 Browser (manual until an E2E tool is approved)
- Fresh profile → workspace without any form; submit a Task; close the tab; return → result shown.
- Two tabs opened at once in a fresh profile → both end on the same workspace after reload; neither keeps showing a different one.
- Reload and browser restart keep the workspace.
- Delete or tamper with the cookie → notice; "Start a new workspace" works.
- Two profiles see only their own Tasks; a copied Task URL shows nothing to the other profile; no response contains lease or policy fields.
- A cross-site page posting to a Server Action changes nothing.
- Response headers include the CSP and other security headers; an injected inline script in test content does not execute.
- Limits reached → generic messages; execution paused → "Agents are paused"; worker stopped → "Taking longer than expected"; truncated output shows the cut-short notice; hidden tab stops polling.

### 12.5 Test-data cleanup (development project only)
Integration tests record every visitor they create and use the username prefix `zz-test-`. The harness requires `SOVEREIGN_TEST_EXPECTED_PROJECT_REF` and a cleanup database URL (no defaults, no fallback). Before deleting it verifies that the URL names the expected project and that `private.test_environment` (one row, created only by a development setup script, never by a migration) holds `environment = 'development'` and the expected ref; any mismatch aborts with no `DELETE`. It deletes only recorded ids that carry the prefix, checks the count and verifies nothing related remains. No cleanup privilege exists in production.

### 12.6 Agent evaluation
Per policy and **per model in its route**: the acceptance package's examples run against the verified provider, account and model with the policy's timeout profile, scored with the agreed rubric; each record (prompt hash, provider identity, account reference, requested and reported model ids, never the key) is referenced by `policy_models.model_evaluation_ref`, and the package by the policy's `evaluation_ref` (§5.2). The reported ids observed here seed `allowed_reported_models`. Live and cost-incurring; each run needs explicit approval. Re-run whenever the prompt content, timeout profile, provider, account or any model in the route changes.

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
| D19 | Provisioning nonce carried through the proxy (identity flow; unrelated to the CSP nonce of D69) | REJECTED | Rec. | Replaced by reload-and-verify (§4.5) |
| D20 | Test-data cleanup safeguards | ACCEPTED | Rec. | §12.5 |
| D21 | Provisioning nonce | REJECTED | Rec. | Orphans accepted; inconsistent UI prevented by D41 |
| D22 | Session events | DEFERRED | Rec. | With rotation |
| D23 | Username editing | DEFERRED | Rec. | |
| D24 | Network HMAC subject | REVISED | Rec. | Exactly 32 bytes, disjoint scope from the global counter; only if D14 enables it |
| D25 | `sovereign.tasks` as single source of truth; five statuses | ACCEPTED | Rec. | |
| D26 | First-beta agent catalogue and instructions | **NEEDS CLIENT DECISION** | — | Acceptance packages (§5.2) |
| D27 | Tools decided per agent; default none | REVISED | Rec. (rule) / **NEEDS CLIENT DECISION** (per agent) | Depends on D26 |
| D28 | Worker claims via `SKIP LOCKED`; no pgmq; worker uses functions only | REVISED | Rec. | Closer to ADR-0003 |
| D29 | Polling for status | ACCEPTED | Rec. | With read limits (D60) |
| D30 | Limit values, token and Task ceilings | **NEEDS CLIENT DECISION** | — | Cost model + verified prices (P6) |
| D31 | Per-visitor limits enforced under a workspace lock | ACCEPTED | Rec. | Atomic under concurrency |
| D32 | AI provider and models | REVISED | **Client** (requirement: OAO; `gpt-6-astra` primary; `auto-vip`, `auto-regular` fallbacks) | **Provider compatibility unverified**; activation gated on §5.11 items 1–18; pinned ids preferred where they exist (D55) |
| D33 | One bounded call per attempt; `max_attempts` 1–3 (proposal 2); queue timeout | ACCEPTED | Rec. | Aligned with constraints |
| D34 | Restrictive markdown allowlist | ACCEPTED | Rec. | Implementation choice P7 |
| D35 | Task-input notice, privacy statement, Task retention | **NEEDS CLIENT DECISION** | — | **Launch blocker** |
| D36 | Backups | **NEEDS CLIENT DECISION** | — | Plan features or accept loss |
| D37 | Hosting; single worker instance | REVISED / **NEEDS CLIENT DECISION** (host) | Rec. (single instance) | Host choice |
| D38 | AI configuration via `AI_API_KEY`, `AI_BASE_URL`, `AI_MODEL`, `AI_ACCOUNT_REF`, worker-only; provider-neutral adapter; rollout and key-rotation procedures | REVISED | **Client** (configurability requested); design Rec. | Key must be rotated |
| D39 | Atomic per-attempt token reservation bound to lease and attempt; error-class settlement; conditional guarantees | ACCEPTED | Rec. | §6.3 |
| D40 | Database-enforced global running cap; single worker instance | ACCEPTED | Rec. | Does not bind a compromised worker (§2) |
| D41 | Post-provisioning reload and workspace-id consistency check | ACCEPTED | Rec. | §4.5 |
| D42 | Agent acceptance package required for every enabled policy, with evaluation bound to prompt hash, provider, account and model | REVISED | Rec. | §5.2 |
| D43 | Idempotency lookup before every other check | ACCEPTED | Rec. | §5.4 |
| D44 | Cancellation can never be undone by retry or recovery | ACCEPTED | Rec. | §5.3, §5.10 |
| D45 | Spend guard: provider-side spend limit on the active account, or documented client acceptance of the compromised-worker financial risk; required before execution | **NEEDS CLIENT DECISION** | — | §5.11 items 12, 14 |
| D46 | `private.agent_policies` as the database source of truth for runnable agent versions | REVISED | Rec. | Now immutable, keyed by `policy_id`, bound to Tasks |
| D47 | Fixed provider error taxonomy with deterministic settlement and retry | REVISED | Rec. | Eight classes: `model_unavailable` added; fallback-eligible classes defined (§5.7, §5.13) |
| D48 | UTC-day token accounting is the enforced bound; rolling 24-hour figure `2C + K·E` is an estimate valid only if billing stops at lease end | REVISED | Rec. | §6.3; depends on verification item 13; rolling window deferred |
| D49 | Storage-limit truncation in `complete_task` with `truncation_reason` | ACCEPTED | Rec. | §5.7 |
| D50 | Prompt hash (RFC 8785 canonical JSON, SHA-256) and immutable policies; approval is the policy row, not the version label | ACCEPTED (new) | Rec. | §5.2, §5.12 |
| D51 | `authorize_send` immediately before each request (pause, policy `enabled`, lease); pausing (`execution_enabled = false`) with stated residual window; provider limit and key revocation as emergency mitigations, not a guaranteed stop | REVISED | Rec. | §6.3, §9.2; effect delay is verification item 16 |
| D52 | Deadline as a database-computed remaining duration anchored on the worker's monotonic clock before the authorising call, with a wall-clock watchdog; host-suspend overlap accepted as residual risk | REVISED | Rec. | §5.6 |
| D53 | Configuration mismatch is transient (requeue, backoff, alert, bounded by queue timeout); disabled policy is permanent; worker gate stops claiming | ACCEPTED (new) | Rec. | §5.5, §5.11, §6.3 |
| D54 | Provider identity includes `AI_ACCOUNT_REF`; spend guard recorded per account; key-rotation procedure | ACCEPTED (new) | Rec. | Operator-attested unless the provider exposes account identity (§5.11 item 14) |
| D55 | Pinned model ids where available; `alias_accepted` only with a recorded client acceptance per model; drift alert only outside `allowed_reported_models` | REVISED / **NEEDS CLIENT DECISION** for each model that is an alias or router (likely `auto-vip`, `auto-regular`) | Rec. | §5.11; depends on item 17 |
| D56 | Per-provider usage mapping into input/output/other, each unit counted once; unknown fields → unusable usage | ACCEPTED (new) | Rec. | §5.11 |
| D57 | Alert level for every error class and operational event | REVISED | Rec. | §5.7; routing alerts added |
| D58 | Security headers and CSP as a launch gate | REVISED | Rec. | §10 A; mechanism fixed by D69 |
| D59 | Per-visitor daily limit counts every Task created that day in any status | ACCEPTED (new) | Rec. | §5.4 |
| D60 | Per-session, per-instance read limits for `getTask`/`listTasks`; polling backoff; separate from Task limits | ACCEPTED (new) | Rec. | §5.9 |
| D61 | Model route: primary `gpt-6-astra`, round-robin fallback between `auto-vip` and `auto-regular`; one model per attempt; route bound to the evaluated policy, every model evaluated | ACCEPTED (new) | **Client** (behaviour and model ids); design Rec. | §5.13; provider support unverified |
| D62 | Fallback-eligible classes (`unbilled_retryable`, `model_unavailable`, `billing_unknown_retryable`) and per-model cooldown | ACCEPTED (new) | Rec. | §5.13 |
| D63 | Round-robin state as an atomic database counter advanced in `reserve_attempt` | ACCEPTED (new) | Rec. | §5.13 |
| D64 | Request timeout profiles 300 s / 1,200 s bound to the policy | ACCEPTED (new) | **Client** (values); per-agent classification **NEEDS CLIENT DECISION** | §5.14; long-request support unverified |
| D65 | Explicit `undici` dispatcher with timeouts above the request timeout; adapter deadline controls | ACCEPTED (new) | Rec. | §5.14; dependency approval (P4) |
| D66 | Lease = request timeout + overhead (no heartbeat); slow running cap; queue timeout from `queued_at`; Task deadline `deadline_at` | ACCEPTED (new) | Rec. | §5.5, §5.14; recovery delay up to ~21 min for slow Tasks accepted |
| D67 | Only sent attempts count toward `max_attempts` and set the final error code; unsent attempts recorded as released | ACCEPTED (new) | Rec. | §5.6 |
| D68 | Spend-limit confirmation on every key rotation; non-secret `AI_KEY_REF`; mandatory evidence reference | ACCEPTED (new) | Rec. | §5.11 |
| D69 | CSP with a per-request nonce issued by `proxy.ts`; nonce-dependent pages rendered dynamically | ACCEPTED (new) | Rec. | §8.2, §8.3, §10 A |

---

## 14. Remaining client decisions and unknowns
| Item | Evidence or answer needed | From |
|---|---|---|
| First-beta agents, instructions, boundaries, examples, rubric (D26) | Approved acceptance packages | Client |
| Tools per agent (D27) | Whether any approved agent needs live data; a tool provider if so | Client |
| AI provider and models (D32, D55, D61, P6) | Provider documentation and a controlled test with a **rotated** key covering §5.11 items 1–18; whether all three model ids exist; pinned, alias or router; client acceptance for each alias | Client + verification |
| Slow-profile agents (D64) | Which agent versions use the 1,200 s profile | Client |
| Long-request support (D64, D65, P5) | Provider, host and network support for 300 s / 1,200 s non-streaming requests | Verification |
| Emergency control delay (D51) | How quickly a spend-limit change and key revocation take effect; effect on in-flight requests | Verification |
| Spend guard (D45) | Whether the provider offers a limit on the account; client's documented acceptance if not | Client + verification |
| Hard ceilings and limit values (D30) | Cost model: expected Tasks per day, tolerated daily spend, verified prices | Client |
| Task retention and privacy notice (D35) | Wording and retention period | Client |
| Hosting and worker topology (D37, P5) | Chosen host(s) | Client |
| Per-network limiting behind the real proxy (D14, P8) | Host documentation and spoofed-header test | Client + verification |
| Session lifetime value (D8) | Confirm or change 90 days | Client |
| Backups (D36, P9) | Plan features or explicit acceptance of loss | Client |
| Driver and connection mode (D16, P1–P4) | Dependency approval and hosted test results | Client / maintainer + verification |
| Sanitiser and canonical-JSON implementation (P7) | Dependency approval or in-house choice | Maintainer |
| Tokenizer bound, hidden-token accounting, usage mapping, post-disconnect billing (§6.3, §5.11) | Provider documentation and measured usage against estimates | Verification |
| Status of the old account-based work in Git (§8.1) | Read-only Git evidence | Client authorises a read-only Git check |

---

## 15. Documents that must change before implementation (not changed by this revision)
- A new ADR recording accountless visitors, the beta Task model, AI provider configuration and binding, model routing, timeout profiles, prompt hashing, and the simplifications above, superseding the affected parts of ADR-0001, ADR-0003, ADR-0005, ADR-0006 and ADR-0011.
- `docs/PRD.md` (accounts, verified-email gate, routes, beta scope).
- `docs/ARCHITECTURE.md` §F.1 (LLM provider), §F.2, §F.5, §F.7, §F.8, §F.9, §H.
- `docs/DATA-MODEL.md` §G (identity tables, Tasks, policies, roles). The G.8 "as implemented" line already misdescribes migration 1.
- `docs/SECURITY.md` §I (trust boundaries, privilege model, AI key handling, compromised-worker risk, security headers, verified-email gate removal).
- `docs/ROADMAP.md` (M2 scope and status, beta launch milestone).
- `.env.example` (placeholders for `AI_*` and the database URLs) — when implementation is approved.
- The M2 hosted-verification runbook (must be rewritten for this design; its previous revisions exist only in a working session, not as a file).
