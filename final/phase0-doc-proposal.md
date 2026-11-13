# Phase 0 — Proposed ADR and living-document changes (NOT APPLIED)

Status: proposal for client approval. Nothing below has been written to the repository.
Source of truth: `docs/identity/M2-SPEC.md` (Revision 8, accepted baseline).
Change rule (docs/README.md): ADR + affected living documents in the same commit + ROADMAP status.

Files that would be modified or created:
1. NEW `docs/DECISIONS/0012-accountless-beta-tasks-and-model-routing.md`
2. `docs/DECISIONS/README.md` (index row + status notes on 0001–0006, 0011)
3. `docs/PRD.md`
4. `docs/ARCHITECTURE.md`
5. `docs/DATA-MODEL.md`
6. `docs/SECURITY.md`
7. `docs/ROADMAP.md`

Note: `DATA-MODEL.md`, `SECURITY.md`, `ROADMAP.md` and `DECISIONS/README.md` already carry uncommitted
edits from the earlier account-based M2 work. These proposals are written against their current working-tree text.

Editing approach (minimal): the blueprint-derived living documents stay as the long-term MVP design.
Each affected section receives a clearly marked **"M2 beta (ADR-0012)"** note stating what is superseded
for the public beta, with a pointer to the exact M2-SPEC section. No blueprint text is deleted except the
one inaccurate "As implemented at M2" paragraph in DATA-MODEL §G.8 (spec §15 says it misdescribes migration 1).

---

## 1. NEW `docs/DECISIONS/0012-accountless-beta-tasks-and-model-routing.md`

```markdown
# 0012 — Accountless public beta: visitor sessions, single-agent Tasks, token reservation and OAO model routing
- Status: Proposed
- Date: 2026-11-13
- Baseline refs: §C.1, §C.4, §D.1, §D.3, §F.1, §F.2, §F.5, §F.7, §F.8, §F.9, §G.1, §G.5–§G.8, §H, §I; ADR-0001, ADR-0002, ADR-0003, ADR-0004, ADR-0005, ADR-0006, ADR-0011
- Specification: `docs/identity/M2-SPEC.md`, Revision 8 (accepted as the initial M2 baseline, 2026-11-13)

## Context
The client confirmed (2026-11-13) that Sovereign's first public beta (about 1–2 months) is a memecoin-related
website that shows a Contract Address as information only, and lets visitors try agents immediately. Visitors
never sign up, log in, verify an email, connect a wallet, sign messages or transact on-chain. The account-based
M2 work (Supabase Auth, verified-email gate, `public.profiles`/`workspaces`/`workspace_members`) does not fit
this product. The client also confirmed an AI provider (OAO, `https://oao.ai.id/v1`), a model route
(primary `gpt-6-astra`; round-robin fallback between `auto-vip` and `auto-regular`) and request timeouts
(300 s standard, 1,200 s slow). Provider compatibility is unverified.

## Decision
Adopt M2-SPEC Revision 8 as the architecture for the public beta. In summary:
1. **Identity (M2-SPEC §4):** accountless visitors with a server-generated id, a display-only generated username,
   one personal workspace per visitor, and a 256-bit HttpOnly bearer session cookie stored only as a SHA-256 hash.
   No Supabase Auth, no recovery across devices. CSRF: required `Origin` + `Sec-Fetch-Site` + `SameSite=Lax`.
2. **Database (§7):** schemas `sovereign` and `private`, neither exposed through the Data API; login roles
   `sovereign_app` (web) and `sovereign_worker` (worker) with no table writes; all writes through SECURITY DEFINER
   functions owned by `NOLOGIN` `sovereign_owner`; RLS keyed on a transaction-local session-token-hash context.
3. **Execution (§5):** single-agent Tasks in `sovereign.tasks` as the only source of truth; a worker claims with
   `SKIP LOCKED` under a global running cap (no pgmq in the beta); fenced leases; one bounded model call per attempt.
4. **Cost control (§6):** per-attempt token reservation against a per-UTC-day ceiling, bound to lease and attempt;
   per-visitor and global limits; a pause setting (`execution_enabled = false`); a provider-side spend guard per
   account and key as a launch condition. Workspace monetary budgets are deferred.
5. **Agents (§5.2, §5.12):** platform-defined agents with client-approved acceptance packages; immutable database
   policies binding prompt hash, output cap, timeout profile, provider, account and model route.
6. **Model routing and timeouts (§5.13, §5.14):** database-selected model per attempt with an atomic round-robin
   counter and per-model cooldown; timeout profiles bound to the policy; lease derived per Task.
7. **Status updates:** polling with per-session read limits instead of Realtime.
8. **Security headers (§10 A, D69):** a per-request CSP nonce issued by `proxy.ts`.

