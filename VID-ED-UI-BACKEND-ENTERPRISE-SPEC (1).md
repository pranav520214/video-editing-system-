# VID-ED — UI, BACKEND & ENTERPRISE SPECIFICATION
### Extends `docs/BUILD_BIBLE.md` and `docs/AGENT_PROMPTS.md`. Read those first — this document assumes their tech stack, repo structure, and Phase 0–8 roadmap as given and does not repeat them except where it changes something.

---

## 0. HOW THIS DOCUMENT FITS

The Build Bible specced a **single-user, local-first desktop app.** That doesn't change — it remains the product's core differentiator and stays true for every free/individual user. This document adds what's needed to sell to a *team* or *company* without breaking that promise. Concretely, it introduces exactly one new thing: a small **Control Plane** (a real backend service) that handles identity, teams, billing, shared brand assets, and audit — and **nothing about video**. Raw footage, transcripts, timelines, and rendered output never touch the Control Plane unless a team explicitly turns on Shared Project Storage (Section 2.1) for collaboration, and even then it's opt-in per-project, clearly labeled, and off by default.

**[ASSUMPTION]** No enterprise requirements were given beyond "enterprise grade," so this document targets the standard bar for B2B SaaS sold into mid-market and enterprise accounts: SSO, RBAC, audit logs, SOC 2 readiness, an admin console, and a self-hosted option for regulated customers. If actual target customers have specific requirements (HIPAA, FedRAMP, a specific SSO vendor), that changes prioritization within Section 3 — flag it, don't silently guess further.

---

# PART A — FRONTEND / UI SPECIFICATION

## A.1 Design Principles

1. **The agent panel is the front door, the timeline is the workshop.** A first-time user should be able to get a finished edit without ever opening the timeline. A power user should be able to ignore the agent panel entirely after the first pass and hand-edit everything. Neither mode is a second-class citizen.
2. **Every AI decision is inspectable, never a black box.** Anywhere the UI shows something an agent produced, there is a one-hover-away answer to "why." This is a hard requirement, not a stretch goal — see A.5, `ReasonTooltip`.
3. **Never block on AI.** Long-running agent/render operations always show real progress and remain cancellable; the UI never shows an unbounded spinner with no way out.
4. **Dense but calm.** This is a professional tool used for hours at a time, not a marketing site — favor information density and low visual noise (muted chrome, high-contrast content) over decorative flourish. Motion is used only to communicate state change (a clip settling into place, a progress bar), never for its own sake.

## A.2 Design Tokens

**[ASSUMPTION]** Exact brand identity (VID-ED's own product branding, as opposed to a *creator's* brand handled by Brand Agent) wasn't specified — tokens below are a professional dark-first creative-tool palette, standard for video/creative software, and should be treated as a starting point a designer can retheme without restructuring the token system itself.

```
Color (dark theme, default):
  --bg-app:         #0E0F12
  --bg-panel:       #17181C
  --bg-elevated:    #1F2126
  --bg-hover:       #262932
  --border-subtle:  #2A2D34
  --border-strong:  #3A3E48
  --text-primary:   #F2F3F5
  --text-secondary: #9A9DA6
  --text-disabled:  #5C5F68
  --accent-primary: #6C5CE7      /* agent/AI-related actions and highlights */
  --accent-render:  #2ECC71      /* render/export/success states */
  --accent-warn:    #F5A623
  --accent-error:   #E85D5D
  --track-video:    #3B82C4
  --track-audio:    #4FAE8C
  --track-caption:  #C48A3B
  --track-effect:   #A85CC4

Color (light theme):
  --bg-app:         #FAFAFA
  --bg-panel:       #FFFFFF
  --bg-elevated:    #FFFFFF
  --bg-hover:       #F0F1F3
  --border-subtle:  #E4E5E9
  --text-primary:   #16171A
  --text-secondary: #6B6E76
  (accent/track colors stay identical across themes for consistency in exported style guides)

Typography:
  --font-ui:        "Inter", system-ui, sans-serif        /* all chrome, labels, panels */
  --font-mono:       "JetBrains Mono", monospace           /* timecodes, JSON inspector, agent reasoning text */
  --text-xs:  11px / 16px
  --text-sm:  13px / 18px
  --text-base:14px / 20px
  --text-lg:  16px / 24px
  --text-xl:  20px / 28px
  --text-2xl: 26px / 34px

Spacing scale: 2, 4, 8, 12, 16, 24, 32, 48, 64 (px) — no arbitrary values outside this scale anywhere in the component library.

Radius: --radius-sm: 4px (buttons, inputs) · --radius-md: 8px (cards, panels) · --radius-lg: 12px (modals)

Elevation: three levels only — flat (panels), raised (dropdowns/popovers, subtle shadow), floating (modals, strong shadow + backdrop dim). Never more than three shadow depths in the whole app; more than that reads as visual noise in a dense editor UI.

Motion: --duration-fast: 100ms (hover/press feedback) · --duration-base: 200ms (panel open/close) · --duration-slow: 320ms (modal/page transitions). Easing: standard ease-out for entrances, ease-in for exits. Timeline playhead motion is NEVER eased — it must track real playback time linearly, or scrubbing will feel laggy/wrong.
```

