# Agentic Workshop — Research & Decision Brief

*Date: 2026-06-07. Status: **research findings + options for input**. Nothing here is committed — this brief exists so Johannes can steer the path before we build.*

## What this is

Research toward a fully-automated agentic "software workshop / AI company": a system that designs, builds, reviews and ships software projects with minimal human input, presented as a multi-project dashboard that tells you **when it needs you**. Constraints fixed earlier:

- **Subscription-first, API ok** — prefer Claude Max/Pro for heavy coding; metered API only for always-on background bits.
- **Hybrid topology** — 24/7 control plane + dashboard on a Linux VPS; a fleet of local dev-box workers join in (Mac Mini first, extensible).
- **Survey then recommend** — adopt vs build is open.

Three inputs were triangulated: (1) external SOTA via a verified deep-research pass, (2) **aina** as a *patterns-only* blueprint (company-owned — ideas, not code), (3) **lifthrasir** as the future integration target and a model of Johannes's own conventions.

---

## TL;DR (the five things that decide the design)

1. **Subscription-driven headless Claude is officially supported — but only for *first-party* tools, and it's now cost-capped.** You can run `claude -p` / the Agent SDK / the official GitHub Action under Pro/Max via an OAuth token (`claude setup-token`). From **June 15 2026** (days away) that usage draws from a **per-user monthly credit** (Pro $20 / Max5x $100 / Max20x $200) that **cannot be shared or pooled and does not roll over**; overflow bills at API rates only if you opt in (default off). [1][2][3]
2. **Critical ToS nuance:** a Jan/Feb 2026 crackdown banned subscription OAuth tokens in **third-party harnesses**. Subscription auth is safe for Anthropic's own `claude` CLI / Agent SDK / GitHub Action — **not** for wrappers like OpenCode/Roo/Crush/Goose. → *Workers must drive the official `claude` CLI, not a third-party harness, to stay subscription-legal.* (Flagged to re-verify; see Gaps.)
3. **The "org of agents" (PM/architect/dev/QA role-play) does not beat a single strong orchestrator for coding.** Evidence: minimal, non-significant gains; ~15× token cost; 14 documented multi-agent failure modes. Multi-agent *shines for research* (parallelizable) and *loses on coding* (dependency-heavy). [4][5][6] → *Single orchestrator + subagents for building; reserve fan-out for the research pipeline.*
4. **Human-in-the-loop gating is mandatory, and approval fatigue is the real risk.** Even Claude's "auto mode" is explicitly "not a drop-in replacement for careful human review on high-stakes" work; users rubber-stamp ~93% of prompts. [7] → *Gate on a few meaningful checkpoints, not everything.*
5. **The cleanest self-hostable spine if we ever want API-based execution is OpenHands (MIT).** But under the subscription-first constraint, the simpler path is to drive the `claude` CLI directly with git worktrees. [8]

---

## 1. Subscription vs API — the linchpin

| Path | Mechanism | Cost model (from Jun 15 2026) | Allowed for our use? |
|---|---|---|---|
| **Subscription, first-party** | `claude setup-token` → `CLAUDE_CODE_OAUTH_TOKEN`; `claude -p`, Agent SDK, `anthropics/claude-code-action` | Per-user capped credit (Pro $20 / Max5x $100 / Max20x $200), no pool/rollover, overflow→API rate if opted in | ✅ heavy coding on workers |
| **Subscription, third-party harness** | OAuth token inside OpenCode/Roo/Crush/Goose/OpenHands | — | ❌ ToS-disallowed |
| **Metered API key** | `ANTHROPIC_API_KEY` | Pay per token; no credit; Anthropic's recommended path for "shared/production automation" | ✅ background jobs, research, cheap models (Haiku) |

**Architectural consequences:**
- **One Claude subscription = one worker's credit pool.** Credits can't be pooled, so you can't fan one Max20x across ten parallel agents and stay within subscription pricing. Parallelism is bounded by (a) how many subscriptions/boxes you have and (b) each box's monthly credit before overflow.
- **Workers run the official `claude` CLI headless**, each authenticated with its own local OAuth token. The control plane never holds the subscription token; it just dispatches work.
- **Background/always-on bits** (triage, scheduling, the research fan-out) use a **metered API key with cheap models** — exactly the split you pre-approved.
- **Practical unknown:** how many agent-hours a $200 Max20x credit actually buys before overflow dominates. This sizes the whole fleet economics and is the #1 open question.

