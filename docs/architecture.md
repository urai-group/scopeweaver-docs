---
title: "Architecture"                # The page title shown in the sidebar and page header
description: "Architecture of the Scopeweaver project"  # Short description of the page
layout: default                  # Layout to use: default, minimal, etc.
---

# Architecture

> [Back to README](README.md) | [Tasks](TASKS.md) | [Contributing](CONTRIBUTING.md) | [Visual Diagrams](docs/architecture/overview.md)

This document describes the internal architecture of Scope Weaver — a platform for building narrow AI agents. The architecture is agent-agnostic: skills are pluggable, the pipeline is generic, and the primitive type system is extensible to any domain. The first agent built on the platform is a **file management agent**, serving as the demonstrator for the AI Day Regensburg competition.

This document is for developers joining the team who need to understand the system before contributing code. Start with the [README](README.md) for a project overview, or see [TASKS.md](TASKS.md) for what to build next.

---

## Background

The project runs on two parallel tracks:

- **Competition prototype** for AI Day Regensburg (mid-April 2026), developed within URAI, the AI student association at the University of Regensburg and OTH Regensburg. Priority: UI/UX impressiveness.
- **Enterprise engagement** the same architecture adapts to embedded software management. Priority: backend robustness, logging, auditability.

| Track | Priority | Definition of Success |
|---|---|---|
| **Competition -- AI Day Regensburg** | UI/UX impressiveness | The system needs to generally work and perform okay. Visual polish and user experience are what matter. Goal: impress the audience and demonstrate what's possible. |
| **Enterprise** | Backend robustness and reliability | Logging, security, auditability, and accurate retrieval matter most. The system must be production-grade and handle real-world complexity. |

---

## System Overview

Scope Weaver is organized into three layers: a frontend for user interaction, an agent pipeline for planning and execution, and a sandbox that manages the file-system state. Every user request flows down through these layers and back up.