## A.3 Screen Inventory

| Screen | Route (internal) | Purpose |
|---|---|---|
| Onboarding / First Run | `/onboarding` | Hardware detection results, model download progress, license/EULA acceptance (Build Bible §10) |
| Project Library | `/` | Grid/list of local projects, "New Project" entry point, recent activity |
| Editor | `/project/:id` | The main workspace — Preview, Timeline, Agent Panel, Inspector (see A.4) |
| Plan Review | modal/panel within Editor | Shown after Orchestrator returns a plan, before agents run |
| Export | modal within Editor | Format/platform selection, progress, output location |
| Brand Kit Manager | `/brand-kit` | View/edit stored creator_profile.brand; for teams, shows org-shared kits (Section 3.11) |
| Settings | `/settings` | Hardware tier override, model management, privacy toggles, cloud integration keys |
| Team / Org (enterprise only) | `/team` | Members, roles, shared assets, usage — thin client of the Control Plane, only present when signed into an org |
| Admin Console | separate web app, not in the desktop bundle | See Section 3.2 |

## A.4 Editor Screen — Detailed Layout

```
┌───────────────────────────────────────────────────────────────────────┐
│ Top Bar: Project name · Undo/Redo · Autosave indicator · Export button│
├───────────────┬───────────────────────────────────┬───────────────────┤
│               │                                     │                 │
│  Agent Panel  │           Preview Player            │    Inspector    │
│  (collapsible,│   (video canvas, before/after       │  (selected clip/│
│   left dock)  │    toggle, transport controls)      │  effect fields, │
│               │                                     │  ReasonTooltip  │
│               │                                     │  source)        │
├───────────────┴─────────────────────────────────────┴─────────────────┤
│  Timeline (tracks: video / audio / caption / effect / overlay)         │
│  Waveform rendering on audio tracks · playhead · zoom/scrub controls   │
└──────────────────────────────────────────────────────────────────────┘
```

- **Agent Panel** default state: a single text input ("Describe what you want") + Creator Profile chips (quick-apply stored style presets) + a scrollable history of past plans run on this project. Collapsible to a thin icon rail for users who want maximum timeline space.
- **Preview Player**: scrubbing here and scrubbing the Timeline playhead are the same state, always in sync — never two independent playheads.
- **Inspector**: context-sensitive. Nothing selected → project-level metadata (resolution, duration, platform target). A clip selected → its transform/effects with real controls (sliders, color pickers, dropdowns bound to the fixed LUT list from Color Agent, etc.) bound through `vided-core` mutation functions per Build Bible Phase 5 — never a raw JSON textarea in the shipped product (a raw-JSON "advanced" toggle is fine for power users, but it's opt-in, not default).
- **Timeline**: tracks are color-coded per A.2's track colors, matching the track `type` field from `timeline.json`. Clips show a thin colored underline whose color encodes which agent produced them (a small legend in the timeline's corner), reinforcing the transparency principle.

## A.5 Core Component Library