## 2. Org-of-agents vs single orchestrator

The romantic "AI company with a PM, architects, and a QA team" underperforms for software delivery:
- MAST taxonomy: multi-agent gains "often minimal," 14 failure modes across system-design / inter-agent-misalignment / task-verification. [4]
- Anthropic's own multi-agent system beat single Opus by 90.2% **on research**, at **~15× tokens**, and explicitly notes coding has "fewer truly parallelizable tasks than research." [5]
- A 2026 orchestration study: 81.2% vs strongest single model 80.5% — statistically non-significant, with real coordination cost; consensus voting *discarded* correct minority answers. [6]

**Takeaway:** model the workshop as **one strong orchestrator per project + subagents + good tool/eval scaffolding** (which is how Claude Code already works), not a role-play org. The "company" metaphor lives in the **dashboard UX** (projects, status, an inbox of decisions), not in the agent topology. Multi-agent fan-out is reserved for **research**, where it genuinely wins.

## 3. Safe autonomous pipeline + HITL gating

Pattern that the sources converge on: **spec → plan → implement (in isolation) → test → review → PR**, with isolation via **git worktrees + a sandbox/container per task** [9][10][11], automated **eval/verification gates** before anything reaches a human, and **approval gates only at meaningful junctions**:
- plan approved before large work starts,
- PR ready for review,
- destructive / irreversible / out-of-sandbox operations,
- repeated failure / agent stuck.

Approval fatigue is the failure mode to design against (93% rubber-stamp rate [7]) — fewer, higher-signal approvals beat a firehose of prompts. Notifications go to where you already are (push / Slack), and the dashboard carries an **approval queue** as a first-class surface.

## 4. The tool landscape (adopt vs build) — *partial; see Gaps*

The verified pass confirmed **OpenHands** as the strongest self-hostable, no-lock-in execution spine (MIT core, composable Python Agent SDK, local→cloud portability, Docker/K8s). [8] The broader tool-by-tool survey (Vibe Kanban, Conductor, Sculptor, Claude Squad, Devin, Factory.ai, container-use/Dagger, Cursor background agents, Roo/Cline, Aider, Copilot coding agent) **did not produce verified claims** this pass (budget) — treat my summaries of those as *unverified* until a focused follow-up.

**Why the subscription constraint pre-decides a lot:** because third-party harnesses can't use subscription auth, adopting OpenHands/Vibe-Kanban/etc. as the *executor* would force you onto **API billing**. To honor *subscription-first on the heavy coding*, the executor should be the **official `claude` CLI**. That pushes the verdict toward **build a thin orchestration + dashboard layer over the `claude` CLI's own headless + worktree primitives**, borrowing patterns (not code) from aina for the control-plane spine, and keeping OpenHands in reserve for API-based or heavily-sandboxed execution.

## 5. Feature-research pipeline

The "new feature research" loop is the natural home for multi-agent fan-out and for cheap-model/API usage. Building blocks seen in the wild: GitHub **spec-kit** (spec-driven dev: spec → plan → tasks → issues), **PRD-to-issues** skills, and **open_deep_research**-style market/technical research agents that emit structured backlog items. [12] Output of the loop = drafted specs / GitHub issues that flow into the same approval queue.

## 6. Observability / dashboard

Self-hostable agent observability standard is **Langfuse** (traces, cost, status) — usable as the trace backend so we don't reinvent it. The custom dashboard then focuses on what's *workshop-specific*: per-project feature status, the worker fleet, and the **decision/approval inbox**. NATS has community **MCP bridges** (e.g. mcp-nats) if we want agents to publish/subscribe directly.

## 7. lifthrasir integration (future)

lifthrasir already exposes everything an external orchestrator needs: REST `/api/v1/*` (reason-audited mutations, batch, search), a **NATS+JetStream** bus (`lifthrasir.{entity}.{action}`), **HMAC webhooks**, and its own **MCP server**. Two clean integration shapes (decide later):
- **Workshop → lifthrasir:** surface workshop work items / approvals as lifthrasir todos or inbox items; you triage them in the life-planner you already use daily.
- **lifthrasir → workshop:** lifthrasir events (or an agent there) file feature requests into the workshop queue.
The audit/`reason` + idempotency conventions and the agent-permission model map almost 1:1 onto the workshop's needs — strong reuse of *your own* patterns, no lock-in.

---

## Recommended architecture (draft — for your input)