- **Frontend** -- Spotlight-style chat interface and real-time graph/pipeline visualization via [Cytoscape.js](https://js.cytoscape.org/).
- **Agent Pipeline** -- the brain: plans the task (Thinker via Gemma 4 E2B), retrieves the right skill (Retriever via EmbeddingGemma), dispatches the declared primitive (Executor).
- **Sandbox + Storage** -- the workspace: the `Sandbox` object owns the file-system graph and both vector indexes and manages their shared lifecycle. The observability stack records everything.

For visual renderings of this architecture (system overview, pipeline sequence, data model, dependency graph, developer workflow), see [`docs/architecture/overview.md`](docs/architecture/overview.md).

---

## Model Stack

Scope Weaver runs two on-device models, both small enough for any modern laptop. No cloud inference is required.

### Thinker -- Gemma 4 E2B

The Thinker is the planning model. It receives a user's natural-language request and produces a structured JSON plan specifying which skills, tools, and parameters to use.

| Property | Value |
|---|---|
| Parameters | 2.3 billion effective (mixture-of-experts 2B architecture) |
| Context window | 128K tokens |
| Capabilities | Native structured JSON output, native function calling |
| License | Apache 2.0 |
| Role | Produces structured JSON plan; specifies skills, tools, params, and cross-step references |
| Runs | Once per user request (planning phase only) |
| Interface | [Ollama Python SDK](https://github.com/ollama/ollama-python) `chat()` with `format=` for grammar-level structured output; [Instructor](https://python.useinstructor.com/) for typed response parsing |

**Why Gemma 4 E2B over Gemma 3 1B?** Gemma 3 1B required careful prompt engineering to produce valid structured JSON and struggled with complex multi-step plans. Gemma 4 E2B provides native structured JSON and function calling out of the box, 128K context (vs 32K), and 2.3B effective parameters -- resolving the critical open question about plan generation reliability. The Apache 2.0 license preserves full commercial flexibility.

### Embedding Model -- EmbeddingGemma 300M

The embedding model converts text (skill descriptions, file contents, query strings) into 768-dimensional vectors for similarity search.

| Property | Value |
|---|---|
| Name | EmbeddingGemma |
| Parameters | ~300M |
| Output | 768-dimensional vectors (truncatable to 128 / 256 / 512) |
| Role | Embeds skill descriptions (SkillIndex), plan steps (retrieval queries), and file contents (ContentIndex) |
| Runs | At startup for index build; once per plan step at runtime; on commit for stale content nodes |

**Why not FunctionGemma?** FunctionGemma (270M) was evaluated as a separate executor model. It is unnecessary: skills define exact method signatures, the Thinker outputs structured plans with skill names and parameters, and execution is deterministic Python via the primitive dispatcher. No LLM is needed at the execution stage.

**Evaluation via large model cross-check:** Because the Thinker is a small on-device model, its planning decisions can be validated by sending the same inputs to a large general-purpose model (e.g. Claude via API) and comparing outputs. This isn't part of the production pipeline — it's an evaluation tool for measuring how often the narrow agent's plan matches what a frontier model would produce, and where it diverges. Useful for identifying systematic weaknesses in prompt design or skill matching.

**Total memory footprint:** approximately 2.5 GB with quantization. Fits on any MacBook and most modern phones.

---

## The Skill System

A skill is the fundamental unit of capability in Scope Weaver. Think of it as a class, not a single function -- each skill packages multiple tools, and each tool declares exactly one primitive it invokes.

**Key concept:** A skill like `manage-files` contains multiple tools within it (list, create, move, rename, delete), and each tool maps to exactly one **primitive**: either a graph method or a retrieval query. See [Contracts & Primitives](#contracts--primitives) for the full primitive type system.

### Skill Folder Structure

```
manage-files/
+-- SKILL.md              # Identity, description, tool declarations with primitive specs
+-- scripts/              # Executable code for deterministic operations
+-- references/           # Detailed docs, loaded on demand
+-- kb/                   # Knowledge base -- domain-specific reference material
+-- errors/               # Known failure modes and remediation guidance
+-- vis.json              # Visualization metadata (how to render each action in the UI)
```

### Skill YAML Frontmatter -- Required Fields

Every SKILL.md begins with YAML frontmatter that the system parses at startup. The `description` field is what gets embedded into the SkillIndex for retrieval matching.

```yaml
---
name: manage-files
id: skill-manage-files-v1
version: "1.0"
description: >
  Core file management operations. Handles listing, creating, editing, moving,
  renaming, and deleting files and directories. Used whenever the user asks to
  view, organize, create, modify, or remove files.
tags: [files, crud, structural]
uses_retrieval: false        # true if any tool invokes a RetrievalPrimitive
kb_chunk_strategy: fixed     # fixed | sentence | paragraph | semantic (inherits global default if omitted)
kb_chunk_size: 512           # overrides CONFIG.chunk_size for this skill's KB only
---
```

### Tool Declaration Format

Each tool in a SKILL.md body declares one primitive. Primitive types are `graph` or `retrieval`.

```markdown
### list -- List files in a directory
primitive: graph
method: get_children
signature: graph.get_children(path: str) -> List[Node]
mutates: false
scope_enforced: true
required_params:
  - name: path
    type: str
optional_params: []
return_type: List[Node]
```

```markdown
### search -- Semantic search over file contents
primitive: retrieval
query_type: content_search
mutates: false
scope_enforced: true
required_params:
  - name: query
    type: str
  - name: top_k
    type: int
return_type: List[Node]
```

### Example: manage-files (structural skill)

This skill uses only `GraphPrimitive` -- all operations mutate or read the file-system graph directly.

```markdown
---
name: manage-files
id: skill-manage-files-v1
version: "1.0"
description: >
  Core file management operations. Handles listing, creating, editing, moving,
  renaming, and deleting files and directories.
tags: [files, crud, structural]
uses_retrieval: false
---

### list
primitive: graph | method: get_children | mutates: false | required: [path]

### create
primitive: graph | method: create_node | mutates: true | required: [path, name, type]

### move
primitive: graph | method: move_node | mutates: true | required: [source, target]

### rename
primitive: graph | method: rename_node | mutates: true | required: [path, new_name]

### delete
primitive: graph | method: delete_node | mutates: true | required: [path]

### get_metadata
primitive: graph | method: get_metadata | mutates: false | required: [path]
```

Skills using `RetrievalPrimitive` (e.g. `search-files`) follow the same format but with `primitive: retrieval | query_type: content_search` instead of `primitive: graph | method: ...`.

For full working examples, browse the skill stubs in the repo:
- [`skills/manage-files/SKILL.md`](skills/manage-files/SKILL.md) — structural skill (GraphPrimitive only)
- [`skills/search-files/SKILL.md`](skills/search-files/SKILL.md) — semantic skill (both primitive types)
- [`skills/skill_template/`](skills/skill_template/) — blank template for authoring new skills

### Skill Discovery Process

At startup the system:

1. Scans all skill folders; extracts `description` from each SKILL.md frontmatter.
2. Embeds each description via `Embedder.embed_document()` into a 768-dimensional vector.
3. Stores vectors in the `SkillIndex` (a `VectorStore` instance).

**Hot-reload:** If a skill's `description` field changes, only that skill is re-embedded. If the embedding model changes, the entire SkillIndex must be rebuilt (as must the ContentIndex -- they share the same `Embedder`).

---

## Sandbox & Virtual File System

The Sandbox is the safe, scoped workspace where all agent operations happen. It provides git-like commit semantics so that no changes reach the real file system until explicitly approved and committed.

### The Sandbox Object

The `Sandbox` is the central environment object. It owns:

- **`FileSystemGraph`** -- structural representation ([NetworkX](https://networkx.org/) DiGraph >=3.6, path strings as node IDs)
- **`ContentIndex`** -- semantic representation (VectorStore of file content vectors, [numpy](https://numpy.org/doc/stable/) cosine)
- **`SkillIndex`** -- semantic representation of skill descriptions (separate VectorStore)
- **`working_root`** -- the scope boundary; no primitive can operate outside this path
- **Staged buffer** -- approved step results awaiting commit
- **Sync state** -- tracks which graph nodes have been modified since their content was last indexed

### Operations

The `Sandbox` exposes a git-like lifecycle through four operations:

| Operation | Description |
|---|---|
| `stage(result)` | Accept an approved `ExecutionResult` into the staged buffer |
| `commit()` | Write all staged changes to the real file system; trigger content re-index for modified nodes |
| `rollback(snapshot_id)` | Restore graph to the given snapshot; clear downstream staged entries |
| `stale_nodes()` | Return nodes whose `modified_at` > `indexed_at` in the ContentIndex |

**Stale node policy:** When a retrieval query hits a stale node, the pipeline surfaces a warning: *"Content index out of date for N nodes -- results may be incomplete. Commit pending changes to reindex."* The pipeline does not block on this by default; the lever `block_on_stale` controls whether it halts or warns-and-continues.

### The FileSystemGraph

At startup, `os.walk(working_root)` snapshots the target directory into a `nx.DiGraph` where nodes are files/directories (with metadata: path, name, extension, size, `modified_at`, `indexed_at`) and edges represent containment. **Node IDs are path strings** for intuitive addressing and debugging. All agent operations mutate this graph only -- never the real file system. Changes reach disk only via `Sandbox.commit()`.

Every mutating primitive call takes a snapshot (`G.copy()`) before executing. The snapshot stack enables rollback to any prior state. JSON export uses `nx.node_link_data(graph)` for serialization to the visualization layer.

### The ContentIndex

A `VectorStore` instance holding embedded representations of file contents within the sandbox. This is distinct from the `SkillIndex` (which holds skill description vectors). Both use the same `Embedder` instance -- same model, same dimensionality, comparable similarity scores. Both use numpy for cosine similarity computation.

The `ContentIndex` enables skills like `search-files` and `remove-duplicates` to operate semantically -- finding files by topic or finding near-duplicate documents -- without exposing raw file bytes to any external service. All inference runs locally via the on-device embedding model.

**Sync:** `ContentIndex` is rebuilt at sandbox initialization and updated on `Sandbox.commit()` for any nodes whose content changed. The `on_mutate` observer on `FileSystemGraph` marks nodes as stale when they are modified, so the commit knows exactly which nodes need re-embedding.

### Scope Enforcement

Every primitive declares `scope_enforced: true/false`. The primitive dispatcher checks this flag before execution and clips all path arguments to `Sandbox.working_root`. A primitive cannot read or write nodes outside the declared scope regardless of what params the Thinker supplies.

---

## Agent Pipeline

The agent pipeline is the 13-stage flow that transforms a user's natural-language instruction into a sequence of approved, committed file-system operations. Each user request passes through every stage.

```
User Input --> Thinker --> Structured Plan (JSON)
                (Gemma 4 E2B        |
                 via ollama SDK     |
                 format= enforced)  |
                          +---------+  (FOR EACH STEP IN PLAN)
                          |
                   Embed Step Query
                   (EmbeddingGemma via ollama SDK)
                          |
                   Match Skill (SkillIndex numpy cosine)
                          |
                   Load Skill (SKILL.md)
                          |
                   Validate Params --> Clarify if missing
                          |
                   Resolve References ($step(n).field -> PipelineContext)
                          |
                   Dispatch Primitive --> GraphPrimitive -> FileSystemGraph
                          |              +-> RetrievalPrimitive -> RetrievalPipeline
                   Snapshot + Log
                   (structlog + OTel + Langfuse)
                          |
                   HITL Confirm (if mutates=true or mode="all")
                          |
                   Stage (if approved) / Rollback (if rejected)
                          |
                   <------+  (next step)
                          |
                   Sandbox.commit() --> Real FS + ContentIndex rebuild
```

### Stage Table

| Stage | Component | What Happens |
|---|---|---|
| 1. Input | Chat interface | User issues a natural-language command |
| 2. Think | Gemma 4 E2B + system prompt (ollama SDK `chat()` with `format=`) | Produces structured JSON plan: skills, tools, params, cross-step references |
| 3. Plan validation | Orchestrator ([Pydantic v2](https://docs.pydantic.dev/latest/) model validation) | Checks JSON is well-formed; skill names exist; reference syntax valid |
| 4. Embed | EmbeddingGemma (ollama SDK `embed()`) | Step description embedded as a query vector |
| 5. Match | SkillIndex numpy cosine | Query vector matched against pre-embedded skill descriptions; confidence check |
| 6. Load | File system | Matched skill's SKILL.md loaded; primitive declarations parsed |
| 7. Validate params | Contracts layer (Pydantic v2) | Required params present; types correct; missing params trigger clarification gate |
| 8. Resolve references | PipelineContext | `$step(n).field` expressions resolved against approved prior results |
| 9. Dispatch | Primitive dispatcher | Routes to `FileSystemGraph` method or `RetrievalPipeline` query based on primitive type |
| 10. Snapshot + Log | Sandbox / [structlog](https://www.structlog.org/) + OTel | Graph snapshot taken; action logged with full metadata; span emitted |
| 11. HITL confirm | HITL interface (WebSocket via [FastAPI](https://fastapi.tiangolo.com/)) | User approves, overrides, or rolls back (behaviour controlled by `hitl_mode` lever) |
| 12. Stage / rollback | Sandbox | Approved results staged; rolled-back results trigger graph restore and clear dependent context entries |
| 13. Commit | Sandbox | All staged results written to real FS; ContentIndex rebuilt for modified nodes |

### Human-in-the-Loop (HITL) Modes

| Mode | Behaviour | Use Case |
|---|---|---|
| `all` | User confirms every step | High-stakes operations, first-time use |
| `key` | Only steps where `primitive.mutates = true` pause for confirmation | Normal usage |
| `bypass` | Agent runs autonomously; user reviews final diff before commit | Trusted, repetitive tasks |

### Cross-Step References

The Thinker's plan may include references to prior step outputs using the syntax `$step(n).field`, where `n` is the step number (1-based) and `field` is a key in that step's `ExecutionResult.data`. The orchestrator resolves these at dispatch time via `PipelineContext`.

```json
{
  "step": 3,
  "skill": "manage-files",
  "tool": "move",
  "params": {
    "source": "$step(1).nodes",
    "target": "~/Downloads/Organized/"
  }
}
```

If the referenced step was rolled back, the dependent step receives `DEPENDENCY_UNAVAILABLE` status and is skipped cleanly -- no error, no halt.

---

## Contracts & Primitives

Every boundary in the pipeline is governed by typed contracts. This section defines the models that skill authors, pipeline developers, and frontend integrators work against. All contracts are Pydantic v2 `BaseModel` classes with full validation from day one. Structured output from the Thinker is enforced at the grammar level via Ollama's `format=` parameter with `model_json_schema()`.

### PipelineConfig -- Levers

All runtime behaviour is controlled through a single config object powered by [pydantic-settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/). Configuration is loaded from YAML files with environment variable overrides.

```python
class PipelineConfig(BaseSettings):
    model_config = {"env_prefix": "SCOPE_WEAVER_"}

    skill_match_threshold: float = Field(0.70)   # min cosine score to accept
    hitl_mode: str = Field("key")                 # all | key | bypass
    thinker_model: str = Field("gemma4:e2b")      # Ollama model tag
    embedding_dims: int = Field(768)              # must match embedding model
    # ... 17 fields total — see config/default.yaml for the full list
```

All 17 fields are documented in [`config/default.yaml`](config/default.yaml) and defined in [`core/config.py`](backend/src/scope_weaver/core/config.py).

### Primitive Type System

A **Primitive** is the atomic unit of what a tool invokes. The rule is strict: one primitive call per tool invocation. Composites are deferred; transactions may come later.

```python
from pydantic import BaseModel

class Primitive(BaseModel):
    primitive_type: str    # "graph" | "retrieval"
    mutates: bool          # drives HITL mode "key"; always False for retrieval
    scope_enforced: bool   # if True, dispatcher clips params to Sandbox.working_root
    return_type: str       # type annotation as string

class GraphPrimitive(Primitive):
    primitive_type: str = "graph"
    method: str = ""       # name of FileSystemGraph method to call

class RetrievalPrimitive(Primitive):
    primitive_type: str = "retrieval"
    mutates: bool = False  # retrieval never mutates
    query_type: str = ""   # "skill_match" | "content_search"
```

**LLMPrimitive** is reserved for future skills that invoke the Thinker model directly (e.g. summarization, classification). Same interface: `primitive_type: "llm"`, `mutates: False`. Adding it requires a new subclass, not a refactor of the dispatcher.

### Core Contracts

These models flow through the entire pipeline. `PlanStep` comes from the Thinker, `ToolDefinition` and `SkillDefinition` are parsed from SKILL.md files, `ExecutionResult` is produced by the dispatcher, and `PipelineContext` threads results across steps.

```python
from typing import Any
from pydantic import BaseModel

class PlanStep(BaseModel):
    step: int              # 1-based ordinal
    description: str       # natural-language description; used as retrieval query
    skill: str             # skill id (e.g. "manage-files")
    tool: str              # tool name within that skill
    params: dict           # may contain $step(n).field reference strings

class ToolDefinition(BaseModel):
    name: str
    primitive: Primitive   # the one primitive this tool invokes
    signature: str         # human-readable type signature
    required_params: list[str]
    optional_params: list[str]
    return_type: str

class SkillDefinition(BaseModel):
    id: str
    name: str
    version: str
    description: str       # the embeddable string; used by SkillIndex
    tags: list[str]
    uses_retrieval: bool   # if True, pipeline ensures ContentIndex is warm before executing
    tools: dict[str, ToolDefinition]
    kb_chunk_strategy: str = "fixed"
    kb_chunk_size: int = 512

class ValidationResult(BaseModel):
    valid: bool
    missing: list[str]     # required params absent from PlanStep.params
    extra: list[str]       # params present but not declared in tool schema

class ExecutionResult(BaseModel):
    ok: bool
    snapshot_id: str       # id of graph snapshot taken before this action
    data: Any              # method-specific return value; available to PipelineContext
    error: str | None = None

class PipelineContext(BaseModel):
    results: dict[int, ExecutionResult]  # { step_index: ExecutionResult }
    # resolve(ref: str) -> Any  parses "$step(n).field" and returns the value
    # A step whose referenced dependency was rolled back returns DEPENDENCY_UNAVAILABLE
```

### Shared Embedder Contract

Both `SkillIndex` and `ContentIndex` must be constructed using the **same `Embedder` instance**. This is structurally enforced: neither `VectorStore` constructs its own embedder -- both receive vectors from the shared `Embedder`. If the embedding model changes, `Sandbox.rebuild_all_indexes()` must be called to invalidate and rebuild both indexes before any retrieval is valid.

---

## Retrieval Architecture

The retrieval system uses a dual-index architecture with a shared embedder. One index holds skill descriptions (for matching user intent to the right skill), and the other holds file content vectors (for semantic search within the sandbox).

```
                    +----------------------------------+
                    |        EmbedderConfig            |
                    |   model, dims, normalization     |
                    +---------------+------------------+
                                    | shared singleton
                    +---------------v------------------+
                    |           Embedder               |
                    |  ollama SDK embed()              |
                    |  embed_document() / embed_query()|
                    +-------+------------------+-------+
                            |                  |
           +----------------v----+    +--------v-----------------+
           |     SkillIndex      |    |     ContentIndex         |
           |   (VectorStore)     |    |     (VectorStore)        |
           |  numpy cosine sim   |    |  numpy cosine sim        |
           | skill descriptions  |    |  file content vectors    |
           | built at startup    |    |  built at sandbox init   |
           | hot-reload per skill|    |  updated on commit       |
           +----------------+----+    +--------+-----------------+
                            |                  |
                    +-------v------------------v----+
                    |       RetrievalPipeline        |
                    |  query_skill(step)             |
                    |  query_content(query, scope)   |
                    |  build_index(skill_store)      |
                    |  hot_reload(skill_id)          |
                    +-------------------------------+
```

### Class Hierarchy

**`Chunker`** -- Splits raw text (SKILL.md, KB docs, reference PDFs) into embeddable chunks. Strategy and size are controlled by `PipelineConfig` with per-skill overrides.

**`Embedder`** -- Wraps the embedding model via the Ollama Python SDK `embed()` method. Shared singleton. `embed_document()` and `embed_query()` are separate methods with different caching semantics.

**`VectorStore`** -- In-memory dict of `{id: vector}` stored as numpy arrays. Query via numpy cosine similarity (200-500x faster than pure Python). Two instances exist: `SkillIndex` and `ContentIndex`. Neither instance is aware of the other. Migration to FAISS or ChromaDB is planned when the skill store grows.

**`Retriever`** -- Composes `Embedder` + `VectorStore`. `retrieve(query, top_k)` returns ranked matches. `is_confident(matches)` returns `True` if the top score exceeds `skill_match_threshold` AND the gap to the second match exceeds 0.05.

**`RetrievalPipeline`** -- Top-level interface used by the orchestrator. Exposes:

- `query_skill(step)` -> `(matches, is_confident)` -- used at stage 5 (Match)
- `query_content(query, scope, top_k)` -> `List[Node]` -- used when a `RetrievalPrimitive` is dispatched; scope clips results to `Sandbox.working_root` or skill-declared scope
- `build_index(skill_store)` -- called once at startup
- `hot_reload(skill_id, new_description)` -- re-embeds a single skill without full rebuild

---

## Worked Example

This walkthrough traces a real user request through the entire pipeline, including cross-step references and an edge case where retrieval confidence is ambiguous.

**User input:** *"Clean up my Downloads folder -- remove duplicates, then organize the remaining files into subfolders by type."*

### Thinker Produces a Plan

The Thinker (Gemma 4 E2B) receives the user input and produces a structured JSON plan via the Ollama Python SDK with `format=` grammar-level enforcement:

```json
[
  { "step": 1, "skill": "manage-files",      "tool": "list",         "params": { "path": "~/Downloads" } },
  { "step": 2, "skill": "remove-duplicates", "tool": "scan",         "params": { "path": "~/Downloads", "strategy": "hash" } },
  { "step": 3, "skill": "remove-duplicates", "tool": "remove",       "params": { "keep": "newest" } },
  { "step": 4, "skill": "manage-files",      "tool": "get_metadata", "params": { "path": "~/Downloads" } },
  { "step": 5, "skill": "manage-files",      "tool": "create",       "params": { "path": "~/Downloads", "types": "$step(4).extensions" } },
  { "step": 6, "skill": "manage-files",      "tool": "move",         "params": { "source": "$step(1).nodes", "by": "extension_category" } }
]
```

Steps 5 and 6 use cross-step references. Step 5 derives folder names from step 4's metadata output; step 6 uses step 1's node list as its working set. If either referenced step was rolled back, the dependent step would receive `DEPENDENCY_UNAVAILABLE` and be skipped.

### Step-by-Step Trace

**Step 1 -- List all files:** Matches `manage-files` (score: 0.94). `GraphPrimitive`, `mutates: false`. Auto-approved under `hitl_mode: "key"`. Returns 47 nodes into `PipelineContext`.

**Step 2 -- Scan for duplicates:** Matches `remove-duplicates` (score: 0.91). Finds 3 duplicate groups. `mutates: false` -- auto-approved. Results staged.

**Step 3 -- Remove duplicates:** `mutates: true` -- pauses for HITL. User inspects: two files flagged as duplicates (`report_v1.pdf`, `report_final.pdf`) share a content hash but represent different versions. User overrides, excluding this group. Agent removes only 2 groups (3 files). Snapshot taken; staged.

**Edge Case -- Retrieval near-miss (Step 5):** EmbeddingGemma matches `manage-files` (score: 0.72) but also returns `organize-by-type` (score: 0.69). Gap is 0.03 -- below the 0.05 confidence gap threshold. Pipeline surfaces both candidates to user: *"I'm not certain which skill to use. Please confirm: manage-files or organize-by-type?"* User selects `manage-files`. Near-miss logged for retrieval quality analysis.

**Steps 4-6:** Metadata retrieved, subfolders created, files moved. Each mutating step pauses for confirmation. All approved. Sandbox staged buffer now holds 6 `ExecutionResult` entries.

**Commit:** `Sandbox.commit()` writes all changes to disk. ContentIndex re-embeds modified nodes. Final report: *"Removed 3 duplicate files (saved 12 MB). Organized 44 files into 4 subfolders."*

---

## Error Taxonomy

The system categorizes errors into four types. Three of the four are resolvable by the user directly within the application, without engineering intervention.

| Type | Cause | Remediation |
|---|---|---|
| **Retrieval -- Near-Miss Ambiguity** | Two or more skills score within the confidence gap threshold. The pipeline cannot confidently select one. | **User-resolvable in-app:** pipeline surfaces top candidates; user selects the correct skill. Selection is logged and may inform future retrieval tuning. |
| **Agentic Error** | Prompts or pipeline logic are flawed. Agent misinterprets the task or sequences steps incorrectly. | **User-fixable in-app:** edit markdown prompts, adjust pipeline order -- all via natural language or the in-app editor. Changes reflect immediately. |
| **Knowledge-Base Error** | Retrieval found the right skill, but its stored content (SKILL.md, KB, references) is wrong or outdated. | **User-fixable in-app:** edit the skill's knowledge base or SKILL.md through the built-in editor. No codebase access needed. Description changes trigger skill re-embedding. |
| **Retrieval -- Systemic Failure** | Wrong skill selected entirely due to a broken index or embedding failure. The system cannot self-diagnose. | **Cannot self-fix.** User flags; escalates to engineering team for investigation. Distinct from near-miss ambiguity, which is a normal operating condition. |

---

## Observability

Everything the pipeline does is logged, traced, and searchable. This is both an operational requirement and a research tool for understanding retrieval quality and agent behaviour.

**Note:** [OpenTelemetry](https://opentelemetry.io/docs/languages/python/) and [Langfuse](https://langfuse.com/docs) are optional dependencies. Install them with `uv sync --group observability`. structlog is a core dependency that is always available.

### What Gets Logged (per action)

| Data | Purpose |
|---|---|
| Timestamp + unique action ID | Ordering and deduplication |
| User prompt and agent response | Full conversation audit trail |
| Model used + tokens consumed | Cost and performance tracking |
| Skill selected + similarity score + runner-up skills | Retrieval quality analysis |
| Primitive type + method / query type + params + result | Execution traceability |
| Cross-step references resolved | Plan coherence tracking |
| Latency (per step and end-to-end) | Performance benchmarking |
| Rollback events + snapshot ID | Accountability: who rolled back what, when |
| HITL overrides and user corrections | Tracking where the agent got it wrong |
| Stale node warnings | ContentIndex sync health |

### Observability Stack

| Layer | Tool | Role |
|---|---|---|
| Structured logging | structlog (processor pipeline) | JSON-formatted structured logs, contextvars-based context propagation |
| Distributed tracing | OpenTelemetry | Full-stack trace spans across API, pipeline, and sandbox layers; integration with structlog |
| LLM observability | Langfuse | Token-level tracing of Thinker calls, prompt/response pairs, retrieval scores, latency breakdowns |

---

## Frontend & Visualization

The frontend combines a macOS-native shell application with a web-based graph visualization, connected to the Python backend via WebSocket.

### Graph Visualization (Cytoscape.js)

Every agent action is visualized -- the file-system graph, pipeline state, skill selection reasoning, and action history with rollback points. Skills include `vis.json` that declares how each action renders.

Cytoscape.js was chosen for its native compound node support: directories contain files as nested graph nodes, mapping naturally to file-system hierarchy without custom layout hacks. The visualization receives live updates from the backend via WebSocket (FastAPI), reflecting agent actions (add/remove/move nodes, highlight active step) in real time.

### macOS Application (SwiftUI)

The frontend is inspired by macOS Spotlight: a global shortcut opens a minimal input bubble, available narrow agents are listed below (or auto-selected), the user types a command, and the agent begins executing with real-time visual feedback.

For the competition prototype, the focus is exclusively on macOS: a SwiftUI native shell with an embedded WKWebView for graph visualization. Linux and Windows support comes later, likely via Tauri.

### UX Principles

| Principle | Implementation |
|---|---|
| Chat-first interaction | Natural language is the primary input mode |
| Agent auto-selection | Narrow agent routed automatically based on input content |
| Clarifying questions | Agent asks when instructions are ambiguous or params are missing |
| Voicing uncertainty | Agent communicates confidence levels and near-misses explicitly |
| Full transparency | Every decision visible in graph and logs |
| In-app editing | Users edit skill files, prompts, and KB directly in the interface |

---

## Technology Decisions

Each choice was made deliberately. The "Enterprise Upgrade" column shows the planned swap for production deployments — the architecture is designed so these swaps don't require changes above the relevant abstraction boundary.

| Component | Choice | Why | Enterprise Upgrade |
|---|---|---|---|
| Thinker LLM | Gemma 4 E2B (2.3B, 128K) | Native structured JSON + function calling; eliminates invalid plan generation risk. Apache 2.0. | Larger MoE variant |
| Embeddings | EmbeddingGemma 300M | 768-dim vectors, Matryoshka-truncatable. Fast on-device. | Same |
| Model serving | Ollama | Simple local server, Metal/CUDA support, pull-based model management. | llama.cpp / vLLM |
| LLM interface | Ollama Python SDK + Instructor | Type-safe calls + Pydantic validation + automatic retries on malformed output. | Same |
| Structured output | Ollama `format=` + `model_json_schema()` | Grammar-level enforcement — model physically cannot produce invalid JSON. | Same |
| Graph | NetworkX DiGraph >=3.6 | Mature, path-string IDs for debugging, `node_link_data()` for JSON export, `G.copy()` snapshots. | Same |
| Vector search | numpy cosine | 200-500x faster than pure Python. Zero complexity. | FAISS / ChromaDB |
| Contracts | Pydantic v2 BaseModel | Validation + serialization + JSON schema built in. Rust core = fast. | Same |
| Config | pydantic-settings | YAML + env vars + `SCOPE_WEAVER_` prefix, all validated against one model. | Same |
| API | FastAPI | Async, WebSocket, OpenAPI docs, native Pydantic integration. | Same |
| Visualization | Cytoscape.js | Native compound nodes (dirs contain files), embeds in WKWebView. | Same |
| macOS frontend | SwiftUI + WKWebView | Native macOS experience for competition demo. | Tauri (cross-platform) |
| Logging | structlog | Processor pipeline, contextvars, JSON/console modes. | Same |
| Tracing | OpenTelemetry (optional) | Vendor-neutral distributed tracing. `uv sync --group observability` | OTel + Jaeger |
| LLM tracing | Langfuse (optional) | Token-level prompt/response tracing. `uv sync --group observability` | Langfuse (self-hosted) |
| Package manager | [uv](https://docs.astral.sh/uv/) with workspaces | 10-100x faster than pip. Lockfile for reproducibility. | Same |
| Build backend | uv_build | Native to uv, no setuptools complexity. | Same |
| Containers | python:3.12-slim | Standard base. DHI deferred (needs registry auth). | Docker Hardened Images |
| Orchestration | Docker Compose + Watch | Dev hot-reload via `docker compose watch`. `--profile urai` for full stack with Ollama. | Kubernetes |
| CI | GitHub Actions | Ruff lint + pytest on Python 3.12. | Same + mypy strict, ghcr.io publish |
| Docs | GitHub Markdown | README, ARCHITECTURE.md, guides — renders natively on GitHub. | Sphinx + Furo + MyST (post-demo) |
| Linting | ruff | Replaces flake8 + isort + pyupgrade. 10-100x faster. | Same |
| Types | mypy (local only) | Catches contract violations at dev time. `task typecheck` locally. | Strict mode in CI (post-demo) |
| Commits | commitizen | Conventional Commits for machine-readable history. | Same |
| Security | Manual review | Dependency auditing deferred for competition sprint. | pip-audit + Renovate + CI audit |

---

## Project Structure

The monorepo follows a src layout with uv workspaces for package management. The directory tree below shows every major component and where it lives.

```text
📦 scope-weaver/
├── ⚙️ pyproject.toml              # Root: uv workspace + tool config (ruff, mypy, pytest)
├── 🐳 compose.yaml                # Docker Compose: dev target + Compose Watch + urai profile
├── ⚙️ Taskfile.yaml               # Task runner (install, lint, test, up, up:urai, check)
├── ⚙️ .pre-commit-config.yaml     # ruff + commitizen hooks
├── 🐍 backend/
│   ├── ⚙️ pyproject.toml          # Package: scope-weaver (uv_build backend, 12 core deps)
│   └── 📂 src/scope_weaver/
│       ├── __init__.py            # __version__ = "0.1.0"
│       ├── __main__.py            # Entry point: uvicorn runner
│       ├── 📂 core/               # Contracts & Config
│       │   ├── config.py          # PipelineConfig (pydantic-settings, YAML + env vars)
│       │   └── contracts.py       # PlanStep, Primitive, ExecutionResult, etc. (Pydantic v2)
│       ├── 🗄️ environment/        # Sandbox & Virtual File System
│       │   ├── sandbox.py         # Sandbox class, git-like lifecycle
│       │   └── fs_graph.py        # FileSystemGraph (NetworkX 3.6+, path string IDs)
│       ├── 🔍 retrieval/          # Retrieval Architecture
│       │   ├── embedder.py        # ollama SDK embed() wrapper
│       │   ├── vector_store.py    # In-memory numpy cosine similarity
│       │   ├── chunker.py         # Text splitting (fixed/sentence/paragraph)
│       │   ├── retriever.py       # Composes Embedder + VectorStore, confidence check
│       │   └── pipeline.py        # RetrievalPipeline, SkillIndex, ContentIndex
│       ├── 🦉 agent/              # Orchestration Loop & LLM
│       │   ├── thinker.py         # ollama SDK chat() + Instructor, Gemma 4 E2B
│       │   ├── orchestrator.py    # Plan validation & execution loop
│       │   ├── context.py         # PipelineContext helpers ($step(n).field resolution)
│       │   └── dispatcher.py      # Routes primitives to Sandbox or Retrieval
│       ├── 🌐 api/                # FastAPI app + routes
│       │   ├── app.py             # FastAPI application factory + lifespan
│       │   ├── routes.py          # POST /run, GET /graph, GET /status, WS /stream
│       │   └── schemas.py         # RunRequest, RunResponse, StreamEvent (API models)
│       └── 📊 observability/      # Logging & Observability
│           ├── logging.py         # structlog processor pipeline (core dep)
│           ├── tracing.py         # OpenTelemetry setup (optional dep)
│           └── langfuse.py        # Langfuse LLM tracing (optional dep)
├── 📜 skills/                     # Skill definitions (outside Python package)
│   ├── skill_template/            # Template for authoring new skills
│   ├── manage-files/              # Core CRUD operations
│   ├── search-files/              # Semantic & structural search
│   ├── organize-by-type/          # Sort by extension
│   ├── remove-duplicates/         # Hash/content dedup
│   ├── batch-rename/              # Pattern-based rename
│   ├── archive-files/             # zip/tar compression
│   ├── analyze-directory/         # Size/count/type stats
│   ├── parse-content/             # Text/metadata extraction
│   ├── skill-search/              # Meta: find skills by description
│   └── skill-create/              # Meta: scaffold new skill
├── 🎨 frontend/
│   ├── macos/                     # Native macOS SwiftUI app
│   │   ├── Package.swift          # Swift Package Manager manifest
│   │   └── Sources/
│   │       ├── ScopeWeaverApp.swift   # Menu bar / global shortcut
│   │       ├── AgentChatView.swift    # Spotlight-style input + HITL
│   │       └── VizWebView.swift       # WKWebView wrapper + JS bridge
│   └── web-viz/                   # Graph Visualization
│       ├── package.json           # cytoscape + fcose + expand-collapse
│       ├── index.html
│       ├── style.css
│       └── src/
│           ├── app.js             # Cytoscape init + event handling
│           ├── bridge.js          # WKWebView <-> Swift bridge
│           └── graph_renderer.js  # Render node_link_data, live updates
├── 🧪 tests/                      # 108 test stubs across 13 files
│   ├── conftest.py                # Root fixtures: config, tmp_dir, mock_ollama
│   ├── unit/                      # 12 test files (contracts, config, graph, sandbox, ...)
│   ├── integration/               # test_pipeline_e2e.py
│   └── benchmarks/                # Phase 0 Ollama/MLX benchmarks (preserved)
├── 🐳 docker/                     # Single multi-stage Dockerfile
│   ├── Dockerfile                 # base -> development -> builder -> production
│   └── scripts/
│       └── setup.sh               # Ollama init sidecar: pull models, smoke test
├── 📘 docs/                       # Architecture diagrams + guides (GitHub Markdown)
│   ├── architecture/              # Mermaid diagrams: system, pipeline, data model, deps, workflow
│   └── guides/                    # skill-authoring, docker-setup
└── ⚙️ config/
    └── default.yaml               # PipelineConfig defaults (17 fields)
```

---

## Platform Vision: Configure, Evaluate, Deploy

The platform is organized around three pillars:

- **Configuration** — Building and customizing agents. The long-term goal is an in-app markdown editor where users create and modify SKILL.md files, system prompts, and knowledge bases directly within the platform — no codebase access needed. This is the "agent building" experience. Changes to skill descriptions trigger automatic re-embedding into the SkillIndex.
- **Evaluation** — Measuring agent quality. Cross-checking narrow agent decisions against a large general-purpose model (see [Model Stack](#model-stack)), analyzing retrieval accuracy, tracking HITL override frequency, and identifying where the agent systematically fails.
- **Deployment** — Getting agents running on-device or in production. Docker, Ollama, CI/CD, and the infrastructure layer.

For the competition prototype, the focus is on getting Configuration and Deployment working end-to-end. The Evaluation pillar and the in-app editor are future work.

## Future & Enterprise Path

These items are out of scope for the competition prototype but are actively informing architecture decisions today.

**On-device deployment:** Both models fit in ~2.5 GB. As NPUs mature, the target is fully local inference on laptops and phones.

**Enterprise engagement:** The same architecture adapted for various knowledge/code base management. E.g., for code bases, Skills would cover code analysis, standards compliance checking, component specification parsing, and code adaptation. The ContentIndex handles automotive standards documents (ISO 26262, AUTOSAR) as semantic search targets within skills -- useful when the problem is a large-context problem.

**Multi-agent expansion:** File management is agent #1. The Spotlight-style UI hosts many narrow agents. The skill marketplace makes capabilities portable across agents. The primitive type system extends cleanly -- `LLMPrimitive` for summarization agents, further types for agents that interact with external APIs.

**Security and data privacy:** Enterprise deployments require access controls, encryption at rest, and comprehensive audit trails. The logging infrastructure and scope enforcement in the Sandbox are designed with this path in mind.

**VectorStore migration:** The `VectorStore` abstraction means the numpy cosine prototype can be swapped for FAISS, ChromaDB, or Elasticsearch hybrid without changing any code above the `RetrievalPipeline` interface.
