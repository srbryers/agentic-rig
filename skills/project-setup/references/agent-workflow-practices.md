# Agent Workflow Practices Reference

This document provides detection signals, architecture recommendations, and anti-patterns for projects that use AI agent frameworks. Use during Phase 1 analysis (Step 1.10) to identify agent projects and during Phase 3 to generate agent-specific CLAUDE.md sections, hooks, skills, and agents.

---

## 1. Detection Signal Groups

Agent projects are identified by four signal groups. Each group that matches increases recommendation depth.

### Group A: SDK Dependencies

Search manifest files for known agent SDK packages.

| Ecosystem | Package / Import | Framework |
|-----------|-----------------|-----------|
| Node.js | `@anthropic-ai/sdk`, `anthropic` | Anthropic SDK |
| Node.js | `openai` | OpenAI SDK |
| Node.js | `@google/generative-ai` | Google AI SDK |
| Node.js | `langchain`, `@langchain/*` | LangChain |
| Node.js | `ai`, `@ai-sdk/*` | Vercel AI SDK |
| Node.js | `llamaindex` | LlamaIndex |
| Node.js | `@mastra/*`, `mastra` | Mastra |
| Python | `anthropic` | Anthropic SDK |
| Python | `openai` | OpenAI SDK |
| Python | `google-generativeai` | Google AI SDK |
| Python | `langchain`, `langchain-*` | LangChain |
| Python | `crewai` | CrewAI |
| Python | `autogen`, `pyautogen` | AutoGen |
| Python | `llama-index`, `llama_index` | LlamaIndex |
| Python | `semantic-kernel` | Semantic Kernel |
| Python | `haystack-ai` | Haystack |
| Python | `dspy`, `dspy-ai` | DSPy |
| Rust | `async-openai` | OpenAI (Rust) |
| Go | `github.com/sashabaranov/go-openai` | OpenAI (Go) |
| .NET | `Microsoft.SemanticKernel` | Semantic Kernel |

### Group B: Session / Chat Storage

Search for files and patterns indicating **AI conversation** persistence. Generic terms like `session` and `thread` are common in non-agent web apps — always look for agent-specific context nearby.

| Signal | How to Detect | False Positive Risk |
|--------|--------------|---------------------|
| Chat history patterns | Grep for `chat_history`, `chat_messages`, `conversation_history`, `message_history` | Low — specific to conversational AI |
| LLM message arrays | Grep for `messages.push` or `addMessage` near LLM SDK imports (Group A packages) | Low — requires co-location with SDK usage |
| Agent session storage | Files or directories named `*chat*`, `*conversation*` with agent SDK imports in same module | Medium — check for co-location with Group A |
| Redis for LLM sessions | `ioredis`/`redis` in deps + `session` AND an agent SDK (Group A) in same project | Medium — Redis alone is not sufficient |
| DB session tables for agents | Migration/schema files with `conversations` or `messages` tables containing `role`/`content` columns | Low — role+content columns are specific to LLM message format |

**Important:** Do NOT match on `session`, `thread`, or `conversation` alone. These terms are ubiquitous in web frameworks (Express sessions, forum threads, email conversations). Only count Group B as matched when signals co-locate with Group A SDK usage or contain LLM-specific patterns (`role`/`content` message structure, `chat_history`).

### Group C: Orchestration Code

Search source files for agent orchestration patterns. Focus on LLM-specific orchestration — generic terms like `workflow` and `pipeline` are common in CI/CD and data processing.

| Signal | How to Detect | False Positive Risk |
|--------|--------------|---------------------|
| State graphs | Grep for `StateGraph`, `state_graph` (LangGraph-specific) | Low — highly specific |
| LLM tool registration | Grep for `register_tool`, `@tool`, `defineTool`, `createTool`, `tool_choice` near Group A SDK imports | Low — LLM-specific API patterns |
| Handoff patterns | Grep for `handoff`, `hand_off`, `transfer_to_agent` | Low — agent-specific terminology |
| Multi-agent configs | Files named `agents.yaml`, `crew.yaml`, `swarm.json`, `agent_config.*` | Low — specific config names |
| LLM chain/runnable | Grep for `RunnableSequence`, `RunnableParallel`, `create_chain`, `langchain` chain patterns | Low — LangChain-specific |

**Important:** Do NOT match on `workflow`, `pipeline`, or `chain` alone. These are common in CI/CD (GitHub Actions workflows), data processing (ETL pipelines), and blockchain projects. Only count Group C as matched when orchestration signals are LLM-specific or co-locate with Group A SDK imports.