```
                 ┌──────────────────────── VPS (control plane, 24/7) ───────────────────────┐
                 │  Go single-binary (clean-room, aina-style patterns)                       │
   you ───────►  │   • Dashboard (embedded Lit): projects · feature status · APPROVAL INBOX  │
  (browser/push) │   • Job queue + scheduler  • project/feature state (Postgres)             │
                 │   • event in/out (NATS)  • Langfuse for traces/cost                        │
                 │   • manifest-sync of skills/rules/hooks → workers (aina pattern)           │
                 └───────▲───────────────────────────────┬──────────────────────────────────┘
                         │ status / approvals / logs      │ dispatch jobs (to a specific box)
                         │                                 ▼
                 ┌───────┴─────────── workers (dev boxes; Mac Mini first) ──────────────────┐
                 │  small Go worker-agent per box:                                           │
                 │   • pulls jobs · git worktree per task · sandbox per task                 │
                 │   • drives the OFFICIAL `claude` CLI headless (subscription OAuth, local) │
                 │   • single orchestrator + subagents (NOT a role-play org)                 │
                 │   • streams status + raises approval requests back to control plane       │
                 └──────────────────────────────────────────────────────────────────────────┘

  background lane (metered API, cheap models): feature-research fan-out · triage · schedulers
  future: control plane ⇄ lifthrasir via REST + NATS + HMAC webhooks + MCP
```

**Adopt-vs-build verdict (draft):** **Build** the control plane, dashboard, and worker-agent (your differentiator, your stack, zero lock-in, subscription-clean). **Adopt/borrow at the edges:** Langfuse (traces), spec-kit (spec→issues), NATS MCP bridge; keep **OpenHands** in reserve for when API-based/sandboxed execution is wanted. **Borrow patterns, not code, from aina** for the single-binary MCP+dashboard spine and manifest-sync fleet model.

---

## Open decisions I need your steer on

1. **Executor:** official `claude` CLI per worker (subscription-clean, my rec) vs an adopted harness like OpenHands (forces API billing)?
2. **Worker transport:** how the VPS dispatches to boxes — pull (worker polls control plane), or push over SSH / a persistent agent connection (Tailscale-style mesh)?
3. **Org vs orchestrator:** confirm single-orchestrator-per-project + subagents, with multi-agent fan-out only in the research lane?
4. **State store:** reuse your lifthrasir-style stack (Postgres + NATS) for the control plane, or keep workshop minimal (SQLite + embedded queue) to start?
5. **lifthrasir coupling in v1:** standalone first and integrate later, or design the integration in from day one?
6. **Build form factor:** single Go binary (aina-pattern) vs Go API + separate TS dashboard?

## Gaps / proposed follow-up research (before committing)

- **Tool-by-tool survey** (Vibe Kanban / Conductor / Sculptor / Claude Squad / Devin / Factory / container-use / Cursor bg agents / Roo-Cline / Aider) — not verified this pass. Worth one focused deep-research run so the adopt-vs-build verdict rests on facts, not my priors.
- **Re-verify the ToS crackdown** specifics (what exactly is allowed under subscription post-Jun-15) — load-bearing for the whole "subscription-first" premise.
- **Cost envelope** of 24/7 subscription automation: agent-hours per $200 Max20x credit before overflow. Sizes the fleet.
- **Distributed-runner mechanics** (queue-to-specific-machine, worktree parallelization, result sync-back) — thin evidence; needs a targeted pass.

---

## Sources

[1] github.com/anthropics/claude-code-action — setup.md (OAuth token) · [2] code.claude.com/docs/en/authentication · [3] support.claude.com/articles/15036540 (Agent SDK with your Claude plan; Jun 15 2026 credits) · [4] arxiv.org/pdf/2503.13657 (MAST multi-agent failure taxonomy) · [5] anthropic.com/engineering/built-multi-agent-research-system · [6] arxiv.org/pdf/2509.23537 (orchestration marginal gains) · [7] anthropic.com/engineering/claude-code-auto-mode · [8] github.com/OpenHands/OpenHands + arxiv.org/pdf/2511.03690 · [9] anthropic.com/engineering/claude-code-sandboxing · [10] mikemcquaid.com/sandboxed-agent-worktrees-...-2026 · [11] penligent.ai/...git-worktrees-runtime-isolation · [12] github.com/github/spec-kit, github.com/langchain-ai/open_deep_research

