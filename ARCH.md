# Workshop — Architecture

*Status: DRAFT v0.1 — Phase 0 foundation, for review before code. Owner: Johannes Tveitan. Date: 2026-06-07.*

Companion: [`research/2026-06-07-agentic-workshop-brief.md`](research/2026-06-07-agentic-workshop-brief.md) (why these choices). This document defines the **module boundaries and interfaces** so the system stays modular, clean, and **additive-by-design** — new capabilities arrive as new adapters behind existing ports, never as a teardown.

---

## 1. Architectural principles

1. **Ports & adapters (hexagonal).** A pure **domain core** (entities + use-cases + port interfaces) depends on nothing infrastructural. Everything concrete — SQLite, NATS, GitHub, the `claude` CLI, Slack, Lit — is an **adapter** plugged into a port. Swapping or adding an implementation is local.
2. **Dependency rule (one direction).** `domain` ← `app` (use-cases) ← `adapters`. Adapters import the domain; the domain never imports an adapter. Enforced by package layout and (later) an import-lint check.
3. **Additive-by-design.** Every capability we can foresee maps to a **named port**. Adding "a new executor", "a new isolation backend", "a new feature backend", "a new notification channel", "lifthrasir integration" = implement an interface + register it. No core edits.
4. **Policy is data, not code.** The two product decisions still open — *what counts as "clean-cut"* and *where the feature tree lives* — are **configurable policy/adapter choices**, not hardcoded branches. Resolving them later is config, not rework.
5. **Everything reversible is audited; everything risky is gated.** Autonomy is only safe because every autonomous action is logged with a reason and a rollback handle, and the risky ones never auto-run.
6. **Two deployables, one module.** The control plane and the worker are separate binaries in one Go module sharing the domain + transport contracts. No duplicated types.

---

## 2. Topology (recap)

```
 you ──(any device: web / push / Slack)──► CONTROL PLANE  (VPS, 24/7, single Go binary)
                                            domain + app + adapters + embedded Lit dashboard
                                            state (SQLite→PG) · dispatch+events (NATS) · MCP · OIDC
                                                    │  jobs on  workshop.job.<worker>
                                                    ▼  status/decisions/results on  workshop.evt.*
 WORKER(s)  (Mac Mini first; add boxes)  ── Tailscale mesh ──  separate Go binary
   subscribes to its own subject · per-task Isolator · Executor (claude CLI | API) · streams back
```

Connectivity = **Tailscale**; dispatch/events = **NATS JetStream** (a worker owns a disjoint subject → "queue to *this* box"). Both are adapters behind ports (§4.7, §4.8) — a different mesh or bus is a swap.

---

## 3. Component map

```
workshop/                         (one Go module)
├── cmd/
│   ├── workshop-server/          control-plane binary (http + mcp + dashboard + dispatcher)
│   ├── workshop-worker/          worker binary (executor + isolation + runner)
│   └── workshop/                 CLI (enroll worker, status, doctor) — aina-pattern, later phase
├── internal/
│   ├── domain/                   ⬅ PURE CORE. entities + ports (interfaces). zero infra imports.
│   │   ├── project.go            Project, Feature(Epic), Task, Step, DoD
│   │   ├── decision.go           Decision (inbox item), Grant
│   │   ├── job.go                Job, Run, AutoAction, Worker
│   │   ├── lesson.go             Lesson, Source, Convention (memory layer)
│   │   └── ports.go              the interfaces in §4
│   ├── app/                      use-cases (orchestration logic) — depends only on domain ports
│   │   ├── planning/             feature → tasks; completeness critic
│   │   ├── dispatch/             pick worker, enqueue, track runs
│   │   ├── autonomy/             the act-vs-ask PolicyEngine (default rule impl)
│   │   └── memory/               lesson write/query use-cases
│   └── adapters/                 concrete implementations of ports
│       ├── store/sqlite/         Store impl (→ store/postgres/ later)
│       ├── bus/nats/             Dispatcher + EventBus impl
│       ├── feature/github/       FeatureStore/IssueSync (GitHub-backed)  ┐ pluggable
│       ├── feature/native/       FeatureStore (workshop-owned)           ┘ (open decision)
│       ├── executor/claudecli/   ExecutorAdapter (subscription, first-party)  ┐ switchable
│       ├── executor/api/         ExecutorAdapter (metered API harness)         ┘
│       ├── isolation/worktree/   Isolator (git worktree)                 ┐ pluggable
│       ├── isolation/containeruse/ Isolator (Dagger container-use)       ┘
│       ├── notify/{web,push,slack}/ NotificationChannel impls
│       ├── auth/                 OIDC (you) + token (workers/CI)
│       ├── mcp/                  MCP tool surface over the app layer
│       └── observ/langfuse/      trace/cost sink
└── web/                          Lit dashboard (embedded via build tag)
```