## Supersedes (for the public beta only; the records stay)
- ADR-0001: Supabase Auth and the Anthropic-first `ModelProvider` choice (the port remains; OAO adapter first).
- ADR-0002: pgmq delivery (replaced by `SKIP LOCKED` claim on `sovereign.tasks`).
- ADR-0003: the `worker` schema and `worker_api_owner` (functions live in `private`, owned by `sovereign_owner`).
- ADR-0004: the monetary budget ledger (replaced by token reservation; monetary budgets deferred).
- ADR-0005: Realtime Broadcast (replaced by polling).
- ADR-0006: account-based personal workspaces (replaced by one visitor workspace).
- ADR-0011: the Supabase Auth / service-role parts. The hosted development project, the psql pgTAP runner and
  its negative control remain in force.

## Consequences
- The two uncommitted migrations `20261112120000_m2_schemas_privileges.sql` and `20261112120100_m2_identity.sql`
  are replaced, subject to hosted-migration evidence and explicit approval.
- The Auth pages, actions, gate, Supabase clients and integration admin layer are removed in Phase 2.
- New dependencies (PostgreSQL driver, Zod, RFC 8785 canonical JSON, markdown sanitiser, `undici`) each need approval
  in the phase that first uses them.
- Pages that need the CSP nonce render dynamically.
- Known accepted risks are listed in M2-SPEC §5.6, §6.3 and the R8 baseline review (billing after abort,
  compromised worker, emergency-control delay, host suspend, slow-Task recovery delay).

## Verification
M2-SPEC §10 (launch checklist), §11 (prerequisites P1–P10), §12 (tests) and §5.11 items 1–18 (provider gates).
No item is verified by this ADR.
```

---

## 2. `docs/DECISIONS/README.md`
Add row:
`| [0012](0012-accountless-beta-tasks-and-model-routing.md) | Accountless public beta: visitor sessions, single-agent Tasks, token reservation, OAO model routing | Proposed |`
(changes to `Accepted (M2 beta)` only after client approval).

Append to the Status cell of 0001, 0002, 0003, 0004, 0005, 0006 and 0011: `; partly superseded for the public beta by 0012`.

---

## 3. `docs/PRD.md`
3a. Insert directly under the title block (before `## C. Product Specification (PRD)`):
```markdown
> **M2 public beta scope (ADR-0012; `docs/identity/M2-SPEC.md` R8).** For the public beta the product is
> accountless: visitors use agents immediately, without sign-up, login, email, wallet or on-chain actions.
> Each visitor gets a display-only generated username and one personal workspace that persists only in the same
> browser profile (no cross-device recovery). The Contract Address on the landing page is informational only.
> Visitors submit single-agent Tasks to platform-defined agents; Forge, Guilds, Convergence, public agent
> publishing, approvals and deliverable versioning are not part of the beta. Where a section below conflicts
> with this note, this note applies to the beta.
```
3b. C.1 item 7 (Billing), append after its last bullet:
`> M2 beta: no verified-email gate and no workspace budgets. Cost is bounded by a daily token ceiling, per-visitor and global limits, a pause setting and a provider-side spend guard (M2-SPEC §6, §5.11).`
3c. After the C.4 table:
`> M2 beta: rows "Auth & profiles", "Workspaces", "Usage & budget", "Quests & Decrees" and "Agent runtime" are replaced by M2-SPEC §4–§6 (accountless sessions, one visitor workspace, single-agent Tasks without tools, token ceilings). Other rows are not in the beta.`
3d. After the D.1 sitemap:
`> M2 beta: no /login, /signup, /reset-password or /auth/callback. The app area is /workspace (Task list, form, results), reached without signing in (M2-SPEC §4.5).`
3e. After D.3 journey 1:
`> M2 beta journey: open the site → /workspace provisions a visitor and workspace (POST, then reload) → choose an agent → submit a Task → see status and output; return later in the same browser (M2-SPEC §4.5, §5.9).`

---

## 4. `docs/ARCHITECTURE.md`
4a. After the F.1 table:
```markdown
> **M2 beta (ADR-0012).** For the public beta: Supabase is used as PostgreSQL only (Auth, Storage, Realtime and the
> Data API unused); background execution claims from `sovereign.tasks` with `SKIP LOCKED` (no pgmq); the LLM row
> is a provider-neutral `ModelProvider` with an OAO adapter first and a database-selected model route
> (`gpt-6-astra` primary, round-robin `auto-vip`/`auto-regular`), all unverified until M2-SPEC §5.11 items 1–18
> are recorded; no web-search tool. Hosting is an open client decision (D37). Zod and the PostgreSQL driver
> need dependency approval. See M2-SPEC §2, §5, §5.11, §5.13.
```
4b. Start of F.2: `> M2 beta diagram: M2-SPEC §2 (browser → Next.js as sovereign_app; one worker as sovereign_worker; AI provider via adapter).`
4c. Start of F.5: `> M2 beta: one bounded model call per attempt, no tools, no approvals; prompt content bound to an approved policy by prompt hash (M2-SPEC §5.6, §5.12).`
4d. Start of F.7: `> M2 beta: Realtime is not used; status is polled with per-session read limits (M2-SPEC §5.9).`
4e. Start of F.8: `> M2 beta: pgmq is not used; claim from sovereign.tasks with SKIP LOCKED, fenced leases, lease = request timeout + overhead (M2-SPEC §5.5, §5.14).`
4f. Start of F.9: `> M2 beta: per-attempt token reservation against a per-UTC-day ceiling (no micro-USD ledger, no workspace budgets) (M2-SPEC §6.3).`
4g. H.1, after "Route handlers in MVP": `> M2 beta: /auth/callback is removed; Server Actions: provisionVisitor, refreshSession, submitTask, getTask, listTasks, cancelTask, startNewWorkspace, forgetThisBrowser (M2-SPEC §8.3).`
4h. Start of H.3: `> M2 beta lifecycle: Task queued → running → succeeded | failed | cancelled (M2-SPEC §5.3).`