*Verification: 6 search angles → 29 sources → 141 claims → 25 adversarially verified (24 confirmed, 1 refuted). The refuted claim: a blanket ToS prohibition on subscription login in Agent-SDK products (1-2, killed). Self-reported Anthropic figures (90.2%, 93%, 17%) and single-preprint results noted as such.*

---

# Part 2 — Tool survey results & final recommendation

*Second verified deep-research pass (2026-06-07): 6 angles → 24 sources → 113 claims → 25 verified (22 confirmed, 3 refuted). Per-tool facts cite primary repos/docs. **Gap:** this pass did not re-surface Anthropic primary sources for the ToS/credit model (Part B) or the cost envelope (Part C) — Part B is covered by Part 1's verified findings; Part C remains the one open empirical question (see below).*

## Comparison table (for this use case)

Legend: ✅ yes · ⚠️ partial/caveat · ❌ no/unfit · "—" unverified

| Tool | License / self-host | Switchable executor | Subscription-legal Claude? | Remote dispatch to a chosen box | Per-task isolation | HITL + dashboard | Verdict |
|---|---|---|---|---|---|---|---|
| **container-use** (Dagger) | Apache-2.0, self-host | ✅ agent-agnostic (MCP) | ✅ **auth-neutral** — executor inside supplies auth | ❌ (no control plane) | ✅ fresh container + branch per agent | ⚠️ git review, no dashboard | **ADOPT** (isolation layer) |
| **Vibe Kanban** | Apache-2.0, self-host (Docker) | ✅ 10+ agents via profiles | ⚠️ undocumented for sub-OAuth | ❌ local | ✅ worktree per attempt | ✅ kanban board | **BORROW** (board UX) |
| **ComposioHQ/agent-orchestrator** | MIT, self-host | ✅ claude-code + 5 others | — (auth undocumented) | ❌ local | ✅ worktree per issue | ⚠️ CLI/issue-driven | **BORROW** (issue→worktree) |
| **Sculptor** (Imbue) | Source-available (Research Preview ToS), not OSI | ✅ **documents sub *or* API** | ⚠️ doesn't say if it wraps first-party CLI | ⚠️ `mngr` remote over SSH/Modal | ✅ container or remote host | ✅ desktop app | **BORROW** (switchable + mngr SSH ref) |
| **Warren** | MIT, self-host | ❌ claude only | ❌ **API key only** | ❌ **single-host** (R-12 unimpl.) | ✅ bwrap sandbox | ✅ control plane + web UI | **BORROW** (control-plane shape) |
| **Meridian** | MIT, self-host | n/a (auth proxy) | ⚠️ SDK-bridge; **ToS-unsettled** | ❌ | n/a | ❌ | **AVOID** for prod (ToS risk) |
| **Terragon (OSS)** | Apache-2.0 snapshot | — | — | cloud sandboxes (dead SaaS) | ✅ container per task | ✅ (was SaaS) | **REFERENCE only** (shut down Feb 2026) |
| **Copilot coding agent** | SaaS | ❌ Copilot only | ❌ | ⚠️ ARC runners, **no macOS** | container | ✅ GitHub UI | **UNFIT** (Mac Mini excluded) |
| **NATS JetStream** | Apache-2.0, self-host | — (substrate) | — | ✅ **disjoint subject filter = queue-to-a-machine** | — | — | **ADOPT** (dispatch substrate) |
| **Tailscale** | freemium, self-host (Headscale) | — (substrate) | — | ✅ mesh VPS↔workers | — | — | **ADOPT** (connectivity) |

**The decisive fact:** no single tool satisfies all of {self-hosted, subscription-first, ToS-legal switchable executor, dispatch-to-a-specific-remote-machine, HITL+dashboard}. Warren is closest on shape but is API-key-only and single-host. Sculptor is the only one documenting subscription-*or*-API switching but is source-available, desktop-centric, and silent on whether it's first-party-CLI-legal. So the control plane + the subscription-legal executor wrapper are **ours to build**; isolation and substrate are **adopt-able**.

## Final adopt-vs-build verdict

