# PROJECT ANALYSIS

## Overview

This document summarizes two parts of the GenericAgent project:

1. How the project defines and uses "skill"
2. What built-in tools the agent framework exposes

The analysis is based on the current repository source code, especially:

- `README.md`
- `GETTING_STARTED.md`
- `agentmain.py`
- `agent_loop.py`
- `ga.py`
- `memory/memory_management_sop.md`
- `memory/skill_search/`
- `assets/tools_schema.json`

---

## 1. What "skill" means in this project

### 1.1 Not a classic plugin system

In this repository, a "skill" is usually not a separately registered plugin with a loader like `load_skill("name")`.

Instead, the main implementation treats skill as part of the memory system:

- L0: meta rules
- L2: global facts
- L3: task SOPs and reusable scripts

The design described in `README.md` is:

> solve a new task -> crystallize execution path into skill -> write to memory layer -> reuse later

So the practical meaning of skill in this project is:

- a reusable SOP file under `memory/*.md`
- a reusable helper script under `memory/*.py`
- discoverability pointers in the memory index

---

## 2. Memory structure behind skills

The layered memory model is defined in `memory/memory_management_sop.md`.

### 2.1 L0 / L1 / L2 / L3

- **L0**: `memory/memory_management_sop.md`
  - rules for what can be remembered and how to update memory
- **L1**: `memory/global_mem_insight.txt`
  - minimal keyword index for discoverability
- **L2**: `memory/global_mem.txt`
  - global stable facts about the environment
- **L3**: `memory/*.md` and `memory/*.py`
  - task-specific SOPs and reusable scripts

The most important part for skills is **L3**.

According to `memory_management_sop.md`:

- `*_sop.md` is used for concise SOPs
- `*.py` is used for reusable helper scripts

This means the project's "skill tree" is mostly a memory-backed collection of SOPs and scripts.

---

## 3. How skills are used at runtime

### 3.1 Startup injects memory pointers into the system prompt

In `agentmain.py`, `get_system_prompt()` appends the result of `get_global_memory()` from `ga.py`.

`ga.py:get_global_memory()` reads:

- `assets/insight_fixed_structure.txt`
- `memory/global_mem_insight.txt`

This injects into the model prompt:

- where memory files are
- where SOPs are
- where global facts are
- which meta SOP controls memory updates

So the agent is primed to **discover skills through files**, not through a plugin registry.

### 3.2 The agent uses normal tools to read and follow a skill

At runtime, there is no dedicated public tool like `run_skill`.

Instead, the agent:

1. reads a memory/SOP file with `file_read`
2. extracts key instructions
3. stores short-term task notes with `update_working_checkpoint`
4. executes work with tools such as `code_run`, `web_scan`, `web_execute_js`, `file_patch`, and `file_write`

### 3.3 Memory/SOP reads are treated specially

In `ga.py`, `do_file_read()` adds a reminder when the path contains `memory` or `sop`:

- if the agent decides to follow the SOP, it should extract key points
- then update working memory

This shows that SOPs are first-class operational guidance for the agent.

### 3.4 Working memory keeps the chosen skill active

`update_working_checkpoint` stores:

- `key_info`
- `related_sop`

Then `_get_anchor_prompt()` in `ga.py` re-injects these into later turns. This keeps the selected skill active during long tasks.

### 3.5 Finished tasks can become new skills

After a meaningful task, `start_long_term_update()` guides the agent to:

- write stable facts into L2
- write reusable task experience into L3 SOPs

This is the main self-evolution path of the framework.

---

## 4. How to use an existing skill

### 4.1 User-level usage

The intended user experience is usually natural language, for example:

- "execute web setup sop"
- "read your SOP and perform this task"
- "configure ADB and save it to memory"

The framework is designed so the agent can inspect its own memory and source code, then choose the right SOP or script.

### 4.2 Direct file-based usage

Because skills are files, they can also be used explicitly by referring to them, for example:

- `memory/web_setup_sop.md`
- `memory/tmwebdriver_sop.md`
- `memory/plan_sop.md`

The Qt frontend also reflects this design. In `frontends/qtapp.py`, the SOP page lists `memory/*.md` directly.

---

## 5. How to add or "install" a new skill

### 5.1 Main system: no separate installer

For the core GenericAgent system, adding a new skill does **not** mean installing a plugin through a registration API.

The standard approach is:

1. add a new SOP or helper script under `memory/`
2. update the memory index if discoverability matters
3. optionally add related environment facts to L2

### 5.2 Recommended forms

#### Option A: SOP file

Create a file like:

- `memory/my_task_sop.md`

Use this for:

- hidden prerequisites
- important pitfalls
- concise repeatable workflows

#### Option B: helper script

Create a file like:

- `memory/my_task_helper.py`

Use this for:

- reusable non-trivial logic
- task automation the agent should not have to reconstruct every time

### 5.3 Make it discoverable

If the new skill should be easier for the agent to recall, update:

- `memory/global_mem_insight.txt`

This file is injected into the system prompt at startup, so it is the main discoverability index.

### 5.4 When to update L2

If the skill depends on stable environment facts, add those to:

- `memory/global_mem.txt`

Examples:

- fixed service endpoints
- non-standard paths
- environment-specific configuration facts

### 5.5 Practical meaning of "install a skill"

In this codebase, "installing" a skill usually means:

- place the SOP or script into `memory/`
- index it in `global_mem_insight.txt` if needed
- let the agent read and reuse it

There is no core code path showing a dedicated installer or skill registry for L3 skills.

---

## 6. The separate `skill_search` package

