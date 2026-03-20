# Episode 6: Agent Harness & Domain-Specific Workflows

This episode transforms the platform from a chat assistant with tool-calling into a **full autonomous agent platform** with two layers: a general-purpose agent harness (Deep Mode) for LLM-driven multi-step tasks, and a domain-specific harness engine for system-enforced deterministic workflows. The first domain implementation is a contract review harness with batched parallel sub-agents, RAG-based playbook discovery, human-in-the-loop context gathering, and DOCX report generation.

## What It Is

**The model is commoditized. Structured enforcement of process is the moat.**

- **Deep Mode (Soft Harness)** — The LLM controls the flow. Planning via todo lists, workspace filesystem, sub-agent delegation, ask-user clarification.
- **Harness Engine (Hard Harness)** — The system controls the flow. Backend state machine with deterministic phases, programmatic validation gates, and curated per-phase tools. The LLM executes within phases but cannot skip, reorder, or bypass them.
- **Contract Review** — 8-phase harness: document intake → classification → human context gathering → playbook RAG → clause extraction → batched risk analysis → redline generation → executive summary + DOCX report.

## What You'll Build

### General-Purpose Agent (Deep Mode)
- **Deep Mode Toggle** — Per-message activation of autonomous agent capabilities
- **Planning System (Todos)** — Agent-managed todo list with real-time sidebar panel
- **Agent Workspace** — Per-thread virtual filesystem for notes, drafts, data, and deliverables
- **Sub-Agent Delegation (`task` tool)** — General-purpose sub-agents with isolated context, shared workspace
- **Ask User Tool** — Agent pauses mid-task for clarifying questions
- **Agent Status Indicators** — Real-time visibility (working, waiting, complete, error)
- **Session Persistence** — All state persisted to DB, survives connection drops

### Domain-Specific Harness Engine
- **Harness Engine** — Backend state machine with 5 phase types (programmatic, llm_single, llm_agent, llm_batch_agents, llm_human_input)
- **Gatekeeper LLM** — Pre-harness conversational agent that validates prerequisites before launching
- **Post-Harness Response LLM** — Generates a conversational summary after completion, then reverts to normal agent mode
- **File Upload to Workspace** — DOCX/PDF upload with text extraction
- **Locked Plan Panel** — System-driven phases with lock icon (LLM cannot modify)
- **Phase-Level SSE Events** — Real-time phase transitions, sub-agent spawns, batch progress

### Contract Review Harness
- **8 Deterministic Phases** — Document intake through executive summary
- **Human-in-the-Loop Context** — Mid-harness pause for user context (side, deadline, focus areas)
- **Playbook RAG Discovery** — Searches knowledge base for relevant standards and policies
- **Batched Parallel Clause Analysis** — 5 concurrent sub-agents per batch, each with isolated context and RAG tools
- **Risk Assessment** — GREEN/YELLOW/RED per clause with rationale and suggested language
- **Redline Generation** — Precise markup for flagged clauses with fallback positions
- **DOCX Report** — Professionally formatted Word document via python-docx sandbox execution
- **Workspace Resumability** — Detects partial progress, resumes from where it left off

## Key Concepts

| Feature | Before | After |
|---------|--------|-------|
| Chat mode | Single-turn completions with tool calling | Autonomous multi-step agent loop with planning |
| Artifacts | Ephemeral (lost on page refresh) | Persisted workspace with browse/view/download |
| Sub-agents | Specialized only (doc analysis, explorer) | General-purpose delegation via `task` tool |
| Workflow enforcement | LLM-driven (suggestive) | System-driven state machine (deterministic) |
| Per-phase context | Full 400-line system prompt, all tools | 5-15 line focused prompt, curated tools |
| Orchestrator overhead | Scales with document size | Fixed ~5k tokens (context in workspace files) |
| User interaction | Send message, wait | Mid-task questions, file upload, interruption |
| Error handling | Tool errors crash the loop | Append-only errors, LLM-driven recovery |
| Deliverables | Text in chat | DOCX reports, structured workspace artifacts |

## New LLM Tools (Deep Mode)

| Tool | Purpose |
|------|---------|
| `write_todos` | Create/replace the agent's todo list |
| `read_todos` | Read current todos (recitation pattern) |
| `write_file` | Create or overwrite a workspace file |
| `read_file` | Read a workspace file |
| `edit_file` | Edit a file via exact string replacement |
| `list_files` | List all workspace files |
| `task` | Delegate work to an isolated sub-agent |
| `ask_user` | Ask the user a question and pause |

## Harness Phase Types

| Type | Description |
|------|-------------|
| `programmatic` | Pure Python code, no LLM (document extraction, data parsing) |
| `llm_single` | Single LLM call with Pydantic-validated structured JSON output |
| `llm_agent` | Multi-round agent loop with tool access (RAG, workspace) |
| `llm_batch_agents` | Batched parallel sub-agents per item (configurable concurrency) |
| `llm_human_input` | Pauses for user input, generates informed questions |

## Contract Review Phases

| Phase | Name | Type | Output |
|-------|------|------|--------|
| 1 | Document Intake | `programmatic` | `contract-text.md` |
| 2 | Classification | `llm_single` | `classification.md` |
| 3 | Gather Context | `llm_human_input` | `review-context.md` |
| 4 | Load Playbook | `llm_agent` | `playbook-context.md` |
| 5 | Clause Extraction | `programmatic` | `clauses.md` |
| 6 | Risk Analysis | `llm_batch_agents` | `risk-analysis.md` |
| 7 | Redline Generation | `llm_batch_agents` | `redlines.md` |
| 8 | Executive Summary | `llm_single` + DOCX | `contract-review-report.md` + `.docx` |

## New Data Models

| Table | Purpose |
|-------|---------|
| `agent_todos` | Per-thread planning todo list |
| `workspace_files` | Per-thread virtual filesystem (text in DB, binary in storage) |
| `harness_runs` | Harness execution state, phase results, status |

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `MAX_DEEP_ROUNDS` | `50` | Maximum agent loop iterations in deep mode |
| `MAX_SUB_AGENT_ROUNDS` | `15` | Maximum rounds for sub-agents |

## Prerequisites

- Completed Episodes 1-5
- Supabase with Postgres + pgvector + Storage
- Docker for sandbox code execution
- `python-docx` and `PyPDF2` installed in backend venv (for text extraction)

## Documentation

See the [PRD](./PRD-Agent-Harness.md) for complete functional specifications.

## Community

Join [The AI Automators community](https://www.theaiautomators.com/) to connect with other builders creating production-grade AI and RAG systems.