---

## 5. `docs/DATA-MODEL.md`
5a. Start of G.1: `> M2 beta tables: M2-SPEC §7.3 (sovereign.visitors, workspaces, visitor_sessions, tasks; private.agent_policies, policy_models, platform_settings, daily_usage, attempt_reservations, route_counters, model_health, provision_counters).`
5b. Start of G.5: `> M2 beta: schemas sovereign and private, neither exposed through the Data API; nothing in public; RLS keyed on the transaction-local session-token hash (M2-SPEC §7.1, §7.5, §7.6).`
5c. Start of G.6: `> M2 beta: sovereign_worker executes seven functions in private (owned by NOLOGIN sovereign_owner); the worker schema and worker_api_owner are not used (M2-SPEC §7.2, §7.4).`
5d. Start of G.7: `> M2 beta: one workspace per accountless visitor, created with the visitor in one transaction; no members table (M2-SPEC §4.5, §7.3).`
5e. G.8: replace the paragraph beginning `- **As implemented at M2** (ADR-0011): …` with:
`- **M2 beta (ADR-0012):** the two uncommitted Auth-based migrations (20261112120000, 20261112120100) are replaced by new migrations per M2-SPEC §7, subject to hosted-migration evidence and explicit approval. New migrations are added per implementation phase and are never edited after they are applied.`

---

## 6. `docs/SECURITY.md`
6a. After I.1 list:
`> M2 beta trust boundaries: M2-SPEC §2 — browser holds only an HttpOnly bearer session; the web server never holds AI_API_KEY; the worker holds AI_API_KEY and can bypass every database control if compromised; only a provider-side limit and key revocation bound spending in that case (emergency mitigations with unverified delay).`
6b. After I.5: `> M2 beta: AI_API_KEY, AI_BASE_URL, AI_MODEL, AI_FALLBACK_MODELS, AI_ACCOUNT_REF and AI_KEY_REF are worker-only; the previously supplied key is treated as exposed and must be rotated; spend-limit confirmation on every rotation (M2-SPEC §5.11). Supabase Auth is not used, so the service-role key has no application role in the beta.`
6c. I.7: replace bullet `- Security headers + CSP (default-src 'self'), CORS closed.` with:
`- Security headers + CSP (default-src 'self'), CORS closed. M2 beta: nonce-based script-src issued per request by proxy.ts, object-src 'none', base-uri 'none', frame-ancestors 'none', form-action 'self', HSTS, nosniff, Referrer-Policy, Permissions-Policy (M2-SPEC §10 A, D69).`
6d. After the I.9 table: `> M2 beta privilege model: M2-SPEC §7.2 and §7.4 (sovereign_app, sovereign_worker, NOLOGIN sovereign_owner; anon/authenticated/service_role have no privileges on sovereign/private).`

---

## 7. `docs/ROADMAP.md`
7a. J.2 M2 row: append to "Done when": ` — **Superseded for the public beta by ADR-0012:** M2 = accountless identity and Task execution per M2-SPEC R8, delivered in Phases 0–8; acceptance = M2-SPEC §10 A and §12.`
7b. Implementation status, replace the M2 Identity row status/notes with:
`| M2 (accountless beta, ADR-0012) | **Re-planned 2026-11-13; Phase 0 in progress** | Baseline: docs/identity/M2-SPEC.md R8 (accepted). Earlier Auth-based M2 work is uncommitted and superseded; its migrations await hosted evidence before replacement. Phases 1–8 not started. |`
7c. Known issues, add rows:
`| D30 limit values not approved | Settings use proposals only | Client decision; proposed: queue_timeout 25 min, task_max_age_standard 20 min, task_max_age_slow 75 min, slow cap 1, max_attempts 2, worker poll ≤ 5 s |`
`| Hosted migration state unknown | Old migrations may or may not exist on the dev project | Operator evidence checklist (Phase 0) |`
