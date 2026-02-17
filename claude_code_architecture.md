> **Important:**
> This document is based on reverse-engineering Claude Code's official documentation and research from various sources across the web. It is intended as a reference for engineers and product managers designing agentic systems.

# Inside Claude Code: The Architecture of an Autonomous Agent

## *A deep dive into the design pillars, primitive tools, and failure-proofing strategies that drive Anthropic's CLI agent.*

*Claude Code is Anthropic's autonomous CLI agent — a terminal-native tool that integrates directly with your local shell, filesystem, and dev environment. It ships with a small set of capability primitives — not 80 specialized tools, not 800. And yet it consistently outperforms agents with hundreds of bespoke integrations. This guide uses Claude Code as a case study to explain why most AI agents fail — and what architectural decisions you can steal for your own products.*


---

***

**The Shift: Why This Matters**

We are entering the third era of LLM applications. We started with **Chatbots** (stateless Q&A), moved to **Workflows** (rigid, code-driven chains like n8n or LangChain), and are now arriving at **Autonomous Agents** (model-driven loops). Claude Code is the first mass-market example of this new architecture.

**TL;DR — The 6 Architectural Shifts**

*   **From Workflows to Loops**: Moving from "Code controls the Model" (DAGs) to "Model controls the Loop" (TAOR). The runtime is dumb; the model is the CEO.
*   **The Harness is the Body**: The AI isn't just a prompt—it's wrapped in a local **Harness** that gives the "Brain" (LLM) a "Body" (Shell, Filesystem, Memory) to act in the real world.
*   **Primitives > Integrations**: Instead of 100 brittle "Jira Plugins," the agent uses **Primitive Tools** (Bash, Grep, Edit) to compose *any* workflow a human engineer can execute.
*   **Context Economy**: The architecture treats the context window as a scarce resource, protecting it with **auto-compaction**, **sub-agents**, and **semantic search** to prevent "Context Collapse."
*   **Solving Universal Failures**: Runaway loops, amnesia, and permission roulette aren't bugs—they are structural constraints. This design turns them into managed features.
*   **Co-Evolution**: The harness is designed to *shrink*. As models get smarter, hard-coded scaffolding (like planning steps) is deleted, making the architecture thinner over time.

***

**Methodology: How I Know**

This architecture was reverse-engineered from runtime transcripts, filesystem artifacts (`~/.claude`), behavioral stress-testing, Anthropic's public documentation and presentations, and my own experience building agentic systems.

> **Disclaimer**: This is an external analysis. The actual internal architecture may differ — I welcome corrections.

## Table of Contents

