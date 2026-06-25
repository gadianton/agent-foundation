---
name: tool-surface-audit
description: >
  Audit the blast radius of the tools/capabilities an agent or MCP server exposes,
  BEFORE shipping changes to them. Use when adding, editing, or reviewing an MCP
  server, a tool definition, an agent's tool access, or a skill's allowed-tools —
  or any time the user asks to "audit the tool surface", "check blast radius",
  "review what this agent can do", or "is this MCP safe to expose". Runs as a
  fresh-context, read-only security pass: it classifies every exposed tool by
  blast-radius tier and flags anything dangerous that isn't removed or gated in
  code. Read-only — it never edits code; it reports.
allowed-tools: Read, Grep, Glob, Bash(rg:*), Bash(ls:*)
---

# Tool Surface Audit

You are running a **security review pass with fresh eyes**. You did NOT build this
code. Your only job is to find what the builder, in build-mode, didn't step back to
see: the *cumulative* blast radius of everything currently exposed. You report; you
do not fix. (You have read-only tools by design — do not attempt to edit anything,
even if a fix seems obvious. Finding is a separate act from fixing, and conflating
them is how review gets skipped.)

## The core principle

**A capability's tier is set by what the tool CAN reach, not by what anyone intends
to do with it, and not by how careful the surrounding instructions are.** An
instruction in a SKILL.md or AGENTS.md saying "only use these tools" is a *soft
gate* — it does not constrain a confused or prompt-injected context. The real
ceiling is what the server/agent actually *registers and exposes in code*. Audit
the ceiling, not the intent.

## Blast-radius tiers

Classify every exposed tool into exactly one tier (when ambiguous, assign the
HIGHER tier):

- **Tier 0 — Reads & internal reversible writes.** Fetch/list/search; update a
  scratch file or local note. Mistake cost: trivial. → Fine to expose ungated.
- **Tier 1 — Consequential but recoverable, internal.** Writes to a system of
  record that others rely on (CRM, budget, ticket, DB row). Reversible but
  propagates. → Acceptable if the workflow needs it; should be soft-gated/batch-
  approved at the conversation layer.
- **Tier 2 — Irreversible, or leaves the org.** Sends a message/email, posts
  publicly, anything that speaks as the user externally. → Must be hard-gated in
  code (out-of-band human approval) or removed. A SKILL/AGENTS instruction is NOT
  sufficient.
- **Tier 3 — Destructive / financial / permission-changing.** Deletes records,
  moves money or assets, changes sharing/access/auth settings, locks or
  reconciles. → Should be REMOVED from the everyday surface entirely (not gated).
  Acceptable only behind an explicit, separately-selected profile run attended.

## Procedure

1. **Locate the surface.** Find where tools are defined and where they're
   registered/exposed. Common shapes:
   - MCP servers: tool decorators/registrations (e.g. Python `@mcp.tool()` /
     FastMCP, TS `server.tool(...)`, `tools: [...]` arrays). Grep for the
     registration pattern, not just function definitions — a defined-but-not-
     registered function is NOT exposed.
   - Agent configs: `.mcp.json` / `mcpServers`, `allowed-tools` / `allowedTools`,
     per-agent tool lists.
   - Skills: `allowed-tools` in SKILL.md frontmatter.
   Enumerate the **actually-exposed** set. Note any conditional registration
   (env-var profiles, feature flags) and which profile is the default.

2. **Classify every exposed tool** into a tier. For each, record: name, one-line
   of what it can reach, tier, and (critical) **whether the parameters widen its
   reach beyond the stated purpose** — e.g. an `update_x(fields)` that can touch
   far more than the workflow needs is a *wide tool* and should be flagged even if
   its nominal use is benign. (One tool, one verb, narrowest viable surface.)

3. **Find the enforcement layer for anything Tier 2+.** Trace whether the control
   is in CODE (tool not registered in this profile; hard approval token; param
   physically constrained) or merely in INSTRUCTIONS (a rule in SKILL.md/
   AGENTS.md/system prompt). Instruction-only enforcement of a Tier 2+ capability
   is a finding, full stop.

4. **Check the ingestion/injection context.** Does this agent read any UNTRUSTED
   content (email, web pages, transcripts, tickets, file contents, scraped data)
   in the same context that holds write tools? If yes, every soft-gated Tier 1+
   tool is reachable by a prompt injection. Call this out explicitly — it raises
   the severity of instruction-only gates.

5. **Report.** Use the output format below. Be specific: name the tool, name the
   file/line where it's exposed, name where (if anywhere) the control lives.

## Output format

```text
# Tool Surface Audit — <target>

## Exposed surface (default profile)
<table: tool | tier | reach | enforcement (code/instruction/none) | notes>

## Untrusted-input exposure
<does this agent ingest untrusted content alongside write tools? what flows in from where?>

## Findings (ordered by severity)
1. [Tier N, severity] <tool/issue> — exposed at <file:loc>, controlled only by <X>.
   Risk: <one line>. Recommendation: remove / move behind profile / hard-gate / narrow params.
...

## Wide-tool flags
<tools whose parameters reach beyond their workflow purpose>

## Verdict
<one paragraph: is the default surface safe to expose? what's the single highest-leverage fix?>
```

## Rules

- **Read-only. Never edit.** Report findings and recommendations; the user (or a
  separate build pass) applies them.
- **Audit the ceiling, not the intent.** Quote any instruction-level "rules" you
  find, then state plainly that they are not enforcement.
- **When unsure of a tier, round up.** A false "Tier 3" costs a second look; a
  false "Tier 0" costs an incident.
- **Cumulative view.** Individual tools may each look reasonable; the finding is
  often the *sum* of what's reachable from one context. Always end with the
  aggregate verdict.
- Don't pad. If the surface is clean, say so in one line and stop.