There is also a separate component under:

- `memory/skill_search/`

This is **not** the main skill runtime of GenericAgent. It is an optional external skill search client.

### 6.1 What it does

`memory/skill_search/skill_search/engine.py` exposes:

- `search(query, env=None, category=None, top_k=10)`
- `get_stats(env=None)`
- `detect_environment()`

It searches an external skill index over HTTP.

### 6.2 Environment variables

The package supports:

- `SKILL_SEARCH_API`
- `SKILL_SEARCH_KEY`

The default API URL in source is:

- `http://www.fudankw.cn:58787`

### 6.3 CLI usage

The CLI entrypoint is:

- `python3 -m skill_search`

Example commands:

```bash
cd memory/skill_search
python3 -m skill_search --env
python3 -m skill_search "python testing"
python3 -m skill_search "docker deployment" --category devops --top 5
```

### 6.4 Python usage

Because this package lives inside the repository and is not installed globally by default, importing it from the repo root requires adjusting `sys.path`.

Example:

```python
import sys
sys.path.append("/path/to/repo/memory/skill_search")
from skill_search import search

results = search("python send email")
```

### 6.5 Important distinction

This package is a **search client for external skills**, not the same thing as the core L3 memory skill mechanism.

The main agent flow does not currently show direct integration that automatically imports and uses `skill_search`.

---

## 7. Built-in tools of the agent framework

The public built-in tools are defined in:

- `assets/tools_schema.json`
- `assets/tools_schema_cn.json`

Their implementations are in `ga.py`.

The framework exposes **9 public built-in tools**.

### 7.1 `code_run`

**Purpose:** execute code or shell commands

Implementation details from `ga.py`:

- `type="python"` -> writes a temporary Python file and executes it
- `type="bash"` on non-Windows
- `type="powershell"` on Windows

This is the framework's built-in mechanism for:

- terminal commands
- scripts
- package installation
- test execution
- git operations

Examples of tasks supported through this tool:

- `git status`
- `pip install requests`
- `npm test`
- `python3 some_script.py`

So the framework does **not** expose a separate terminal tool. Terminal-like behavior is provided through `code_run`.

### 7.2 `file_read`

**Purpose:** read file content

Supports:

- path selection
- line offsets
- line counts
- keyword-based contextual search
- line number display

### 7.3 `file_patch`

**Purpose:** patch a file by replacing a unique text block

This is the precise edit tool for local changes.

Important behavior:

- `old_content` must match exactly
- the match must be unique
- whitespace and indentation matter

### 7.4 `file_write`

**Purpose:** write whole-file content

Modes:

- `overwrite`
- `append`
- `prepend`

This is better for:

- new files
- large rewrites
- output artifacts

### 7.5 `web_scan`

**Purpose:** inspect the current browser page and tab list

Returns simplified page information and supports:

- tab listing
- tab switching
- plain text only mode

This is part of the browser automation stack.

### 7.6 `web_execute_js`

**Purpose:** execute JavaScript inside the controlled browser

This is the main browser control tool for:

- DOM interaction
- data extraction
- navigation logic
- browser-side automation

### 7.7 `update_working_checkpoint`

**Purpose:** short-term task memory

Stores important current-task data such as:

- user requirements
- key constraints
- file paths
- progress
- related SOP names

This is injected into later turns to reduce context loss during long tasks.

### 7.8 `ask_user`

**Purpose:** interrupt and ask the user a question

Used when:

- the task requires a human decision
- information is missing
- automation is blocked

### 7.9 `start_long_term_update`

**Purpose:** begin long-term memory distillation

Used when:

- a task produced reusable facts
- new preferences or environment knowledge were verified
- a repeatable workflow should be written into memory

---

## 8. How tool execution is orchestrated

The execution loop is implemented in `agent_loop.py`.

High-level flow:

1. system prompt is created
2. user input is sent to the LLM
3. the LLM returns tool calls
4. `BaseHandler.dispatch()` routes each tool call to `do_<tool_name>()` in `ga.py`
5. the tool result is fed back into the next model turn
6. the loop continues until the task is done

If the model returns no explicit tool call, the engine uses an internal fallback path:

- `no_tool`

This is not one of the normal public tools; it is an internal engine behavior.

---

## 9. Most important practical conclusions

### 9.1 Skills

- Skills are primarily memory-backed SOPs and scripts
- They live in `memory/`
- They are discovered through memory files, not a classic registry
- New skills are effectively "installed" by adding files and indexing them

### 9.2 Tools

- The framework exposes 9 public built-in tools
- `code_run` is the built-in way to execute terminal commands
- File editing is split between `file_patch` and `file_write`
- Browser control is split between `web_scan` and `web_execute_js`
- Memory persistence is handled by `update_working_checkpoint` and `start_long_term_update`

### 9.3 `skill_search`

- `memory/skill_search/` is an optional external skill search client
- It is separate from the main memory-based skill mechanism
- It is usable through CLI or manual Python import, but is not shown as automatically wired into the core runtime

---

## 10. Reference file list

- `README.md`
- `GETTING_STARTED.md`
- `agentmain.py`
- `agent_loop.py`
- `ga.py`
- `assets/tools_schema.json`
- `assets/tools_schema_cn.json`
- `assets/insight_fixed_structure.txt`
- `memory/memory_management_sop.md`
- `memory/web_setup_sop.md`
- `memory/tmwebdriver_sop.md`
- `frontends/qtapp.py`
- `memory/skill_search/SKILL.md`
- `memory/skill_search/skill_search/engine.py`
- `memory/skill_search/skill_search/__main__.py`