- **BUILD:** (1) the VPS control plane — dispatcher, project/feature state, **approval inbox**, dashboard; (2) the **worker-agent**; (3) the **executor abstraction** — a thin wrapper that runs the **first-party `claude` CLI** for the subscription path (ToS-safe) and an API harness for the API path, selected per job/budget.
- **ADOPT:** **container-use** (auth-neutral per-task container+branch isolation — composes with either executor); **NATS JetStream** for dispatch (disjoint subject per machine = "queue to *this* box"; you already run NATS for lifthrasir); **Tailscale** for VPS↔worker connectivity; **Langfuse** for traces/cost; **spec-kit** for spec→issues in the research lane.
- **BORROW (patterns, not code):** Vibe Kanban / agent-orchestrator (board + issue→worktree UX), Warren (control-plane shape), Sculptor's `mngr` (remote-over-SSH executor) and its switchable-executor config, **aina** (single-binary MCP+dashboard spine + manifest-sync of skills/rules/hooks to workers).
- **AVOID / UNFIT:** Meridian and any third-party-harness-on-subscription-OAuth path (ToS-unsettled) for anything production; Copilot coding agent (no macOS runners); Terragon (dead).

## Recommended v1 architecture (concrete — includes the deferred form-factor call)

**Form factor:** **single Go binary** for the control plane (aina-pattern: REST + MCP + embedded Lit dashboard), with **NATS** as the external dispatch/event substrate and **SQLite to start** (→ Postgres if the fleet grows). The worker is a **separate small Go binary**. This blends your two options: single-binary simplicity for the control plane, but NATS for dispatch because it's the *verified* queue-to-a-machine mechanism and you already operate it.

```
 you ──(browser / push / Slack)──► VPS control plane (single Go binary, 24/7)
                                     • Lit dashboard: projects · feature status · APPROVAL INBOX
                                     • REST + MCP · project/feature state (SQLite→PG)
                                     • dispatcher → NATS JetStream (subject per worker)
                                     • Langfuse (traces/cost) · Caddy · Tailscale node
                                            │  jobs on  workshop.job.<machine>
                                            ▼
 worker-agent (Mac Mini first; add boxes later) — Tailscale node
   • subscribes to its own disjoint subject (queue-to-THIS-box)
   • per task: container-use (Dagger) container + branch  [or git worktree]
   • executor abstraction:  ┌ subscription: official `claude` CLI (local OAuth token)
                            └ API: API harness (OpenHands/Aider) for burst/cheap jobs
   • single orchestrator + subagents (NOT a role-play org)
   • streams status + raises approvals back over NATS → branch/PR out

 background lane (metered API + Haiku): feature-research fan-out · triage · schedulers
 future: control plane ⇄ lifthrasir via REST + NATS + HMAC webhooks + MCP (standalone-first)
```

## Suggested phase plan (for sign-off)

- **Phase 0 — Spikes (days):** (a) confirm `claude` CLI headless works under your subscription post-Jun-15 via `claude setup-token`; (b) **measure the cost envelope** — run real coding tasks on the Mac Mini and watch credit burn (this is the one unresolved number and it gates fleet sizing).
- **Phase 1 — Walking skeleton:** control-plane binary (project/feature model + NATS job queue + minimal dashboard) + one Mac Mini worker driving `claude` CLI in a worktree; submit a job by hand → get a branch/PR back.
- **Phase 2 — Switchable executor + safety:** executor abstraction (CLI ⇄ API), container-use isolation, **approval inbox** with push/Slack notifications, HITL gates at plan/PR/destructive/stuck.
- **Phase 3 — Research lane:** multi-agent feature-research fan-out on API/Haiku → drafted specs/GitHub issues (spec-kit), feeding the same approval queue.
- **Phase 4 — Fleet + observability:** add workers (disjoint-subject routing), Langfuse traces, dashboard polish.
- **Phase 5 — lifthrasir integration:** surface approvals/work items via lifthrasir REST+NATS+webhooks+MCP.

## The one remaining open question

**Cost envelope (Part C)** — how many agent-hours a $100/$200 monthly credit buys under automation — was not verifiable from sources (community data is thin and the Jun-15 credit model is brand new). Recommendation: **don't research it further — measure it** in Phase 0 on your actual Mac Mini. That empirical number, not a blog estimate, should size the fleet.

*Sources (Part 2): github.com/BloopAI/vibe-kanban · github.com/ComposioHQ/agent-orchestrator · github.com/dagger/container-use · github.com/imbue-ai/sculptor (+ mngr) · github.com/jayminwest/warren · github.com/rynfar/meridian · github.com/terragon-labs/terragon-oss · github.blog/changelog/2025-10-28 (Copilot ARC) · natsbyexample.com/examples/jetstream/workqueue-stream · docs.nats.io/jetstream.*

