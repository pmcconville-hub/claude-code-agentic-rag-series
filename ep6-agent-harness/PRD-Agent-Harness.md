# Episode 6 — PRD: Agent Harness & Domain-Specific Workflows

## Overview

This episode transforms the platform from a chat assistant with tool-calling into a **full autonomous agent platform with domain-specific workflow enforcement**. It builds two layers:

1. **General-Purpose Agent Harness (Deep Mode)** — A soft harness where the LLM controls the flow. Planning via todo lists, a per-thread workspace filesystem, general-purpose sub-agent delegation, and mid-task user clarification. The user activates this via a "Deep Mode" toggle.

2. **Domain-Specific Harness Engine** — A hard harness where the **system** controls the flow. A backend state machine drives deterministic phase transitions with programmatic validation gates. The LLM executes within each phase but cannot skip, reorder, or bypass phases. This is the architectural moat: plugins and skills can suggest steps, but a harness **enforces** them.

3. **Contract Review Harness** — The first domain harness implementation. An 8-phase workflow that takes a contract document, classifies it, gathers user context, loads playbook materials via RAG, extracts clauses, runs batched parallel risk analysis with isolated sub-agents, generates redlines, produces an executive summary, and outputs a downloadable DOCX report.

The key insight: **the model is commoditized, but structured enforcement of process is where value lives.** A domain harness guarantees every step completes and validates before advancing — the LLM cannot decide to skip due diligence.

---

## Part 1: General-Purpose Agent Harness (Deep Mode)

### Feature 1.1: Deep Mode Activation

#### What It Does

Adds a per-message "Deep Mode" toggle next to the Send button. When activated, the agent gains planning tools, workspace filesystem access, general-purpose sub-agent delegation, and an ask-user capability — transforming it from a single-turn assistant into an autonomous agent that plans and executes multi-step tasks.

#### How It Works

1. **Per-message toggle** — A toggle button next to Send controls whether the current message triggers deep mode. Per-message, not per-thread.

2. **Backend branching** — When `deep_mode=true`, the backend assembles an extended system prompt with planning/workspace/delegation/ask-user instructions, loads deep mode tools alongside existing tools, and uses a higher iteration limit (`MAX_DEEP_ROUNDS`) for the agent loop.

3. **When OFF** — Current behavior unchanged. No extra tools, no extra system prompt, no token overhead.

4. **Per-message persistence** — The `deep_mode` flag is stored on each message record so the UI can reconstruct panel visibility when loading thread history.

#### Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `MAX_DEEP_ROUNDS` | `50` | Maximum agent loop iterations in deep mode |

#### Design Decisions

- **KV-cache friendly** — Deep mode system prompt sections are deterministic and stable (no timestamps, no volatile data). Todo state flows through tools, not the system prompt, preserving cache-friendly stable prefixes.
- **Additive only** — Deep mode extends the existing agent loop. No refactoring of existing tool-calling behavior.

---

### Feature 1.2: Planning System (Todos)

#### What It Does

Gives the agent the ability to plan multi-step tasks by creating a structured todo list, then track progress as it works through each step. The user sees the plan updating in real-time in a sidebar panel.

#### How It Works

1. **Todo tools** — `write_todos` (replaces the full todo list for the thread) and `read_todos` (returns the current list). Full-replacement pattern — the model sends the complete updated list each time.

2. **Recitation pattern** — The system prompt instructs the agent to call `read_todos` after completing each step, reinforcing plan awareness and preventing drift during long sessions.

3. **Adaptive replanning** — The agent can add, remove, or rewrite tasks mid-execution based on what it discovers.

4. **Real-time UI** — Every `write_todos` or `read_todos` call emits a `todos_updated` SSE event. The Plan Panel in the sidebar updates immediately.

#### Data Model

New `agent_todos` table scoped to thread (`thread_id` FK). Each todo has `content`, `status` (pending/in_progress/completed), `position`. Full replacement on write. RLS enforced.

#### User-Facing UI

- **Plan Panel** — Sidebar panel showing todos with real-time status updates
- **Status indicators** — Visual differentiation for pending, in-progress, completed
- **History persistence** — Loading a thread with deep mode history shows last known state

