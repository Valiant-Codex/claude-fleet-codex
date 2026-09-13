---
type: template
title: Agent Template
description: How to scaffold a new <ORG> agent and what a good agent brain contains (v3 — one always-on CLAUDE.md per agent). Also the single statement of what loads into a session at launch.
tags:
- template
- agent
- okf
status: active
timestamp: 2026-07-24T00:00:00Z
---
# Agent Template

New agents are scaffolded from the **`<ORG>/kb-agent-template`** repository and provisioned onto
a box with `infra/scripts/provision-agent` (which clones the brain + `kb-agent-shared` as a sibling and
wires the `shared/` symlink — no submodule commands). This document is the checklist for what a brain
should contain and how the pieces fit. For the *why* and boundary contract of a new agent, write a
decision in `decisions/` first (a dated `add-<agent>-<role>-agent.md` record is the worked example).

## Repo shape (`kb-agent-<role>-<name>`)

```
CLAUDE.md            ← THE contract: identity, scope, gates, threat model. Symlinked to
                       ~/CLAUDE.md. See "What loads at launch" below.
memory/              ← durable memory (distilled-memory.md + auto/); see policies/memory-policy.md
tools/               ← tool/MCP registry (README.md); secrets are ${ENV} placeholders, never committed
skills/              ← folder-per-skill (<name>/SKILL.md); fleet-common ones symlink shared/skills/*
deploy/              ← topics.tsv, claude-settings.json, output-styles/<agent>.md (runtime wiring)
.mcp.json            ← MCP servers with ${ENV} placeholders — no secrets. NOT read when the session
                       runs with cwd=~ unless passed via --mcp-config (see docs/context-budget.md)
shared -> ../kb-agent-shared   ← committed symlink to the sibling clone (global governance)
```

## What loads at launch

This is the only place in the fleet's documents that states it. Every other document that mentions
what an agent has in context at launch points here instead of restating it.

Five inputs reach a session without the agent asking, and nothing else does:

1. **`CLAUDE.md`**, symlinked to `~/CLAUDE.md`; the topic session runs with cwd = `~`. Block-level HTML
   comments are stripped before injection, so maintainer notes cost no context.
2. **What `CLAUDE.md` `@`-imports**, expanded in full at launch: relative paths resolve against the
   importing file, to a depth of four hops, and an import through the committed `shared` symlink loads
   without an approval dialog. In this fleet that is `memory/distilled-memory.md` and
   `shared/fleet-conduct.md`.
3. **The skills' names and descriptions** — never their bodies, which load when a skill is invoked.
4. **The output style**, `deploy/output-styles/<agent>.md`, symlinked under `~/.claude/output-styles/`.
   Voice and register live there, not in `CLAUDE.md`.
5. **The auto-memory index**, `MEMORY.md` in the runtime's memory directory: the first 200 lines or
   25 KB, whichever comes first; everything past that is silently dropped.

Policies, `owner-profile.md`, `bootstrap.md`, decision records and skill bodies are read on demand or
not at all. A rule that must bind before the agent knows to look for it therefore lives in `CLAUDE.md`
or in a file it imports, and nowhere else.

Limits and levers. The runtime skips a `CLAUDE.md` above 4 MiB; the vendor's guidance is to keep each
file under 200 lines, and imports count in full. There is no byte budget in this fleet: the field
evidence behind release 0.8.10 is that raw length has no
measurable effect on adherence and the number of competing instructions does. So the discipline is
fewer, unambiguous rules — not fewer bytes — and emphasis rationed to the lines whose breach is
expensive, because if many lines are emphasised none stands out. The only always-on number with a
loss attached is the index limit in item 5, and `agentic-divergence-check` asserts it with the vendor's
constants pinned to the docs page they were read from. When the harness changes what it loads, this
section changes and the pointers do not.

## Identity: one file, because one file is what loads

`CLAUDE.md` **is** the agent. It carries, in this order: who the agent is (identity, principles), its
scope and delegation map, the untrusted-content prime directive, the human-confirm gates, the
autonomous-OK list, the threat model, and the safety invariants. Voice and register are in the output
style (item 4 above). `provision-agent` symlinks `CLAUDE.md` to `~/CLAUDE.md`, and the topic unit sets
`WorkingDirectory=%h`, so this is the file the runtime loads.