---

# Part 3 — The core vision, made concrete

*This part exists to lock the big idea before any build. It leans on Johannes's own assets as clean-room patterns (NOT code reuse): aina's **Learning Graph** spec (durable lessons surfaced to agents before they act) and lifthrasir's Phase-5 agent design (**autonomy grants #381**, **tool safety #387**, **AIHint #380**, **agent memory #385**, **agent eval #386**). aina is company-owned — concepts only.*

## North Star (one sentence)

**One control plane I can open on any device that runs my Claude subscription as a one-person software house — acting on its own for every clean-cut decision, pulling me in only where my judgment actually changes the outcome, and always able to show me exactly where we are in each feature so nothing slips.**

Three properties fall out of that, and the rest of the system serves them:
1. **Any device** — the dashboard is the product; the laptop is just one more client.
2. **Autonomy by exception** — silence is the default for clean-cut work; interrupt only when it matters.
3. **Legible progress, many projects at once** — multiple projects run in parallel; the home view is the portfolio, and one tap into any project shows its progression and what it needs from me.

## Navigation model — portfolio → project → feature

The top-level object is the **project** (usually one repo). Projects run **in parallel** across the worker fleet; the dashboard's job is to make N-at-once legible and to route your attention.

**Level 1 — Portfolio (home, glanceable on a phone):** every project as a card — overall progress, a **"needs you" badge** (count of open Decision-Inbox items), what's actively building right now, and health (failing/blocked/idle). Sort so anything *blocked on you* floats to the top. One unified Decision Inbox across all projects also lives here.

```
WORKSHOP                                   ⬤ 3 need you
┌── Calendar app ──────── 62% ──┐  ┌── Tidder v2 ───────── 28% ──┐
│ 🔄 2 building · ⏸ 1 needs you │  │ 🔄 1 building · ✅ on track  │
└──────────────────────────────┘  └──────────────────────────────┘
┌── Marketing site ────── 90% ──┐  ┌── Lifthrasir connector ─ 5% ─┐
│ ⏸ 2 need you (deploy?)        │  │ 🔬 research phase            │
└──────────────────────────────┘  └──────────────────────────────┘
```

**Level 2 — Project (tap a card):** that project's **feature/epic tree with progression**, its Decision Inbox (filtered to this project), the live activity (which worker is on what, in which worktree), recent AUTO actions (with rollback), and the memory/lessons relevant here. This is the "where are we / what's left / what's blocked" screen.

```
◀ Calendar app                          62% ▸ ⏸ 1 needs you
 Features
  ▸ Calendar sharing      62%  🔄 in review · ⏸ pricing of shared tier?
  ▸ Recurring events      100% ✅ shipped v1.3
  ▸ Mobile polish          0%  ⬜ queued
 Needs you (1)   ▸ "Pricing of shared tier?"   [approve · tweak · later]
 Activity        mac-mini · share-ui · worktree #14 · 4m ago
 Recent auto     ✓ merged dependabot bump  ✓ formatted  (undo)
```

## Pillar 1 — Autonomy without rubberstamping

The failure mode (Part 1) is approval fatigue: ask about everything → you rubber-stamp → approvals become noise. The fix is a **decision classifier** that decides *act vs ask* per action, plus **standing grants** that let you widen autonomy as trust grows. This is lifthrasir's autonomy-grant model (#381: default ask-then-act; user grants autonomous action per scope) fused with tool-safety (#387: blast radius, dry-run, rollback) and aina's confidence/provenance signal.

**The act-vs-ask decision (per proposed action):**

```
AUTO (act, log it, no interrupt)  when ALL hold:
  • reversibility  = easy   (git-revertable / no external side-effects)
  • blast radius   = low    (within the task's worktree/branch/sandbox)
  • confidence     = high   (matches a known convention/lesson; tests green)
  • inside a standing GRANT you've given for this (project, action-type)

ASK (raise to the Decision Inbox)  when ANY hold:
  • irreversible / out-of-sandbox (deploy, prod data, spend money, send external msg, delete)
  • ambiguous (conflicts with a lesson, or no precedent — aina's AMBIGUOUS tier)
  • a real plan fork (architecture choice, scope tradeoff, dependency add)
  • repeated failure / agent stuck (N retries) or eval gate failed
  • new territory not yet covered by a grant
```