---

### Feature 1.3: Agent Workspace (Virtual Filesystem)

#### What It Does

Provides a per-thread virtual workspace where the agent captures intermediate artifacts — research notes, drafts, data files, and deliverables. The user can browse, read, and download workspace files from a sidebar panel. Sandbox-generated files (code execution outputs) also appear in the workspace.

#### How It Works

1. **Workspace tools** — `write_file` (create/overwrite), `read_file` (read content), `edit_file` (exact string replacement), `list_files` (list all files).

2. **Dual storage model** — Text files stored directly in the database for fast access (<100ms). Binary files stored in Supabase Storage with metadata in the database.

3. **Sandbox integration** — Sandbox-generated files automatically create workspace entries with `source="sandbox"`. Solves the existing problem where sandbox file download links disappeared on page refresh.

4. **Shared access** — Sub-agents share the same workspace, enabling them to read context and write results.

5. **Workspace decoupled from deep mode** — Workspace panels are visible whenever a thread has workspace files, regardless of whether deep mode is active. This allows the harness engine (Part 2) to use workspace independently.

#### Data Model

New `workspace_files` table. Unique constraint on `(thread_id, file_path)`. `content` for text, `storage_path` for binary. `source` discriminator: `agent`, `sandbox`, `upload`. RLS enforced.

#### File Path Conventions

- Relative paths, forward slashes only (e.g., `notes/research.md`, `data/players.csv`)
- No path traversal, no leading `/`, no backslashes
- Max 500 characters, max 1MB text content
- Recommended directories: `notes/`, `data/`, `drafts/`, `deliverables/`

#### User-Facing UI

- **Workspace Panel** — Sidebar panel with file list, sizes, source indicators
- **File Viewer** — Click text file to view, click binary to download
- **Real-time updates** via SSE events
- **REST endpoints** — `GET /threads/{id}/files` (list), `GET /threads/{id}/files/{path}` (read/download)

---

### Feature 1.4: Sub-Agent Delegation (Task Tool)

#### What It Does

A general-purpose `task` tool lets the agent delegate focused work to sub-agents with isolated context windows. Sub-agents share the workspace but have their own message history.

#### How It Works

1. **Clean context** — Each sub-agent starts fresh with only its task description.
2. **Tool inheritance** — Same tools as parent, minus `task` (no recursion) and minus `write_todos`/`read_todos`.
3. **Shared workspace** — Read from and write to the same thread workspace.
4. **Result propagation** — Last assistant message text returned as tool result.
5. **Failure isolation** — Errors return as tool results, never crash the parent loop.

#### Tool Interface

- `task(description: str, context_files: list[str])` — `context_files` optionally pre-loads workspace files into sub-agent context.

#### Existing Sub-Agents Preserved

Additive — existing `analyze_document` and `explore_knowledge_base` remain unchanged.

---

### Feature 1.5: Ask User (Mid-Task Clarification)

#### What It Does

An `ask_user` tool pauses execution to ask the user a clarifying question. The agent waits for the response before continuing.

#### How It Works

1. Agent calls `ask_user` with a question.
2. Loop pauses, `ask_user` SSE event delivered, `agent_status` signals `waiting_for_user`.
3. User types their answer, delivered as the `ask_user` tool result (not a new top-level message).
4. Agent loop resumes.

---

### Feature 1.6: Agent Status and Error Handling

- **Status indicators** — `working`, `waiting_for_user`, `complete`, `error`
- **Append-only errors** — Failed tool calls remain in context for the LLM to learn from
- **LLM-driven recovery** — No automatic retries. The agent decides: retry, try alternative, or escalate via `ask_user`
- **Sub-agent failure isolation** — Errors return as tool results, parent continues
- **Loop exhaustion** — At `MAX_DEEP_ROUNDS`, system forces the agent to summarize and deliver
- **Interruption safety** — Users can stop anytime, all completed work persisted

---

### Feature 1.7: Session Persistence

All state persisted to the database as work progresses:
- Messages + tool calls persisted after each loop iteration
- Todos persisted on every `write_todos` call
- Workspace files persisted on every write/edit
- `deep_mode` flag persisted per message

