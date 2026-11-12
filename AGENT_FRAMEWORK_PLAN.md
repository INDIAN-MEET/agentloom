# AgentFlow (working name): A Config-First TypeScript Framework for Building AI Agents

> **One-line promise:** *Describe your agent in one file. Run it in one command. See exactly what it did.*

This document covers three things:
1. **Why** we are building this (the problem)
2. **What** we are building (features and developer experience)
3. **How** we will build it (a numbered, step-by-step implementation plan, no code)

Follow the steps in order. Each step lists **Why**, **Approach**, **Deliverables**, and **Done when**, so you always know what to build, why it exists, and when to move on.

---

## Table of Contents
1. [The Problem](#1-the-problem)
2. [What We Are Solving](#2-what-we-are-solving)
3. [Target Users](#3-target-users)
4. [Competitive Landscape and Differentiator](#4-competitive-landscape-and-differentiator)
5. [Feature List](#5-feature-list)
6. [Developer Experience](#6-developer-experience)
7. [Architecture Overview](#7-architecture-overview)
8. [Tech Stack and Reasons](#8-tech-stack-and-reasons)
9. [Design Principles](#9-design-principles)
10. [Implementation Plan (Step by Step)](#10-implementation-plan-step-by-step)
11. [Milestones](#11-milestones)
12. [Risks and Mitigations](#12-risks-and-mitigations)
13. [Success Metrics](#13-success-metrics)
14. [Open Decisions Log](#14-open-decisions-log)
15. [Glossary](#15-glossary)

---

## 1. The Problem

Every developer who builds an AI agent today rebuilds the same plumbing from scratch:

| Pain point | What it looks like in practice |
|---|---|
| **Boilerplate** | Managing shared state, calling the LLM, parsing outputs, routing to the next step, retrying on failure. Every project repeats it. |
| **Steep learning curve** | Existing frameworks introduce many concepts (graphs, reducers, channels, checkpointers) before a developer gets a first working agent. |
| **Tool wiring is manual** | Connecting MCP servers or tools to an agent needs glue code: discovery, schema conversion, execution, error handling. |
| **Routing logic is scattered** | The "if the user says X go to Y" logic lives inside code, so it is hard to read, review, or change. |
| **Debugging is painful** | When an agent misbehaves, developers cannot easily see which path it took, what each step received, or why. |
| **Switching providers is costly** | Moving from one model provider to another often means rewriting parts of the app. |
| **Pause/resume and human approval are hard** | Persisting state and waiting for a human decision needs custom infrastructure. |

**Result:** developers spend most of their time on plumbing instead of the actual behavior of their agent.

---

## 2. What We Are Solving

We provide a **declarative layer** for agents. The developer describes *what* the agent is, and the framework handles *how* it runs.

The developer describes:
- The prompts
- The steps (nodes) and how they connect (the flow or graph)
- The routing rules
- The tools (including MCP servers)
- The shared state shape

The framework provides:
- Validation of the description, with clear errors
- State management
- Execution (running nodes, branching, looping)
- LLM calls through a single interface
- Tool and MCP integration
- Retries, limits and error handling
- Tracing of every step
- Persistence and resume

**Core idea:** boilerplate becomes configuration, and the agent's behavior becomes readable and visible.

---

## 3. Target Users

1. **Application developers** who want to add an agent to their product without learning a big framework (primary).
2. **Startups and small teams** who need to ship agents quickly and iterate on prompts and flow.
3. **Developers new to agents** who want sensible defaults and templates.
4. **Advanced developers** who want less boilerplate but still need full control through custom code.

---

## 4. Competitive Landscape and Differentiator

Existing tools to study (verify current status before finalizing, since this space moves fast):
- LangGraph.js (graph-based, powerful, steeper learning curve)
- Mastra (TypeScript-first agent framework)
- Vercel AI SDK (model layer with agent features)
- OpenAI Agents SDK, CrewAI, AutoGen, Google ADK, Pydantic AI (other ecosystems and approaches)

**We will not try to beat them at everything.** Choose and own one of these positions:

| Option | Positioning |
|---|---|
| A (recommended) | **Simplest path from idea to running agent**, driven by a single config file. |
| B | **Best debugging and tracing experience** in the ecosystem. |
| C | **A niche focus**, for example support bots or internal tools. |

**Recommendation:** combine A and B: *simple to define, easy to see what happened.* Decide this formally in Step 1.

---

## 5. Feature List

### 5.1 MVP features (must have)
- **Declarative agent spec** (YAML or JSON) plus a typed TypeScript builder API as an equivalent alternative
- **Schema validation** with human-friendly error messages
- **Typed shared state** that flows between nodes
- **Node types:** LLM node, tool node, router (conditional) node, custom-code node
- **Graph executor:** sequential flow, branching, loops, end conditions
- **Unified LLM interface** (switch provider by changing one setting)
- **MCP integration:** declare servers in config; tools are auto-discovered and attached
- **Safety limits:** max steps, max retries, timeouts, optional token/cost budget
- **Tracing:** every step is recorded (inputs, outputs, tokens, duration, route chosen)
- **CLI:** init, run, validate, trace
- **Templates:** a few ready-to-run example agents

### 5.2 Version 1.x features (should have)
- **Checkpointing and resume** (in-memory, then SQLite)
- **Human-in-the-loop** node (pause for approval or input)
- **Streaming output**
- **Sub-agent / nested graph** node
- **Dev mode** with auto-reload when files change
- **Test utilities** (mock LLM, scenario tests)
- **Config includes** (reuse prompts and fragments across files)

### 5.3 Future features (nice to have)
- Visual builder (drag and drop UI that edits the spec)
- Web trace viewer
- Hosted runtime and deployment helpers
- Plugin system for custom node types
- Evaluation tools (compare prompt versions)
- Long-term memory and retrieval integrations
- Multi-agent patterns (supervisor, handoff)

---

## 6. Developer Experience

### What the developer should feel
1. **Fast start:** `init` creates a working project, one API key is the only setup, and `run` shows a result in about five minutes.
2. **Readable:** the whole agent flow can be understood by reading one file.
3. **Safe:** mistakes are caught at validation time with precise messages, not at runtime with stack traces.
4. **Visible:** after a run, they can see exactly what happened at every step.
5. **Never stuck:** any step can be replaced with their own TypeScript function.

### Example of the experience (conceptual, not code)
- Open the agent file, see: a prompt, three steps, one router, two tools.
- Run it, watch each step print as it executes.
- Open the trace, see that the router chose "refund" because of a specific field in state.
- Change one prompt, run again, compare.

### Why developers will trust it
| Trust factor | How we deliver it |
|---|---|
| Works immediately | Templates run with only an API key |
| Understandable errors | Messages name the node, field and fix |
| Transparent | Full trace for every run |
| No lock-in | Provider-agnostic, standard MCP, escape hatch to plain code |
| Reliable | Test suite for the engine, example agents in CI |
| Documented | Working example first, theory afterwards |

---

## 7. Architecture Overview

Layers from top (what the developer touches) to bottom (infrastructure):

```
 Developer-facing layer
 ┌───────────────────────────────────────────────┐
 │ Agent Spec (YAML/JSON)   |  Builder API (TS)  │
 └───────────────────────────────────────────────┘
                     │
 Validation & compile layer
 ┌───────────────────────────────────────────────┐
 │ Schema validation → Normalized spec → Graph   │
 │ (checks: missing edges, unknown nodes, loops) │
 └───────────────────────────────────────────────┘
                     │
 Runtime layer
 ┌───────────────────────────────────────────────┐
 │ Executor (run loop)  ◄──►  State Manager      │
 │   │                                           │
 │   ├─ LLM Node ──► LLM Adapter                 │
 │   ├─ Tool Node ─► Tool Registry ─► MCP Client │
 │   ├─ Router Node                              │
 │   └─ Custom Code Node                         │
 └───────────────────────────────────────────────┘
                     │
 Cross-cutting services
 ┌───────────────────────────────────────────────┐
 │ Event bus → Tracer │ Checkpointer │ Limits    │
 └───────────────────────────────────────────────┘
                     │
 Interfaces
 ┌───────────────────────────────────────────────┐
 │ CLI  │  Programmatic API  │  (later) UI/Server│
 └───────────────────────────────────────────────┘
```

**Key idea:** the spec is compiled into an internal graph, and the executor only knows about the internal graph. This means the YAML format, the builder API, and a future visual UI can all feed the same engine.

---

## 8. Tech Stack and Reasons

| Concern | Choice | Why |
|---|---|---|
| Language | TypeScript (strict mode) | Type safety, npm distribution, shared types with a future UI |
| Validation and types | Zod | Runtime validation plus inferred types from a single source of truth |
| LLM access | Vercel AI SDK (verify current version) | Already unifies providers, streaming and tool calling, so we do not build adapters |
| MCP | Official MCP TypeScript SDK | Standard client, supports multiple transports |
| Spec format | YAML (primary), JSON (also) | Human-readable; YAML parsing library needed |
| CLI | commander or citty | Lightweight, well-known |
| Persistence | In-memory first, then SQLite, later Postgres/Redis | Start simple, abstract behind an interface |
| Testing | Vitest | Fast, TypeScript-friendly |
| Build/packaging | tsup (or similar) with a monorepo (pnpm workspaces) | Clean separation of core, CLI and templates |
| Lint/format | ESLint + Prettier | Consistency |
| CI | GitHub Actions | Run tests and example agents on every change |

**Decision note:** verify the current versions and APIs of these libraries in Step 1, because they change often.

---

## 9. Design Principles

1. **Config first, code as escape hatch.** Common cases need no code; unusual cases are never blocked.
2. **Single source of truth.** The Zod schema defines validation, types and documentation.
3. **Fail early, fail clearly.** Catch problems at validation time with actionable messages.
4. **Everything observable.** Every step emits events; tracing is not optional bolt-on.
5. **Small core.** The engine stays small; features are layered on through interfaces.
6. **Interfaces before implementations.** LLM, tools, persistence and tracing are all behind interfaces so they can be swapped.
7. **Deterministic where possible.** Routing and state updates should be predictable and testable; only the LLM is non-deterministic.
8. **Safe by default.** Step limits, timeouts and budgets are on by default.
9. **Boring and stable public API.** Keep the public surface small; mark everything else internal.

---

## 10. Implementation Plan (Step by Step)

**How to read this plan:** steps are numbered in the order you should build them. Each step depends only on earlier steps. Do not skip ahead, and do not move on until the **Done when** criteria are met.

**Phases overview**

| Phase | Steps | Outcome |
|---|---|---|
| 0. Research and decisions | 1-3 | Clear scope, repo ready |
| 1. Foundation (spec and state) | 4-7 | A validated spec and a state model |
| 2. Core engine | 8-15 | A working executor with core node types |
| 3. MCP and tools | 16-18 | Agents can use MCP tools |
| 4. Observability | 19-21 | Every run is traceable |
| 5. Persistence and human-in-the-loop | 22-25 | Pause, resume, approvals |
| 6. Developer experience | 26-30 | CLI, templates, dev mode, public API |
| 7. Quality | 31-33 | Tests, examples, evaluation |
| 8. Docs and release | 34-36 | Public release |
| 9. After MVP | 37-40 | Growth roadmap |

---

### PHASE 0: Research and Decisions

#### Step 1: Study competitors and finalize positioning
- **Why:** Building without knowing the landscape risks duplicating something that already exists. A clear differentiator guides every later trade-off.
- **Approach:**
  - Build one small agent (a support bot with a router and one tool) in 2 or 3 existing frameworks.
  - Record: lines of config/code, concepts learned, time to first run, quality of debugging.
  - List what felt painful; those pains become your feature priorities.
  - Check current versions of the AI SDK, MCP SDK and competitor frameworks.
- **Deliverables:** A short comparison document and a one-paragraph positioning statement.
- **Done when:** You can complete the sentence "Developers should choose this over X because ___" in one honest sentence.

#### Step 2: Define MVP scope and non-goals
- **Why:** Frameworks die from scope creep. A written boundary protects you.
- **Approach:**
  - Choose the MVP features from section 5.1 only.
  - Write explicit non-goals (for example: no UI, no hosting, no multi-agent in the MVP).
  - Define three target example agents the MVP must support (see Step 32).
- **Deliverables:** Scope document with "in" and "out" lists.
- **Done when:** Every MVP feature maps to at least one example agent, and nothing in the list is "just in case."

#### Step 3: Set up repository, tooling and conventions
- **Why:** Consistent tooling early prevents painful refactors later.
- **Approach:**
  - Create a monorepo with separate packages: **core** (engine), **cli**, **templates/examples**, and **docs**.
  - Configure strict TypeScript, linting, formatting, test runner and CI.
  - Define naming conventions, folder layout, commit style and a license.
  - Decide the package name and check availability on npm.
- **Deliverables:** Empty but building repo, CI passing on a placeholder test.
- **Done when:** A fresh clone installs, builds and passes CI with one command each.

---

### PHASE 1: Foundation (Spec and State)

#### Step 4: Design the agent spec (on paper first)
- **Why:** The spec is your product's user interface. Getting it wrong is the most expensive mistake, because users will write it by hand.
- **Approach:**
  - List every concept a user must express: agent name, model settings, state shape, nodes, edges, router rules, tools, MCP servers, limits.
  - Write 3 hand-made example specs (simple chatbot, tool-using agent, routed support bot) *before* writing any schema. If they feel awkward to write, redesign.
  - Choose names that read like plain English.
  - Decide how routing is expressed: simple rules (match a state field), LLM-decided routes, or custom function.
  - Decide how prompts reference state values (templating rules).
  - Define versioning: the spec carries a version number so future changes do not break old files.
- **Deliverables:** Spec design document with the three example specs and a field-by-field description.
- **Done when:** A teammate can read an example spec and explain the flow without any documentation.

#### Step 5: Implement the spec schema and types
- **Why:** The schema turns your design into enforceable rules and automatically gives you TypeScript types.
- **Approach:**
  - Translate the design into schemas, one per concept (agent, node, edge, router, tool, MCP server, limits).
  - Use a discriminated structure for node types so each type has its own required fields.
  - Derive all TypeScript types from the schemas, never write them twice.
  - Add default values (for example default limits) in the schema.
- **Deliverables:** Schema module with exported types.
- **Done when:** All three hand-written example specs parse successfully, and deliberately broken variants are rejected.

#### Step 6: Build the loader and the friendly-error layer
- **Why:** First impressions depend on error quality. Raw schema errors are confusing; developers need messages that say what is wrong and how to fix it.
- **Approach:**
  - Build a loader that reads YAML or JSON files and returns a validated spec.
  - Add **semantic validation** beyond the schema (things a schema cannot express): every edge points to an existing node, every node is reachable, a start node exists, at least one path reaches the end, router outcomes are all covered, referenced tools and state fields exist, and loops have an exit or a step limit.
  - Convert errors into friendly messages that include: file, line if possible, node name, what is wrong, a suggested fix.
  - Add a "did you mean" suggestion for typos in names.
- **Deliverables:** Loader, semantic validator, error formatter, and a catalog of error messages.
- **Done when:** Ten deliberately wrong specs each produce a clear, specific, actionable message.

#### Step 7: Design and implement the state model
- **Why:** State is the "memory" flowing between nodes. A clear model makes execution predictable and debugging possible.
- **Approach:**
  - Define state as a typed object defined in the spec (fields, types, defaults).
  - Include built-in fields: message history, current node, step counter, run id, and error info.
  - Decide update rules: nodes return *partial updates*, and the framework merges them. Define merge behavior per field (replace, append for lists like messages).
  - Make state **immutable per step**: each step produces a new snapshot. This enables tracing, checkpoints and replay.
  - Validate state after every update so corruption is caught at the step that caused it.
  - Define template resolution: how `{{field}}` style references in prompts are filled from state, and what happens when a field is missing (clear error, not silent empty text).
- **Deliverables:** State manager module with snapshot, merge and template resolution.
- **Done when:** Unit tests cover merging, defaults, validation failures, and template edge cases (missing fields, nested fields).

---

### PHASE 2: Core Engine

#### Step 8: Define the node contract
- **Why:** If every node type obeys the same contract, the executor stays tiny and new node types are easy to add.
- **Approach:**
  - Define what every node receives (current state snapshot, run context, services such as LLM and tools) and what it returns (a partial state update, plus optional routing hint and metadata).
  - Define lifecycle events each node emits (started, completed, failed).
  - Define how a node signals errors versus "retry me."
  - Define a registry so node types can be looked up by name.
- **Deliverables:** Node interface specification and node registry.
- **Done when:** A trivial "echo" node can be written against the contract in a few lines and registered.

#### Step 9: Build the graph compiler
- **Why:** The executor should never read raw spec files. A compiled, normalized graph is faster to run and easier to validate, and it lets multiple input formats (YAML, builder API, UI) share one engine.
- **Approach:**
  - Convert the validated spec into an internal graph: nodes by id, outgoing edges, conditional edges, start and end markers.
  - Pre-resolve references (node to implementation, tool names to tool objects).
  - Detect cycles and confirm each has a termination condition.
  - Produce a readable text or diagram description of the graph (useful for `validate` and docs).
- **Deliverables:** Compiler module and a graph description output.
- **Done when:** Each example spec compiles, and a graph summary is printed that matches the author's intent.

#### Step 10: Build the executor (run loop)
- **Why:** This is the heart of the framework. It must be small, predictable and well tested.
- **Approach:**
  - Loop: pick current node, run it, merge its update into state, decide the next node, repeat until end or a stop condition.
  - Support **sequential flow, conditional branching and loops**.
  - Enforce global limits: maximum steps, total timeout, optional budget.
  - Emit events at every transition (run started, node started/completed/failed, route chosen, run finished).
  - Support cancellation (abort signal).
  - Return a final result object: final state, status, and the run id.
  - Keep it free of LLM-specific logic. It only knows the node contract.
- **Deliverables:** Executor module.
- **Done when:** Using only fake nodes, tests prove: linear flow, branching, a loop that exits correctly, a loop that hits the step limit and fails clearly, and cancellation.

#### Step 11: Build the LLM adapter layer
- **Why:** Isolating model access behind your own small interface means you can change the underlying SDK or add providers without touching nodes.
- **Approach:**
  - Define a minimal interface: send messages (with optional tools and structured-output schema), receive text, tool calls, token usage and finish reason; plus a streaming variant.
  - Implement it using the AI SDK for the first providers (pick two, for example one major hosted provider and one other, to prove provider-independence).
  - Resolve model settings from the spec (provider, model name, temperature, max tokens), with agent-level defaults and per-node overrides.
  - Read API keys from environment variables only; give a clear error if one is missing.
  - Normalize errors (rate limit, auth, timeout, invalid response) into your own error types.
  - Add a **mock provider** that returns scripted responses (needed for tests in Step 31).
- **Deliverables:** LLM interface, two real adapters, one mock adapter.
- **Done when:** The same test passes against the mock and against each real provider by changing only one setting.

#### Step 12: Implement the LLM node
- **Why:** This is the most commonly used node, and it proves the whole chain: spec to state to LLM and back.
- **Approach:**
  - Build the prompt from the node's instruction text, resolved state values, and (optionally) message history.
  - Call the LLM adapter and write the output to a state field named in the spec.
  - Support **structured output**: if the node declares an output schema, request and validate it, retry once or twice if invalid, then fail clearly.
  - Handle retries for transient errors with backoff, within the retry limit.
  - Record token usage and latency in the node's metadata.
- **Deliverables:** LLM node.
- **Done when:** A one-node agent answers a question end to end, and a structured-output node returns validated data or a clear failure.

#### Step 13: Implement the router node
- **Why:** Routing is the thing developers hand-code most often. Making it declarative is a big part of the value.
- **Approach:**
  - Support three routing modes, ordered from simplest to most flexible:
    1. **Rule-based:** compare a state field against values or conditions.
    2. **LLM-based:** the model picks one of the allowed route labels (constrained to the declared labels).
    3. **Function-based:** a custom function returns the label.
  - Always require a **default route** or fail validation, so no run gets stuck.
  - Record the chosen route *and the reason* in the trace.
  - Validate at compile time that every possible label has a target.
- **Deliverables:** Router node with the three modes.
- **Done when:** A support-bot example routes correctly by rule and by LLM, and an unmapped label is caught during validation.

#### Step 14: Implement the custom-code node (escape hatch)
- **Why:** No config format covers everything. Without an escape hatch, advanced users leave.
- **Approach:**
  - Let the spec reference a TypeScript function by file and export name.
  - Pass the function the same context as built-in nodes (state, services), and require it to return a partial state update.
  - Validate the returned update against the state schema.
  - Isolate errors so a bug in user code produces a clear message pointing to their file.
  - Provide a helper for authors to define functions with proper types.
- **Deliverables:** Custom node plus loading mechanism.
- **Done when:** An example agent mixes built-in nodes with two custom functions, and type errors in user code are reported clearly.

#### Step 15: Error handling, retries and limits (engine-wide)
- **Why:** Agents fail in unpredictable ways (rate limits, bad outputs, tool outages). Reliability is a major reason to adopt a framework.
- **Approach:**
  - Define an error taxonomy: validation, configuration, provider, tool, user-code, limit-exceeded, cancelled.
  - Define per-node policy: retry count, backoff, timeout, and on-failure behavior (fail the run, go to a fallback node, or continue with a default).
  - Define global safeguards: maximum steps, maximum duration, maximum tokens or cost.
  - Make every failure appear in the trace with context.
  - Make the final run status explicit (completed, failed, cancelled, paused).
- **Deliverables:** Error types, policy handling in the executor, limit enforcement.
- **Done when:** Tests simulate each failure type and confirm correct behavior, messages and trace entries.

---

### PHASE 3: MCP and Tools

#### Step 16: Build the tool abstraction and registry
- **Why:** MCP tools, built-in tools and user-defined tools should all look identical to nodes. This decouples the engine from MCP specifics.
- **Approach:**
  - Define a tool as: name, description, input schema, and an execute action.
  - Build a registry that collects tools from all sources and prevents name collisions (namespacing by source).
  - Allow per-node tool allow-lists, so each LLM node only sees the tools it should (this improves quality and safety).
  - Support user-defined tools written in TypeScript.
- **Deliverables:** Tool interface and registry.
- **Done when:** A local TypeScript tool can be declared, allow-listed to a node, and called by the LLM node.

#### Step 17: Build the MCP client manager
- **Why:** Connection lifecycle (start, reuse, close, reconnect) is where MCP integrations usually break.
- **Approach:**
  - Read MCP servers from the spec (transport type, command or URL, environment, headers, authentication).
  - Connect on run start (lazily or eagerly, as configured), keep connections open for the run, and close cleanly on finish or failure.
  - Handle timeouts, reconnect attempts and a clear message if a server is unreachable.
  - Keep secrets out of the spec: reference environment variables instead.
  - Support more than one server per agent.
- **Deliverables:** MCP manager with lifecycle handling.
- **Done when:** An agent connects to a real MCP server, lists its tools, and shuts down without leaving orphaned processes or connections.

#### Step 18: Discover MCP tools, convert them, and wire up tool calling
- **Why:** This is the "just list the server URL and it works" magic that sells the framework.
- **Approach:**
  - On connection, fetch the server's tool list and convert each tool's schema into your internal tool format.
  - Register them in the tool registry with a namespace (for example the server's name).
  - Implement the **tool-calling loop** inside the LLM node: the model requests a tool, the framework validates the arguments, executes it, feeds the result back, and repeats until the model gives a final answer or a tool-call limit is reached.
  - Also implement a standalone **tool node** for deterministic cases where the flow, not the model, decides to call a tool.
  - Handle tool errors gracefully: return the error to the model so it can recover, and log it in the trace.
  - Add a configurable approval requirement for sensitive tools (connects to Step 25).
- **Deliverables:** Discovery, schema conversion, tool-calling loop, tool node.
- **Done when:** An agent uses at least two tools from a real MCP server to answer a question, and tool failures are shown clearly in output.

---

### PHASE 4: Observability

#### Step 19: Build the event system
- **Why:** If every important moment is an event, tracing, logging, UI streaming and metrics all become simple subscribers instead of code sprinkled through the engine.
- **Approach:**
  - Define a fixed catalog of event types with consistent fields (run id, node id, timestamp, payload).
  - Cover: run lifecycle, node lifecycle, routing decisions, LLM calls (request/response/usage), tool calls, state updates, errors, retries, limits.
  - Provide a subscribe mechanism usable by user code.
  - Make payload redaction configurable (so secrets and sensitive text can be hidden).
- **Deliverables:** Event definitions and emitter.
- **Done when:** A test run produces a complete, ordered event stream that tells the whole story of the run.

#### Step 20: Build the tracer and trace storage
- **Why:** The trace is your differentiator. It must be structured and complete enough to replay and compare runs.
- **Approach:**
  - Convert events into a structured trace: run summary, list of steps, each with input state, output update, duration, tokens, cost estimate, tool calls and route chosen.
  - Store traces as files per run in a local folder by default.
  - Compute totals (time, tokens, estimated cost) and highlight the slowest and most expensive steps.
  - Design the trace format to be stable and documented, since a future UI will read it.
- **Deliverables:** Tracer, trace format specification, local storage.
- **Done when:** After any run, a single trace file fully explains it, and two runs of the same agent can be diffed.

#### Step 21: Build the terminal trace viewer and live output
- **Why:** Developers must see value in the first five minutes. Readable terminal output does that without building a UI.
- **Approach:**
  - **Live mode:** print each step as it runs (node name, short result, time, tokens).
  - **Post-run view:** a command to show a past trace as a tree or timeline, with the ability to drill into a step.
  - Show the router decision and reason clearly.
  - Add verbosity levels (quiet, normal, verbose).
- **Deliverables:** Terminal renderer and trace viewing command.
- **Done when:** A new user can answer "why did the agent go down this path?" in under a minute using only the output.

---

### PHASE 5: Persistence and Human-in-the-Loop

#### Step 22: Define the checkpoint interface and in-memory store
- **Why:** Abstract first, implement second. This keeps storage swappable.
- **Approach:**
  - Define what a checkpoint contains: run id, state snapshot, next node, step counter, status.
  - Define operations: save, load latest, list for a run, delete.
  - Decide when checkpoints are saved (after every node by default).
  - Implement an in-memory store for tests and quick runs.
- **Deliverables:** Checkpointer interface and in-memory implementation.
- **Done when:** An executor run saves a checkpoint after every step, and a test can restore any of them.

#### Step 23: Implement SQLite persistence
- **Why:** Real agents need to survive restarts; SQLite is zero-setup and good enough for most local and small-scale use.
- **Approach:**
  - Implement the same interface on SQLite.
  - Version the stored format so future changes can migrate.
  - Store conversation threads separately from runs so a conversation can continue across multiple runs.
  - Keep a note for later: Postgres/Redis adapters follow the same interface.
- **Deliverables:** SQLite checkpointer, thread concept.
- **Done when:** Killing the process mid-run and restarting resumes from the last completed step.

#### Step 24: Implement resume, replay and time-travel
- **Why:** These are powerful debugging tools and rare in simple frameworks.
- **Approach:**
  - **Resume:** continue a stopped or failed run from the last checkpoint.
  - **Replay:** re-run from any earlier checkpoint, optionally after editing the prompt, to test a fix without repeating earlier steps.
  - Make sure side-effecting tool calls are flagged so replay does not silently repeat them; require confirmation.
- **Deliverables:** Resume and replay in the engine and CLI.
- **Done when:** You can fail a run at step 5, fix the cause, resume, and see only steps 5 onward run.

#### Step 25: Implement the human-in-the-loop node
- **Why:** Real products need approval before sensitive actions (sending emails, refunds, deleting data).
- **Approach:**
  - A node that pauses the run, saves a checkpoint, and marks the status as "paused" with a description of what is needed.
  - Define how a response is provided (CLI prompt, API call) and how the run resumes with that response merged into state.
  - Support approve, reject (route to an alternative path), or provide-edited-value.
  - Allow tools to be flagged "requires approval" so the same mechanism protects them.
  - Add a timeout policy for unanswered approvals.
- **Deliverables:** Human node and approval flow.
- **Done when:** An example agent asks for approval before a tool call, pauses, survives a restart, and continues correctly after approval or rejection.

---

### PHASE 6: Developer Experience

#### Step 26: Build the CLI core
- **Why:** The CLI is the daily interface. It must be fast, discoverable and consistent.
- **Approach:**
  - Commands: **init** (create project from template), **validate** (check a spec without running), **run** (execute with input), **trace** (view past runs), **resume**, **graph** (print flow diagram), **doctor** (check environment, API keys, MCP connectivity).
  - Provide helpful `--help` text, sensible defaults, and consistent exit codes for scripting and CI.
  - Support passing input from arguments, a file, or interactively.
  - Add colored, readable output with a plain mode for logs.
- **Deliverables:** CLI package with the commands above.
- **Done when:** A new user can go from install to a running agent using only CLI hints and no docs.

#### Step 27: Build project templates
- **Why:** Templates teach by example and make the five-minute promise real.
- **Approach:**
  - Create at least three: **simple chat agent**, **tool-using agent with an MCP server**, **routed support bot with human approval**.
  - Each template includes: spec, sample input, environment file example, short README, and a note on what to try changing.
  - Keep each template minimal and heavily commented.
  - Make `init` pick a template interactively.
- **Deliverables:** Template set wired into `init`.
- **Done when:** Each template runs successfully on a clean machine in under five minutes by someone who has not seen the project.

#### Step 28: Build dev mode (watch and auto-reload)
- **Why:** Fast feedback loops make prompt and flow iteration enjoyable.
- **Approach:**
  - Watch the spec, prompts and custom code files.
  - On change: re-validate immediately and show errors; if valid, optionally re-run with the last input.
  - Keep the previous trace available for comparison.
- **Deliverables:** Dev command.
- **Done when:** Editing a prompt and saving results in a new run and visible output within seconds.

#### Step 29: Build the programmatic API and TypeScript builder
- **Why:** Many developers will embed the framework inside an existing app (a web server, a Slack bot), not run it from a CLI.
- **Approach:**
  - Expose a small, stable API: load a spec (file or object), create an agent, run it (with input, thread id), stream events, resume, and subscribe to events.
  - Provide a **typed builder** that produces the same internal spec so TypeScript users get autocomplete and compile-time checking.
  - Ensure both entry points go through the same validation and compiler.
  - Document which exports are public and which are internal.
- **Deliverables:** Public API surface and builder.
- **Done when:** An example Express (or similar) server runs an agent per request and streams progress to the client.

#### Step 30: Add streaming and config reuse
- **Why:** Streaming is expected in chat UIs, and reuse keeps large specs maintainable.
- **Approach:**
  - Stream partial LLM tokens and events through the API and CLI.
  - Support including shared prompt files and reusable fragments, and environment-specific overrides (development vs production).
  - Keep validation aware of included files so errors still point to the right file.
- **Deliverables:** Streaming support, include and override mechanism.
- **Done when:** A large spec can be split into multiple files, and the streamed output works in the CLI and the example server.

---

### PHASE 7: Quality

#### Step 31: Build testing utilities and the engine test suite
- **Why:** Developers will trust a framework they can test. You will trust your own code the same way.
- **Approach:**
  - **Engine tests:** executor, state merging, routing, retries, limits, cancellation, checkpoint/resume, error taxonomy.
  - **Validation tests:** a large set of good and bad specs with expected messages.
  - **Mock LLM and mock tools** for deterministic tests.
  - **Scenario testing helper for users:** run an agent with scripted LLM responses and assert on the path taken and final state.
  - **Contract tests:** the same suite runs against every LLM adapter and every checkpoint store.
  - Run tests in CI on multiple Node versions and operating systems.
- **Deliverables:** Test suite, user-facing test helpers, CI configuration.
- **Done when:** Coverage on the engine core is high and meaningful, and CI is reliable (no flaky tests).

#### Step 32: Build three real example agents (the proof)
- **Why:** Examples prove the framework works for real tasks and become your best marketing and documentation.
- **Approach:**
  - **Customer support bot:** classify the request, route to FAQ, order lookup (tool) or refund (human approval).
  - **Research agent:** plan, search via an MCP tool, summarize, and produce a structured report.
  - **Internal tool agent:** an agent that queries a data source through MCP and answers team questions.
  - For each, record the line count and compare against the same agent in a competing framework (from Step 1).
  - Run these examples automatically in CI against the mock provider, and occasionally against real providers.
- **Deliverables:** Three examples, a comparison table.
- **Done when:** Every example works end to end, and the comparison table is honest and reproducible.

#### Step 33: Add basic evaluation helpers
- **Why:** Prompt changes can silently break behavior. Even simple regression checks add confidence.
- **Approach:**
  - Allow a file of test cases (input plus expected route or expected output properties).
  - Run the agent over all cases and report pass or fail, cost and time.
  - Compare results between two versions of a spec.
- **Deliverables:** Eval command and format.
- **Done when:** You can change a prompt and see, in one command, which cases got better or worse.

---

### PHASE 8: Documentation and Release

#### Step 34: Write the documentation
- **Why:** Docs decide adoption. Developers read the first page, and if it does not work, they leave.
- **Approach:**
  - **Quickstart:** install, init, run, and see a trace, all on one page. Show the working result first.
  - **Concepts:** state, nodes, routing, tools, tracing, persistence, each short and example-driven.
  - **Guides:** add an MCP server, add human approval, write a custom node, switch providers, test your agent, deploy inside a server.
  - **Reference:** spec fields, CLI commands, API, event catalog, error catalog.
  - **Troubleshooting:** common errors and fixes (generated from the error catalog).
  - Test the quickstart on someone unfamiliar and fix every point where they hesitate.
- **Deliverables:** Docs site or docs folder.
- **Done when:** A developer who has never seen the project completes the quickstart unassisted.

#### Step 35: Pre-release hardening
- **Why:** Rough edges in the first release shape your reputation.
- **Approach:**
  - Review the public API for naming consistency and minimal surface.
  - Security review: secrets handling, safe loading of user code, redaction in traces, tool permission model, dependency audit.
  - Performance sanity check: startup time, memory in long loops, large state.
  - Cross-platform check (macOS, Linux, Windows).
  - Define versioning and a changelog policy; start at a pre-1.0 version and say so honestly.
- **Deliverables:** Hardening checklist, completed.
- **Done when:** All checklist items are resolved or consciously deferred and documented.

#### Step 36: Private beta, then public release
- **Why:** Watching real developers use it reveals problems you cannot see yourself.
- **Approach:**
  - Recruit 5 to 10 developers. Give them a task, say nothing, and watch where they stall. Collect their feedback.
  - Fix the top issues, then publish to npm with a clear README, demo video or GIF, and the comparison table.
  - Announce where agent developers gather, with the honest "what it is and is not" message.
  - Set up an issue template, contribution guide and a feedback channel.
- **Deliverables:** Published package, announcement, feedback loop.
- **Done when:** Strangers successfully build agents and report issues you can act on.

---

### PHASE 9: After the MVP (Roadmap)

#### Step 37: Web trace viewer
- **Why:** The terminal viewer proves value; a visual timeline and graph view makes it exceptional.
- **Approach:** A local web app that reads the stable trace format, shows the graph with the executed path highlighted, and allows comparing runs. Build it on the existing trace format, so no engine changes are needed.

#### Step 38: Visual builder
- **Why:** Lowers the barrier for less experienced developers and enables non-developers to participate.
- **Approach:** A drag-and-drop editor that reads and writes the same spec format and uses the same validator. Spec to UI and UI to spec must round-trip without loss.

#### Step 39: Sub-agents and multi-agent patterns
- **Why:** Larger systems need composition.
- **Approach:** Allow a node to run another agent spec as a sub-graph with defined input and output mapping, then build supervisor and handoff patterns as templates, not as core engine complexity.

#### Step 40: Plugins, more integrations and optional hosting
- **Why:** Ecosystem growth multiplies value.
- **Approach:** A plugin interface for custom node types, storage backends and trace exporters; adapters for Postgres/Redis; optional export of traces to standard observability tools; and later, a hosted runtime if there is demand.

---

## 11. Milestones

| Milestone | Steps | What you can demo |
|---|---|---|
| **M1: Validated spec** | 1-7 | Specs load, validate and give friendly errors; state model works |
| **M2: First agent runs** | 8-15 | A multi-node agent with routing and custom code runs from a spec |
| **M3: Tools and MCP** | 16-18 | The agent uses real MCP tools |
| **M4: Visible execution** | 19-21 | Live output and a post-run trace explain every decision |
| **M5: Durable and safe** | 22-25 | Pause, resume, approval, survive restarts |
| **M6: Developer-ready** | 26-30 | CLI, templates, dev mode, embeddable API |
| **M7: Proven** | 31-33 | Tests, three real examples, comparisons, evals |
| **M8: Public** | 34-36 | Docs and release |

Tip: each milestone is a natural point to demo, get feedback, and decide whether to continue as planned.

---

## 12. Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Competitors already cover this | Low adoption | Commit to a sharp differentiator (Step 1); focus on time-to-first-agent and trace quality |
| Config becomes too limiting | Advanced users leave | Custom-code node, function-based routing, programmatic API |
| Config becomes too complex | Defeats the purpose | Keep spec small; test every addition against the three hand-written examples |
| Fast-changing LLM and MCP ecosystem | Breakage | Own interfaces for LLM and tools; contract tests; pin and monitor dependency versions |
| Non-deterministic LLM makes tests flaky | Unreliable CI | Mock provider for CI; real-provider tests separated and optional |
| Security (secrets in traces, unsafe tools, user code) | Trust loss | Redaction, tool allow-lists, approvals, environment-variable secrets, security review |
| Scope creep | Never ships | Written non-goals (Step 2); defer extras to Phase 9 |
| Solo-developer burnout | Project stalls | Milestones with visible demos; release early; ask for feedback early |

---

## 13. Success Metrics

**Developer experience**
- Time from install to first successful run: **under 5 minutes**
- Lines of definition compared with a raw LangGraph-style implementation of the same agent: **significantly fewer** (measure in Step 32)
- Share of validation errors that include an actionable fix: **near 100% on the catalog**

**Quality**
- Engine tests stable in CI with no flaky tests
- All example agents pass in CI

**Adoption (after release)**
- Number of developers completing the quickstart
- Issues and questions that indicate real usage
- Number of external templates or contributions

**Reliability**
- Percentage of runs that end in an explicit status (never "hangs")
- Resume success rate after forced restarts

---

## 14. Open Decisions Log

Resolve these as you go, and write down the decision and the reason:

1. **Final name and npm package name**
2. **Primary spec format:** YAML only, or YAML plus JSON, and how strictly the builder mirrors it
3. **Routing expression style:** how rich the rule language should be before you point users to custom functions
4. **Template syntax for state references** in prompts
5. **LLM layer:** commit to the AI SDK, or hide it fully behind your interface from day one
6. **Default persistence:** none, in-memory, or SQLite enabled by default
7. **Trace location and format:** local files versus embedded database
8. **Licensing:** permissive open source versus another model
9. **Sub-agent timing:** MVP or after
10. **Supported Node versions and platforms**

---

## 15. Glossary

| Term | Meaning |
|---|---|
| **Agent** | A program that uses an LLM to decide actions, often with tools, across multiple steps |
| **Spec** | The declarative file that describes an agent |
| **Node** | One step in the flow (LLM call, tool call, router, custom code, human approval) |
| **Edge** | A connection from one node to the next |
| **Router** | A node that chooses the next node based on rules, an LLM, or a function |
| **State** | The shared data passed between nodes |
| **Checkpoint** | A saved snapshot of state and position, used to pause and resume |
| **MCP** | Model Context Protocol, a standard way for agents to connect to external tools and data |
| **Trace** | The structured record of everything that happened during a run |
| **Human-in-the-loop** | A pause point where a person approves or provides input |
| **Escape hatch** | The ability to write plain code when config is not enough |

---

### Suggested working order (summary)
**Decide (1-3) → Define the spec and state (4-7) → Build the engine (8-15) → Add tools and MCP (16-18) → Make it visible (19-21) → Make it durable (22-25) → Make it pleasant (26-30) → Prove it (31-33) → Ship it (34-36) → Grow it (37-40).**
