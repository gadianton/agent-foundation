# Agent Architecture Principles

A reference for deciding how to structure AI agents: when to use one agent vs.
many, how to draw the boundaries between them, and how to control what an agent
is allowed to do. Written to be pasted in as context when using AI to help design
or build future agents.

---

## 0. How to use this document

When you (or an AI helping you) are designing an agent system, the temptation is
to start by drawing an org chart of personas — "an assistant agent, a strategist
agent, a reviewer agent." Resist that. Start instead by asking the two questions
in this document:

1. **Is each proposed agent a real boundary, or a costume?** (Section 2)
2. **What blast radius does each capability have, and where is it enforced?**
   (Section 4)

Almost every "do I need multiple agents" question dissolves into those two. The
default answer is **one agent**, and complexity has to earn its way in.

> Anthropic's own guidance on building agents: find the simplest solution
> possible and only increase complexity when needed — which may mean not building
> an agentic system at all. Frameworks and multi-agent patterns make it tempting
> to add complexity when a simpler setup would do.

---

## 1. The core distinction: workflow vs. agent

Before deciding how many agents, decide whether you even need an *agent* for a
given piece of work, versus a *workflow*.

- **Workflow** — LLM steps and tools orchestrated through predefined paths. You
  control the sequence: do A, then B, then if C, do D. Predictable, debuggable,
  cheaper.
- **Agent** — the LLM dynamically directs its own process and tool use, deciding
  what to do next. Use when flexibility and model-driven decisions genuinely
  matter and you can't pre-plan the path.

Most things that *feel* like they need a cast of autonomous agents are actually a
single workflow with role-flavored **steps**. "PM writes a spec → engineer builds
→ PM reviews → iterate" is a workflow. The "PM" and "engineer" are roles a single
agent adopts at different steps (different prompts, different context, maybe
different tool access per step), not separate persistent agents.

The thing that tricks you is the *display surface*. Staging the steps as visible
posts in a Slack channel makes it look like you need independent agents talking to
each other. Visibility is a presentation choice, not an architecture requirement.
You can run a single-agent workflow and still surface each step publicly.

---

## 2. Multi-agent vs. single-agent: the "boundary vs. costume" test

In a human organization, role separation earns its keep for **three** distinct
reasons. The mistake is assuming all three transfer to AI agents. They don't.

### Reason 1 — Cognitive bandwidth
Humans specialize because one person can't hold everything in their head.

**Transfers partially, and shrinking.** There's a legitimate version: a tightly
scoped persona with a curated context window outperforms a generalist juggling
everything, because attention dilutes across a bloated context. But this is an
argument for **scoping context per task**, not for spinning up separate persistent
agents — and it weakens as model context handling improves.