Reconnection loads state from DB via REST. User sends follow-up to resume — agent reads its todos and workspace to continue.

---

## Part 2: Domain-Specific Harness Engine

### Feature 2.1: Harness Engine (Backend State Machine)

#### What It Does

A harness engine that drives domain-specific workflows with **deterministic phase transitions**, **programmatic validation gates**, and flexible execution within each phase. The system advances phases — not the LLM. This is the fundamental difference from the general-purpose deep mode where the LLM controls the flow.

#### Phase Types

| Phase Type | Description |
|------------|-------------|
| `programmatic` | Pure Python code, no LLM. Document extraction, data parsing. |
| `llm_single` | Single LLM call with structured JSON output via Pydantic schema validation. |
| `llm_agent` | Multi-round agent loop (like task sub-agents) with tool access. |
| `llm_batch_agents` | Batched parallel sub-agents — spawns isolated task agents per item (e.g., per clause) with configurable concurrency. |
| `llm_human_input` | Pauses the harness for user input. Generates a context-gathering question, waits for response. |

#### Phase Definition

Each phase is a typed dataclass:
- `name`, `description` — human-readable
- `phase_type` — one of the types above
- `system_prompt_template` — focused, per-phase prompt (5-15 lines vs. the 400-line generic prompt)
- `tools` — curated list of tools for this phase (only what's needed)
- `output_schema` — Pydantic model for structured output validation
- `validator` — optional programmatic validation function
- `workspace_inputs` — which workspace files this phase reads
- `workspace_output` — which workspace file this phase writes
- `batch_size` — for `llm_batch_agents`, concurrency limit (default 5)
- `post_execute` — optional async callback after phase completion (e.g., DOCX generation)

#### Harness Engine Architecture

1. **Phase dispatch** — Engine dispatches by `PhaseType` enum. Each type has its own execution strategy.
2. **Workspace-based context passing** — Phases read from and write to workspace files. No inline `$prior_results` dumps. The orchestrator passes file paths, not content.
3. **Thin orchestrator** — The main thread tracks only phase transitions and workspace references. Token count reflects orchestrator overhead (~5k tokens), not total analysis work.
4. **Structured output enforcement** — `llm_single` phases use OpenAI `response_format: { type: "json_schema" }` for guaranteed structured output. Pydantic validation before advancing.
5. **Phase timeout** — Configurable per phase (120s for `llm_single`, 300s for `llm_agent`). Timeout = phase failure.
6. **Cancellation support** — Cancel event checked between rounds/phases. Clean shutdown on client disconnect.

#### Data Model

New `harness_runs` table:
- `thread_id` (FK), `harness_type`, `status` (pending/running/paused/completed/failed)
- `current_phase` (index), `phase_results` (JSONB — phase index → structured output)
- `input_file_ids` (workspace files used as input)
- Unique constraint: one active run per thread
- RLS enforced via thread ownership

#### Harness Registry

Simple dict-based registry. Adding a new harness = adding a file to `harnesses/`, registering it, and the engine handles everything else.

#### SSE Events

| Event | When |
|-------|------|
| `harness_phase_start` | Phase begins (index, name, description) |
| `harness_phase_complete` | Phase completes (index, name, result summary) |
| `harness_phase_error` | Phase fails (index, name, error) |
| `harness_complete` | All phases done (harness type, overall result) |
| `harness_batch_start` | Batch of sub-agents begins (batch index, size, total items) |
| `harness_batch_complete` | Batch completes (results count) |
| `harness_sub_agent_start` | Individual sub-agent spawned (clause ref, description) |
| `harness_sub_agent_complete` | Sub-agent finished (clause ref, result) |
| `harness_human_input_required` | Harness paused for user input |

---

### Feature 2.2: Gatekeeper LLM (Pre-Harness)

#### What It Does

A conversational agent that validates prerequisites before the harness starts. No tools — purely conversational. Checks whether required files are uploaded, greets the user, and triggers the harness when everything is ready.

#### How It Works

1. **Stateless across messages** — Each message checks harness run state. No active/completed run = gatekeeper handles it.
2. **Sentinel-based trigger** — Gatekeeper streams normally (character-by-character). When prerequisites are met, it ends its response with `[TRIGGER_HARNESS]`. The sentinel is stripped before display and the harness begins in the same SSE stream.
3. **Multi-turn** — Can go back-and-forth with the user ("Please upload your contract first").
4. **Configurable per harness** — `HarnessPrerequisites` dataclass with `requires_upload`, `upload_description`, `harness_intro`. Harnesses without prerequisites skip the gatekeeper entirely.

---

### Feature 2.3: Post-Harness Response LLM

#### What It Does

After harness completion, an LLM generates a conversational summary — key findings, risk level, where to find the full report. The agent then reverts to normal mode with full tools for follow-up questions.

#### How It Works

1. Phase results loaded into system prompt (not a fabricated user message).
2. LLM streams a concise summary (~500 tokens) referencing the workspace report.
3. Persisted as a separate assistant message.
4. Follow-up messages route through normal LLM loop with phase results available in context.
5. Token budget: phase results exceeding 30,000 characters are truncated (last 2 phases kept in full detail).

---

### Feature 2.4: Human-in-the-Loop Context Gathering

#### What It Does

Mid-harness pause that gathers user context informed by what the harness has already discovered. The harness generates intelligent questions based on prior phase results, then waits for the user's response before continuing.

#### How It Works

1. The `llm_human_input` phase type generates a question using classification results (contract type, parties, governing law).
2. Question streams as a normal chat message (not in the phase panel).
3. Harness sets status to `paused`.
4. User responds in chat — response captured, written to workspace file, phase marked complete, harness resumes.
5. Questions are specific: Which side are you on? Deadline pressure? Focus areas? Deal context?

---

### Feature 2.5: Batched Parallel Sub-Agents

#### What It Does

For phases that process multiple items (e.g., analyzing 15 clauses), the engine spawns isolated sub-agents in configurable batches. Each agent has clean context, shared workspace access, and streams real-time progress.

#### How It Works

1. Items parsed from workspace file (e.g., `clauses.md` contains a JSON array of extracted clauses).
2. Chunked into batches of `batch_size` (default 5).
3. Each batch runs concurrently via `asyncio.gather()` using existing `run_task_agent()` pattern.
4. Sub-agent events stream in real-time via `asyncio.Queue` pattern — not delayed until batch completes.
5. Results accumulated into workspace output file per batch.
6. Each sub-agent reads only what it needs from workspace (playbook context, review context).

#### Resumability

If interrupted mid-batch, the engine detects partial workspace output (e.g., 10 of 15 clause assessments in `risk-analysis.md`), computes remaining items, and resumes from where it left off.

---

### Feature 2.6: File Upload to Chat Workspace

#### What It Does

Enables uploading DOCX and PDF files directly into a chat thread's workspace. Required for domain harnesses that process user-provided documents.

#### How It Works

1. Upload endpoint: `POST /threads/{thread_id}/files/upload`
2. Binary files uploaded to Supabase Storage, metadata stored in `workspace_files` with `source='upload'`.
3. Text extraction utilities (`python-docx`, `PyPDF2`) extract content for the harness to process.
4. File upload button appears in chat input when a harness mode is active.

---

### Feature 2.7: Plan Panel Integration

#### What It Does

Reuses the existing Plan Panel (from deep mode) to show harness phases, but with a **locked** variant — a lock icon communicates that the plan is system-driven, not LLM-controlled.

#### How It Works

1. Harness engine writes phases to `agent_todos` with content prefix (e.g., `[Contract Review] Clause Extraction`).
2. Plan Panel shows phases progressing: pending → in_progress → completed.
3. `write_todos`/`read_todos` tools stripped from harness phase LLM calls — the LLM cannot modify the plan.
4. Lock icon in panel header communicates immutability.

---

## Part 3: Contract Review Harness

### The First Domain Implementation

An 8-phase deterministic workflow that takes a contract document and produces a comprehensive review with risk assessment, redlines, and a downloadable DOCX report.

### Phase Architecture

| Phase | Name | Type | Reads From | Writes To |
|-------|------|------|------------|-----------|
| 1 | Document Intake | `programmatic` | Upload file | `contract-text.md` |
| 2 | Contract Classification | `llm_single` | `contract-text.md` | `classification.md` |
| 3 | Gather Context | `llm_human_input` | `classification.md` | `review-context.md` |
| 4 | Load Playbook | `llm_agent` (RAG tools) | `classification.md`, `review-context.md` | `playbook-context.md` |
| 5 | Clause Extraction | `programmatic` (with internal LLM) | `contract-text.md` | `clauses.md` |
| 6 | Risk Analysis | `llm_batch_agents` (batch=5) | `clauses.md`, `playbook-context.md`, `review-context.md` | `risk-analysis.md` |
| 7 | Redline Generation | `llm_batch_agents` (batch=5) | `risk-analysis.md` (YELLOW/RED only), `review-context.md` | `redlines.md` |
| 8 | Executive Summary | `llm_single` + DOCX post-execute | `progress.md`, `risk-analysis.md`, `redlines.md`, `review-context.md` | `contract-review-report.md` + DOCX |

### Phase Details

**Phase 1 — Document Intake** (`programmatic`): Reads the uploaded file, extracts text via `python-docx` (DOCX) or `PyPDF2` (PDF), writes full text to `contract-text.md`.

**Phase 2 — Contract Classification** (`llm_single`): Classifies the contract by type, parties, effective/expiration dates, governing law, jurisdiction. Output validated against Pydantic schema (at least 2 parties, non-empty type). Writes `classification.md`.

**Phase 3 — Gather Context** (`llm_human_input`): Pauses the harness. Uses classification results to generate informed questions: Which side are you on? Deadline pressure? Focus areas? Deal context? User's response written to `review-context.md`.

**Phase 4 — Load Playbook** (`llm_agent`, max 10 rounds): Uses RAG tools (`search_documents`, `analyze_document`) to discover relevant playbook materials from the knowledge base. Reads first ~100 lines of each to assess relevance. Writes `playbook-context.md` with doc IDs, titles, summaries, and clause category mappings.

**Phase 5 — Clause Extraction** (`programmatic` with internal LLM calls): Extracts every distinct clause. For large contracts (>50k tokens), splits into overlapping chunks, runs LLM per chunk, merges and deduplicates. Writes `clauses.md` as JSON array. Categories: Liability, Indemnification, IP, Data Protection, Confidentiality, Warranties, Term/Termination, Governing Law, Insurance, Assignment, Force Majeure, Payment, Other.

**Phase 6 — Risk Analysis** (`llm_batch_agents`, batch_size=5): Spawns isolated sub-agents per clause. Each agent assesses risk (GREEN/YELLOW/RED) against playbook standards, provides rationale, suggests alternative language. Agents have RAG tools to deep-read specific playbook docs. Results accumulated into `risk-analysis.md`. User sees real-time progress: "Analyzing clause 8/15" with nested tool calls per agent.

**Phase 7 — Redline Generation** (`llm_batch_agents`, batch_size=5): Processes only YELLOW/RED assessments. Each agent generates precise redline markup: original text, proposed replacement, rationale, fallback positions. Results in `redlines.md`.

**Phase 8 — Executive Summary** (`llm_single` + `post_execute`): Reads all workspace artifacts, generates executive summary with overall risk, recommendation, key findings, risk breakdown. Writes `contract-review-report.md`. Then a `post_execute` callback runs a Python script in the sandbox to generate a professionally formatted DOCX report (title page, risk breakdown, redline table, acceptable clauses, next steps).

### DOCX Report Generation

The `post_execute` hook on Phase 8 generates a downloadable Word document:
- **Title page** — "CONFIDENTIAL", contract type, risk rating badge, prepared-for party
- **Executive summary** — Overall risk, recommendation, risk breakdown (RED/YELLOW/GREEN counts)
- **Key findings** — Numbered list from executive summary
- **Detailed redline table** — Each clause with risk level (color-coded), original text, proposed text, rationale
- **Acceptable clauses** — GREEN clauses with "no changes recommended"
- **Recommended next steps**
- Generated via `python-docx` in the sandbox. Non-fatal — if sandbox is unavailable, the LLM summary is still saved.

### Key Differentiators

1. **Load Playbook as a first-class RAG phase** — Real demonstrable RAG against the user's knowledge base
2. **Batched parallel clause agents** — 5 concurrent sub-agents with real-time visibility per clause
3. **Human-in-the-loop context gathering** — Mid-workflow, informed by classification results
4. **Thin orchestrator** — ~5k tokens regardless of contract size (context offloaded to workspace)
5. **Workspace-based resumability** — Interrupted harness detects partial progress and resumes
6. **Downloadable DOCX deliverable** — Professional report ready for stakeholder distribution

---

## Cross-Cutting Concerns

### Security

- All new tables (`agent_todos`, `workspace_files`, `harness_runs`) have Row-Level Security
- Users only see data for their own threads
- Sub-agents operate within the same user context — no privilege escalation
- Model provider API keys remain server-side only

### SSE Streaming

All events follow the existing pattern: `data: {"type": "<event_name>", ...payload}\n\n`. Deep mode and harness events include `thread_id`. No changes to transport — just additional event types within the existing `StreamingResponse` async generator.

### Model Provider Compatibility

Works identically with any OpenAI-compatible API provider (OpenRouter, Ollama). Provider selection is configuration, not code.

### Observability

`progress.md` written by the harness engine (single writer) after each phase transition. Contains status AND intermediate detail — classification summary, clause count/categories, risk tally, redline count. Workspace files serve as the source of truth for intermediate progress.

---

## Summary of New LLM Tools (Deep Mode)

| Tool | Purpose |
|------|---------|
| `write_todos` | Replace the full todo list for the current thread |
| `read_todos` | Read the current todo list (recitation pattern) |
| `write_file` | Create or overwrite a workspace file |
| `read_file` | Read workspace file content |
| `edit_file` | Edit a workspace file via exact string replacement |
| `list_files` | List all workspace files for the thread |
| `task` | Delegate work to a sub-agent with isolated context |
| `ask_user` | Ask the user a clarifying question and pause |

All tools are only available when Deep Mode is ON. Harness phases have their own curated tool sets.

---

## New Data Models

| Table | Purpose |
|-------|---------|
| `agent_todos` | Per-thread todo list for agent planning |
| `workspace_files` | Per-thread virtual filesystem (text + binary) |
| `harness_runs` | Harness execution state — phases, results, status |

### Modified Data

| Table | Change |
|-------|--------|
| `messages` | New `deep_mode` boolean, `harness_mode` text columns |

---

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `MAX_DEEP_ROUNDS` | `50` | Maximum agent loop iterations in deep mode |
| `MAX_TOOL_ROUNDS` | `25` | Maximum tool rounds in standard mode |
| `MAX_SUB_AGENT_ROUNDS` | `15` | Maximum rounds for sub-agents |

---

## Phased Delivery

| Phase | What |
|-------|------|
| Epic 1 | Deep mode database, todo service, system prompt, toggle, plan panel |
| Epic 2 | Workspace database, service, sandbox re-engineering, SSE events, workspace panel |
| Epic 3 | Sub-agent delegation (task tool), SSE events, UI indicator |
| Epic 4 | Ask-user tool, agent interruption, error handling, status visibility |
| Epic 5 | Tool integration (RAG, sandbox, skills adapters), session persistence |
| Domain Harness | Harness engine, file upload, gatekeeper LLM, contract review phases |
| Harness V2 | Context-isolated sub-agents, batched parallel execution, workspace-based context, human-in-the-loop, DOCX generation |

---

## Post-MVP Features (Phase 2)

- Context auto-compaction/summarization for very long sessions
- Human-in-the-loop tool approval for sensitive operations
- Agent memory across sessions (persistent per-user instructions)
- Background/async agent runs (loop continues server-side)
- Stall detection and auto-replan
- Full SSE reconnection with automatic loop resumption
- Additional domain harnesses (NDA generator, vendor assessment, compliance check)
