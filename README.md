# agent-foundation

Portable standards for designing, building, and reviewing AI agents.

This repo is not an application. It's a small set of documents and skills that
encode **how I think agents should be structured and constrained** — meant to be
carried into any project where I'm creating or using agents, so the same
discipline applies whether I'm building an MCP server, wiring up a multi-agent
workflow, or reviewing what an existing agent is allowed to do.

The throughline across everything here: **start with one agent, prefer workflows
over autonomy, split only on blast radius, and enforce the dangerous boundaries
in code — not in prompts.**

## What's here

| Path | What it is | When to reach for it |
|------|------------|----------------------|
| [agent-architecture-principles.md](agent-architecture-principles.md) | The reference doc. How to decide one agent vs. many, where to draw boundaries, and how to control what an agent can do. | Designing a new agent system, or sanity-checking an existing one. Paste it in as context when using AI to help design or build agents. |
| [.claude/skills/tool-surface-audit/](.claude/skills/tool-surface-audit/SKILL.md) | A read-only review skill that classifies every tool an agent/MCP exposes by blast-radius tier and flags anything dangerous that isn't removed or gated in code. | Before shipping changes to an MCP server, a tool definition, an agent's tool access, or a skill's allowed-tools. |

The two are designed to work together: the principles doc defines the
blast-radius tiering system (Tier 0–3); the audit skill is the enforcement pass
that holds a real codebase against it.

## The core ideas, in brief

These are the load-bearing rules. The doc develops each one in full.

- **Workflow before agent.** Most things that feel like they need autonomous
  agents are a single workflow with role-flavored *steps*. Reach for a true agent
  only when you genuinely can't pre-plan the path.
- **Boundary, not costume.** Before splitting work across multiple agents, ask
  whether each proposed agent has a genuinely different capability/permission
  boundary — or just a different personality. Personalities are prompts, not
  agents.
- **Split on blast radius, not topic.** When separation *is* warranted, the line
  that matters is almost always "the part that observes and thinks" vs. "the part
  that acts on the world."
- **Tier every capability by what it *can* reach, not what you intend.** Tier 0
  (reads/reversible) runs ungated; Tier 1 (recoverable) is soft/batch-gated;
  Tier 2 (irreversible/leaves the org) gets a hard mechanical gate; Tier 3
  (destructive/financial/permission-changing) is removed entirely.
- **Enforcement location is everything.** A soft gate is an instruction the model
  can be argued out of. A hard gate lives in code, out-of-band. Skills organize
  and document controls; they can never *be* the control standing between a
  hijacked context and an irreversible action.
- **MCP = the keyring; skill = the note on the keyring.** Your security floor is
  the set of tools you actually register. The skill only buys focus and lower
  error rates on top of that floor.

## How to use this

**Designing a new agent** — read
[agent-architecture-principles.md](agent-architecture-principles.md), and walk the
decision checklist in §7. Bring it along as context for any AI helping you build.

**Reviewing or shipping tool/MCP changes** — run the `tool-surface-audit` skill as
a fresh-context, read-only pass over the code before you ship. It reports; it
doesn't fix.

**Adopting these standards in another project** — copy the principles doc in as
context, and install the skill into that project's `.claude/skills/` (or your
global `~/.claude/skills/`) so the audit is available wherever you work.
