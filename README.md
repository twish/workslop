# Workshop

An automated, agentic **software workshop** — a one-person software house run from a single control plane you can open on any device. It builds, reviews, and ships software across **multiple projects in parallel**, acting autonomously on clean-cut decisions and pulling you in only where your judgment changes the outcome — while always showing exactly where each feature stands so nothing slips.

**Status (2026-06-07):** research complete; **Phase 0 — architecture foundation**, in review. No application code yet.

## Start here (read in this order)

1. [`PROJECT_PLAN.md`](PROJECT_PLAN.md) — vision, decisions log, phase plan, open questions, **current status & next action**.
2. [`ARCH.md`](ARCH.md) — architecture: ports & adapters, module boundaries, the seams that keep it additive.
3. [`research/2026-06-07-agentic-workshop-brief.md`](research/2026-06-07-agentic-workshop-brief.md) — the full research (why every choice was made): subscription-vs-API reality, tool survey + adopt-vs-build verdict, and the core-vision design (Part 3).
4. [`CLAUDE.md`](CLAUDE.md) — conventions for working in this repo (rigor bar, git identity, stack).

## In one breath

- **Subscription-first**: heavy coding runs on the Claude subscription via the first-party `claude` CLI (ToS-safe); metered API only for background bits.
- **Hybrid topology**: a 24/7 control plane (VPS) + a fleet of worker dev-boxes (Mac Mini first, extensible) over Tailscale + NATS.
- **Build the spine, adopt the edges**: build the control plane + worker + switchable-executor; adopt container-use / NATS / Langfuse; borrow *patterns* (not code) from aina & lifthrasir.
- **Differentiator**: a memory/learning layer so the workshop accumulates *your* judgment and asks less over time.
