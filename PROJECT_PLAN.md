# Workshop — Project Plan

*Owner: Johannes Tveitan. Last updated: 2026-06-07. This is the cross-device source of truth for where we are and what's next.*

---

## ▶ Current status & next action

- **Status:** Research complete (two verified deep-research passes). **Phase 0 — architecture foundation** written ([`ARCH.md`](ARCH.md)), awaiting review/sign-off.
- **Next action:** review `ARCH.md` (module boundaries + the nine ports). On sign-off → scaffold the repo per ARCH §3 as a *compiling skeleton* (`domain/ports.go` + `nop`/in-memory adapters), then run the Phase-0 spikes on the Mac Mini.
- **Nothing is committed to a build yet.** Everything so far is docs/decisions.

---

## North Star

> One control plane I can open on any device that runs my Claude subscription as a one-person software house — acting on its own for every clean-cut decision, pulling me in only where my judgment changes the outcome, and always able to show me exactly where each feature stands so nothing slips.

### Three pillars (full detail in the research brief, Part 3)

1. **Autonomy without rubberstamping** — an act-vs-ask classifier (AUTO when reversible + low-blast-radius + high-confidence + inside a standing grant; ASK when irreversible / ambiguous / plan-fork / stuck). Standing grants widen autonomy as trust grows; every AUTO action is logged + reversible.
2. **Feature-set progress tracking** — feature = first-class epic→task→step tree with a Definition-of-Done checklist + a completeness-critic pass. Answers "where are we / did we miss anything" by construction.
3. **Memory / learning layer** (the differentiator) — a clean-room knowledge store (aina Learning-Graph pattern) the orchestrator queries before acting and writes to after; your approvals become grants/lessons, so it asks less over time.

### Navigation (parallel projects is core)

**Portfolio → Project → Feature(epic) → Task → Step.** Level 1 = portfolio home (every project a card: %-progress, "needs you" badge, what's building, health; blocked-on-you floats up; one unified cross-project Decision Inbox). Level 2 = tap a project → its feature tree + progression, project-filtered Decision Inbox, live worker activity, recent AUTO actions (with rollback), relevant lessons. Phone-first.

---

## Decisions log

| # | Decision | Choice | Date |
|---|---|---|---|
| D1 | Auth/cost model | Subscription-first (first-party `claude` CLI), metered API only for background bits | 2026-06-07 |
| D2 | Topology | Hybrid: VPS control plane + worker dev-boxes (Mac Mini first, extensible) over Tailscale + NATS | 2026-06-07 |
| D3 | Executor | **Switchable** per job: `claude` CLI (subscription) ⇄ API harness | 2026-06-07 |
| D4 | Org topology | Single orchestrator + subagents per project; multi-agent fan-out only in the research lane | 2026-06-07 |
| D5 | Build vs adopt | **Build** control plane + worker + switchable executor; **adopt** container-use / NATS / Langfuse / spec-kit; **borrow patterns** from aina & lifthrasir | 2026-06-07 |
| D6 | Form factor | Single Go binary control plane (REST+MCP+embedded Lit) + external NATS + SQLite→PG; worker = separate Go binary | 2026-06-07 |
| D7 | lifthrasir coupling | Standalone first; integrate later via an adapter (REST+NATS+webhooks+MCP) | 2026-06-07 |
| D8 | Architecture | Ports & adapters (hexagonal); dependency rule `domain ← app ← adapters`; everything foreseeable is a port | 2026-06-07 |
| D9 | Build bar | Rigorous, modular, additive-by-design, no quick fixes (see CLAUDE.md) | 2026-06-07 |

---

## Open questions (resolve as config/adapters — no rework, per D8)

- **Q1 — what "clean-cut" means:** the default reversible+low-blast+high-confidence+granted spine, OR a phase-gated model ("spec always needs me; after a signed spec it runs free to PR")? → resolved as `PolicyEngine` config.
- **Q2 — feature tree location:** GitHub issues/milestones as source of truth (workshop reads/annotates) OR workshop owns epic→task state and mirrors to GitHub? → resolved by which `FeatureStore` adapter is wired.
- **Q3 — cost envelope (empirical):** how many agent-hours a $100/$200 monthly credit buys before overflow. → **measure** in Phase 0 on the Mac Mini, don't research.

---

## Phase plan

- **Phase 0 — Foundation & spikes** *(in progress)*
  1. ✅ Architecture written (`ARCH.md`).
  2. ☐ Architecture reviewed/agreed.
  3. ☐ Repo skeleton per ARCH §3 — port interfaces stubbed + `nop`/in-memory adapters that compile.
  4. ☐ Spikes on Mac Mini: confirm `claude` CLI headless under subscription (`claude setup-token`); measure cost envelope. (Harness provided; you run it.)
- **Phase 1 — Walking skeleton:** control-plane binary (project + feature/epic tree + DoD as first-class model; NATS job queue; minimal portfolio+project dashboard) + one Mac Mini worker driving `claude` CLI in a worktree; submit a job by hand → get a branch/PR back.
- **Phase 2 — Switchable executor + safety + Decision Inbox:** executor abstraction (CLI ⇄ API); container-use isolation; the act-vs-ask PolicyEngine + standing grants; Decision Inbox with push/Slack; AUTO-action audit + rollback. *(Pulled forward — it's the heart of the value.)*
- **Phase 3 — Research lane + memory:** multi-agent feature-research fan-out (API/Haiku) → drafted specs/issues (spec-kit); seed the memory/learning layer from approvals + merged PRs.
- **Phase 4 — Fleet + observability:** add workers (disjoint-subject routing); Langfuse traces/cost; dashboard polish.
- **Phase 5 — lifthrasir integration:** surface approvals/work items via lifthrasir REST + NATS + webhooks + MCP (as an adapter).

---

## Pointers

- Full research + citations: [`research/2026-06-07-agentic-workshop-brief.md`](research/2026-06-07-agentic-workshop-brief.md) (Part 1 landscape, Part 2 tool survey + verdict, Part 3 core vision).
- Architecture + the nine ports: [`ARCH.md`](ARCH.md).
- Pattern sources (clean-room, not code): aina (single-binary MCP+dashboard spine, manifest sync, Learning Graph) and lifthrasir (autonomy grants #381, tool safety #387, audit/reason/idempotency, agent memory #385).
