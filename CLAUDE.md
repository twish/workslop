# Workshop — Conventions for Claude

Personal project of Johannes Tveitan. An automated agentic software workshop. See [`README.md`](README.md) for the elevator pitch and [`PROJECT_PLAN.md`](PROJECT_PLAN.md) for status & next action.

**NOT a Mediaflow project.** This is personal, unrelated to Mediaflow. Global rules about Mediaflow conventions (aina MCP, `mfui-web-components`, `.mediaflow.*` environments, Mediaflow CI/CD/style guides) **do not apply here**. Where this project borrows ideas from `aina` or `lifthrasir`, it borrows **patterns only, clean-room — never code** (aina is company-owned; lifthrasir is a separate personal project).

## Read before working

- [`PROJECT_PLAN.md`](PROJECT_PLAN.md) — vision, decisions log, phases, open questions, **current status**.
- [`ARCH.md`](ARCH.md) — the architecture. **Respect the ports & adapters boundaries; do not violate the dependency rule.**
- [`research/2026-06-07-agentic-workshop-brief.md`](research/2026-06-07-agentic-workshop-brief.md) — why every choice was made (subscription/ToS, tool survey, core vision).

## The bar (non-negotiable)

Rigorous, modular, clean, **additive-by-design** — new ideas plug in as new adapters behind existing ports; they never force a teardown. No quick fixes. Architecture before code; review the design before implementing. Quality over speed. If a new idea doesn't fit an existing port, add a *new port* deliberately rather than threading a special case through the core.

**Architecture invariants (from ARCH.md):**
- Dependency rule, one direction only: `domain ← app ← adapters`. The `domain` package imports nothing infrastructural.
- Every foreseeable capability is a port (executor, isolation, feature store, policy, store, memory, dispatch, events, notifications). Implement + register; don't edit the core.
- Policy is data, not code: the act-vs-ask "clean-cut" rules and the feature-tree location are pluggable choices, not hardcoded branches.
- Everything reversible is audited (reason + rollback handle); everything risky is gated.

## Git

- **Identity:** `tveitan <johannes@tveitan.se>` (personal). If repo-local git config isn't set to this, set it before committing.
- **No AI-attribution trailers.** Do not add `Co-Authored-By: Claude`, `Generated with Claude Code`, or similar to commits or PR bodies.
- Commit/push only when asked.

## Stack (decided — see ARCH.md §8)

Go (control-plane single binary + worker binary, one module) · NATS JetStream (dispatch + events) · SQLite→Postgres (`modernc`, CGO-free) · Lit + web components (embedded dashboard) · Caddy + Tailscale · Google OIDC + tokens · Langfuse (traces) · `mark3labs/mcp-go` (MCP) · **adopt** container-use (Dagger) for task isolation.

## Working style (his)

Issue-driven (GitHub issues/milestones), design-first, self-hosted, conversational/high-level delegation, least-privilege permissions.