### Group D: Agent File Structure

Check for directory patterns and config files common in agent projects. Require multiple signals or co-location with Group A — individual directories like `tools/` are common in non-agent repos.

| Signal | How to Detect | False Positive Risk |
|--------|--------------|---------------------|
| Agent directories | `agents/`, `agent/` directories containing files that import Group A SDKs | Medium — verify file contents |
| Prompt directories | `prompts/`, `prompt_templates/` directories exist | Low — fairly specific to LLM apps |
| Prompt files | Files with `.prompt`, `.jinja` extensions in a project with Group A deps | Medium — Jinja is used outside LLM context |
| Agent configs | `agent.yaml`, `crew.yaml`, `agents.json`, `swarm.config.*` | Low — specific config names |

**Important:** A `tools/` directory alone is NOT sufficient for Group D — it is common in build tooling, CLI tools, and general utility code. Only count `tools/` if it contains files that reference Group A SDK packages or LLM-specific patterns (tool schemas with `description`/`parameters` matching OpenAI function calling format).

### Confidence Thresholds

| Groups Matched | Confidence | Action |
|---------------|------------|--------|
| 0–1 | Low | Skip agent-specific recommendations |
| 2 | Medium | Include core recommendations (Section 6 core items) |
| 3+ | High | Include full recommendations (Section 6 core + extended items) |

Record the matched group count (0–4) and specific signals found for use in Phase 2 reporting.

---

## 2. Session Management

Session management covers the lifecycle of conversation state: creation, retrieval, updates, corrections, and cleanup.

### Lifecycle Operations

| Operation | Description | Key Consideration |
|-----------|-------------|-------------------|
| Create | Initialize new session with ID, metadata, empty history | Generate unique session IDs; set TTL or max-turn limits |
| Retrieve | Load session by ID for continued conversation | Handle missing sessions gracefully (expired, deleted) |
| Append | Add new turns (user + assistant) to session | Validate turn ordering; enforce max history length |
| Correct | Modify or remove previous turns (user edits, tool errors) | Maintain turn consistency; log corrections |
| Clear | Reset session history while preserving metadata | Useful for "start over" without losing session identity |
| Delete | Permanently remove session and all associated data | Respect data retention policies; cascade to related data |

### Storage Backend Selection

Choose the simplest backend that meets the project's requirements. Start simple and migrate as volume grows.

| Backend | Best For | Complexity | Notes |
|---------|----------|------------|-------|
| **Markdown files** | Dev, prototyping, single-agent, low-volume | Minimal | Human-readable, Git-trackable, no dependencies. One `.md` file per session in `sessions/` dir. Use YAML frontmatter for metadata, structured markdown for turns. |
| **JSON files** | Dev, structured data, programmatic access | Minimal | One `.json` file per session. Easier to parse than markdown. |
| **SQLite** | Single-server, moderate volume | Low | No setup required. Good for local dev and staging. |
| **PostgreSQL** | Production, multi-server, high volume | Medium | JSONB columns for flexible message schema. Required for concurrent access. |
| **Redis + PostgreSQL** | Production, high performance | High | Redis for hot sessions (<5ms reads), Postgres for durability. |

### Markdown Session File Format

For projects using file-based session storage:

```markdown
---
id: sess_abc123
user_id: user_789
created: 2024-01-15T10:30:00Z
updated: 2024-01-15T11:45:00Z
status: active
---

# Session: sess_abc123

## Turns

### Turn 1 — user (10:30:00Z)
Hello, I need help with my order

### Turn 2 — assistant (10:31:00Z)
I'd be happy to help. Could you provide your order number?

## Context
- Topic: Order inquiry
- Key facts: Customer is premium tier

## Tool Outputs
- `get_order(456)` → status: shipped, tracking: 1Z999...
```

### Session ID Conventions

| Format | Example | Use Case |
|--------|---------|----------|
| UUID v4 | `550e8400-e29b-41d4-a716-446655440000` | Default; guaranteed unique |
| Prefixed | `sess_abc123` | Human-readable; easy to filter in logs |
| Composite | `user_789:sess_abc123` | Multi-tenant; encodes ownership |

---

## 3. Context Window Management

Strategies for keeping conversation context within model token limits.

### Strategy Selection