The worker binary imports `domain` + the executor/isolation adapters + the bus client. The server imports everything else. Shared contracts (job/event payloads) live in `domain`.

---

## 4. The ports (the seams that keep it additive)

Illustrative Go signatures — the contract matters, not the exact shape. Each is the extension point for a class of future idea.

**4.1 ExecutorAdapter** — *runs an agent on a prepared workspace.* The switchable-executor requirement lives here.
```go
type ExecutorAdapter interface {
    Name() string                       // "claude-cli", "api"
    Run(ctx, ExecRequest) (<-chan ExecEvent, error) // streams thoughts/tool-calls/decisions/results
    Capabilities() ExecCaps             // subscription? sandboxed? cost-model
}
```
`claudecli` drives the first-party `claude` CLI headless (subscription, ToS-safe); `api` drives an API harness. Selection is per-job (budget/policy). New model/harness = new adapter.

**4.2 Isolator** — *gives a task a clean, conflict-free workspace and collects the result.*
```go
type Isolator interface {
    Prepare(ctx, Repo, TaskRef) (Workspace, error) // worktree or fresh container+branch
    Collect(ctx, Workspace) (Result, error)        // diff/branch/PR
    Discard(ctx, Workspace) error
}
```
`worktree` and `containeruse` (Dagger) implementations. New isolation backend = new adapter.

**4.3 FeatureStore + IssueSync** — *where the epic→task tree lives and how it mirrors to issues.* The open "GitHub-owned vs workshop-owned" decision is just which adapter is wired.
```go
type FeatureStore interface {
    Upsert(ctx, Feature) error
    Tree(ctx, ProjectID) ([]Feature, error)   // epics→tasks→steps + DoD state
    Progress(ctx, ProjectID) (Progress, error)
}
type IssueSync interface { Push(ctx, Feature) error; Pull(ctx, ProjectID) ([]Feature, error) }
```

**4.4 PolicyEngine** — *the act-vs-ask classifier.* "Clean-cut" is its configuration, not a hardcode.
```go
type PolicyEngine interface {
    Decide(ctx, ProposedAction, []Grant, Confidence) Verdict // AUTO | ASK(reason) | BLOCK
}
```
Default rule impl: AUTO iff reversible ∧ low-blast-radius ∧ high-confidence ∧ covered-by-grant; ASK on irreversible/ambiguous/plan-fork/stuck. Phase-gated variants or a learned policy = another impl. Grants are domain data you edit from the dashboard.

**4.5 Store** — *persistence for all domain aggregates,* behind one interface (aina KnowledgeStore discipline). `sqlite` now, `postgres` when triggers hit. Sub-stores: Projects, Features, Decisions, Grants, Jobs/Runs, Lessons, Workers.

**4.6 KnowledgeStore / LessonSource** — *the memory layer.* `Query(scope)` before acting (feeds Confidence), `Write(lesson)` after. `LessonSource` adapters ingest from approvals, merged PRs, specs, skills (aina Learning-Graph pattern, clean-room).

**4.7 Dispatcher** — *route a job to a chosen worker* (NATS disjoint subject) and track its Run.

**4.8 EventBus** — *publish/subscribe domain events* (`feature.progressed`, `decision.raised`, `run.finished`, `auto.acted`) for the dashboard, notifications, and audit. NATS impl; the future **lifthrasir** integration subscribes here / bridges to lifthrasir's own NATS — an adapter, never a core dep.

**4.9 NotificationChannel** — *deliver a Decision-Inbox item where you are* (web, push, Slack) and accept the one-tap reply (approve/tweak/later).

---

## 5. Domain model (core entities)