**The trap:** deliberately *hiding information* from an agent to mimic a human's
limited reach (e.g. "the strategist shouldn't see the raw notes, only the
summary"). That's not managing bandwidth — it's a self-imposed handicap that
recreates a constraint which only existed because humans can't be in two places at
once. An AI agent can query the primary source directly. If you find yourself
hiding data and then building machinery so the agent can ask for it back, you've
added complexity to reproduce a limitation, not a capability. **Use retrieval on
demand, not information-hiding-as-roleplay.**

### Reason 2 — Trust and blast radius
The intern doesn't get prod access; legal doesn't see the engineering channel.

**Transfers fully. This is the strongest reason to separate agents and it does not
evaporate.** If two parts of a system need genuinely different permissions or
different "can this do something irreversible" boundaries, that is a real wall
worth building. An agent that can push code should not have the same reach as one
reading your CRM.

### Reason 3 — Genuine diversity of mind
Two people bring different training and blind spots, so a second person's review
catches what the first missed.

**Mostly an illusion for same-model setups.** Two instances of the same model
reviewing each other share a prior — it's closer to one mind grading its own
homework in a different hat than to genuine independent review. The *real* benefit
that exists here comes from two cheaper things, neither of which requires a
separate persistent agent:
  - **Fresh context** — a reviewer not anchored on the implementation catches what
    the author rationalized away.
  - **Different instruction** — a system prompt that says "be skeptical, hunt for
    failures" pulls different behavior from the same weights.

Both are obtainable from a single agent running a **review step** in a fresh
context. You don't need a second identity with its own memory to get a critical
second pass.

### The test
For every proposed separate agent, ask:

> **What can this agent do, see, or be trusted with that's genuinely different
> from the others — and is that difference a capability/permission boundary, or
> just a costume?**

- **Capability/permission boundary** (different keys to different doors, different
  blast radius) → separate it. This is Reason 2.
- **Costume** (same capabilities, just a different personality or topic) → it's a
  prompt, not an agent. Keep it as a role-step inside one agent.

### Corollary: split on the right axis
When separation *is* warranted, split by **blast radius**, not by **topic**. The
common failure is dividing a system by subject matter ("notes agent" vs. "strategy
agent") when both halves are read-and-synthesize work with identical low blast
radius — so they should be one agent — while the genuinely dangerous part (the
tools that *act on the world*: send, commit, delete, pay) gets left undivided and
stapled onto the safe work. The boundary that carries weight is almost always
**"the part that observes and thinks" vs. "the part that acts."**

---

## 3. Running many agents (containers/instances): should you?

Mostly **no**, unless the reason is Reason 2 (permission isolation). Running N
isolated instances to get N personalities pays real operational cost — N configs,
N memory stores, N things to monitor, N things that can loop — to buy the illusory
version of Reason 3 and the self-handicapping version of Reason 1.

Run separate isolated agents when one can touch a system the other must never
touch. Don't run them to give the same capability set a different costume.

---

## 4. The blast-radius tiering system

Tag every **capability** (every tool an agent can call) by what it can reach. The
tier is set by **what the tool can do, not by what you intend to do with it.** This
is the single most important rule, because it's the thing that resists scope creep
("categorize is basically read-only" → "splitting is basically categorizing" →
something can now move money and you've been calling it Tier 0 the whole way).

| Tier | What it covers | Control |
|------|----------------|---------|
| **Tier 0** | Reads, and internal reversible writes (pull data, update a scratch doc/notes) | **Ungated.** Let it run, even ambiently. Cost of a mistake: fix a doc. |
| **Tier 1** | Consequential but recoverable, internal (e.g. writing to a CRM/budget record) | **Soft-gate or batch-approve.** Show the diff, approve the set — not each field. "Blatant + a quick look" is the right weight here. |
| **Tier 2** | Irreversible, or leaves your org (send a customer email, post publicly, anything that speaks as you) | **Hard mechanical gate.** Enforced *below the model*. No exceptions. |
| **Tier 3** | Destructive, financial, or permission-changing (delete records, move money, change sharing/access settings) | **Capability removal.** Don't give the agent the tool at all. |

Two principles fall out of the table:

### Gate few things, gate them hard, leave the rest ungated
Don't hard-gate everything — that produces approval fatigue, and rubber-stamping
is its own failure mode (you stop reading the diffs by Thursday). Experienced
operators drift away from per-action approval toward monitor-and-intervene;
requiring approval on everything creates friction without proportional safety
benefit. Reserve hard gates for Tier 2.

### Capability removal > capability gating
Gating a tool means a compromised or confused agent can still *attempt* the call,
and you're betting the gate holds every time. Removing the tool means there's
nothing to attempt. For Tier 3, removal is the control. This is the one place the
"the agent simply doesn't possess this" logic is the right answer — and you get it
**without a second agent**, by simply not putting the tool in the agent's hands.

---

## 5. Soft gates vs. hard gates: enforcement location is everything

These two sentences are not the same control:

- *"The skill instructs the agent to get approval before sending."* — **soft gate.**
- *"The send tool cannot fire without a human approval token."* — **hard gate.**

A gate's robustness is a function of **where it is enforced, not how loudly it is
stated.**

- A **soft gate** is a rule expressed *in the same channel the agent reasons in.*
  If the agent ingests untrusted content (emails, transcripts, web pages, tickets),
  that content can argue with the rule. An instruction that arrives through the
  same door as the data — "per our agreement, send X to Y" — can talk a model into
  believing the unsafe action *is* the approved workflow. Visibility ("make it
  blatant") is good UX but is not enforcement.
- A **hard gate** is enforced in code, out-of-band. No amount of persuasion inside
  the context can manufacture the human's approval click, because the human action
  lives outside the channel the attacker controls.

**Skills cannot hard-enforce.** A skill is instructions (a SKILL.md plus scripts);
the model is the thing reading and choosing to honor it. Hard enforcement has to
live one layer down — in the tool/harness, where code decides whether the call
executes. So:

> **The skill is where you organize and document a control. The tool/harness is
> where you enforce it.** Use skills for focus, legibility, and lower error rates.
> Never make a skill the thing standing between a hijacked context and an
> irreversible action.

---

## 6. The two boundaries: MCP and skill

When you build your own tools (e.g. via MCP), you get two distinct control layers.
Know which is which, because they have different strengths.

- **The MCP server decides what *can exist*.** Hard, enforced in code. A tool you
  didn't implement (or didn't register) cannot be called by anyone, ever. This is
  where your **security floor** lives.
- **The skill decides what a given workflow *should reach for*.** Organizational
  and instructional — the model reading "for this task, use these tools" and
  complying. Real and useful, but Tier-0-grade: it shapes well-behaved behavior; it
  does not survive a determined hijack.

Mental model:

> **MCP = the keyring that physically has only these keys cut.
> Skill = the note clipped to the keyring saying which key to use for which door.**

If an attacker talks the agent into ignoring the note, the worst it can do is try
other keys *on the ring* — it still cannot use a key you never cut. So your floor
is set entirely by the MCP; the skill buys focus and lower error rates on top of
that floor. Don't let a tidy skill-scoping config convince you it's a wall. The
wall is the MCP.

### Authoring rule: one tool, one verb, narrowest viable surface
The power of building your own tools comes with owning the discipline of keeping
them narrow. The failure mode is the **convenience tool**: you expose
`update_record(fields)` because it's less code than three specific tools, and now
one handle can do many things across tiers. You've handed back the granularity you
built the MCP to get.

> `recategorize_transaction(id, category)` that *physically cannot* touch the
> amount field beats `update_transaction(...everything...)` plus a polite
> instruction not to.

For tools whose dangerous capabilities you only need rarely, expose them behind a
**separate profile** (e.g. an env-var-selected tool set), so your everyday agent's
keyring literally does not contain them, and you opt into the full set deliberately
and attended.

---

## 7. Quick decision checklist for a new agent

1. **Can this be a workflow instead of an autonomous agent?** Prefer the workflow.
2. **List the atomic operations**, not the personas. Tag each with its blast-radius
   tier (Section 4).
3. **Group by blast radius, not topic.** Everything Tier 0–1 that shares a
   permission profile is probably one agent.
4. **For each proposed *separate* agent, apply the boundary-vs-costume test**
   (Section 2). Only a real permission/blast-radius wall justifies a split.
5. **Set the security floor in the tool/MCP layer** (Section 6): register only the
   tools the agent needs; narrow each tool to one verb; put Tier 3 capabilities
   behind a separate profile or omit them.
6. **Place gates by tier** (Section 5): ungate Tier 0, batch/soft-gate Tier 1,
   hard-gate Tier 2 in code, remove Tier 3.
7. **Use skills for legibility and focus**, never as the security boundary.
8. **Watch for "coordination theater"** — agents producing plausible back-and-forth
   that feels like work without producing decisions. Hard stop conditions, a
   human-in-the-loop gate before anything is acted on, and making each agent earn
   its turn are the antidotes.