| Strategy | How It Works | Best For | Trade-off |
|----------|-------------|----------|-----------|
| Trimming | Drop oldest turns beyond a limit | Simple bots, low-context tasks | Loses early context |
| Summarization | Periodically summarize older turns into a condensed block | Long conversations, support bots | Costs extra LLM calls |
| Hybrid | Keep last N turns verbatim + summary of older turns | Most production agents | Balanced cost/quality |
| Sliding + Pinned | Keep system prompt + pinned messages + sliding window of recent turns | Agents with important instructions | Most control, more complexity |

### Token Budget Allocation

| Component | Budget Share | Notes |
|-----------|-------------|-------|
| System prompt | 10–15% | Instructions, persona, constraints |
| Memory / context | 15–25% | Retrieved facts, session summaries, user preferences |
| Conversation | 40–50% | Recent turns (user + assistant) |
| Tool definitions + outputs | 10–20% | Available tools schema + recent tool results |
| Headroom | 10–15% | Buffer for model response generation |

### Tool Output Handling

- Truncate large tool outputs to essential fields before adding to context
- Store full tool outputs externally; include only summaries in conversation
- Set per-tool output size limits (e.g., max 500 tokens per tool result)

### File-Based Session Context Loading

For markdown/file-backed sessions:
- Keep the full conversation history in the session file on disk
- Load only the last N turns into the model's context window
- Write a `## Summary` section at the top of the file for older turns
- On session load, include: system prompt + summary + last N turns

---

## 4. Memory Architecture

Three-tier model for agent memory, from simplest to most complex.

### Session Memory (Short-Term)

The current conversation. For file-based backends, this is the session file itself.

- Automatically managed by session lifecycle operations
- Cleared when session ends or is deleted
- No special infrastructure needed

### Episodic Memory (Medium-Term)

Facts and events extracted from past sessions that may be relevant to future interactions.

| Backend | Implementation |
|---------|---------------|
| Markdown files | `memory/` directory with one file per topic or user. Append extracted facts as bullet points. |
| JSON files | Structured fact store with timestamps and source session IDs |
| Database | Dedicated `memories` table with embeddings for semantic search |
| Vector store | Pinecone, Weaviate, pgvector for semantic retrieval |

### Semantic Memory (Long-Term)

Structured knowledge about users, entities, and domain concepts.

| Backend | Implementation |
|---------|---------------|
| Markdown/JSON files | Structured files for user preferences, entity data, domain knowledge |
| Database | Normalized tables for entities with relationships |
| Knowledge graph | Neo4j or similar for complex entity relationships |

### Scale-Up Path

```
Markdown files → JSON files → SQLite → PostgreSQL → PostgreSQL + Vector DB
```

Start with the simplest backend. Migrate when:
- File I/O becomes a bottleneck (>100 concurrent sessions)
- You need semantic search across memories
- You need multi-server access to session data

---

## 5. Workflow Patterns

Common execution patterns in multi-step agent workflows.

### Multi-Phase Execution

| Pattern | Description | Example |
|---------|-------------|---------|
| Sequential | Steps execute in order; each receives previous output | Research → Draft → Review → Publish |
| Router | A dispatcher agent routes to specialized sub-agents | Triage agent → Support / Sales / Technical |
| Fan-out | Parallel execution with result aggregation | Search 3 sources simultaneously, merge results |
| Iterative loop | Repeat until quality threshold met | Generate → Evaluate → Refine → Evaluate → ... |

### Checkpoint / Resume

For long-running workflows that may be interrupted:
- Save workflow state at each phase boundary
- Use markdown checkpoint files (similar to `/checkpoint` skill pattern)
- Include: current phase, completed steps, intermediate results, next action
- On resume: load checkpoint, verify state consistency, continue from last completed step

### Correction Workflows

When a tool call or agent step produces incorrect results:
- **Pop-and-retry**: Remove the bad turn from history, re-run with adjusted prompt
- **Append correction**: Keep the bad turn, add a correction instruction, continue
- **Rollback to checkpoint**: Revert to last known good state, re-execute

---

## 6. CLAUDE.md Section Recommendations

Sections to add to generated CLAUDE.md files based on agent signal group count.

### Core Sections (2+ Groups Matched)

| Section | Content |
|---------|---------|
| Agent Architecture | SDK used, agent types (single/multi/swarm), orchestration pattern, tool registration approach |
| Session Management | Storage backend, session ID format, lifecycle operations, TTL/cleanup policy |
| Context Management | Window strategy (trim/summarize/hybrid), token budget split, tool output handling |

### Extended Sections (3+ Groups Matched)