| Component | Key props | States |
|---|---|---|
| `TimelineTrack` | `type`, `clips[]`, `zoom` | default, dragging-clip-over, locked |
| `TimelineClip` | `clip`, `selected`, `producedByAgent` | idle, selected, trimming, dragging, invalid (fails validator) |
| `ReasonTooltip` | `reason: string`, `trigger: string \| null`, `agentName: string` | Appears on hover/focus over any agent-produced element; always renders the agent's own `reason`/`trigger` field verbatim, never a UI-invented explanation |
| `AgentPlanCard` | `plan: OrchestratorOutput`, `onRun`, `onEditInstruction` | pending-review, running, partial-failure (shows which agent failed, offers retry per Build Bible Phase 3's retry/fallback logic), complete |
| `AgentProgressRow` | `agentName`, `status: queued\|running\|done\|error`, `durationMs` | one row per dispatched agent, shown live during a run |
| `WaveformCanvas` | `audioFeatures`, `zoom` | rendered once from cached `audio_features`, never re-analyzed live |
| `PreviewPlayer` | `timeline`, `currentTime`, `beforeAfterMode` | loading, playing, paused, buffering (render cache miss), error |
| `Inspector.ClipPanel` | `clip`, `onChange` | reflects live validator state — a field that would break validation (Build Bible Phase 1) shows inline, not just on export attempt |
| `BrandKitPicker` | `kits[]`, `activeKitId`, `scope: personal\|org` | For org users, org-scope kits show a lock icon on fields the org policy has pinned (Section 3.11) |
| `CommandPalette` | global `Cmd/Ctrl+K` | fuzzy search over actions, recent projects, and agent instructions ("run: make this faster paced") |
| `ExportDialog` | `platformPresets[]`, `progress` | Mirrors Shorts Agent's fixed platform-constant table (Agent Prompt Pack Agent 14) exactly — preset list must never drift from that table |
| `SafeZoneOverlay` | `platform` | Toggleable overlay on the Preview Player showing the exact UI safe-zone rectangle from Shorts Agent's fixed constants, so a human can see why a caption was repositioned |

## A.6 Interaction & Keyboard Shortcuts

| Action | Shortcut |
|---|---|
| Play/pause | `Space` |
| Split clip at playhead | `S` |
| Undo / Redo | `Cmd/Ctrl+Z` / `Cmd/Ctrl+Shift+Z` |
| Zoom timeline in/out | `+` / `-` or trackpad pinch |
| Command palette | `Cmd/Ctrl+K` |
| Toggle before/after | `B` |
| Run agent plan | `Cmd/Ctrl+Enter` from the Agent Panel input |
| Nudge selected clip 1 frame | `Left` / `Right` |
| Delete selected clip (ripple) | `Shift+Delete` |
| Delete selected clip (lift, leave gap) | `Delete` |

All shortcuts are re-bindable in Settings; the table above defines shipped defaults only.

## A.7 Frontend State Architecture

Zustand stores, one per domain, each with a narrow responsibility (mirrors the "single responsibility per crate" discipline from the Build Bible):

- `useProjectStore` — current project metadata, save/dirty state.
- `useTimelineStore` — the live `timeline.json` in memory, exposes only mutation actions that go through `vided-core` via Tauri commands (never mutated directly in JS state and then "synced" — the Rust core is the source of truth, the store is a reactive mirror of it).
- `useAgentRunStore` — current plan, per-agent progress, retry state.
- `useSelectionStore` — currently selected clip/effect for the Inspector.
- `useOrgStore` — present only when signed into a Control Plane org session; empty/absent for local-only users (this store must be fully optional at the type level — the app must compile and run with zero org awareness for the free/individual product).

## A.8 Accessibility (WCAG 2.1 AA baseline)

- Full keyboard operability of the timeline (select, move, trim, delete clips without a mouse).
- Every icon-only button has an `aria-label`; every color-coded track/state also has a non-color indicator (icon or pattern) for colorblind users — do not rely on `--track-video` vs `--track-audio` color alone to distinguish track types.
- Minimum contrast ratio 4.5:1 for body text, 3:1 for large text/UI chrome, checked against both theme token sets in A.2.
- Respect OS-level "reduce motion" setting — falls back to instant transitions, no eased animation.
- Preview Player supports caption rendering that mirrors what Caption Agent will actually burn in, so low-vision/deaf/hard-of-hearing users reviewing a project can rely on in-app captions being accurate, not a placeholder.

## A.9 Theming

Light/dark ship by default. For enterprise customers with a white-label/custom-branding entitlement (Section 3.3), the token file (A.2) is swappable at the org level via the Control Plane's org settings — this reuses the same CSS-variable architecture, no separate theming engine needed.

---

# PART B — BACKEND / CONTROL PLANE SPECIFICATION

## B.1 Scope Boundary — read this before anything else in Part B

**The Control Plane never receives:** raw video/audio files, transcripts, `video.cache.json` contents, `timeline.json` contents, or rendered output — for any user, on any plan — **unless** a project has Shared Project Storage explicitly enabled (an opt-in, per-project, clearly-labeled toggle, off by default, available only on Team/Enterprise plans, and itself just an encrypted object-storage bucket scoped to that org, never inspected or processed by any Control Plane service).

**The Control Plane exists to handle:** identity (who are you, what org, what role), entitlements (what plan, what feature flags, how many seats), shared brand assets that are genuinely small (logos, LUT files, font references — kilobytes, not gigabytes of footage), billing, audit logs of *account-level* actions (who invited whom, who changed a policy — never "what did you edit"), and license/model distribution metadata.

This boundary is the single most important design decision in this document — violating it anywhere (e.g., a "helpful" analytics event that includes a transcript snippet) undermines the entire local-first positioning from Build Bible Section 1. Treat it with the same weight as the Agent Prompt Pack's hard safety rules.

## B.2 Tech Stack

| Layer | Choice | Why |
|---|---|---|
| API service | **Node.js + TypeScript, Fastify** | Keeps the whole non-performance-critical surface (frontend + control plane) in one language, easing maintenance for a small team building the full repo; Fastify over Express for built-in schema validation (JSON Schema/TypeBox) matching this project's "validate everything at the boundary" philosophy |
| Database | **PostgreSQL** | Relational integrity for orgs/users/roles/entitlements is a natural fit; row-level security used for tenant isolation (B.4) |
| Cache / session store | **Redis** | Session tokens, rate-limit counters, short-lived SSO state |
| Background jobs | **BullMQ** (Redis-backed) | Entitlement recalculation, invite emails, usage rollups, SCIM sync jobs |
| Object storage (opt-in Shared Project Storage + brand assets only) | **S3-compatible** (AWS S3 for SaaS; MinIO for self-hosted, Section 3.9) | |
| Auth | **OIDC/SAML via a vetted library** (e.g. `openid-client` for OIDC, `samlify` or equivalent for SAML) — never hand-rolled crypto for auth flows | Auth correctness bugs are unusually high-severity; use audited libraries, verify current maintenance status before depending on one |
| API contract format | **OpenAPI 3.1**, generated into TypeScript client types consumed by `apps/desktop`'s `useOrgStore` and by the separate Admin Console app | Single source of truth shared across two frontends |

## B.3 Service Architecture

```
services/control-plane/
├── auth-service/        # OIDC/SAML login, session issuance, SCIM provisioning endpoint
├── org-service/         # Orgs, teams, members, roles, invites
├── billing-service/      # Plan/seat management, Stripe (or equivalent) integration, usage metering for cloud-generation calls
├── asset-service/        # Org-shared Brand Kits, DAM-lite (Section 3.11) — small files only, size-capped
├── audit-service/        # Append-only account-level audit log, export endpoint
├── entitlement-service/  # Resolves "what can this user/org do" — feature flags, seat counts, plan tier — cached aggressively, this is read on nearly every desktop-app startup so it must be fast and available even if other services degrade
└── gateway/               # Single public API surface, auth middleware, rate limiting, routes to the above internal services
```

Each service is a separate deployable (even if they start out as modules in one process for the SaaS MVP — **[ASSUMPTION]**: ship as a modular monolith behind the `gateway/` first, split into real separate deployments only once a specific service's load actually requires independent scaling; premature microservices here would slow down getting to a working enterprise MVP for no real benefit at initial scale).

## B.4 Data Model (core tables)

```sql
-- Tenant isolation: every table below carries org_id and uses Postgres row-level
-- security policies keyed on org_id, enforced at the DB layer, not just app-layer checks.

orgs            (id, name, plan_tier, sso_config_id, created_at, data_residency_region)
users           (id, email, name, auth_provider, created_at)
org_members     (org_id, user_id, role, invited_by, joined_at)
teams           (id, org_id, name)
team_members    (team_id, user_id)
brand_kits      (id, org_id, name, fonts JSONB, colors JSONB, logo_asset_ref, pinned_fields JSONB, created_by, updated_at)
                 -- pinned_fields: which fields an org admin has locked so individual creators can't override (feeds A.5's BrandKitPicker lock icons)
entitlements    (org_id, feature_flag, enabled, seat_limit, cloud_generation_monthly_cap_usd)
audit_events    (id, org_id, actor_user_id, event_type, target, metadata JSONB, created_at)
                 -- event_type examples: "member.invited", "role.changed", "brand_kit.updated",
                 -- "sso.config_changed", "cloud_generation.threshold_reached" — never a video-content event
usage_records   (org_id, user_id, metered_item, quantity, occurred_at)
                 -- metered_item examples: "cloud_generation_call", "render_minutes_local" (local render
                 -- minutes are counted for plan-limit purposes on some tiers but the video itself is never uploaded)
shared_projects (id, org_id, team_id, storage_ref, created_by, opted_in_at)
                 -- exists ONLY when a project owner explicitly enables Shared Project Storage;
                 -- storage_ref points at an org-scoped encrypted bucket object, opaque to every
                 -- Control Plane service except the one narrow sync client in the desktop app
```

## B.5 API Contract (representative endpoints — full spec lives in `proto/openapi/control-plane.yaml`, generated, not hand-maintained prose)

```
POST   /v1/auth/sso/initiate
POST   /v1/auth/sso/callback
GET    /v1/orgs/:orgId/entitlements          # cached client-side, polled infrequently, must degrade gracefully offline (B.1: app must remain fully usable offline for all local features)
POST   /v1/orgs/:orgId/members/invite
PATCH  /v1/orgs/:orgId/members/:userId/role
GET    /v1/orgs/:orgId/brand-kits
PUT    /v1/orgs/:orgId/brand-kits/:kitId
GET    /v1/orgs/:orgId/audit-events?since=&type=
POST   /v1/orgs/:orgId/usage/cloud-generation     # called only at the moment a cloud generation call is made, per Build Bible §15.2's double-gate — records the metered event, does not carry any video content or prompt content beyond what's needed for billing (e.g. "1 image generation call", not the actual generated image or prompt text)
GET    /v1/billing/portal-session                 # returns a hosted billing-portal URL, actual card data never touches VID-ED's own servers
```

Every request requires a signed session token (short-lived JWT, refreshed via a longer-lived refresh token stored in the OS keychain client-side, same pattern as Build Bible §12's API-key-in-keychain rule). Every response is validated against its OpenAPI schema on both server (request) and client (response) sides — same "validate everything at the boundary" discipline used for agent JSON throughout this whole project.

## B.6 Auth & Identity

- **Individual/free users:** no Control Plane account required at all — the desktop app works fully standalone, exactly as specced in the Build Bible, forever. Signing into an org is strictly additive.
- **SSO:** OIDC and SAML 2.0 support for org login; **[ASSUMPTION]** initial launch partners with the two most commonly required in mid-market/enterprise procurement — Okta and Azure AD/Entra ID — validated first, generic OIDC/SAML support following the same code path for any other IdP.
- **SCIM 2.0** for automated user provisioning/deprovisioning from the customer's IdP — deprovisioning must revoke the desktop app's org session within one polling interval (target: 15 minutes) even if the user's device is offline at the moment of deprovisioning, by having the entitlement check fail closed the next time the app can reach the Control Plane, and by capping how long a session token is valid before requiring re-validation.
- **MFA:** delegated to the IdP for SSO orgs; TOTP-based MFA available directly for orgs not using SSO.

## B.7 Sync Protocol (desktop ⇄ control plane)

A single, narrow sync client in `apps/desktop` (a new `useOrgStore`-backed module, per A.7) is the **only** code path in the entire desktop app permitted to make Control Plane network calls. It syncs, on login and periodically thereafter: entitlements, org-level brand kits, audit-relevant account actions the user takes (e.g. "user updated org brand kit," never "user trimmed clip at 00:14"), and — only if the specific project has opted in — the Shared Project Storage blob referenced in B.4. Every other part of the desktop app has zero network awareness of the Control Plane, enforced at the module-boundary/lint level (an ESLint rule forbidding imports of the org sync client from anywhere outside `useOrgStore` and its own module).

## B.8 Deployment Topology

- **SaaS control plane:** single region to start (**[ASSUMPTION]** `us-east`), multi-region read replicas added per Section 3.7's SLA needs once there's real enterprise demand for it — do not over-build multi-region infra before there's a customer requiring it.
- **Self-hosted control plane** (Section 3.9): the same codebase, packaged as a Docker Compose bundle (small customers) or a Helm chart (larger regulated customers running their own Kubernetes) — this is why the modular-monolith-behind-a-gateway decision in B.3 matters: it must be operable as a single deployable unit by a customer's own ops team, not require them to run eight microservices.

## B.9 Observability

OpenTelemetry traces/metrics from every Control Plane service, structured JSON logs, no video-content or PII beyond what's already listed as legitimately stored (B.4) ever appears in a log line — log statements go through a redaction helper that strips anything resembling a token, email in a non-designated field, or file path outside known-safe categories.

---

# PART C — ENTERPRISE-GRADE REQUIREMENTS

## C.1 RBAC Model

| Role | Scope | Can do |
|---|---|---|
| Org Owner | org | Everything below, plus billing changes, org deletion, SSO config |
| Org Admin | org | Manage members/roles, brand kits, audit log access, entitlement/policy toggles (e.g. org-wide disable of cloud generation per Build Bible §15.2) |
| Team Lead | team | Manage team membership, team-scoped brand kit overrides within org policy limits |
| Editor | team/project | Full editing rights on projects they're a member of; cannot change org/team policy |
| Viewer | team/project | Read-only access to Shared Project Storage projects (review/comment, no edit) — this is the role a client/producer reviewer gets |
| Billing Admin | org | Billing portal access only, no editing/member-management rights — separates finance stakeholders from editorial control cleanly |

Permission checks are enforced **server-side** on every Control Plane endpoint (never trust a client-side role check alone) and additionally enforced by Postgres row-level security (B.4) as a defense-in-depth layer, so a bug in application-layer authorization logic cannot alone leak cross-tenant data.

## C.2 Admin Console

A **separate web application** (`apps/admin-console`, React + TypeScript, deployed independently of the desktop app and the marketing site) for Org Owners/Admins:

- Member management, role assignment, SCIM status.
- Brand Kit editor with field-pinning (B.4's `pinned_fields`).
- Usage dashboard: seats used, cloud-generation spend against the configured monthly cap (`entitlements.cloud_generation_monthly_cap_usd`), local render-minute counts where relevant to the plan.
- Audit log viewer with filter/export (CSV/JSON) — satisfies typical procurement/security-review requirements.
- Policy toggles: org-wide enable/disable of each cloud integration named in Build Bible §15.2, org-wide enable/disable of Shared Project Storage, SSO/SCIM configuration.
- Support/ticket entry point, tied to the support tier in C.7.

The Admin Console **never** displays project content, timelines, or footage — consistent with B.1, it only manages the account-level surface.

## C.3 Billing & Licensing Model

- Individual/free tier: no Control Plane account, no billing, full local feature set per the Build Bible (this must remain genuinely full-featured, not artificially crippled, so it continues to function as the funnel into Team/Enterprise).
- **Team tier:** per-seat subscription, unlocks org brand kits, Shared Project Storage, basic RBAC (Editor/Viewer), standard support.
- **Enterprise tier:** adds SSO/SCIM, full RBAC, audit log export, self-hosted option, custom cloud-generation spend caps, dedicated support/SLA (C.7), optional white-label theming (A.9).
- Billing itself is handled by a hosted billing provider (Stripe Billing or equivalent) — VID-ED's own systems never store raw payment card data (B.5).
- Cloud-generation usage (Veo/Imagen/Runway/Flux/Claude API calls from B-roll/Voice agents) is metered per B.4's `usage_records` and billed either pass-through at cost-plus-margin or against the org's configured monthly cap, whichever the contract specifies — **[ASSUMPTION]** exact pricing model is a business decision outside this document's scope; the metering/enforcement plumbing above supports either model without further engineering rework.

## C.4 Compliance & Certifications Roadmap

| Target | Priority | Notes |
|---|---|---|
| SOC 2 Type II | High — table stakes for most enterprise procurement | Requires the audit logging (C.10), access control (C.1), and encryption (C.5) work below to already be in place before an audit engagement can even start; budget 6–12 months of evidence collection once controls are live |
| GDPR / CCPA | High | Data residency region field already in `orgs` table (B.4); right-to-erasure implemented as a hard-delete job across `users`, `org_members`, `usage_records`, `audit_events` (audit events for erasure requests themselves are retained per legal requirement, anonymized against the erased user) |
| ISO 27001 | Medium | Natural follow-on after SOC 2, shares much of the same control set |
| HIPAA readiness | Not pursued by default | **[ASSUMPTION]** out of scope unless a specific customer segment requires it — would add BAA support and PHI-specific handling requirements on top of everything above; flag explicitly if this becomes a real requirement rather than silently scoping it in |

## C.5 Security Program

- **Encryption in transit:** TLS 1.2+ everywhere, HSTS on all Control Plane domains.
- **Encryption at rest:** Postgres encryption at rest (cloud provider-managed keys for SaaS; customer-managed keys optional for self-hosted per C.9), S3 bucket encryption for any Shared Project Storage / brand assets.
- **Dependency scanning:** automated (e.g. `npm audit`/`cargo audit`/`pip-audit`) in CI (extends Build Bible §14's pipelines), blocking merge on any critical/high unpatched vulnerability with no available fix acknowledged.
- **Penetration testing:** annual third-party pen test of the Control Plane once it's handling real customer org data, plus before any SOC 2 audit engagement.
- **Vulnerability disclosure:** a published `security.txt`/disclosure policy and a monitored inbox — standard enterprise procurement checklist item, cheap to stand up early.
- **Secrets management:** no secrets in source control ever (enforced via a pre-commit/CI secret-scanning check); Control Plane secrets in a real secrets manager (cloud provider's, e.g. AWS Secrets Manager, for SaaS; a self-hosted option like Vault documented for the on-prem package in C.9).

## C.6 Data Residency & Retention

- `orgs.data_residency_region` lets an enterprise customer pin their Control Plane data (and, if Shared Project Storage is used, their asset bucket) to a specific region — required by many EU customers.
- Retention defaults: audit events retained 2 years (configurable up per contract, never down below what compliance requires), usage records retained per billing/legal need, Shared Project Storage objects retained per the org's own policy (a project owner-controlled deletion, not silently expired).

## C.7 SLA & Support Tiers

| Tier | Uptime target (Control Plane API — note: this SLA covers account/identity/billing services only, since local editing/rendering has no network dependency and thus no relevant "uptime" concept) | Support |
|---|---|---|
| Team | Best-effort, no formal SLA | Email, standard business-hours response |
| Enterprise | 99.9% monthly | Priority queue, named contact, target first-response times defined per contract |

The framing above matters: because the actual editing product works offline, a Control Plane outage degrades org-management features (inviting a new member, updating a shared brand kit) but **never** blocks someone from opening the desktop app and editing — entitlement checks (B.5) must fail open for already-authenticated, already-entitled local sessions during a Control Plane outage, re-validating once connectivity returns, rather than locking a paying customer out of their own tool because of a backend blip.

## C.8 Disaster Recovery & Backup

- Automated daily Postgres backups, point-in-time recovery enabled, backup restore tested on a real schedule (quarterly), not just configured and assumed to work.
- RPO target: 1 hour. RTO target: 4 hours for the SaaS Control Plane (again — this bounds account/billing/org-management recovery time, not local editing, which is unaffected).

## C.9 Self-Hosted / On-Premise Package

- Docker Compose bundle (Control Plane services from B.3 + Postgres + Redis + MinIO) for smaller regulated customers who want to run the Control Plane themselves without a Kubernetes footprint.
- Helm chart for larger customers running their own Kubernetes, with documented resource requests/limits, horizontal pod autoscaling config for the API gateway, and a clear upgrade/migration runbook (schema migrations must be forward-only, reversible-by-restore, and tested against the previous two minor versions before every release, per standard enterprise upgrade-safety expectations).
- The desktop app's org sync client (B.7) points at a configurable Control Plane base URL, so a self-hosted customer simply repoints their fleet at their own instance — no separate desktop app build required for self-hosted vs SaaS customers.

## C.10 Audit Logging Spec

Every `audit_events` row (B.4) is **append-only** at the database level (no `UPDATE`/`DELETE` grants on that table for the application role — only `INSERT`, plus a separate, heavily-restricted retention-cleanup job role for the erasure case in C.4). Event schema:

```json
{
  "id": "uuid",
  "org_id": "uuid",
  "actor_user_id": "uuid or null (null for system-initiated events, e.g. an automated entitlement recalculation)",
  "event_type": "member.invited | member.role_changed | brand_kit.updated | sso.config_changed | policy.cloud_generation_toggled | billing.plan_changed | erasure.completed",
  "target": "string identifying what was acted on (a user id, a brand kit id, etc.)",
  "metadata": {"...event-specific fields, never video content or file paths outside brand-kit asset references..."},
  "created_at": "ISO 8601 timestamp"
}
```
Exportable as CSV/JSON from the Admin Console (C.2), filterable by date range and event_type — this is frequently a literal procurement checklist requirement, not a nice-to-have.

## C.11 Centralized Brand Kit / DAM-lite

Extends `creator_profile.brand` from the Agent Prompt Pack's Shared Data Contracts: an org-level `brand_kits` table (B.4) that Brand Agent's input can be populated from instead of (or blended with) a purely personal profile, when the acting user belongs to an org. **Field-pinning** (`pinned_fields`) lets an org admin lock specific fields (e.g. logo asset, primary colors) so individual editors can't drift off-brand, while leaving others (e.g. secondary accent choices) open for creative flexibility — surfaced in the UI via A.5's `BrandKitPicker` lock icons. Assets themselves (logo files, LUT files, font files) are size-capped (**[ASSUMPTION]** 50MB per asset) and stored in the org's S3-compatible bucket — this is the one legitimate, intentional exception to "no media in the Control Plane," scoped narrowly to brand assets rather than footage, and still opt-in at the org level.

## C.12 Enterprise Rollout — Phase Numbering (continues Build Bible Section 6)

- **Phase 9 — Control Plane MVP:** auth-service + org-service + entitlement-service behind the gateway, Postgres schema from B.4, desktop app's `useOrgStore` sync client, basic RBAC (Org Owner/Admin/Editor only, Team Lead/Viewer/Billing Admin follow in Phase 10). Definition of Done: an org can be created, members invited, an org-scoped Brand Kit created and consumed by Brand Agent in a real edit, entitlement fail-open verified by killing the Control Plane mid-session and confirming editing continues uninterrupted.
- **Phase 10 — Admin Console + Billing:** `apps/admin-console` built, billing-service integrated with the hosted billing provider, full RBAC matrix (C.1), audit log viewer/export.
- **Phase 11 — SSO/SCIM + Compliance Hardening:** OIDC/SAML, SCIM provisioning, the encryption/secrets/dependency-scanning items in C.5 fully in place, ready to begin SOC 2 evidence collection.
- **Phase 12 — Self-Hosted Package:** Docker Compose + Helm chart (C.9), upgrade/migration runbook tested.

Each of these phases follows the exact same discipline as Build Bible Phases 0–8: no phase marked done without its Definition of Done met and its tests passing in CI — the Never List (Build Bible §13) applies here without modification, including its rule against hallucinated dependencies, which matters especially for the SSO/SAML libraries named in B.2 given how security-sensitive that surface is.

---

## APPENDIX — REPO STRUCTURE ADDITIONS

```
vided/
├── apps/
│   ├── desktop/                  # unchanged from Build Bible, plus a new src/org/ module (useOrgStore + sync client, per B.7)
│   └── admin-console/            # NEW — separate React+TS app, Section C.2
├── services/
│   ├── vision-sidecar/           # unchanged
│   └── control-plane/            # NEW — Section B.3's services
│       ├── gateway/
│       ├── auth-service/
│       ├── org-service/
│       ├── billing-service/
│       ├── asset-service/
│       ├── audit-service/
│       └── entitlement-service/
├── proto/openapi/control-plane.yaml   # NEW — OpenAPI source of truth for B.5
├── infra/                         # NEW — Docker Compose bundle + Helm chart, Section C.9
│   ├── docker-compose.selfhosted.yml
│   └── helm/vided-control-plane/
└── docs/
    ├── AGENT_PROMPTS.md
    ├── BUILD_BIBLE.md
    └── UI_BACKEND_ENTERPRISE_SPEC.md  # this document
```

## CROSS-DOCUMENT CONSISTENCY RULES

1. Any change to `creator_profile.json`'s schema (Agent Prompt Pack) must be mirrored in `brand_kits` (B.4) and vice versa — they are meant to merge cleanly for org users, per C.11.
2. The Never List (Build Bible §13) and Prime Directives (Build Bible §0) apply to every service added by this document without exception — a Control Plane hallucinated dependency or a stubbed auth check is exactly as unacceptable as a stubbed render function.
3. B.1's scope boundary is the one rule in this entire three-document set that should require the strongest justification to ever change — if a future feature seems to need video content in the Control Plane, treat that as a decision requiring explicit human sign-off, not something to infer from a feature request's convenience.