**Standing grants** are how you turn rubberstamping into trust over time: "always auto-merge green dependabot-style chores in project X", "auto-open PRs but never auto-merge", "auto-deploy to staging, never prod without me". Every AUTO action is still **logged and reversible** (audit + one-click rollback), so autonomy is *recoverable*, not blind. This is exactly lifthrasir's `reason`-audited, idempotent, permission-scoped mutation model applied to code work.

**The learning loop tightens autonomy:** the memory layer (Pillar 3) feeds the "confidence" input. The more lessons/conventions captured, the more decisions qualify as clean-cut, the less it has to ask. Approvals you make become new grants/lessons → the system asks about that class of thing less next time.

## Pillar 2 — Feature-set progress tracking (the thing you ask for most)

The pain: *"where are we in this larger feature, did we miss anything?"* The fix is to make the **feature/epic a first-class object** with a live, queryable structure — not a chat you have to reconstruct.

**Model:** each feature is an **epic → tasks → steps** tree with explicit state and a **Definition of Done checklist**. It's issue-driven (your style): the epic maps to a GitHub milestone/issue set; tasks map to issues; the workshop keeps the live status the dashboard renders.

```
Feature "Calendar sharing"  ▸ 62% ▸ 1 blocked-on-you
 ├─ ✅ spec approved              (you signed off 2d ago)
 ├─ ✅ API: share endpoints       PR #14 merged
 ├─ 🔄 web: share UI              worker mac-mini · in review by reviewer-agent
 ├─ ⏸  needs you: pricing of shared tier?   ← Decision Inbox item
 ├─ ⬜ tests: permission matrix
 └─ ⬜ docs
 Definition of Done: [x]api [x]events []ui []tests []docs []changelog
 Open questions (3) · Risks (1) · "Have we done this before?" → 2 related lessons
```

**Why this kills the "did we miss anything" problem:**
- The **DoD checklist** is explicit and machine-tracked — the system *knows* docs/tests aren't done and won't call the feature finished.
- A **completeness critic** pass (cheap API/Haiku) runs against the epic before "done": *what's in the spec but has no task? what task has no test? what's unverified?* — surfacing gaps as new tasks. (This is the "completeness critic" pattern, applied to features.)
- The memory layer answers **"have we built/decided this before?"** (aina's prize query) so you don't redo or contradict past work.
- Everything rolls up two levels: a **portfolio of projects** (each with %-complete + a "needs you" badge), and inside each project its **features**, each with the live task tree one tap away (see Navigation model above).

## Pillar 3 — Memory / learning layer (the enabler of 1 & 2)

A workslop-native, clean-room **knowledge store** (aina Learning-Graph pattern; SQLite + the abstracted-store discipline) that the orchestrator queries before acting and writes to after:
- **Conventions / decisions / lessons** per project — feeds Pillar 1's "confidence" and lets the agent act in your style without asking.
- **"Have we tried this?" / "have we built this?"** — feeds Pillar 2's not-missing-anything.
- **Provenance + bi-temporal validity** — lessons can go stale; decisions supersede. Borrowed straight from the spec.
- Sources to ingest later: your merged PRs, the specs you approve, the approvals you make (each approval is a lesson), and your existing CLAUDE.md/skills.

This is the differentiator. Tools like Vibe Kanban orchestrate agents; none of them carry *your* accumulated judgment so the fleet gets more autonomous over time. That's what makes it *your* one-person house and not just a parallel-agent runner.

## "Any device" — concretely

- The control-plane binary serves a **responsive web dashboard** (works on phone) behind Caddy + Tailscale — open it anywhere, no app store.
- **Push + Slack** for the Decision Inbox: a clean-cut-exception arrives as a notification with enough context to **approve/redirect from your phone in one tap** (approve · tweak · take-over-later). Anthropic's own guidance: notifications go where you already are.
- Auth: Google OIDC for you (web) + tokens for workers/CI (aina's multi-auth pattern). One identity, every device.

## What this changes about the v1 plan

It re-prioritizes the phases around the two pillars (the rest of Part 2's architecture stands):
- **Pull the Decision Inbox + autonomy classifier forward** into Phase 2 (it's the heart of the value, not a late add).
- **Make the feature/epic tree + DoD a Phase 1 data-model citizen**, even before multi-worker — because "where are we" is the daily need.
- **Seed the memory layer early** (Phase 2-3) from approvals + merged PRs, so autonomy compounds.

These don't change the build/adopt verdict or the topology — they sharpen *what the dashboard is for*.
