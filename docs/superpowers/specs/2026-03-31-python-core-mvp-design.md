# Python Core Engine MVP Design

**Date:** 2026-03-31

**Repository:** `https://github.com/CHZarles/cc2cc`

**Goal:** Translate the core execution architecture of Claude Code into a Python implementation that preserves the original system's layering and control flow while simplifying product-specific complexity.

## Summary

The Python port will translate the source snapshot's core engine, not the entire product surface.

The MVP will focus on the headless execution path:

- command-line entrypoint
- command and tool registries
- session-scoped query engine
- multi-turn model/tool query loop
- minimal runtime state
- transcript persistence
- core local tools

The MVP will not include the original terminal UI, plugin system, MCP, bridge, remote sessions, or agent swarms.

## Source Translation Strategy

The project will use:

- **Approach A as the main strategy:** preserve the original system's architectural boundaries
- **Approach B as a local implementation tactic:** simplify internals where Python can express the same behavior more cleanly

This means the port should remain easy to compare against the source snapshot while avoiding a line-by-line TypeScript rewrite.

The intended mapping is:

- [`src/main.tsx`](/tmp/cc/claude-code/src/main.tsx) -> Python CLI bootstrap
- [`src/commands.ts`](/tmp/cc/claude-code/src/commands.ts) -> Python command registry
- [`src/tools.ts`](/tmp/cc/claude-code/src/tools.ts) -> Python tool registry
- [`src/QueryEngine.ts`](/tmp/cc/claude-code/src/QueryEngine.ts) -> Python session orchestrator
- [`src/query.ts`](/tmp/cc/claude-code/src/query.ts) -> Python multi-turn query loop

## Scope

### In Scope

- Create a dedicated Python project under `python-core-mvp/`
- Use the GitHub repository `CHZarles/cc2cc` as the canonical implementation repo
- Provide a headless CLI entrypoint
- Implement a registry-driven command and tool system
- Implement a source-faithful `QueryEngine` / `query_loop` split
- Implement a provider abstraction with at least one real provider
- Support structured tool calls and tool result reinjection
- Support transcript persistence to local disk
- Support a minimal permission model
- Implement the following tools:
  - `bash`
  - `file_read`
  - `file_write`
  - `file_edit`
- Add tests for registries, query loop behavior, providers, tools, and end-to-end flow

### Out of Scope

- Ink or REPL UI
- MCP
- plugin marketplace
- bridge or IDE integration
- remote session infrastructure
- SSH mode
- multi-agent orchestration
- skills system
- GrowthBook and runtime feature flag parity
- full product-level command parity

## Project Layout

The initial repository layout will be:

```text
python-core-mvp/
├── pyproject.toml
├── README.md
├── src/cccore/
│   ├── cli.py
│   ├── config.py
│   ├── models.py
│   ├── messages.py
│   ├── commands.py
│   ├── tools.py
│   ├── query_engine.py
│   ├── query_loop.py
│   ├── providers/
│   │   ├── base.py
│   │   └── anthropic.py
│   ├── runtime/
│   │   ├── app_state.py
│   │   ├── permissioning.py
│   │   └── transcript.py
│   ├── tool_impls/
│   │   ├── bash.py
│   │   ├── file_read.py
│   │   ├── file_write.py
│   │   └── file_edit.py
│   └── utils/
│       ├── fs.py
│       └── subprocess.py
└── tests/
```

## Architectural Principles

### 1. Preserve the original control flow

The port must preserve the original engine shape:

1. user input enters through the CLI
2. the session object records the message
3. the query engine prepares the provider request
4. the query loop calls the model
5. model tool calls trigger local tool execution
6. tool results are added back into message history
7. the model continues until it returns a final answer

The MVP must center the implementation around this loop instead of wrapping an existing Python agent framework.

### 2. Keep boundaries clear

The port will keep four stable boundaries:

- `provider`: talks to external model APIs
- `query_loop`: coordinates the model/tool round-trip
- `tools`: executes local capabilities
- `runtime`: holds session state, transcript logic, and permissions

Each layer must be able to change internally without forcing changes in the others.

### 3. Translate behavior, not incidental complexity

The port will preserve source concepts that matter:

- registry-based discovery
- message-driven execution
- session state
- tool call / tool result cycles
- transcript persistence
- permission checks

The port will deliberately drop product-specific complexity that does not matter for the MVP:

- UI state trees
- feature flag branches
- remote-control pathways
- plugin and marketplace machinery

## Core Runtime Model

### Messages

The Python runtime will use an internal message model rather than raw provider payloads everywhere.

Minimum message categories:

- `system`
- `user`
- `assistant`
- `tool`

Messages may contain:

- plain text content
- structured tool call data
- structured tool result data

The internal history will be the source of truth for the engine. Providers will translate to and from that history shape rather than owning message state directly.

### Session State

The session state will be intentionally smaller than the original product's `AppState`.

Minimum state:

- current working directory
- accumulated message history
- transcript output path
- permission mode
- turn count
- provider selection and model configuration

This is enough to support a stable headless engine without inheriting UI-specific state.

### Tool Model

Each tool definition must include:

- stable name
- description
- input schema
- enablement check
- read-only vs write behavior
- permission check
- execution function

This mirrors the original tool registry concept while using Python-native data models.

## Module Responsibilities

### `cli.py`

Responsibilities:

- parse command-line arguments
- build runtime configuration
- load commands and tools
- create the provider and query engine
- execute a prompt in headless mode
- print final output and exit status

The CLI will only target non-interactive usage in the MVP.

### `commands.py`

Responsibilities:

- register headless commands
- expose a simple lookup mechanism
- keep command metadata separate from execution logic

The MVP command layer will stay small and utility-oriented. It is included mainly to preserve source structure rather than to reach feature parity.

### `tools.py`

Responsibilities:

- hold the canonical tool registry
- expose enabled tools for a session
- provide lookup by name
- keep tool metadata independent from provider-specific schemas

### `query_engine.py`

Responsibilities:

- own the lifecycle of one conversation session
- maintain mutable message history
- prepare system prompt and runtime context
- call the query loop for each submitted user prompt
- persist transcripts
- return the final answer to the CLI

This module is the Python equivalent of the source session orchestrator, not the place where individual tool execution rules should live.

### `query_loop.py`

Responsibilities:

- perform one multi-turn model/tool exchange
- call the provider
- detect tool calls
- dispatch tool execution
- add tool results back into history
- stop when the provider returns a final assistant answer

This loop is the most important part of the translation. It must remain readable and close in intent to the TypeScript source.

### `providers/base.py`

Responsibilities:

- define the provider contract
- normalize provider responses into internal message and tool call structures
- hide external API differences from the rest of the engine

### `providers/anthropic.py`

Responsibilities:

- implement the first real provider
- map internal tool specs to Anthropic-compatible tool definitions
- convert Anthropic responses back into internal tool calls or final text

The provider abstraction will be designed so future providers can be added without changing the query loop contract.

### `runtime/app_state.py`

Responsibilities:

- define the minimal runtime session structures
- keep engine state out of CLI parsing code

### `runtime/permissioning.py`

Responsibilities:

- centralize permission checks
- support minimal modes:
  - `default`
  - `accept_edits`
  - `bypass_permissions`

### `runtime/transcript.py`

Responsibilities:

- write session history to local disk
- use a simple append-friendly format such as JSONL
- avoid coupling transcript persistence to provider payload formats

## Query Execution Design

The Python execution path will be:

1. CLI receives a prompt
2. `QueryEngine.submit(prompt)` records a user message
3. `QueryEngine` builds the session context and provider request
4. `query_loop.run(...)` sends the current history to the provider
5. the provider returns either:
   - assistant text
   - one or more tool calls
6. if tool calls are returned:
   - execute tools serially
   - convert each result into an internal tool-result message
   - append results to history
   - call the provider again
7. if no tool calls are returned:
   - finalize the response
   - persist transcript state
   - return the final answer

### Planned Simplifications

The MVP will intentionally simplify the source implementation in these ways:

- synchronous request/response control flow instead of full streaming event choreography
- serial tool execution instead of streaming tool executors
- simple transcript persistence instead of full resume infrastructure
- smaller permission surface
- no REPL interruption model

These simplifications are acceptable because they do not change the essential engine behavior.

## Provider Design

The first provider implementation will follow the source-oriented design rather than a generic chat abstraction.

Provider responsibilities:

- accept internal message history
- accept the active tool list
- return either text output or tool calls
- preserve enough structured information for iterative tool use

The provider interface should be stable enough for later addition of:

- OpenAI-compatible providers
- test providers and fake providers
- offline fixtures for deterministic tests

## Tools Included in the MVP

### `bash`

Purpose:

- execute local shell commands

Constraints:

- enforce timeout handling
- pass through permission checks
- capture stdout, stderr, and exit code
- expose failures as structured tool results

### `file_read`

Purpose:

- read file contents from disk

Constraints:

- support path validation relative to the working directory
- return bounded, explicit output

### `file_write`

Purpose:

- create or overwrite files

Constraints:

- require write permissions unless bypassed
- return a structured success result

### `file_edit`

Purpose:

- apply targeted text replacements

Constraints:

- keep the MVP scope narrow
- support deterministic, testable behavior
- prefer explicit old/new replacement semantics over open-ended patch parsing

## Permissions

The MVP permission model will be intentionally narrow.

Supported modes:

- `default`: read tools allowed, write and shell tools require explicit confirmation rules
- `accept_edits`: file writes and edits allowed, other sensitive actions still checked
- `bypass_permissions`: all tools allowed

This keeps the permission surface understandable while preserving the source system's basic safety model.

## Transcript Design

The engine will persist transcript records in JSONL.

Each record should include enough information to reconstruct:

- role
- content
- tool call metadata
- tool result metadata
- timestamps

The transcript layer is for observability and future resumption work. It does not need to reproduce the full original log format in the MVP.

## Testing Strategy

The MVP test suite must prove the translated architecture works, not just that modules import successfully.

Required coverage:

- registry tests
  - command registration
  - tool registration
  - lookup behavior
- provider tests
  - internal-to-provider request mapping
  - provider response normalization
- query loop tests
  - plain assistant response
  - one tool call round-trip
  - multi-turn tool call continuation
- tool tests
  - `bash`
  - `file_read`
  - `file_write`
  - `file_edit`
- transcript tests
  - message persistence format
- end-to-end test
  - fake provider issues a tool call
  - local tool runs
  - tool result re-enters the loop
  - final assistant answer is returned

## Verification Standard

The MVP is considered complete only when the following real flow works:

1. a prompt is submitted through the CLI
2. the provider returns a tool call
3. the tool executes locally
4. the tool result is added back into history
5. the provider returns a final answer
6. the session transcript is written to disk
7. tests cover the control path and core tools

## Repository and Commit Workflow

The repository workflow is part of the design.

Rules:

- all work happens in `CHZarles/cc2cc`
- each meaningful increment is committed separately
- commit messages use Conventional Commits
- every non-trivial commit includes a detailed body

Commit body structure:

- `Why:` reason for the increment
- `What:` modules or behaviors changed
- `How:` implementation approach
- `Verification:` exact commands and outcomes

Example:

```text
feat(engine): add Python query loop skeleton

Why:
- establish a source-faithful MVP execution path before tool integrations

What:
- add query engine session state
- add provider abstraction
- add basic query loop without tool execution

How:
- translate the TypeScript QueryEngine/query split into Python modules
- keep message history and turn state in dataclasses

Verification:
- pytest tests/test_query_engine.py -q
- python -m cccore.cli --help
```

## Implementation Order

The recommended execution order is:

1. bootstrap the repository and Python package layout
2. add core data models and provider interfaces
3. add the query engine and query loop
4. add the initial tool registry and tool implementations
5. add transcript and permission runtime modules
6. wire the headless CLI
7. add end-to-end tests

This order preserves fast feedback and keeps each increment independently verifiable.

## Risks and Mitigations

### Risk: accidental rewrite instead of translation

Mitigation:

- keep module boundaries visibly aligned with the source
- avoid importing external agent orchestration frameworks

### Risk: over-scoping the MVP

Mitigation:

- treat UI, MCP, plugins, remote control, and swarms as explicit non-goals

### Risk: tool/provider coupling becomes too tight

Mitigation:

- provider returns normalized tool calls
- query loop owns execution
- tools stay provider-agnostic

### Risk: Python implementation drifts into a monolith

Mitigation:

- keep `query_engine.py` and `query_loop.py` separate
- keep runtime and tool modules small and single-purpose

## Decision Record

The design decisions fixed by this document are:

- the Python port targets the headless core engine first
- source-faithful architecture matters more than line-for-line translation
- the `QueryEngine` / `query_loop` split will be preserved
- the first real provider will be Anthropic-oriented
- the first tools will be `bash`, `file_read`, `file_write`, and `file_edit`
- the project will be built in a dedicated directory named `python-core-mvp/`
- development will proceed as a sequence of small, well-documented commits
