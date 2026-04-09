---
title: Home
layout: home
---

# SCOPEWEAVER

The purpose of this website is to document the design and implementation of
scopeweaver project for developer. This would allow for:

1. Situational awarness on the state of the project
2. Efficient future planning
3. Unambigous implementation across all independent teams
4. Single source of truth

## What is Scopeweaver

> A narrow-agent platform for reliable, on-device file and context management.

Scope Weaver is a multi-agent system designed to handle large contexts by decomposing 
domain knowledge into specialized, modular skills with embedded retrieval. Originally 
conceptualized to help manage complex embedded software development (like C++ codebases 
and automotive standards), the platform utilizes small, focused models to orchestrate 
tasks safely and deterministically.

Our first implementation is a **file management agent** that serves as the foundation 
for future, more complex capabilities.

<!-- 

Some of these features are speculative, we might not have the time to implement
all of them.

## Key Features

- **100% Local Execution:** Powered by on-device models (Gemma 3 1B-IT for planning, 
    EmbeddingGemma for retrieval) with a total memory footprint of approximately 2.2 GB.
- **Skill-Based Architecture:** Skills act as deterministic classes containing specific 
    tools, keeping LLM hallucinations out of the execution layer.
- **Virtual File-System Sandbox:** All agent actions happen within a safe, isolated 
    FileSystemGraph (NetworkX).
- **Git-like Safety Lifecycle:** Changes are staged and can be rolled back to specific 
    snapshot IDs; real file system mutations only occur upon explicit commits.
- **Human-in-the-Loop (HITL**): Adjustable oversight modes ranging from confirming every 
    step (all), to only mutating actions (key), to full autonomy (bypass).
- **Live Visualization:** Real-time visual feedback of the file-system graph, pipeline 
    state, and agent reasoning. -->

## Planned features

1. Skill based architecture: skills act as deterministic classes containing specific
    tools
2. Virtual file system sandbox: the agent shall act withing an isolated environment,
    as a guarrail
3. Git like safety lifecycle: changes made by the agent can be rolled back to specific
    snapshots
4. Text files support: the context window of the LLM consumes the text files in the system
    for efficient and informed operations
5. Human in the loop: ultimate will and authority is maintained by the user


## System Architecture

The platform is strictly separated into three core layers:

#### 1. Frontend (macOS Native)

A SwiftUI application featuring a minimal, Spotlight-style chat interface. It embeds a 
WebView to render the interactive file-system graph and action timeline in real-time.

#### 2. Agent Pipeline (The Brain)

The orchestration loop where a user prompt is processed. The pipeline handles task 
planning (via the Thinker model), skill retrieval, parameter validation, and deterministic 
execution via the primitive dispatcher.

#### 3. Sandbox & Storage (The Environment)

The working environment containing the FileSystemGraph (structural state), ContentIndex 
(semantic state of files), and SkillIndex (semantic state of available tools). It handles 
all commit/rollback semantics and system logging.

## Repository Structure

Scope Weaver follows a clean monorepo architecture separating the interface, backend logic, 
and the modular skill definitions.

```
📁 scope-weaver/
├── 📄 README.md
├── 📁 docs/                 # Project documentation & architectural specs
│
├── 📁 skills/               # The Skill System (Agent capabilities)
│   ├── 📁 skill_template/   # Skeleton template for new skills
│   ├── 📁 manage-files/     # Core CRUD operations (list, create, move, etc.)
│   └── 📁 search-files/     # Semantic & structural search
│
├── 📁 backend/              # Python Backend (Agent Pipeline & Sandbox)
│   ├── 📄 pyproject.toml    # Dependencies
│   ├── 📄 main.py           # API/WebSocket Server
│   ├── 📁 scope_weaver/     # Core pipeline, retrieval, and environment logic
│   └── 📁 tests/            # Pytest suite
│
└── 📁 frontend/             # UI / UX & Visualization
    ├── 📁 ScopeWeaverMac/   # Native macOS SwiftUI App
    └── 📁 web-viz/          # Graph Visualization (vis.js/D3) loaded via WebView
```

## Developing Skills

Skills in Scope Weaver are not individual tools, but rather domain-specific "classes" 
that declare exact method signatures and invoke deterministic primitives.

To add a new capability to the agent, copy the skills/skill_template folder. You must 
define a SKILL.md file containing the YAML frontmatter (description, tags) and Markdown-based 
tool declarations. The pipeline automatically embeds the description into the SkillIndex on 
startup, allowing the agent to dynamically route user requests to your new skill based on 
semantic similarity.