```
Project   (≈ one repo)         id, name, repo, status, settings, workspace
 └─ Feature (Epic)             id, project_id, title, spec_ref, dod[], status, progress, blocked_on_user
     └─ Task                   id, feature_id, title, status, assignee(worker), worktree_ref, pr_ref
         └─ Step               id, task_id, label, done
Decision  (inbox item)         id, project_id, feature_id?, kind, question, options[], context, status
Grant     (standing autonomy)  id, project_id?, scope(action-type), bounds, created_at  — widens AUTO
Job/Run   (unit of dispatch)   id, task_id, worker, executor, isolator, status, cost, trace_ref
AutoAction(audited autonomy)   id, run_id, what, reason, reversible, rollback_handle, at
Lesson/Source (memory)         id, statement, rationale, scope, confidence, provenance, t_valid_*
Worker    (fleet node)         id, name, subject, host, caps, last_seen
```
Invariants: a Feature is **never "done" while any DoD item is unchecked**; every `AutoAction` has a `reason` + `rollback_handle`; every mutation carries an actor (you | agent | worker).

---

## 6. Control flow — one feature, idea → shipped

1. **Intake** → a Feature is created (from you, or the research lane). Spec drafted; **ASK** gate: you approve the spec (a real plan fork).
2. **Plan** (`app/planning`) → Feature decomposed into Tasks+Steps with a DoD; completeness critic flags gaps as Tasks.
3. **Dispatch** (`app/dispatch` → Dispatcher) → each Task queued to a worker subject. Worker `Isolator.Prepare` → `ExecutorAdapter.Run`.
4. **Act/ask loop** → for each proposed action the worker proposes, `PolicyEngine.Decide`: **AUTO** → do + record `AutoAction` (reversible); **ASK** → raise a `Decision` to the Inbox + notify you. Memory (`KnowledgeStore.Query`) supplies Confidence so more becomes AUTO over time.
5. **Verify** → tests/eval gates; `Isolator.Collect` → branch/PR; reviewer-agent pass.
6. **Progress** → DoD ticks; `feature.progressed` event updates the dashboard portfolio + project view.
7. **Close** → only when DoD complete + critic clean. Approvals you made along the way → `LessonSource` → memory, so next time it asks less.

---

## 7. Extension points — "build further without tearing down"

| Future idea | How it plugs in (no core change) |
|---|---|
| New coding model / harness | implement `ExecutorAdapter`, register |
| Different sandbox/isolation | implement `Isolator` |
| Feature tree GitHub-owned **or** native | wire `feature/github` or `feature/native` (same `FeatureStore` port) |
| Change what "clean-cut" means | swap/config `PolicyEngine` (or edit Grants) |
| New notification surface | implement `NotificationChannel` |
| New memory/lesson source | implement `LessonSource` |
| SQLite → Postgres | implement `Store`, flip config |
| Different mesh / bus | implement `Dispatcher`/`EventBus` |
| **lifthrasir integration** | an adapter on `EventBus`/REST — subscribes/bridges; core untouched |
| New worker machine | run `workshop-worker`, it registers + gets a subject |

If a future idea does **not** fit an existing port, that's the signal to add a *new port* deliberately — not to thread a special case through the core.

---

## 8. Tech choices (decided; see brief)

Go (control plane single binary + worker binary) · NATS JetStream (dispatch+events) · SQLite→Postgres (`modernc`, CGO-free) · Lit + web components (embedded dashboard) · Caddy + Tailscale · Google OIDC + tokens · Langfuse (traces) · `mark3labs/mcp-go` (MCP) · **adopt** container-use (isolation) · **borrow patterns** from aina (single-binary spine, manifest sync, Learning Graph) and lifthrasir (autonomy grants, tool safety, audit/reason/idempotency) — **patterns, not code; both are owned elsewhere.**

## 9. Out of core (deliberately)

Role-play "org of agents" (evidence-weak for coding — single orchestrator + subagents instead); multi-agent fan-out *except* in the research lane; any third-party harness on a subscription OAuth token (ToS-unsettled). These can return later as adapters if the evidence changes — but not in the core.

## 10. Phase 0 scope

1. **This document** reviewed + agreed.
2. Repo skeleton matching §3 with **port interfaces stubbed** (`domain/ports.go`) and `nop`/in-memory adapters so the wiring compiles and is testable before any real adapter.
3. **Spikes (need the Mac Mini + your subscription):** (a) confirm `claude` CLI headless under your plan via `claude setup-token`; (b) **measure the cost envelope** on real tasks. I'll provide the harness; you run it on the box.

*Next after sign-off: scaffold §3 with compiling interface stubs + in-memory adapters (the walking skeleton's spine), then Phase 1.*