- [**Part 1: The Problem**](#part-1-the-problem--why-most-ai-agents-fail) — 8 failures of autonomous agents
- [**Part 2: The Approach**](#part-2-the-approach--five-design-pillars) — 5 pillars of failure-proofing
- [**Part 3: The Evolution**](#part-3-the-evolution--from-api-calls-to-agents) — From workflows to loops
- [**Part 4: The Architecture**](#part-4-the-architecture--inside-the-harness) — Inside the harness
    - [§4.1 TAOR Loop](#41-the-agentic-loop-taor)
    - [§4.2 Primitive Tools](#42-primitive-tools-the-power-of-simplicity)
    - [§4.3 Permissions](#43-the-permission-system)
    - [§4.4 Memory](#44-the-memory-system--never-start-from-zero)
    - [§4.5 Context](#45-context-window-management)
    - [§4.6 Task Tracking](#46-task-management--preventing-context-rot)
    - [§4.7 Continuity](#47-session-continuity)
    - [§4.8 UX Layer](#48-the-ux-layer--dual-use-information-design)
- [**Part 5: Extensibility**](#part-5-extensibility--the-declarative-extension-model) — No-code extensions
- [**Part 6: Scorecard**](#part-6-what-you-should-steal) — What to steal for your product
- [**Part 7: Takeaways**](#part-7-key-takeaways) — For PMs and Engineers

### Deep Dive: Technical Reference
- [**D1. Skills**](#d1-skills--prompt-macros)
- [**D2. Sub-Agents**](#d2-sub-agents--isolated-workers)
- [**D3. Agent Teams**](#d3-agent-teams--multi-session-parallelism)
- [**D4. Hooks**](#d4-hooks--deterministic-lifecycle-events)
- [**D5. MCP**](#d5-mcp--the-universal-service-connector)
- [**D6. Plugins**](#d6-plugins--bundle-everything)

### Appendix
- [A. Steerability](#a-steerability--engineering-personality)
- [B. Permission Modes](#b-permission-modes-reference)
- [C. Context Strategies](#c-context-strategies-reference)
- [D. Feature Layering](#d-feature-layering--how-everything-composes)
- [E. Isolation Spectrum](#e-the-isolation-spectrum)

---

## Part 1: The Problem — Why Most AI Agents Fail

Before looking at Claude Code's architecture, it's worth understanding *why* it exists. These failure modes aren't unique to coding agents — they plague **every** agentic system: customer support bots, research assistants, data analysis agents, workflow automators, and internal copilots. If you're building anything that gives an LLM a loop and tools, you will hit these walls.

The table below categorizes the eight structural failure modes that plague autonomous systems—from 'Runaway Loops' where agents burn cash without value, to 'Context Collapse' where memory degradation leads to hallucinations. These aren't just bugs; they are architectural bottlenecks.

```
  ┌──────────────────────────────────────────────────────────────────────────┐
  │                     UNIVERSAL AGENT FAILURE MODES                        │
  │                                                                          │
  │  1. RUNAWAY LOOPS        Agent burns tokens forever, no kill switch      │
  │  2. CONTEXT COLLAPSE     Stuffs everything into one window, hallucinates │
  │  3. PERMISSION ROULETTE  Either asks about everything or trusts blindly  │
  │  4. AMNESIA              Forgets your context between sessions           │
  │  5. MONOLITHIC CONTEXT   One conversation does research + plan + execute │
  │  6. HARD-CODED BEHAVIOR  Extending requires changing source code         │
  │  7. BLACK BOX            Can't see what it did, intercept, or audit      │
  │  8. SINGLE-THREADED      One task, one agent, no delegation              │
  └──────────────────────────────────────────────────────────────────────────┘
```

These aren't theoretical. A customer support agent with #2 starts contradicting its own answers mid-conversation. A research agent with #5 loses track of which sources it already checked. A coding agent with #1 burns hundreds of dollars on a single stuck task. Every agent without persistent memory (#4) makes the user re-explain context every morning, whether that's a codebase, a customer account, or an internal policy.

The question Claude Code answers is: what architectural decisions address *all eight* simultaneously? And because these are universal failure modes, the answers transfer to any agent you build.

> **TL;DR — The Architecture in One Sentence**: A model-agnostic harness that gives any tool-calling LLM filesystem access, a shell, layered memory, and declarative extensibility — all within a bounded autonomous loop governed by composable permissions.

---

## Part 2: The Approach — Five Design Pillars

Claude Code's architecture is organized around five principles. Every feature in the product maps back to at least one.

Claude Code answers these failure modes with five core design pillars: **Model-Driven Autonomy** (the loop), **Context as a Scarce Resource** (compaction), **Layered Memory** (persistence), **Declarative Extensibility** (no-code skills), and **Composable Permissions** (trust). The diagram below maps each pillar to the specific failure mode it solves.

```
  ┌──────────────────────────────────────────────────────────────────────────┐
  │                                                                          │
  │  ┌────────────────┐   The MODEL decides next steps — not a hard-coded   │
  │  │ 1. MODEL-DRIVEN│   orchestrator. The runtime is a simple loop:       │
  │  │    AUTONOMY     │   Think → Act → Observe → Decide.                   │
  │  └────────────────┘   --> Solves #1 (Runaway Loops), #6 (Hard-coded)    │
  │                                                                          │
  │  ┌────────────────┐   Context window is the scarcest resource.          │
  │  │ 2. CONTEXT AS  │   Auto-compaction, sub-agents, forked contexts,     │
  │  │    A RESOURCE   │   and semantic tool search all exist to protect it.  │
  │  └────────────────┘   --> Solves #2 (Collapse), #5 (Monolithic)         │
  │                                                                          │
  │  ┌────────────────┐   6 layers of memory load at session start.         │
  │  │ 3. LAYERED     │   Project conventions, personal preferences,        │
  │  │    MEMORY       │   auto-learned patterns — the agent never starts    │
  │  └────────────────┘   from zero.                                        │
  │                       --> Solves #4 (Amnesia)                           │
  │                                                                          │
  │  ┌────────────────┐   Skills, agents, hooks, MCP, plugins — all         │
  │  │ 4. DECLARATIVE │   configured via .md and .json files. No code       │
  │  │    EXTENSIBILITY│   changes needed to add new capabilities.           │
  │  └────────────────┘   --> Solves #6 (Hard-coded), #7 (Black Box)        │
  │                                                                          │
  │  ┌────────────────┐   Tool-level allow/deny/ask with glob patterns.     │
  │  │ 5. COMPOSABLE  │   Six permission modes from "ask everything" to     │
  │  │    PERMISSIONS  │   "bypass all." Scoped per tool, per path.          │
  │  └────────────────┘   --> Solves #3 (Roulette)                          │
  │                                                                          │
  └──────────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: The Evolution — From API Calls to Agents

Claude Code sits at the third stage of how LLM applications have matured. Understanding this progression explains *why* the architecture looks the way it does.

```
  ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
  │  SINGLE-LLM     │      │   WORKFLOWS     │      │     AGENTS      │
  │   FEATURES      │      │                 │      │                 │
  │                 │      │                 │      │   Human         │
  │  In ──▶ LLM    │─────▶│  In ──▶ Code    │─────▶│     ↕           │
  │         ──▶ Out │      │    orchestrates │      │   LLM ──▶ Act  │
  │                 │      │    LLM calls    │      │         ──▶ Obs│
  │                 │      │    ──▶ Out      │      │     loop...     │
  └─────────────────┘      └─────────────────┘      └─────────────────┘

    Summarization           Code controls the         LLM controls its
    Classification          LLM (DAGs, chains)        OWN trajectory
    Extraction
                            Deterministic flow        Non-deterministic
                            Predictable               Adaptive
                            Rigid                     Flexible
```

*Figure: The architectural shift from rigid code-driven workflows (Step 2) to autonomous model-driven loops (Step 3). Note how control inverts: code no longer orchestrates the model; the model orchestrates the tools.*

**The key shift**: In workflows, *code* decides what the LLM does next. In agents, the *model* decides. This is the fundamental architectural choice — the runtime is a dumb loop, and all intelligence lives in the model.

> **Why Workflows Fail**: Workflows are brittle. A "Research → Plan → Code" DAG works until the research reveals you need to clarify requirements first. In a workflow, that path doesn't exist. This is exactly **Failure Mode #6 (Hard-Coded Behavior)**. Only an autonomous loop can adapt to the unexpected.

Most coding tools today — Cursor, Copilot, Windsurf — are still hybrids. They use workflows for features like tab-completion and inline edits, then switch to an agentic loop for "chat" mode. Claude Code's *core interaction model* is agentic — loop-first — even when the UI adds workflow-like affordances (inline diffs, plan review, IDE integration).

> **[For PMs] A note on "bounded agency"**: In practice, most enterprise deployments land somewhere *between* workflows and full autonomy. High-risk actions (deploy to prod, delete a database) are still locked behind deterministic gates — that's what Hooks (§D4) and the permission system (§4.3) provide. Claude Code's architecture supports this hybrid: the *model* decides the path, but *deterministic guardrails* bound what it can actually do. The loop is autonomous; the environment is not.

### The Architecture is Designed to Shrink

Unlike traditional software where features accumulate, Claude Code's harness is built to *thin out* with each model generation. Boris (creator of Claude Code) describes this as **model-harness co-evolution**:

*   When Sonnet 4.5 shipped, he deleted ~2,000 tokens from the system prompt because the model no longer needed them.
*   When Claude 4.0 launched, they removed roughly half the system prompt.
*   Features like Plan Mode may eventually be un-shipped when the model can infer from user intent that it should plan first.
*   They **actively un-ship tools**: the OS tool was removed once Bash could enforce the same permission boundaries.

The co-evolution is organic, not managed. Anthropic's researchers use Claude Code daily → they hit model limits → those limits inform training → the next model is better → the harness sheds scaffolding. This is the strongest signal that the architecture is sound: *it gets simpler as models improve*.

> **[For Engineers] The design principle**: If you're building an agent harness, design each feature as if it might be deleted in 3 months. If you can't easily remove it, you've coupled too tightly. The Claude Code team's north star: build the most premium experience for the current model, even if that means throwaway work — because what matters is the experience, not the code.

---

## Part 4: The Architecture — Inside the Harness

### The Big Picture

The system is a **Harness** — a runtime shell that wraps any sufficiently capable LLM with tools, memory, and orchestration support. The model is pluggable.

**Philosophy: Radical Simplicity.** The harness runs **locally** on the user's machine — no cloud sandboxes, no mandatory virtualization. This was a deliberate founding choice. From the engineering team:

> *"With every design decision, we almost always pick the simplest possible option. What's the simplest answer to 'where do you run commands?' It's locally."*

This means the agent uses local tools (git, npm, docker) natively, with zero bridging. Safety originally came entirely from the permissions system. As the product matured, an **optional sandboxing layer** was added (`sandbox.enabled` in settings.json) — providing network restrictions, domain allowlists, and command exclusions — but the default path remains local execution governed by permissions.

The diagram below visualizes the **Harness**—the local runtime that wraps the LLM. Key components include the **Skills** layer (tools/prompts), the **Agent Core Loop** (Think/Act/Observe), and the pluggable **Model Layer**. Notice how the harness is structurally model-agnostic, requiring only tool-use capabilities from the underlying LLM.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                  HARNESS                                    │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                                SKILLS                                 │  │
│  │                                                                       │  │
│  │  ┌──────────────┐    ┌─────────────────┐    ┌──────────────────────┐  │  │
│  │  │    Tools     │    │     Prompts     │    │     File System      │  │  │
│  │  │  MCP         │    │  Core Agent     │    │  Process Files       │  │  │
│  │  │  Custom      │    │  Custom         │    │  JIT Code            │  │  │
│  │  │  File System │    │  Workflows      │    │  Example Output      │  │  │
│  │  └──────────────┘    └─────────────────┘    └──────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│         │                                               │                   │
│         ▼                                               ▼                   │
│  ┌─────────────────────────────────────┐      ┌─────────────────────┐       │
│  │          Agent Core Loop            │      │    Enhancements     │       │
│  │                                     │      │                     │       │
│  │   Think ──▶ Act ──▶ Observe         │◀────▶│  Subagents          │       │
│  │     ▲                │              │      │  Auto-Compacting    │       │
│  │     └───── Decide ───┘              │      │  Hooks              │       │
│  │                                     │      │  Memory             │       │
│  └──────────────────┬──────────────────┘      └─────────────────────┘       │
│                     │                                                       │
└─────────────────────┼───────────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            MODEL LAYER (Pluggable)                          │
│                                                                             │
│  The harness supports Claude models and Anthropic-compatible endpoints     │
│  (e.g. via ANTHROPIC_BASE_URL). Its prompts are tuned for Claude, but      │
│  the architecture only requires one LLM capability: tool-use               │
│  (function calling). The design is structurally model-agnostic.            │
│                                                                             │
│       ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐         │
│       │  Haiku   │    │  Sonnet  │    │   Opus   │    │ Anthropic│         │
│       │  (fast)  │    │  (smart) │    │  (deep)  │    │-compat.  │         │
│       └──────────┘    └──────────┘    └──────────┘    │endpoints │         │
│                                                       └──────────┘         │
└─────────────────────────────────────────────────────────────────────────────┘
```

> **Key insight for engineers**: The harness needs exactly one capability from the LLM: tool-use (function calling). Claude Code today ships with Claude models and Anthropic-compatible endpoints. But the *architectural pattern* — a dumb loop + primitive tools + layered memory — transfers to any model. The prompt engineering and steerability techniques would need adaptation per model, but the design itself is portable.

---

### 4.1 The Agentic Loop (TAOR)

The heart of the system. A dead-simple loop that hands *all* decision-making to the model.

This flowchart illustrates the **TAOR** loop (Think-Act-Observe-Repeat). Crucially, the 'Decide' step is owned by the model, not a hard-coded state machine. Guardrails like `maxTurns` and Permission Gates act as external constraints on this autonomous loop.

```
                    ┌──────────────────────────────────────────────────────┐
                    │                   AGENTIC LOOP                       │
                    │                                                      │
  User Prompt ─────▶│  ┌─────────┐    ┌─────────┐    ┌───────────┐       │
                    │  │  THINK  │───▶│   ACT   │───▶│  OBSERVE  │       │
                    │  │         │    │         │    │           │       │
                    │  │ Receive │    │ Text OR │    │ Append    │       │
                    │  │ context │    │ Tool    │    │ results   │       │
                    │  │ + reason│    │ calls   │    │ to history│       │
                    │  └─────────┘    └─────────┘    └─────┬─────┘       │
                    │       ▲                              │              │
                    │       │         ┌──────────┐         │              │
                    │       │         │  DECIDE  │         │              │
                    │       │         │          │◀────────┘              │
                    │       │         │ end_turn │                        │
                    │       │         │   OR     │                        │
                    │       └─────────│ tool_use │                        │
                    │    (tool_use)   └────┬─────┘                        │
                    │                      │ (end_turn)                   │
                    │                      ▼                              │
                    │               ┌────────────┐                        │
                    │               │   OUTPUT   │────────▶ Final Reply   │
                    │               └────────────┘                        │
                    └──────────────────────────────────────────────────────┘

                    Guard Rails:
                    ├── maxTurns cap prevents runaway loops
                    ├── Permission gates pause for human approval
                    └── Model decides when to stop, NOT the orchestrator
```

**Why this matters**: The orchestrator has *no* domain logic. It doesn't know about code, files, testing, or deployment. It just runs the loop. All intelligence comes from (a) the model, (b) the system prompt, and (c) the tools available. This makes the architecture infinitely flexible — change the prompt and tools, and the same loop can do code review, data analysis, or customer support.

#### Task Sizing: Tuning the Autonomy Dial

The TAOR loop runs the same way every time — but how much *human input* it needs varies dramatically. Boris uses a three-tier mental model:

| Tier | Human Input | Implementation |
|:--|:--|:--|
| **Easy** | None — delegate entirely | Tag `@claude` on a GitHub issue. It ships the PR while you do other work. |
| **Medium** | Align on approach first | `Shift+Tab` into plan mode → review/edit the plan → then `Shift+Tab` into auto-accept and let it implement. |
| **Hard** | User drives, Claude assists | Use Claude for research, prototyping, and tests — but you write the core logic yourself. |

The boundary between tiers **shifts with every model**: what was "hard" with Sonnet 3.5 becomes "medium" with 4.0 and may become "easy" with the next generation. Plan mode itself may eventually be un-shipped when the model can infer the right autonomy level from context.

> **Power-user technique: Throwaway prototyping.** Even if your spec is vague, let Claude implement it once. Watch where it misunderstands. Use those learnings to write a much better spec for the real implementation. This is free with agents — you'd never throw away an engineer's first pass — and it consistently produces better plans than pure upfront spec-writing.

> **Power-user technique: "Ask me questions."** The model doesn't naturally ask clarifying questions — it tries to infer your intent. In plan mode, Boris explicitly prompts: *"We're brainstorming. Please ask me questions if there's anything you're unsure about."* This produces dramatically better plans.

---

### 4.2 Primitive Tools: The Power of Simplicity

This is the most underrated design decision in the entire architecture. Claude Code is built on a small number of **capability primitives** — read, write, execute, connect — implemented via a modest set of built-in tools. The early versions shipped with just 8; the count has grown as features like `TodoWrite`, `AskUserQuestion`, and `WebFetch` were added. But the philosophy remains the same: *generic OS primitives, not domain-specific wrappers*.

Instead of bespoke integrations (e.g., 'RunPytest'), Claude Code relies on four **Capability Primitives**: Read, Write, Execute, and Connect. The diagram below shows how `Bash` acts as a universal adapter, allowing the model to compose any workflow from standard CLI tools.

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │                    THE PRIMITIVE TOOL PHILOSOPHY                     │
  │                                                                     │
  │  Instead of building specialized tools like "CreateReactComponent"  │
  │  or "RunPytestSuite," Claude Code gives the model the SAME tools   │
  │  a human developer uses: read files, write files, run commands.    │
  │                                                                     │
  │  ┌─── READ (understand) ────────────────────────────────────────┐  │
  │  │                                                              │  │
  │  │   Read ── read any file            "cat, less, head"         │  │
  │  │   Glob ── find files by pattern    "find, ls"                │  │
  │  │   Grep ── search file contents     "grep, ripgrep"           │  │
  │  │                                                              │  │
  │  └──────────────────────────────────────────────────────────────┘  │
  │                                                                     │
  │  ┌─── WRITE (modify) ──────────────────────────────────────────┐  │
  │  │                                                              │  │
  │  │   Write ── create / overwrite       "echo > file"            │  │
  │  │   Edit ─── surgical patch           "sed, patch"             │  │
  │  │                                                              │  │
  │  └──────────────────────────────────────────────────────────────┘  │
  │                                                                     │
  │  ┌─── EXECUTE (anything) ──────────────────────────────────────┐  │
  │  │                                                              │  │
  │  │   Bash ─── run any shell command    THE UNIVERSAL TOOL       │  │
  │  │            npm test, git commit,    Every CLI tool ever       │  │
  │  │            docker build, curl...    written is now available  │  │
  │  │                                                              │  │
  │  └──────────────────────────────────────────────────────────────┘  │
  │                                                                     │
  │  ┌─── CONNECT ─────────────────────────────────────────────────┐  │
  │  │                                                              │  │
  │  │   WebFetch ── HTTP requests          "curl"                   │  │
  │  │   Task ────── spawn sub-agent       "fork"                   │  │
  │  │                                                              │  │
  │  └──────────────────────────────────────────────────────────────┘  │
  │                                                                     │
  │  WHY THIS WORKS:                                                    │
  │  ┌──────────────────────────────────────────────────────────────┐  │
  │  │ Bash("npm test")           ──▶  runs your test suite        │  │
  │  │ Bash("docker compose up")  ──▶  starts your infra          │  │
  │  │ Bash("git diff HEAD~3")    ──▶  reviews recent changes     │  │
  │  │ Bash("psql -c 'SELECT'")   ──▶  queries your database      │  │
  │  │ Bash("kubectl get pods")   ──▶  checks your cluster        │  │
  │  │                                                              │  │
  │  │ No special tools needed. Bash IS the universal adapter.      │  │
  │  └──────────────────────────────────────────────────────────────┘  │
  └─────────────────────────────────────────────────────────────────────┘
```

> **The takeaway for builders**: You don't need to build a "ToolForEveryAPI." Give a capable model filesystem access and a shell, and it can compose *any* workflow from existing CLI tools. Bash is the most powerful tool in the agent's kit because *every developer tool ever built* is accessible through it. The model brings the intelligence; the shell brings the surface area.

An important corollary: the Claude Code team **actively deletes tools**. With every smarter model release, they remove prompt instructions and tool wrappers that the model no longer needs. When Claude 4.0 launched, they deleted roughly **half the system prompt**. This is the opposite of what most teams do — and it's a sign that the architecture is working. As models improve, the harness gets *thinner*, not thicker.

> **[For Engineers] Why Agentic Search, Not Vector Embeddings?** Claude Code initially used vector embeddings for code search but abandoned them. The reasons: (1) Continuous reindexing is fragile — local changes, stale indices, race conditions. (2) Embeddings expand the security and deployment surface area, which is a hard sell for enterprise adoption. (3) **Agentic search — where the model iteratively uses Bash, Grep, and Glob to navigate codebases — reaches the same accuracy with a far simpler deployment story.** If users want semantic search, they can add it via MCP — but the default is deliberately embedding-free. This is the primitive-tools philosophy applied to retrieval: let the model compose its own search strategy from basic tools rather than building a specialized RAG pipeline.

---

### 4.3 The Permission System

With great power (Bash!) comes the need for fine-grained control. The permission system is what makes it safe to give an AI a shell.

**How it works**: The client performs **static analysis** on every tool call before execution. It matches the command/path against a multi-tiered whitelist (Global > Project > User). No call reaches the OS without passing through the resolver.

Every tool call passes through the **Permission Resolver** before execution. As shown below, this static analysis layer checks Deny rules first, then Allow rules, and finally the current Permission Mode (e.g., 'Plan' vs 'Default') to determine if the action proceeds automatically or requires user approval.

```
  Tool Call: Bash("rm -rf /tmp/build")
       │
       ▼
  ┌───────────────────────────────────────────────────────────────┐
  │                    PERMISSION RESOLVER                         │
  │                                                               │
  │  1. Check DENY rules ─── Write(/etc/*) ──▶ BLOCKED (stop)    │
  │       │ (no match)                                            │
  │       ▼                                                       │
  │  2. Check ALLOW rules ── Bash(npm test) ──▶ AUTO-RUN (go)    │
  │       │ (no match)                                            │
  │       ▼                                                       │
  │  3. Check Permission Mode                                     │
  │       │                                                       │
  │       ├── default ────────▶ ASK user                          │
  │       ├── acceptEdits ────▶ ASK (bash), AUTO (edits)          │
  │       ├── plan ───────────▶ BLOCKED (write tools)             │
  │       ├── dontAsk ────────▶ AUTO-RUN (if in allow list)       │
  │       └── bypassPermissions ▶ AUTO-RUN (everything)           │
  │                                                               │
  └───────────────────────────────────────────────────────────────┘

  Rule Format:   Tool(glob-specifier)
  Examples:      Bash(npm *)     Read(/src/**)     Write(/etc/*)
```

The trust spectrum — from `plan` (read-only) to `bypassPermissions` (full auto) — is what separates a toy demo from a production tool. Teams share `settings.json` files that whitelist common commands so every developer doesn't have to approve `npm test` individually.

> **[For Engineers] A known limitation**: Static analysis of shell commands is inherently imperfect. An obfuscated `rm -rf` piped through `base64 | sh` won't be caught by glob-pattern matching. The permission system is a strong *policy* layer, not a sandbox. For defense-in-depth, pair it with the optional sandboxing layer (`sandbox.enabled`) and `PreToolUse` hooks that can inspect and block commands programmatically before execution.

---

### 4.4 The Memory System — Never Start From Zero

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    SESSION CONTEXT WINDOW                        │
  │                                                                 │
  │  At session start, all layers are loaded and merged:            │
...
  │  All layers are ADDITIVE — they merge, they don't override.     │
  │  When instructions conflict, the LLM uses judgment              │
  │  (more specific > more general).                                │
  └─────────────────────────────────────────────────────────────────┘
```

This diagram stacks the six layers of context loaded at session start—from global organization policies to specific user quirks. These layers are additive, ensuring the agent never starts from a blank slate:
1.  **L1 Managed**: Organization-wide policies (`/Library/.../CLAUDE.md`)
2.  **L2 Project**: Repository conventions (`./CLAUDE.md`)
3.  **L3 Rules**: Feature-specific instruction sets (`./.claude/rules/*.md`)
4.  **L4 User**: Personal preferences (`~/.claude/CLAUDE.md`)
5.  **L5 Local**: Git-ignored overrides (`./CLAUDE.local.md`)
6.  **L6 Auto-Memory**: Learned patterns from previous sessions (`MEMORY.md`)

**The Auto-Memory loop** is particularly elegant: the agent notices patterns in how you work (preferred test framework, code style, architectural patterns) and writes them to `MEMORY.md`. Next session, it reads them back.

> **The Context File Pattern**: `CLAUDE.md` isn't just a readme — it's injected into the context window on *every* turn. This single feature reportedly makes a "night and day" difference in performance. It grounds the model in project-specific realities (e.g., "Use pnpm," "Ignore legacy/") that cannot be inferred from code alone. This pattern is now an industry standard: Cursor has `.cursorrules`, Windsurf has `rules/`, and nearly every agent framework has adopted some variant.
>
>
> **Advanced Pattern: The Map, Not The Encyclopedia.** A monolithic `CLAUDE.md` eventually rots. Advanced implementations treat the root file as a strictly minimal *Table of Contents* (pointer to `docs/design`, `docs/api`, `docs/plans`) rather than a full instruction manual. This forces the agent to navigate the documentation structure intentionally, preventing context window pollution with irrelevant rules.

---

### 4.5 Context Window Management

The context window is a fixed-size resource (typically 200K tokens). Without management, it fills up and the agent starts hallucinating or losing track of earlier decisions.

As the 200k token window fills (illustrated below), the generic **Context Manager** triggers auto-compaction. It replaces the raw transcript with a lossy summary, preserving the *decisions* while freeing up space for new work.

```
  ┌────────────────────────────────────────────────────────────────┐
  │                     CONTEXT WINDOW (200K tokens)               │
  │                                                                │
  │  ████████████████████████████████████████████████████████░░░░░  │
  │  ◀──────────── approaching context limit ──────────────▶       │
  │                  ⚠ AUTO-COMPACTION TRIGGERS                    │
  │                                                                │
  │  ┌──────────────────────┐     ┌──────────────────────────────┐ │
  │  │  BEFORE              │     │  AFTER                       │ │
  │  │                      │     │                              │ │
  │  │  Turn 1: explored    │     │  ┌────────────────────────┐  │ │
  │  │  Turn 2: read 40     │────▶│  │ SUMMARY:               │  │ │
  │  │          files       │     │  │ • Explored /src         │  │ │
  │  │  Turn 3: planned     │     │  │ • Edited auth module    │  │ │
  │  │  Turn 4: edited      │     │  │ • Tests passing         │  │ │
  │  │  Turn 5: tested      │     │  │ • Working on refactor   │  │ │
  │  │  Turn 6: refactored  │     │  └────────────────────────┘  │ │
  │  │  ...50 more turns    │     │                              │ │
  │  └──────────────────────┘     └──────────────────────────────┘ │
  │                                                                │
  │  ██████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │
  │  ◀─ ~10% ─▶  ◀──────────── 90% FREE ──────────────────────▶   │
  └────────────────────────────────────────────────────────────────┘
```

> **[For Engineers] Compaction is lossy — by design.** The summary replaces raw conversation turns, so exact line numbers, full file contents, and intermediate reasoning are discarded. This is a deliberate trade-off: you lose granularity but preserve *intent* and *decisions*. The model can always re-read files with the `Read` tool if it needs exact content. The `PreCompact` hook (§D4) lets you inject critical context that *must* survive compaction — think of it as pinning important facts before the summary is generated. For long-running tasks, the `TodoWrite` tool (§4.6) acts as a structured scratchpad that persists independently of compaction.

---

### 4.6 Task Management — Preventing Context Rot

When sessions get long, models forget the plan. This is "context rot" — the agent enthusiastically starts a 10-step task, but by step 7, it's lost track of steps 8-10.

Claude Code solves this not just with prompts, but with a **dedicated tool**:

*   **The Todo Tool**: The model manages its own task list via a `TodoWrite` tool. It explicitly adds, updates, and completes tasks — like a developer managing a checklist.
*   **Why it works**:
    *   **External Memory**: The list persists even if the context window is compacted. It's a structured scratchpad that survives summarization.
    *   **Course Correction**: The model can "interleave thinking" — creating a plan, discovering a missing dependency mid-task, adding a new item, and resuming. This is dramatically better than rigid multi-agent handoffs (Planner → Coder → QA).
    *   **System Reminders**: If the list is empty, an invisible `<system-reminder>` tag nudges the model: *"Your todo list is empty. If working on a task, use TodoWrite."* The user never sees this.

---

### 4.7 Session Continuity

Sessions aren't disposable. They persist, branch, and roll back — just like git.

Sessions are structured like git branches. The diagram below shows how a user can 'rollback' to a previous checkpoint (CP3) or 'fork' a session to explore a new path (Session B) without losing the original state.

```
  Session A (branch: main)
  ┌──────────────────────────────────────────────────────────────┐
  │  CP1          CP2          CP3                               │
  │   ●────────────●────────────●────────────── ▶ current state  │
  │   │            │            │                                │
  │   │            │            └── git snapshot (before edit)   │
  │   │            └── git snapshot                              │
  │   └── git snapshot                                           │
  │                                                              │
  │   Esc key ──▶ rollback to CP3                                │
  │   --continue ──▶ resume this session                         │
  └──────────────────────────────────────────────────────────────┘
          │
          │  --fork-session
          ▼
  Session B (branched from CP2)
  ┌──────────────────────────────────────────────────────────────┐
  │   ●────────────●──────────────── ▶ independent exploration   │
  │  CP1          CP2                                            │
  └──────────────────────────────────────────────────────────────┘
```

---

### 4.8 The UX Layer — Dual-Use Information Design

A critical but often-overlooked architectural decision: **how much do you show the human?** Claude Code's answer is a principle called **"collapsed but available"** — minimize by default, but never hide information.

The UX design follows a **Three-Layer Model**:
1.  **Default View (Minimal)**: Collapsed tool calls for readability.
2.  **Full Transcript (Ctrl+O)**: Exact inputs/outputs for deep inspection.
3.  **Ambient Teaching**: Contextual cues (e.g., "Thinking mode active") that appear only when relevant.

```
  ┌────────────────────────────────────────────────────────────────────────┐
  │                     INFORMATION DISPLAY LAYERS                          │
  │                                                                        │
  │  Layer 1: DEFAULT VIEW (minimal)                                       │
  │  ┌──────────────────────────────────────────────────────────────────┐   │
  │  │  Tool calls shown as collapsed summaries                        │   │
  │  │  "Read 12 files" / "Ran npm test" — one line each               │   │
  │  └──────────────────────────────────────────────────────────────────┘   │
  │           │                                                            │
  │           ▼  Ctrl+O (raw transcript mode)                              │
  │  Layer 2: FULL TRANSCRIPT                                              │
  │  ┌──────────────────────────────────────────────────────────────────┐   │
  │  │  Exact view the model sees — every tool call, every output      │   │
  │  │  Complete inspectability for debugging and understanding        │   │
  │  └──────────────────────────────────────────────────────────────────┘   │
  │                                                                        │
  │  Layer 3: AMBIENT TEACHING                                             │
  │  ┌──────────────────────────────────────────────────────────────────┐   │
  │  │  Tips shown while Claude works — surface features in context    │   │
  │  │  Notification bar: new models, changelog, thinking mode status  │   │
  │  │  "Use Ctrl+O to expand" appears only when relevant              │   │
  │  └──────────────────────────────────────────────────────────────────┘   │
  └────────────────────────────────────────────────────────────────────────┘
```

The deeper principle: **everything in the UI is dual-use** — designed for both the human and the model simultaneously. What the user sees, the model also sees. Slash commands can be called by either party. The bash mode (`!` prefix) shows output to both human and model.

This has two consequences:

1.  **Tools designed for humans work for the model, too.** Making a tool understandable to an engineer makes it understandable to the LLM. There's no separate "model API" — the interface is shared.
2.  **The model teaches the user the tool.** When the model calls a slash command, the user sees it happen and learns "oh, I can do that too." Users can also ask Claude Code about its own features — it knows how to look up its own docs.

> **[For PMs] Progressive disclosure, not progressive complexity.** The key UX decision is that features are revealed *contextually*, not upfront. "Use Ctrl+O to expand" appears only when there's a collapsed tool result — not as a blanket instruction. Tips appear while Claude works, not as a tutorial. The result: new users aren't overwhelmed, but power users can go as deep as they want.

---

## Part 5: Extensibility — The Declarative Extension Model

This is where Claude Code becomes a *platform*, not just a tool. There are six extension mechanisms, and **none require writing code** — everything is `.md` files, `.json` config, or shell scripts.

> **The product philosophy behind this openness: Latent Demand.** Boris describes this as the primary way Claude Code decides what to build next. The idea: build a product that is **hackable enough that power users abuse it for use cases it wasn't designed for** — then productize the abuse. Every extension point in the table below exists because someone hacked Claude Code to do something unexpected: people wrote slash commands for non-coding workflows → the team formalized them. People piped Claude Code output like a Unix utility → the team built the SDK. People used Claude Code for blog writing, expense filing, and market research → the team rebranded from "Claude Code SDK" to "Claude Agent SDK." The extension model isn't just architecture — it's a **demand-sensing mechanism**.

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                     EXTENSION MODEL                                   │
...
  │                                                                      │
  │  ALL ARE DECLARATIVE. No TypeScript. No Python. No build step.       │
  └──────────────────────────────────────────────────────────────────────┘
```

The table above illustrates the six declarative extension points. Note that from simple instructions to full plugins, every customization uses Markdown or JSON—no TypeScript required:
1.  **CLAUDE.md**: Persistent interactions
2.  **Skills**: Propmt macros & workflows
3.  **Sub-Agents**: Isolated worker agents
4.  **Hooks**: Lifecycle event scripts
5.  **MCP Servers**: External service bridges
6.  **Plugins**: Bundles of all the above

**Why this matters for adoption**: Declarative config means a PM can write a `CLAUDE.md` file. A DevOps engineer can add a hook. A platform team can publish a plugin. Nobody needs to fork the product or write an adapter. This is the Terraform model applied to AI tooling.

---

## Part 6: What You Should Steal

Regardless of whether you use Claude Code, build a competitor, or design agentic features in your own product — these patterns transfer.

### If You're Building an Agent

|Pattern|Why It Works|
|:--|:--|
| **TAOR loop + primitive tools** | ~50 lines of loop logic + a shell gives you infinite surface area. Don't build 100 tools. |
| **Sub-agents for isolation** | Don't force one context to do research AND implementation. Solves **#2 (Context Collapse)** and **#5 (Monolithic Context)**. |
| **Todo tool for task tracking** | Prevents context rot. The model manages its own scratchpad — not a rigid planner. |
| **Context file (CLAUDE.md)** | Inject project-specific ground truth on every turn. Performance goes from frustrating to magical. |
| **Layered memory** | Don't make the user re-explain. Load org → project → user → auto-learned context at startup. |
| **Hooks for determinism** | Lint on save, audit on shell, gate on deploy — *without* the LLM. Deterministic where you need guarantees. |

### If You're Evaluating AI Agents


Use the 8 failure modes as a **scorecard**:

| Failure Mode | Question to Ask |
|:--|:--|
| Runaway loops | Does the tool have a turn limit? Can I kill a stuck session? |
| Context collapse | Does it manage context window size? How? |
| Permission roulette | Can I whitelist safe commands and block dangerous ones? |
| Amnesia | Does it remember my project across sessions? |
| Monolithic context | Can it delegate subtasks to isolated contexts? |
| Hard-coded behavior | Can I extend it without writing code? |
| Black box | Can I intercept, audit, or hook into its actions? |
| Single-threaded | Can it run parallel tasks? |

### If You're a PM Designing AI Features

| Insight | What It Means For Your Product |
|:--|:--|
| **Permissions are UX** | The trust spectrum (read-only → ask → auto → bypass) is how you ship AI to enterprises. Without it, you're stuck in demo mode. |
| **Memory is a product feature** | Users expect agents that learn. Auto-memory isn't over-engineering — it's table stakes for retention. |
| **"Delete code on model upgrade"** | As models get smarter, your scaffolding should shrink. If your product gets *more* complex with each model release, your architecture is wrong. |

---

## Part 7: Key Takeaways

### For Product Managers

| Insight | Why It Matters |
|:--|:--|
| **The model is the brain, the harness is the body** | You can swap the brain (model) without rebuilding the body (tools, memory, permissions). Plan for model-agnosticism. |
| **Generic beats specific** | A small set of capability primitives outperforms 100 specialized tools. Invest in composability, not coverage. |
| **Memory is a product feature** | Users expect agents that remember. A 6-layer memory system is not over-engineering — it's table stakes. |
| **Permissions are UX** | The trust spectrum (plan → default → dontAsk → bypass) is what makes the difference between "toy" and "production-ready." |
| **Extensibility is adoption** | Declarative config (`.md` files, not code) means non-engineers can extend the system. This dramatically expands your user base. |

### For Engineers

| Insight | Why It Matters |
|:--|:--|
| **Build a dumb loop, not a smart orchestrator** | The TAOR loop has ~50 lines of logic. All intelligence is in the model + prompt. This is dramatically easier to maintain and debug. |
| **Bash is your most powerful tool** | Don't build tool wrappers for `npm test` or `git commit`. Give the model a shell and let it compose. |
| **Context is your hardest constraint** | Every architectural decision — sub-agents, compaction, forked contexts, tool search — exists to manage a single 200K-token budget. Design for it from day one. |
| **Think in layers, not monoliths** | Memory (6 layers), permissions (tool + specifier + scope), extensions (skill → agent → team) — the pattern is always layered composition. |
| **Hooks are underrated** | Deterministic scripts at lifecycle events give you linting, auditing, security gates, and telemetry — without touching the LLM. |
| **Delete code when models improve** | If you're adding scaffolding with every release, you're fighting the model. The harness should get *thinner* over time. |

### Closing the Loop: All 8 Solved

Remember the 8 universal failure modes from Part 1? Here's where each one gets killed:

| # | Failure Mode | What Solves It | Section |
|:--|:--|:--|:--|
| 1 | **Runaway Loops** | `maxTurns` cap + model-driven stop signal (not hard-coded exit) | §4.1 TAOR Loop |
| 2 | **Context Collapse** | Auto-compaction at ~50% + sub-agents with isolated context windows | §4.5 + D2 |
| 3 | **Permission Roulette** | 6 permission modes + tool-level allow/deny/ask with glob patterns | §4.3 Permissions |
| 4 | **Amnesia** | 6-layer memory system loads at session start; auto-memory persists learned patterns | §4.4 Memory |
| 5 | **Monolithic Context** | Sub-agents fork isolated TAOR loops; Agent Teams run parallel peers | D2 + D3 |
| 6 | **Hard-Coded Behavior** | Declarative extensions (Skills, Agents, Hooks, MCP, Plugins) — no code changes needed | §Part 5 |
| 7 | **Black Box** | Hooks fire at every lifecycle event; deterministic scripts for audit, lint, and gates | D4 Hooks |
| 8 | **Single-Threaded** | Sub-agents (child delegation) + Agent Teams (parallel peers) | D2 + D3 |

Every architectural choice in this guide exists to solve one or more of these problems. If you're building your own agent — in any domain — use this table as your checklist.

### The Architecture in One Sentence

> **A model-agnostic harness that gives any tool-calling LLM filesystem access, a shell, layered memory, and declarative extensibility — all within a bounded autonomous loop governed by composable permissions.**

---
---

# Deep Dive: Technical Reference

*The sections below provide granular detail on each extension mechanism. If the main article gave you the "what" and "why," this section gives you the "how."*

---

## D1. Skills — Prompt Macros

Skills are reusable instruction bundles that inject context or trigger workflows via `/slash-commands`.

```
  my-skill/
  ├── SKILL.md            # Instructions (required)
  ├── reference.md        # Extra docs
  ├── examples/           # Example outputs
  └── scripts/            # Executable scripts

  Invoke:   /deploy                    (slash command)
            /review my-file.ts         (with arguments)
            Auto-triggered by Claude   (reads description field)
```

| Key Field | Purpose |
|:--|:--|
| `disable-model-invocation: true` | Pure context injection, zero LLM cost |
| `context: fork` | Run in isolated context (doesn't pollute main) |
| `allowed-tools` | Restrict available tools during execution |

---

## D2. Sub-Agents — Isolated Workers

Sub-agents run their own TAOR loop in a separate context window and return a summary. They are the primary mechanism for **context isolation** — offloading heavy research, testing, or exploration without polluting the main conversation.

### Built-in Sub-Agents

Claude Code ships with three built-in sub-agents, each optimized for a different job:

| Built-in Agent | Model | Tools | Use Case |
|:--|:--|:--|:--|
| **Explore** | Haiku (fast) | Read-only (Read, Grep, Glob) | File discovery, codebase exploration |
| **Plan** | Inherited | Read-only (Read, Grep, Glob) | Codebase research for planning |
| **General-purpose** | Inherited | All tools | Complex research, multi-step operations |

### How Sub-Agents Work

The diagram below demonstrates **Context Isolation**: The main agent delegates a task (e.g., "explore") to a Sub-Agent. The Sub-Agent runs its own isolated TAOR loop, consumes tokens, and returns only a *summary* to the parent—protecting the main context window from pollution.

```
  MAIN AGENT                          SUB-AGENT (isolated)
  ┌────────────────┐                  ┌──────────────────────┐
  │                │   Task(explore)  │                      │
  │  "I need to    │────────────────▶│  Own TAOR loop        │
  │   explore the  │                 │  Scoped tools         │
  │   codebase"    │                 │  Own maxTurns         │
  │                │                 │  Own compaction       │
  │                │                 │  Own MEMORY.md        │
  │                │   summary only  │                      │
  │  "Found 3      │◀────────────────│  20 turns of work     │
  │   files..."    │                 │  (stays inside)       │
  └────────────────┘                  └──────────────────────┘
  Context cost:                       Full context used,
  just the summary                    then discarded
```

### Foreground vs Background Execution

*   **Foreground**: Blocks the main conversation. Permission prompts and questions pass through to the user.
*   **Background**: Runs concurrently while the user keeps working. Permissions are collected *upfront* before launch, then auto-denied if not pre-approved. Background agents can't ask clarifying questions — the tool call simply fails and the agent continues. MCP tools are unavailable in background mode.

Press `Ctrl+B` to send a running foreground agent to the background.

### Custom Sub-Agents (YAML Frontmatter)

Custom sub-agents are defined as `.md` files with YAML frontmatter:

```yaml
---
name: code-reviewer
description: Expert code reviewer. Use proactively after code changes.
tools: Read, Glob, Grep, Bash
disallowedTools: Write, Edit
model: sonnet            # or opus, haiku, inherit
permissionMode: default  # or acceptEdits, dontAsk, plan, delegate, bypassPermissions
maxTurns: 25
skills:
  - api-conventions
  - error-handling-patterns
memory: user             # or project, local
---

You are a senior code reviewer. When invoked:
1. Run git diff to see recent changes
2. Focus on modified files
3. Provide feedback by priority: Critical → Warnings → Suggestions
```

**Storage scopes:** `~/.claude/agents/` (user-level), `.claude/agents/` (project-level), or via `--agents` CLI flag.

### Key Capabilities

| Feature | How It Works |
|:--|:--|
| **Persistent Memory** | Set `memory: user` → agent writes learned patterns to `~/.claude/agent-memory/<name>/MEMORY.md`. First 200 lines auto-loaded on next invocation. |
| **Skill Preloading** | Set `skills: [api-conventions]` → injects skill instructions into the sub-agent's context before it starts. |
| **Tool Scoping** | `tools` whitelist AND `disallowedTools` blacklist. One sub-agent can restrict another: `tools: Task(worker, researcher)`. |
| **Chaining** | *"Use code-reviewer to find issues, then use optimizer to fix them."* — sequential delegation. |
| **Resumability** | Transcripts persist in `~/.claude/projects/{project}/{session}/subagents/`. Ask Claude to *"continue that code review"* and it resumes with full prior context. |
| **Auto-Compaction** | Sub-agents compact independently as they approach context limits. Main conversation compaction does *not* affect sub-agent transcripts. |

---

## D3. Agent Teams — Multi-Session Parallelism

Agent Teams are entirely separate Claude Code instances coordinating via a shared filesystem. Unlike sub-agents (which are child processes), teammates are **peer processes** that can communicate bidirectionally.

> **Status**: Agent Teams are currently experimental. Enable via `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in environment or `settings.json`.

### When to Use Agent Teams (vs Sub-Agents)

| Dimension | Sub-Agents | Agent Teams |
|:--|:--|:--|
| **Process model** | Child of main agent | Independent peer processes |
| **Communication** | Summary returned at end | Ongoing message/broadcast IPC |
| **Context** | Isolated, discarded | Independent, persistent |
| **Coordination** | Sequential delegation | Shared task list, self-claiming |
| **Best for** | Research, exploration, reviews | Parallel implementation across modules |

### Architecture

Unlike Sub-Agents, **Agent Teams** run as parallel peer processes. As visualized below, a **Lead Agent** assigns work via a Shared Task List, while independent agents (Auth, Tests, Docs) execute simultaneously in separate terminal panes, coordinating via message passing.

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                        LEAD AGENT                                │
  │                     (Delegate Mode)                              │
  │                                                                 │
  │   "Implement auth, tests, and docs"                             │
  │        │              │              │                           │
  │        ▼              ▼              ▼                           │
  │   ┌─────────┐   ┌─────────┐   ┌─────────┐                      │
  │   │  Auth   │   │  Tests  │   │  Docs   │  ◀── Shared Task List │
  │   └────┬────┘   └────┬────┘   └────┬────┘  ~/.claude/tasks/     │
  └────────┼──────────────┼──────────────┼──────────────────────────┘
           │              │              │
     ┌─────┴─────┐  ┌─────┴─────┐  ┌────┴──────┐
     │  tmux #1  │  │  tmux #2  │  │  tmux #3  │
     │  Claude   │◀─┤  Claude   │◀─┤  Claude   │ ◀── Peer messaging
     │  Instance │─▶│  Instance │─▶│  Instance │
     │  Own ctx  │  │  Own ctx  │  │  Own ctx  │ ◀── Independent
     └───────────┘  └───────────┘  └───────────┘
```

**Configuration**: `~/.claude/teams/{team-name}/config.json`

### Display Modes

| Mode | How It Works | Requirements |
|:--|:--|:--|
| **In-process** | All teammates inside your terminal. `Shift+Up/Down` to select, type to message. | Any terminal |
| **Split panes** | Each teammate gets its own pane for full visibility. Click into a pane to interact. | `tmux` or iTerm2 |

Set via `settings.json`: `{ "teammateMode": "in-process" }` or `claude --teammate-mode in-process`.

### Coordination Mechanisms

*   **Shared Task List**: All agents see task status. Teammates **self-claim** the next unassigned, unblocked task when they finish one.
*   **Message**: Send a message to one specific teammate.
*   **Broadcast**: Send to all teammates simultaneously (use sparingly — cost scales with team size).
*   **Automatic Idle Notification**: When a teammate finishes and stops, it automatically notifies the lead.
*   **Plan Approval**: You can require teammates to get plan approval before making changes: *"Spawn an architect teammate. Require plan approval before they make any changes."*

### Quality Gate Hooks

Two team-specific hooks enforce quality without involving the LLM:

| Hook | Trigger | Use Case |
|:--|:--|:--|
| `TeammateIdle` | Teammate about to go idle | Exit code 2 → send feedback, keep them working |
| `TaskCompleted` | Task being marked complete | Exit code 2 → prevent completion, request fixes |

**Entropy Management**: For long-running projects, "Garbage Collection" agents are a powerful *architectural pattern* enabled by Team Agents. You can configure a low-cost background teammate to scan for architectural drift (e.g., deprecated patterns, TODOs, stale docs) and open refactoring PRs automatically. This prevents the "AI Slop" accumulation that plagues purely generative workflows.

### Use Cases

*   **Parallel feature implementation**: Auth, tests, and docs each owned by a different teammate.
*   **Research with competing hypotheses**: Three teammates test different theories and converge on an answer.
*   **Cross-layer coordination**: Frontend, backend, and infrastructure changes in parallel.
*   **Parallel code review**: Multiple reviewers examine different aspects simultaneously.

---

## D4. Hooks — Deterministic Lifecycle Events

Hooks are scripts that run **outside** the LLM loop — zero AI, pure determinism. They're the observability and guardrail layer.

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                       HOOK INJECTION POINTS                       │
  │                                                                  │
  │   ○ SessionStart ─────────────────────────────────────────┐      │
  │                                                           ▼      │
  │   User message ──▶ ○ UserPromptSubmit (can transform)     │      │
  │                               │                           │      │
  │                          ┌────┴────┐                      │      │
  │                          │  THINK  │                      │      │
  │                          └────┬────┘                      │      │
  │               ○ PreToolUse ◀──┘  (can block/modify)       │      │
  │               ○ PermissionRequest (can auto-approve/deny) │      │
  │                          ┌────┴────┐                      │      │
  │                          │   ACT   │                      │      │
  │                          └────┬────┘                      │      │
  │              ○ PostToolUse ◀──┤                            │      │
  │              ○ PostToolUseFailure ◀──┘                     │      │
  │               ○ PreCompact (can inject context)            │      │
  │                          ┌────┴────┐                      │      │
  │                          │ DECIDE  │── tool_use ──▶ loop   │      │
  │                          └────┬────┘                      │      │
  │                          ○ Stop                           │      │
  │                          ○ SessionEnd ◀───────────────────┘      │
  └──────────────────────────────────────────────────────────────────┘

  Example: Auto-lint after every file write
  ┌──────────────────────────────────────────┐
  │  "PostToolUse": [{                       │
  │    "matcher": { "tool_name": "Write" },  │
  │    "command": "eslint --fix $FILE"       │
  │  }]                                      │
  └──────────────────────────────────────────┘
```

The diagram above maps the execution flow of hooks. Note closely where they act—`PreToolUse` can block an action before it happens (security), while `PostToolUse` can react to it (observability). The `SessionStart` and `SessionEnd` hooks allow for environment setup and teardown.

> **Pro Tip: Linter-Driven Remediation.** Don't just fail a build. Write custom linters that output *remediation instructions* directly into the agent's context. Instead of "Error: Line 14," the output should be "Error: Line 14 uses a deprecated API. Replace `foo()` with `bar()`." This turns the linter into a teacher.

---

## D5. MCP — The Universal Service Connector

MCP (Model Context Protocol) lets the agent talk to any external service — databases, APIs, SaaS tools — through a standard protocol.

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                          CLAUDE CODE                              │
...
  │   100+ tools?  ──▶  Semantic Tool Search  ──▶  Only inject relevant defs
  └──────────────────────────────────────────────────────────────────┘
```

MCP abstracts away the proprietary APIs of your tools. As shown, it supports three transport modes depending on where the tool lives:
1.  **Stdio**: For local tools (databases, CLI apps).
2.  **HTTP**: For remote servers (SaaS integrations).
3.  **SSE**: For streaming updates (logs, real-time feeds).

> **Killer Use Case: Runtime Inspection.** One of the most powerful MCP applications is a "Chrome DevTools MCP" or "LogQL MCP." This gives the agent *eyes* — allowing it to inspect the DOM of a running localhost server, grab console errors, or query structured logs. Without this, the agent is coding blind. With it, the agent can validate its own fixes.

---

## D6. Plugins — Bundle Everything

Plugins package skills, agents, hooks, and MCP servers into distributable, installable units.

```
  my-plugin/
  ├── plugin.json         # Metadata, version, dependencies
  ├── skills/             # /my-plugin:deploy, /my-plugin:review
  ├── agents/             # Custom sub-agents
  ├── hooks/              # Lifecycle scripts
  └── mcp-servers/        # Service connectors
```

This structure enables the "One-Click DevOps" experience. Instead of asking a developer to configure ESLint, pre-commit hooks, and a database connector separately, you simply ship a `standard-compliance` plugin that configures all of them instantly.

Marketplaces allow discovering and installing community plugins. Plugin skills are namespaced (`my-plugin:deploy`) to avoid conflicts.

---
---

# Appendix

## A. Steerability — Engineering Personality

Claude Code's "tasteful" and "eager" personality is carefully engineered through system prompt structure, not model fine-tuning.

**Shouting still works.** The most effective way to prevent bad behavior is emphasis in the prompt:
*   `IMPORTANT: DO NOT ADD ***ANY*** COMMENTS unless asked`
*   `VERY IMPORTANT: You MUST avoid using search commands like find...`

**Tone & Style is a prompt section.** A dedicated markdown section defines the persona:
*   *"If you cannot help, do not explain why — it comes across as preachy."*
*   *"No emojis unless explicitly requested."*

**XML tags for structure.** The massive system prompt uses XML for semantic parsing:
*   `<system-reminder>` — injected at end of turns to reinforce rules
*   `<good-example>` / `<bad-example>` — few-shot heuristic training (e.g., "Use absolute paths, don't `cd`")

---

## B. Permission Modes Reference

| Mode | Behavior | Trust Level |
|:--|:--|:--|
| `plan` | Read-only, no writes at all | 🔒 Lowest |
| `default` | Ask before edits and shell | 🔒 Standard |
| `acceptEdits` | Auto-approve file edits, ask for shell | 🔓 Medium |
| `dontAsk` | Auto-approve everything in allow list | 🔓 High |
| `bypassPermissions` | Skip all checks (managed orgs only) | 🔓 Maximum |

---

## C. Context Strategies Reference

```
  Strategy              How It Works                         When to Use
  ─────────────────     ──────────────────────────────       ──────────────────────
  Auto-Compaction       LLM summarizes at ~50% usage         Always on (automatic)
  Manual Compact        /compact <focus area>                 User wants targeted trim
  Sub-Agent             Offload to isolated context           Heavy research/exploration
  Forked Context        context: fork in skill                Skill that would pollute ctx
  Logic-Only Skill      disable-model-invocation: true        Pure instruction injection
  MCP Tool Search       Semantic search for relevant tools    Servers with 100+ tools
```

---

## D. Feature Layering — How Everything Composes

When the same feature is defined at multiple levels, the resolution strategy depends on the feature type:

```
  CLAUDE.md files         Skills / Subagents         MCP Servers         Hooks
  ────────────────        ──────────────────         ───────────         ─────
  ┌─── Managed ───┐      ┌─── Managed ────┐  WIN   ┌── Local ──┐ WIN  All sources
  ├─── Project ───┤      ├─── CLI Flag ───┤   │    ├── Proj ───┤  │   fire for
  ├─── Rules ─────┤ ALL  ├─── Project ────┤   │    └── User ───┘  │   matching
  ├─── User ──────┤ ADD  ├─── User ───────┤   │                   │   events.
  ├─── Local ─────┤  UP  └─── Plugin ─────┘   ▼    Override by    ▼
  └─── Auto ──────┘                                 name           MERGE
                          Override by name                         (all run)
  LLM resolves            (highest priority
  conflicts               scope wins)
```

This logic chart is complex but critical. It explains why your `CLAUDE.md` instructions might conflict with a Plugin.
*   **CLAUDE.md**: Everything is added to context (additive).
*   **Skills/Agents/MCP**: Name collisions are resolved by priority (Project > User > Plugin).
*   **Hooks**: *Everyone* runs. If a plugin adds a pre-commit hook and you add one too, both execute.

---

## E. The Isolation Spectrum

```
                        SAME CONTEXT              ISOLATED CONTEXT
                    ┌─────────────────┐      ┌──────────────────────┐
  SAME PROCESS      │                 │      │                      │
                    │     SKILLS      │      │    SUB-AGENTS        │
                    │  (inject here)  │      │  (separate loop)     │
                    │  Zero overhead  │      │  Return summary      │
                    └─────────────────┘      └──────────────────────┘

                                             ┌──────────────────────┐
  SEPARATE PROCESS                           │                      │
  (tmux)                                     │    AGENT TEAMS       │
                                             │  (separate sessions) │
                                             │  Task board + IPC    │
                                             └──────────────────────┘

  Cost:            Low ◀──────────────────────────────────────▶ High
  Isolation:       None ◀─────────────────────────────────────▶ Full
```

This chart helps you choose the right tool. Need simple instruction macros? Use **Skills** (cheap, same context). Need to research a topic without filling the window? Use **Sub-Agents** (isolated context). Need parallel implementation of features? Use **Agent Teams** (isolated processes).

---


