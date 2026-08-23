---
name: fable-orchestration
description: 'Orchestration and delegation policy for Claude Code running under a Claude Pro ($20/mo) subscription — not the Max ($200/mo) tier the upstream Avenox policy assumes. Governs when the main-loop model (Sonnet 5) delegates to Opus vs a cheaper reader tier vs Codex lanes (via codex-fleet/omp-fleet). Triggers on: "delegate this", "spawn a subagent", "use opus", "fan out", "which model should do this", or any multi-model routing decision inside a Claude Code session.'
metadata:
  short-description: Claude-side model routing policy (Pro-tier economics)
  source: 'Adapted from avenoxai/avenoxskills skills/fable-orchestration (Max-tier original); adapted 2026-08-23 for Claude Pro'
---

# Fable Orchestration — Pro-tier adaptation

Upstream source: [`avenoxai/avenoxskills/skills/fable-orchestration`](https://github.com/avenoxai/avenoxskills/blob/main/skills/fable-orchestration/SKILL.md).

The upstream policy is written for a **Claude Max ($200/mo)** stack, where Opus
is described as an "unlimited" delegate tier. On **Claude Pro ($20/mo)** —
what this account actually runs, no `ANTHROPIC_API_KEY` fallback — Opus is
**not unlimited**. Its quota is materially tighter than Sonnet's (shorter
5h/weekly caps), so treating it as a cheap high-fan-out reader burns the
account's scarcest budget on recon instead of judgment. Everything below is
the upstream rule set with that one assumption replaced.

Core law unchanged: **the scarce tier does not do pleb work, and is not
spawned aggressively as a sub-agent.** On Pro, "scarce" now means Opus itself,
not just the main loop.

## The hard rules (Pro-adapted)

1. **Opus as sub-agent: rare and explicit, never for high-volume fan-out.**
   Default to an explicit `model: 'sonnet'` (or `'haiku'` for pure mechanical
   reads) on Agent tool calls. Only pass `model: 'opus'` when the delegated
   task is itself judgment-heavy and the main loop's own Sonnet pass genuinely
   can't carry it — treat it the same way the upstream policy treats spawning
   the main-loop model itself.
2. **Three tiers, not two.** Main loop (**Sonnet 5**, whatever Claude Code is
   actually running), reader/delegate tier (**Sonnet 5** by default for
   recon/context-gathering — same model as the main loop, just parallelized;
   drop to **Haiku 4.5** for pure mechanical high-fan-out sweeps where
   reasoning quality barely matters), and **Codex gpt-5.x lanes** (via
   `codex-fleet` / `omp-fleet`, unaffected by this — separate subscription,
   separate economics). Opus sits above all three as a scarce escalation tier,
   not a routine delegate.
3. **The main loop keeps the big picture.** Architecture, specs,
   contract-sensitive design, integration/conflict resolution, final
   synthesis, judgment calls — done in the main loop (Sonnet), same as
   upstream. Mechanical, scoped, parallelizable work gets delegated to
   Sonnet/Haiku readers or Codex execution lanes.

## Choosing the delegate

- **Codex (`gpt-5.x`, via `codex-fleet`/`omp-fleet`) — default for
  substantial implementation lanes**, unchanged from upstream. Runs on the
  Codex subscription, not Claude quota, so it's the cheapest place to put
  large mechanical or well-specified execution work regardless of Claude plan
  tier.
- **Sonnet 5 (in-session subagent) — default for context gathering.**
  Exploration, codebase mapping, research sweeps, review passes where the
  brief can be loose. This replaces upstream's "Opus for context gathering" —
  same role, cheaper tier, because Pro's Opus budget can't absorb that volume.
- **Haiku 4.5 — high-fan-out mechanical reads only.** "Where does X live",
  inventory sweeps, first-pass grep-and-summarize across many files. Use when
  the task is closer to lookup than reasoning.
- **Opus — reserved, explicit, judgment-heavy only.** Ambiguity resolution a
  Sonnet pass couldn't settle, a second opinion on a consequential decision,
  or a task the operator explicitly names Opus for. Never the default; never
  spawned in a loop.

Rule of thumb: **context gathering → Sonnet (Haiku for pure mechanical
volume); execution once the spec is complete → Codex; judgment / synthesis /
spec-writing → the main loop; Opus → rare, named exceptions only.**

## Exceptions

- If the operator explicitly names a model for a scoped task, honor it for
  that task only, then return to this policy.
- The operator can override any of this per session; absent that, this policy
  stands.
- If the account's plan changes (e.g. upgrades to Max), re-evaluate this file
  against the upstream original rather than assuming the Pro-tier tiering
  still applies — the whole point of this adaptation is that tier availability
  drives the routing, not a fixed preference for Opus or Sonnet.