**Use absolute `~/github/<org>/<brain>/…` paths.** Repo-relative paths do not resolve from cwd=`~`.

Who the owner and the org are lives once, fleet-wide, in `shared/owner-profile.md` — never duplicated
per agent.

### Why this is one file and not three (2026-08-07)

Until 2026-08-07 identity was split across `SOUL.md` (durable: who the agent is) and `OPERATING.md`
(mutable: what it does), with a thin `CLAUDE.md` pointing at both. The split was principled and the
principle was sound. It was also **not what happened at runtime.**

A fleet audit measured the loading path and found there is none: no `@`-import, no hook, no
`~/.claude/CLAUDE.md`, no wrapper flag. `SOUL.md` and `OPERATING.md` entered context **only if the
model chose to obey an instruction telling it to read them.** On the reference deployment's own
transcripts, agents read their `OPERATING.md` in 22–54% of substantial sessions and their `SOUL.md` in
16–47%. Meanwhile 45–70% of each `OPERATING.md` was content that must bind *before* the agent knows to
look for it: the confirm gates, the threat model, the delegation map, the never-commit-secrets
invariant. **The brakes were out of context most of the time.** Every `SOUL.md` even asserted
`Loaded every session alongside OPERATING.md` in its own frontmatter, which was false.

The lesson generalizes past this framework: **a layer that loads only by instruction is not a layer,
it is a suggestion.** If a rule must hold, it belongs in the file the runtime reads by itself. Splitting
by *durability* (rarely-changed identity vs mutable contract) is a good instinct for authoring and a bad
one for loading — git already gives you the durability distinction, for free, without a second file.

### The one-place rule

A fact belongs in the always-loaded layer **only if the agent cannot know to go looking for it.**
Everything else is retrieved on demand, and is stated exactly once.

Always-on, therefore stated verbatim in `CLAUDE.md`: identity, the rules that must bind *before* the
agent knows a policy exists (untrusted-content; data residency and not-a-gestor where they apply), the
confirm gates and their autonomous-OK counterweight, and the delegation map, because an agent cannot
lazily retrieve the knowledge that a task belongs to someone else.

Fleet-common always-on rules are the one exception to "verbatim": they live once in
`shared/fleet-conduct.md` and every `CLAUDE.md` imports them with `@shared/fleet-conduct.md`. A rule
goes there only if it holds for any model and any agent; tuning for one model belongs in that agent's
output style.

Retrieved on demand, therefore stated once in its own file and merely pointed at: everything else —
link lists, path mechanics, historical reasoning, deep procedure. Procedure specifically belongs in a
**skill**, which is the one retrievable layer the runtime advertises by itself.

This is a correctness rule, not a budget. Measured on the reference deployment, a fresh session loads
about 2% of the context window, with MCP schemas deferred: there is nothing meaningful to reclaim, so
do not trim to save tokens. What a duplicated read order costs is not tokens but truth — three copies
drift, each one looks authoritative, and the fleet carried two standing bugs from exactly that:
repo-relative paths that do not resolve from cwd=`~`, and a step ordering a read of `CLAUDE.md`, which
the harness has already loaded before the agent reads anything.

## Skills

Folder-per-skill: `skills/<name>/SKILL.md` with `name` + `description` frontmatter (the `description`
states *what* and *when*). `~/.claude/skills` is a whole-directory symlink to `skills/`, so new skills
are auto-discovered. Fleet-common skills live once in `kb-agent-shared/skills/` and are symlinked in
(`ln -s ../shared/skills/<name> skills/<name>`). See `policies/skills-policy.md` and the `skillify` skill.

## Cross-cutting rules

- **Boundaries:** keep each agent in its lane; name the agent that owns work outside scope. Cross-agent
  handoffs are relayed by <OWNER> in chat (the sending agent writes the brief as Markdown, <OWNER> pastes it
  into the receiving agent's session) — there is no handoff directory.
- **Access:** one agent = one Unix user + one GitHub bot account; least-privilege per
  `policies/github-access-policy.md` and the root agent's `manage-agents` skill (`references/github-access.md`).
- **Governance freshness:** `shared/` stays current via kb-sync (no submodule commands). Runtime and
  portability: the root agent's `manage-agents` skill (`references/agent-ops-and-portability.md`).