| Section | Content |
|---------|---------|
| Memory Architecture | Memory tiers in use, storage backends per tier, retrieval strategy |
| Workflow Patterns | Execution pattern (sequential/router/fan-out), checkpoint strategy, error recovery |
| Tool Management | Tool directory structure, registration pattern, input/output schemas, testing approach |
| Prompt Management | Prompt file location, templating system, variable substitution, version control |
| Testing Agent Workflows | How to test tool calls, mock LLM responses, validate conversation flows, test memory retrieval |

---

## 7. Hook Recommendations

Hooks to recommend for agent projects.

### PreToolUse Hooks

| Signal | Matcher | Action | Rationale |
|--------|---------|--------|-----------|
| Session storage files detected | `Write\|Edit` | Block edits to `sessions/`, `conversations/`, `chat_history/` | Session files should be managed by application code, not edited directly |
| Vector store indexes detected | `Write\|Edit` | Block edits to vector index files (`.index`, `.faiss`, `chroma/`) | Index files are generated by the vector store, not edited manually |
| Prompt template directory detected | `Write\|Edit` | Warn (not block) on edits to `prompts/`, `prompt_templates/` | Prompt changes can have wide impact; encourage review |

### PostToolUse Hooks

| Signal | Matcher | Action | Rationale |
|--------|---------|--------|-----------|
| Session schema files detected | `Write\|Edit` | Warn on changes to session/message type definitions | Schema changes can break existing sessions |

---

## 8. Skill and Agent Recommendations

### Skills

| Signal | Skill Name | Invocation | Description | When to Recommend |
|--------|-----------|------------|-------------|-------------------|
| `tools/` directory with 3+ files | new-tool | `/new-tool` | Scaffold a new agent tool with input/output schema, handler, and test stub | Tool directory pattern established |
| `prompts/` directory detected | new-prompt | `/new-prompt` | Create a new prompt template with variables and metadata | Prompt template pattern established |

### Agents

| Signal | Agent Name | Agent File | Model | Tools | Description | When to Recommend |
|--------|-----------|------------|-------|-------|-------------|-------------------|
| Session storage detected | session-auditor | `session-auditor.md` | haiku | Read, Grep, Glob | Audits session storage for unbounded growth, missing TTLs, cross-tenant data leaks | Session/chat storage signals found (Group B) |
| Prompt templates detected | prompt-reviewer | `prompt-reviewer.md` | haiku | Read, Grep, Glob | Reviews prompt templates for injection risks, missing variables, inconsistent formatting | Prompt directory or template files found (Group D) |

---

## 9. Common Anti-Patterns

Document these in CLAUDE.md when agent patterns are detected, to guide future development.

| Anti-Pattern | Problem | Recommendation |
|-------------|---------|----------------|
| All tools loaded at once | Wastes context window tokens on irrelevant tool schemas | Load tools conditionally based on conversation state or agent role |
| Unbounded session growth | Sessions grow indefinitely, eventually exceeding context limits | Set max turn limits; implement summarization or trimming |
| Shared memory across tenants | User A's data leaks into User B's context | Namespace all memory by tenant/user ID; validate access on retrieval |
| Hardcoded prompts in source | Prompt changes require code deploys; hard to version and A/B test | Use external prompt files with variable substitution |
| No session cleanup | Expired sessions accumulate, consuming storage | Implement TTLs or scheduled cleanup jobs |
| Synchronous tool execution | Sequential tool calls when parallel execution is possible | Use fan-out pattern for independent tool calls |
| Large binary outputs inline | Storing full API responses or file contents inside session markdown | Reference external files for large outputs; store only summaries inline |

---

## 10. Compound Signals

Combinations of agent signals with other project signals that trigger additional recommendations.

| Compound Signal | Additional Recommendation | Rationale |
|----------------|--------------------------|-----------|
| Agent SDK + Redis | Session Management sections + Redis MCP server | Redis is likely used for session caching; MCP enables direct inspection |
| Agent SDK + Vector DB (pinecone, weaviate, chromadb, pgvector) | Memory Architecture sections + episodic memory guidance | Vector DB indicates semantic memory; needs architecture documentation |
| Agent SDK + test framework (jest, pytest, vitest) | Testing Agent Workflows section in CLAUDE.md | Tests exist; document how to test agent-specific patterns (mocking LLMs, validating flows) |
| Agent SDK + Docker | Session storage isolation notes in Deployment section | Containerized agents need volume mounts or external storage for sessions |
| Agent SDK + queue system (bull, celery, rabbitmq) | Workflow Patterns section with async execution guidance | Async task processing indicates complex workflow orchestration |
