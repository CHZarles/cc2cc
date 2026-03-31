# Python Core Engine MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Python MVP that translates Claude Code's core headless engine architecture into a source-faithful, testable implementation under `python-core-mvp/`.

**Architecture:** The implementation preserves the original `main -> registries -> QueryEngine -> query loop -> tools` control path while simplifying the product surface to a headless CLI. Shared dataclasses define the engine contract, providers translate between internal messages and external APIs, the query loop owns tool round-trips, and the query engine owns session state plus transcript persistence.

**Tech Stack:** Python 3.11+, `pytest`, `httpx`, `argparse`, dataclasses, JSONL transcripts

---

## File Structure

The implementation will create and use these files:

- `python-core-mvp/pyproject.toml`
  Packaging metadata, editable install, and test dependencies.
- `python-core-mvp/README.md`
  Setup instructions, CLI usage, and architecture summary.
- `python-core-mvp/src/cccore/__init__.py`
  Package version and public exports.
- `python-core-mvp/src/cccore/config.py`
  Runtime configuration loading from CLI and environment.
- `python-core-mvp/src/cccore/messages.py`
  Internal message and tool-call dataclasses plus JSON-serializable records.
- `python-core-mvp/src/cccore/models.py`
  Shared engine dataclasses for tools, providers, command metadata, and results.
- `python-core-mvp/src/cccore/commands.py`
  Headless command metadata registry.
- `python-core-mvp/src/cccore/tools.py`
  Tool registry helpers and default tool construction.
- `python-core-mvp/src/cccore/query_loop.py`
  Multi-turn provider/tool execution loop.
- `python-core-mvp/src/cccore/query_engine.py`
  Session orchestration and transcript-aware prompt submission.
- `python-core-mvp/src/cccore/cli.py`
  Headless command-line entrypoint.
- `python-core-mvp/src/cccore/providers/base.py`
  Provider protocol definition.
- `python-core-mvp/src/cccore/providers/anthropic.py`
  Anthropic Messages API implementation and mapping helpers.
- `python-core-mvp/src/cccore/runtime/app_state.py`
  Minimal session state.
- `python-core-mvp/src/cccore/runtime/permissioning.py`
  Minimal permission mode enforcement.
- `python-core-mvp/src/cccore/runtime/transcript.py`
  JSONL transcript writing helpers.
- `python-core-mvp/src/cccore/tool_impls/bash.py`
  Shell execution tool.
- `python-core-mvp/src/cccore/tool_impls/file_read.py`
  Read-only file tool.
- `python-core-mvp/src/cccore/tool_impls/file_write.py`
  File creation and overwrite tool.
- `python-core-mvp/src/cccore/tool_impls/file_edit.py`
  Deterministic string-replacement edit tool.
- `python-core-mvp/src/cccore/utils/fs.py`
  Shared path validation and text file helpers.
- `python-core-mvp/src/cccore/utils/subprocess.py`
  Shared subprocess execution helper.
- `python-core-mvp/tests/test_smoke.py`
  Package import smoke test.
- `python-core-mvp/tests/test_core_models.py`
  Shared model and config coverage.
- `python-core-mvp/tests/test_registries_and_permissioning.py`
  Command/tool registry plus permission mode tests.
- `python-core-mvp/tests/test_tool_impls.py`
  Local tool behavior tests.
- `python-core-mvp/tests/test_query_loop.py`
  Tool round-trip loop tests with a fake provider.
- `python-core-mvp/tests/test_query_engine.py`
  Session state and transcript persistence tests.
- `python-core-mvp/tests/test_anthropic_provider.py`
  Anthropic payload mapping tests.
- `python-core-mvp/tests/test_cli.py`
  Headless CLI tests using a fake provider.

## Task 1: Bootstrap The Python Package

**Files:**
- Create: `python-core-mvp/pyproject.toml`
- Create: `python-core-mvp/README.md`
- Create: `python-core-mvp/src/cccore/__init__.py`
- Test: `python-core-mvp/tests/test_smoke.py`

- [ ] **Step 1: Write the failing smoke test**

```python
# python-core-mvp/tests/test_smoke.py
from cccore import __version__


def test_package_exposes_version() -> None:
    assert __version__ == "0.1.0"
```

- [ ] **Step 2: Run the smoke test to verify it fails**

Run:

```bash
cd /tmp/cc/cc2cc
pytest python-core-mvp/tests/test_smoke.py -q
```

Expected:

```text
E   ModuleNotFoundError: No module named 'cccore'
```

- [ ] **Step 3: Create the package skeleton**

```toml
# python-core-mvp/pyproject.toml
[build-system]
requires = ["setuptools>=69", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "cccore"
version = "0.1.0"
description = "Python core engine MVP translated from Claude Code architecture"
readme = "README.md"
requires-python = ">=3.11"
dependencies = [
  "httpx>=0.27,<1.0",
]

[project.optional-dependencies]
dev = [
  "pytest>=8.0,<9.0",
]

[project.scripts]
cccore = "cccore.cli:main"

[tool.setuptools]
package-dir = {"" = "src"}

[tool.setuptools.packages.find]
where = ["src"]

[tool.pytest.ini_options]
pythonpath = ["src"]
testpaths = ["tests"]
```

~~~~markdown
# python-core-mvp/README.md
# cccore

`cccore` is a Python MVP that translates the core headless execution architecture
from Claude Code into a smaller, testable implementation.

## Status

This package intentionally targets the headless core engine only:

- CLI bootstrap
- command and tool registries
- query engine
- multi-turn query loop
- local tool execution
- transcript persistence

## Development

```bash
cd /tmp/cc/cc2cc/python-core-mvp
python -m venv .venv
. .venv/bin/activate
pip install -e .[dev]
pytest -q
```

## Initial Layout

The package will grow around `src/cccore/` and `tests/`.
~~~~

```python
# python-core-mvp/src/cccore/__init__.py
__all__ = ["__version__"]

__version__ = "0.1.0"
```

- [ ] **Step 4: Run the smoke test to verify it passes**

Run:

```bash
cd /tmp/cc/cc2cc/python-core-mvp
pip install -e .[dev]
pytest tests/test_smoke.py -q
```

Expected:

```text
1 passed
```

- [ ] **Step 5: Commit the bootstrap**

```bash
cd /tmp/cc/cc2cc
git add python-core-mvp/pyproject.toml python-core-mvp/README.md python-core-mvp/src/cccore/__init__.py python-core-mvp/tests/test_smoke.py
git commit -m "feat(python-core): bootstrap package skeleton" -m "Why:
- establish the Python package boundary before translating engine code
- create a stable editable-install workflow for the remaining tasks

What:
- add pyproject metadata and dev dependencies
- add the initial README
- add the cccore package version export
- add a smoke test for package import

How:
- use setuptools with a src layout
- expose a console script entrypoint that will be implemented later
- keep the first test minimal and focused on package availability

Verification:
- cd python-core-mvp && pip install -e .[dev]
- cd python-core-mvp && pytest tests/test_smoke.py -q"
```

## Task 2: Add Shared Models, Messages, And Provider Contracts

**Files:**
- Create: `python-core-mvp/src/cccore/config.py`
- Create: `python-core-mvp/src/cccore/messages.py`
- Create: `python-core-mvp/src/cccore/models.py`
- Create: `python-core-mvp/src/cccore/providers/base.py`
- Test: `python-core-mvp/tests/test_core_models.py`

- [ ] **Step 1: Write the failing core model tests**

```python
# python-core-mvp/tests/test_core_models.py
from pathlib import Path

from cccore.config import RuntimeConfig
from cccore.messages import Message, ToolCall, make_tool_result_message


def test_message_to_record_includes_tool_calls() -> None:
    message = Message(
        role="assistant",
        content="Running a tool",
        tool_calls=[ToolCall(id="call_1", name="bash", arguments={"command": "pwd"})],
    )

    record = message.to_record()

    assert record["role"] == "assistant"
    assert record["tool_calls"] == [
        {"id": "call_1", "name": "bash", "arguments": {"command": "pwd"}}
    ]


def test_make_tool_result_message_marks_error_flag() -> None:
    message = make_tool_result_message(
        tool_call_id="call_1",
        tool_name="bash",
        content="permission denied",
        is_error=True,
    )

    assert message.role == "tool"
    assert message.tool_call_id == "call_1"
    assert message.tool_name == "bash"
    assert message.is_error is True


def test_runtime_config_reads_environment(monkeypatch) -> None:
    monkeypatch.setenv("CCCORE_PROVIDER", "anthropic")
    monkeypatch.setenv("CCCORE_MODEL", "claude-test-model")
    monkeypatch.setenv("CCCORE_PERMISSION_MODE", "accept_edits")
    monkeypatch.setenv("ANTHROPIC_API_KEY", "test-key")

    config = RuntimeConfig.from_env(cwd=Path("/tmp/project"))

    assert config.cwd == Path("/tmp/project")
    assert config.provider_name == "anthropic"
    assert config.model == "claude-test-model"
    assert config.permission_mode == "accept_edits"
    assert config.anthropic_api_key == "test-key"
```

- [ ] **Step 2: Run the core model tests to verify they fail**

Run:

```bash
cd /tmp/cc/cc2cc/python-core-mvp
pytest tests/test_core_models.py -q
```

Expected:

```text
E   ModuleNotFoundError: No module named 'cccore.config'
```

- [ ] **Step 3: Implement shared runtime contracts**

```python
# python-core-mvp/src/cccore/messages.py
from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Any, Literal


def _utc_now() -> str:
    return datetime.now(timezone.utc).isoformat()


@dataclass(slots=True)
class ToolCall:
    id: str
    name: str
    arguments: dict[str, Any]

    def to_record(self) -> dict[str, Any]:
        return {
            "id": self.id,
            "name": self.name,
            "arguments": self.arguments,
        }


@dataclass(slots=True)
class Message:
    role: Literal["system", "user", "assistant", "tool"]
    content: str = ""
    tool_calls: list[ToolCall] = field(default_factory=list)
    tool_call_id: str | None = None
    tool_name: str | None = None
    is_error: bool = False
    timestamp: str = field(default_factory=_utc_now)

    def to_record(self) -> dict[str, Any]:
        return {
            "role": self.role,
            "content": self.content,
            "tool_calls": [tool_call.to_record() for tool_call in self.tool_calls],
            "tool_call_id": self.tool_call_id,
            "tool_name": self.tool_name,
            "is_error": self.is_error,
            "timestamp": self.timestamp,
        }


def make_user_message(content: str) -> Message:
    return Message(role="user", content=content)


def make_assistant_message(
    content: str = "",
    *,
    tool_calls: list[ToolCall] | None = None,
) -> Message:
    return Message(role="assistant", content=content, tool_calls=tool_calls or [])


def make_tool_result_message(
    *,
    tool_call_id: str,
    tool_name: str,
    content: str,
    is_error: bool = False,
) -> Message:
    return Message(
        role="tool",
        content=content,
        tool_call_id=tool_call_id,
        tool_name=tool_name,
        is_error=is_error,
    )
```

```python
# python-core-mvp/src/cccore/models.py
from __future__ import annotations

from collections.abc import Callable, Sequence
from dataclasses import dataclass
from pathlib import Path
from typing import Any, Literal, Protocol

from cccore.messages import Message


PermissionMode = Literal["default", "accept_edits", "bypass_permissions"]


@dataclass(slots=True)
class ToolContext:
    cwd: Path
    permission_mode: PermissionMode


@dataclass(slots=True)
class ToolExecutionResult:
    tool_call_id: str
    tool_name: str
    content: str
    is_error: bool = False


ToolRunner = Callable[[dict[str, Any], ToolContext], ToolExecutionResult]


@dataclass(slots=True)
class ToolDefinition:
    name: str
    description: str
    input_schema: dict[str, Any]
    readonly: bool
    requires_confirmation: bool
    enabled: bool
    run: ToolRunner


@dataclass(slots=True)
class CommandDefinition:
    name: str
    description: str


@dataclass(slots=True)
class ProviderTurn:
    assistant_message: Message
    stop_reason: str | None = None


@dataclass(slots=True)
class QueryLoopResult:
    history: list[Message]
    final_message: Message
    turns: int


class ModelProvider(Protocol):
    name: str

    def complete(
        self,
        *,
        system_prompt: str,
        messages: Sequence[Message],
        tools: Sequence[ToolDefinition],
        model: str | None,
    ) -> ProviderTurn:
        ...
```

```python
# python-core-mvp/src/cccore/config.py
from __future__ import annotations

import os
from dataclasses import dataclass
from pathlib import Path

from cccore.models import PermissionMode


@dataclass(slots=True)
class RuntimeConfig:
    cwd: Path
    provider_name: str
    model: str | None
    permission_mode: PermissionMode
    transcript_dir: Path
    max_turns: int
    max_output_tokens: int
    anthropic_api_key: str | None = None

    @classmethod
    def from_env(cls, cwd: Path | None = None) -> "RuntimeConfig":
        resolved_cwd = cwd or Path.cwd()
        permission_mode = os.getenv("CCCORE_PERMISSION_MODE", "default")
        if permission_mode not in {"default", "accept_edits", "bypass_permissions"}:
            raise ValueError(f"Unsupported permission mode: {permission_mode}")

        return cls(
            cwd=resolved_cwd,
            provider_name=os.getenv("CCCORE_PROVIDER", "anthropic"),
            model=os.getenv("CCCORE_MODEL") or os.getenv("ANTHROPIC_MODEL"),
            permission_mode=permission_mode,
            transcript_dir=Path(
                os.getenv(
                    "CCCORE_TRANSCRIPT_DIR",
                    str(resolved_cwd / ".cccore" / "transcripts"),
                )
            ),
            max_turns=int(os.getenv("CCCORE_MAX_TURNS", "8")),
            max_output_tokens=int(os.getenv("CCCORE_MAX_OUTPUT_TOKENS", "1024")),
            anthropic_api_key=os.getenv("ANTHROPIC_API_KEY"),
        )
```

```python
# python-core-mvp/src/cccore/providers/base.py
from __future__ import annotations

from cccore.models import ModelProvider, ProviderTurn

__all__ = ["ModelProvider", "ProviderTurn"]
```

- [ ] **Step 4: Run the core model tests to verify they pass**

Run:

```bash
cd /tmp/cc/cc2cc/python-core-mvp
pytest tests/test_core_models.py -q
```

Expected:

```text
3 passed
```

- [ ] **Step 5: Commit the shared contracts**

```bash
cd /tmp/cc/cc2cc
git add python-core-mvp/src/cccore/config.py python-core-mvp/src/cccore/messages.py python-core-mvp/src/cccore/models.py python-core-mvp/src/cccore/providers/base.py python-core-mvp/tests/test_core_models.py
git commit -m "feat(core-models): add shared engine contracts" -m "Why:
- define the internal data model before building registries, tools, or query execution
- keep later modules aligned on one set of message and provider contracts

What:
- add runtime config loading
- add message and tool-call dataclasses
- add shared models for tools, commands, providers, and query results
- add a provider base export

How:
- use dataclasses for engine state and payloads
- keep provider contracts provider-agnostic at the package boundary
- serialize messages through a stable JSON-ready record format

Verification:
- cd python-core-mvp && pytest tests/test_core_models.py -q"
```

## Task 3: Add Registries And Permission Enforcement

**Files:**
- Create: `python-core-mvp/src/cccore/commands.py`
- Create: `python-core-mvp/src/cccore/runtime/permissioning.py`
- Create: `python-core-mvp/src/cccore/tools.py`
- Test: `python-core-mvp/tests/test_registries_and_permissioning.py`

- [ ] **Step 1: Write the failing registry and permission tests**

```python
# python-core-mvp/tests/test_registries_and_permissioning.py
import pytest
from pathlib import Path

from cccore.commands import build_default_commands, get_command
from cccore.models import ToolContext, ToolDefinition, ToolExecutionResult
from cccore.runtime.permissioning import ensure_tool_allowed
from cccore.tools import get_enabled_tools, get_tool


def _dummy_tool(*, name: str, readonly: bool, requires_confirmation: bool, enabled: bool = True) -> ToolDefinition:
    return ToolDefinition(
        name=name,
        description=f"{name} tool",
        input_schema={"type": "object", "properties": {}},
        readonly=readonly,
        requires_confirmation=requires_confirmation,
        enabled=enabled,
        run=lambda arguments, context: ToolExecutionResult(
            tool_call_id="dummy",
            tool_name=name,
            content="ok",
            is_error=False,
        ),
    )


def test_command_registry_finds_named_command() -> None:
    command = get_command(build_default_commands(), "run")
    assert command is not None
    assert command.description == "Execute a prompt through the query engine"


def test_tool_registry_filters_disabled_tools() -> None:
    enabled = get_enabled_tools(
        [
            _dummy_tool(name="file_read", readonly=True, requires_confirmation=False),
            _dummy_tool(name="bash", readonly=False, requires_confirmation=True, enabled=False),
        ]
    )

    assert [tool.name for tool in enabled] == ["file_read"]


def test_default_mode_blocks_confirmed_tool() -> None:
    tool = _dummy_tool(name="bash", readonly=False, requires_confirmation=True)

    with pytest.raises(PermissionError, match="bash"):
        ensure_tool_allowed(tool, ToolContext(cwd=Path("/tmp"), permission_mode="default"))


def test_accept_edits_allows_file_write() -> None:
    tool = _dummy_tool(name="file_write", readonly=False, requires_confirmation=True)
    context = ToolContext(cwd=Path("/tmp"), permission_mode="accept_edits")

    ensure_tool_allowed(tool, context)


def test_bypass_permissions_allows_shell() -> None:
    tool = _dummy_tool(name="bash", readonly=False, requires_confirmation=True)
    context = ToolContext(cwd=Path("/tmp"), permission_mode="bypass_permissions")

    ensure_tool_allowed(tool, context)


def test_get_tool_returns_matching_definition() -> None:
    tools = [
        _dummy_tool(name="file_read", readonly=True, requires_confirmation=False),
        _dummy_tool(name="file_write", readonly=False, requires_confirmation=True),
    ]

    assert get_tool(tools, "file_write") is not None
    assert get_tool(tools, "missing") is None
```

- [ ] **Step 2: Run the registry tests to verify they fail**

Run:

```bash
cd /tmp/cc/cc2cc/python-core-mvp
pytest tests/test_registries_and_permissioning.py -q
```

Expected:

```text
E   ModuleNotFoundError: No module named 'cccore.commands'
```

- [ ] **Step 3: Implement command and tool registries**

```python
# python-core-mvp/src/cccore/commands.py
from __future__ import annotations

from collections.abc import Sequence

from cccore.models import CommandDefinition


def build_default_commands() -> list[CommandDefinition]:
    return [
        CommandDefinition(name="run", description="Execute a prompt through the query engine"),
        CommandDefinition(name="tools", description="List the enabled local tools"),
        CommandDefinition(name="transcript", description="Print the current transcript directory"),
    ]


def get_command(
    commands: Sequence[CommandDefinition],
    name: str,
) -> CommandDefinition | None:
    for command in commands:
        if command.name == name:
            return command
    return None
```

```python
# python-core-mvp/src/cccore/runtime/permissioning.py
from __future__ import annotations

from cccore.models import ToolContext, ToolDefinition


def ensure_tool_allowed(tool: ToolDefinition, context: ToolContext) -> None:
    if not tool.enabled:
        raise PermissionError(f"Tool '{tool.name}' is disabled")

    if context.permission_mode == "bypass_permissions":
        return

    if not tool.requires_confirmation:
        return

    if context.permission_mode == "accept_edits" and tool.name in {"file_write", "file_edit"}:
        return

    raise PermissionError(
        f"Tool '{tool.name}' requires a less restrictive permission mode than '{context.permission_mode}'"
    )
```

```python
# python-core-mvp/src/cccore/tools.py
from __future__ import annotations

from collections.abc import Sequence

from cccore.models import ToolDefinition


def get_enabled_tools(tools: Sequence[ToolDefinition]) -> list[ToolDefinition]:
    return [tool for tool in tools if tool.enabled]


def get_tool(
    tools: Sequence[ToolDefinition],
    name: str,
) -> ToolDefinition | None:
    for tool in tools:
        if tool.name == name:
            return tool
    return None
```

- [ ] **Step 4: Run the registry tests to verify they pass**

Run:

```bash
cd /tmp/cc/cc2cc/python-core-mvp
pytest tests/test_registries_and_permissioning.py -q
```

Expected:

```text
6 passed
```

- [ ] **Step 5: Commit the registries and permission layer**

```bash
cd /tmp/cc/cc2cc
git add python-core-mvp/src/cccore/commands.py python-core-mvp/src/cccore/runtime/permissioning.py python-core-mvp/src/cccore/tools.py python-core-mvp/tests/test_registries_and_permissioning.py
git commit -m "feat(registry): add command, tool, and permission helpers" -m "Why:
- registries and permission checks are the stable glue between the engine and local capabilities
- the query loop needs these helpers before tool execution can be implemented

What:
- add the headless command registry
- add tool lookup and enabled-tool filtering
- add minimal permission mode enforcement
- add tests covering registry lookup and permission behavior

How:
- keep command metadata separate from CLI execution
- keep permission logic mode-based and deterministic for the headless MVP
- avoid pulling provider logic into registry modules

Verification:
- cd python-core-mvp && pytest tests/test_registries_and_permissioning.py -q"
```

## Task 4: Implement The Core Local Tools

**Files:**
- Create: `python-core-mvp/src/cccore/utils/fs.py`
- Create: `python-core-mvp/src/cccore/utils/subprocess.py`
- Create: `python-core-mvp/src/cccore/tool_impls/bash.py`
- Create: `python-core-mvp/src/cccore/tool_impls/file_read.py`
- Create: `python-core-mvp/src/cccore/tool_impls/file_write.py`
- Create: `python-core-mvp/src/cccore/tool_impls/file_edit.py`
- Modify: `python-core-mvp/src/cccore/tools.py`
- Test: `python-core-mvp/tests/test_tool_impls.py`

- [ ] **Step 1: Write the failing tool implementation tests**

```python
# python-core-mvp/tests/test_tool_impls.py
from pathlib import Path

from cccore.models import ToolContext
from cccore.tool_impls.bash import build_bash_tool
from cccore.tool_impls.file_edit import build_file_edit_tool
from cccore.tool_impls.file_read import build_file_read_tool
from cccore.tool_impls.file_write import build_file_write_tool
from cccore.tools import build_default_tools


def test_file_write_and_read_round_trip(tmp_path: Path) -> None:
    context = ToolContext(cwd=tmp_path, permission_mode="bypass_permissions")
    file_write = build_file_write_tool()
    file_read = build_file_read_tool()

    write_result = file_write.run({"path": "notes.txt", "content": "hello"}, context)
    read_result = file_read.run({"path": "notes.txt"}, context)

    assert write_result.is_error is False
    assert read_result.content == "hello"


def test_file_edit_replaces_exact_match(tmp_path: Path) -> None:
    target = tmp_path / "notes.txt"
    target.write_text("alpha beta gamma", encoding="utf-8")
    context = ToolContext(cwd=tmp_path, permission_mode="bypass_permissions")

    result = build_file_edit_tool().run(
        {"path": "notes.txt", "old": "beta", "new": "delta"},
        context,
    )

    assert result.is_error is False
    assert target.read_text(encoding="utf-8") == "alpha delta gamma"


def test_bash_tool_captures_stdout(tmp_path: Path) -> None:
    context = ToolContext(cwd=tmp_path, permission_mode="bypass_permissions")
    result = build_bash_tool().run({"command": "printf 'hello'"}, context)
    assert result.content == "hello"


def test_default_tool_registry_contains_expected_names() -> None:
    names = [tool.name for tool in build_default_tools()]
    assert names == ["bash", "file_read", "file_write", "file_edit"]
```

- [ ] **Step 2: Run the tool tests to verify they fail**

Run:

```bash
cd /tmp/cc/cc2cc/python-core-mvp
pytest tests/test_tool_impls.py -q
```

Expected:

```text
E   ModuleNotFoundError: No module named 'cccore.tool_impls.bash'
```

- [ ] **Step 3: Implement the local tool set**

```python
# python-core-mvp/src/cccore/utils/fs.py
from __future__ import annotations

from pathlib import Path


def resolve_path(cwd: Path, raw_path: str) -> Path:
    resolved = (cwd / raw_path).resolve()
    if cwd.resolve() not in {resolved, *resolved.parents}:
        raise ValueError(f"Path escapes working directory: {raw_path}")
    return resolved


def read_text_file(path: Path) -> str:
    return path.read_text(encoding="utf-8")


def write_text_file(path: Path, content: str) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(content, encoding="utf-8")
```

```python
# python-core-mvp/src/cccore/utils/subprocess.py
from __future__ import annotations

import subprocess
from dataclasses import dataclass
from pathlib import Path


@dataclass(slots=True)
class CompletedShellCommand:
    stdout: str
    stderr: str
    returncode: int


def run_shell_command(
    *,
    command: str,
    cwd: Path,
    timeout_seconds: int = 30,
) -> CompletedShellCommand:
    completed = subprocess.run(
        command,
        cwd=cwd,
        shell=True,
        capture_output=True,
        text=True,
        timeout=timeout_seconds,
        check=False,
    )
    return CompletedShellCommand(
        stdout=completed.stdout,
        stderr=completed.stderr,
        returncode=completed.returncode,
    )
```

```python
# python-core-mvp/src/cccore/tool_impls/file_read.py
from __future__ import annotations

from cccore.models import ToolContext, ToolDefinition, ToolExecutionResult
from cccore.utils.fs import read_text_file, resolve_path


def build_file_read_tool() -> ToolDefinition:
    def run(arguments: dict[str, object], context: ToolContext) -> ToolExecutionResult:
        path = resolve_path(context.cwd, str(arguments["path"]))
        content = read_text_file(path)
        return ToolExecutionResult(
            tool_call_id=str(arguments.get("_tool_call_id", "")),
            tool_name="file_read",
            content=content,
            is_error=False,
        )

    return ToolDefinition(
        name="file_read",
        description="Read a text file from the working directory",
        input_schema={
            "type": "object",
            "properties": {"path": {"type": "string"}},
            "required": ["path"],
        },
        readonly=True,
        requires_confirmation=False,
        enabled=True,
        run=run,
    )
```

```python
# python-core-mvp/src/cccore/tool_impls/file_write.py
from __future__ import annotations

from cccore.models import ToolContext, ToolDefinition, ToolExecutionResult
from cccore.utils.fs import resolve_path, write_text_file


def build_file_write_tool() -> ToolDefinition:
    def run(arguments: dict[str, object], context: ToolContext) -> ToolExecutionResult:
        path = resolve_path(context.cwd, str(arguments["path"]))
        write_text_file(path, str(arguments["content"]))
        return ToolExecutionResult(
            tool_call_id=str(arguments.get("_tool_call_id", "")),
            tool_name="file_write",
            content=f"Wrote {path.relative_to(context.cwd)}",
            is_error=False,
        )

    return ToolDefinition(
        name="file_write",
        description="Create or overwrite a text file in the working directory",
        input_schema={
            "type": "object",
            "properties": {
                "path": {"type": "string"},
                "content": {"type": "string"},
            },
            "required": ["path", "content"],
        },
        readonly=False,
        requires_confirmation=True,
        enabled=True,
        run=run,
    )
```

```python
# python-core-mvp/src/cccore/tool_impls/file_edit.py
from __future__ import annotations

from cccore.models import ToolContext, ToolDefinition, ToolExecutionResult
from cccore.utils.fs import read_text_file, resolve_path, write_text_file


def build_file_edit_tool() -> ToolDefinition:
    def run(arguments: dict[str, object], context: ToolContext) -> ToolExecutionResult:
        path = resolve_path(context.cwd, str(arguments["path"]))
        old = str(arguments["old"])
        new = str(arguments["new"])
        original = read_text_file(path)
        if original.count(old) != 1:
            raise ValueError(
                f"Expected exactly one occurrence of {old!r} in {path.name}"
            )
        updated = original.replace(old, new, 1)
        write_text_file(path, updated)
        return ToolExecutionResult(
            tool_call_id=str(arguments.get("_tool_call_id", "")),
            tool_name="file_edit",
            content=f"Edited {path.relative_to(context.cwd)}",
            is_error=False,
        )

    return ToolDefinition(
        name="file_edit",
        description="Replace one exact string inside a text file",
        input_schema={
            "type": "object",
            "properties": {
                "path": {"type": "string"},
                "old": {"type": "string"},
                "new": {"type": "string"},
            },
            "required": ["path", "old", "new"],
        },
        readonly=False,
        requires_confirmation=True,
        enabled=True,
        run=run,
    )
```

```python
# python-core-mvp/src/cccore/tool_impls/bash.py
from __future__ import annotations

from cccore.models import ToolContext, ToolDefinition, ToolExecutionResult
from cccore.utils.subprocess import run_shell_command


def build_bash_tool() -> ToolDefinition:
    def run(arguments: dict[str, object], context: ToolContext) -> ToolExecutionResult:
        completed = run_shell_command(
            command=str(arguments["command"]),
            cwd=context.cwd,
            timeout_seconds=int(arguments.get("timeout_seconds", 30)),
        )
        content = completed.stdout if completed.returncode == 0 else completed.stderr or completed.stdout
        return ToolExecutionResult(
            tool_call_id=str(arguments.get("_tool_call_id", "")),
            tool_name="bash",
            content=content.rstrip("\n"),
            is_error=completed.returncode != 0,
        )

    return ToolDefinition(
        name="bash",
        description="Execute a shell command inside the working directory",
        input_schema={
            "type": "object",
            "properties": {
                "command": {"type": "string"},
                "timeout_seconds": {"type": "integer"},
            },
            "required": ["command"],
        },
        readonly=False,
        requires_confirmation=True,
        enabled=True,
        run=run,
    )
```

```python
# python-core-mvp/src/cccore/tools.py
from __future__ import annotations

from collections.abc import Sequence

from cccore.models import ToolDefinition
from cccore.tool_impls.bash import build_bash_tool
from cccore.tool_impls.file_edit import build_file_edit_tool
from cccore.tool_impls.file_read import build_file_read_tool
from cccore.tool_impls.file_write import build_file_write_tool


def build_default_tools() -> list[ToolDefinition]:
    return [
        build_bash_tool(),
        build_file_read_tool(),
        build_file_write_tool(),
        build_file_edit_tool(),
    ]


def get_enabled_tools(tools: Sequence[ToolDefinition]) -> list[ToolDefinition]:
    return [tool for tool in tools if tool.enabled]


def get_tool(
    tools: Sequence[ToolDefinition],
    name: str,
) -> ToolDefinition | None:
    for tool in tools:
        if tool.name == name:
            return tool
    return None
```

- [ ] **Step 4: Run the tool tests to verify they pass**

Run:

```bash
cd /tmp/cc/cc2cc/python-core-mvp
pytest tests/test_tool_impls.py -q
```

Expected:

```text
4 passed
```

- [ ] **Step 5: Commit the local tools**

```bash
cd /tmp/cc/cc2cc
git add python-core-mvp/src/cccore/utils/fs.py python-core-mvp/src/cccore/utils/subprocess.py python-core-mvp/src/cccore/tool_impls/bash.py python-core-mvp/src/cccore/tool_impls/file_read.py python-core-mvp/src/cccore/tool_impls/file_write.py python-core-mvp/src/cccore/tool_impls/file_edit.py python-core-mvp/src/cccore/tools.py python-core-mvp/tests/test_tool_impls.py
git commit -m "feat(tool-impls): add bash and file tools" -m "Why:
- the MVP needs a representative local tool surface before the query loop can execute provider-issued tool calls
- these four tools cover the source translation target for the first milestone

What:
- add shared filesystem and subprocess helpers
- add bash, file_read, file_write, and file_edit tools
- wire the default tool registry
- add direct tests for each tool path

How:
- keep file tools deterministic and bounded
- implement file_edit as a single exact replacement to avoid patch parser scope
- return structured ToolExecutionResult objects from every tool

Verification:
- cd python-core-mvp && pytest tests/test_tool_impls.py -q"
```

## Task 5: Implement The Query Loop

**Files:**
- Create: `python-core-mvp/src/cccore/query_loop.py`
- Test: `python-core-mvp/tests/test_query_loop.py`

- [ ] **Step 1: Write the failing query loop tests**

```python
# python-core-mvp/tests/test_query_loop.py
from pathlib import Path

from cccore.messages import Message, ToolCall, make_assistant_message, make_user_message
from cccore.models import ProviderTurn
from cccore.query_loop import run_query_loop
from cccore.tool_impls.file_write import build_file_write_tool


class FakeProvider:
    name = "fake"

    def __init__(self) -> None:
        self.calls = 0

    def complete(self, *, system_prompt, messages, tools, model):
        self.calls += 1
        if self.calls == 1:
            return ProviderTurn(
                assistant_message=make_assistant_message(
                    "Writing the file now",
                    tool_calls=[
                        ToolCall(
                            id="call_1",
                            name="file_write",
                            arguments={"path": "out.txt", "content": "hello"},
                        )
                    ],
                ),
                stop_reason="tool_use",
            )
        return ProviderTurn(
            assistant_message=make_assistant_message("Finished writing the file"),
            stop_reason="end_turn",
        )


def test_query_loop_executes_tool_and_returns_final_message(tmp_path: Path) -> None:
    result = run_query_loop(
        provider=FakeProvider(),
        system_prompt="You are a test provider",
        initial_messages=[make_user_message("Write hello to out.txt")],
        tools=[build_file_write_tool()],
        cwd=tmp_path,
        permission_mode="bypass_permissions",
        model="test-model",
        max_turns=4,
    )

    assert result.final_message.content == "Finished writing the file"
    assert (tmp_path / "out.txt").read_text(encoding="utf-8") == "hello"
    assert [message.role for message in result.history] == ["user", "assistant", "tool", "assistant"]


def test_query_loop_returns_plain_assistant_message_without_tools(tmp_path: Path) -> None:
    class TextOnlyProvider:
        name = "fake"

        def complete(self, *, system_prompt, messages, tools, model):
            return ProviderTurn(
                assistant_message=make_assistant_message("No tools needed"),
                stop_reason="end_turn",
            )

    result = run_query_loop(
        provider=TextOnlyProvider(),
        system_prompt="You are a test provider",
        initial_messages=[make_user_message("Say hello")],
        tools=[],
        cwd=tmp_path,
        permission_mode="default",
        model="test-model",
        max_turns=2,
    )

    assert result.final_message.content == "No tools needed"
    assert [message.role for message in result.history] == ["user", "assistant"]
```

- [ ] **Step 2: Run the query loop tests to verify they fail**

Run:

```bash
cd /tmp/cc/cc2cc/python-core-mvp
pytest tests/test_query_loop.py -q
```

Expected:

```text
E   ModuleNotFoundError: No module named 'cccore.query_loop'
```

- [ ] **Step 3: Implement the multi-turn query loop**

```python
# python-core-mvp/src/cccore/query_loop.py
from __future__ import annotations

from collections.abc import Sequence
from pathlib import Path

from cccore.messages import Message, make_tool_result_message
from cccore.models import (
    ModelProvider,
    PermissionMode,
    QueryLoopResult,
    ToolContext,
    ToolExecutionResult,
    ToolDefinition,
)
from cccore.runtime.permissioning import ensure_tool_allowed
from cccore.tools import get_enabled_tools, get_tool


def _execute_tool_call(
    *,
    tool_name: str,
    tool_call_id: str,
    arguments: dict[str, object],
    tools: Sequence[ToolDefinition],
    context: ToolContext,
) -> Message:
    tool = get_tool(tools, tool_name)
    if tool is None:
        return make_tool_result_message(
            tool_call_id=tool_call_id,
            tool_name=tool_name,
            content=f"Unknown tool: {tool_name}",
            is_error=True,
        )

    try:
        ensure_tool_allowed(tool, context)
        result: ToolExecutionResult = tool.run(
            {**arguments, "_tool_call_id": tool_call_id},
            context,
        )
        return make_tool_result_message(
            tool_call_id=result.tool_call_id or tool_call_id,
            tool_name=result.tool_name,
            content=result.content,
            is_error=result.is_error,
        )
    except Exception as exc:
        return make_tool_result_message(
            tool_call_id=tool_call_id,
            tool_name=tool_name,
            content=str(exc),
            is_error=True,
        )


def run_query_loop(
    *,
    provider: ModelProvider,
    system_prompt: str,
    initial_messages: Sequence[Message],
    tools: Sequence[ToolDefinition],
    cwd: Path,
    permission_mode: PermissionMode,
    model: str | None,
    max_turns: int,
) -> QueryLoopResult:
    history = list(initial_messages)
    enabled_tools = get_enabled_tools(tools)
    context = ToolContext(cwd=cwd, permission_mode=permission_mode)

    for turn_index in range(1, max_turns + 1):
        provider_turn = provider.complete(
            system_prompt=system_prompt,
            messages=history,
            tools=enabled_tools,
            model=model,
        )
        assistant_message = provider_turn.assistant_message
        history.append(assistant_message)

        if not assistant_message.tool_calls:
            return QueryLoopResult(
                history=history,
                final_message=assistant_message,
                turns=turn_index,
            )

        for tool_call in assistant_message.tool_calls:
            tool_result_message = _execute_tool_call(
                tool_name=tool_call.name,
                tool_call_id=tool_call.id,
                arguments=tool_call.arguments,
                tools=enabled_tools,
                context=context,
            )
            history.append(tool_result_message)

    raise RuntimeError(f"Query loop exceeded max_turns={max_turns}")
```

- [ ] **Step 4: Run the query loop tests to verify they pass**

Run:

```bash
cd /tmp/cc/cc2cc/python-core-mvp
pytest tests/test_query_loop.py -q
```

Expected:

```text
2 passed
```

- [ ] **Step 5: Commit the query loop**

```bash
cd /tmp/cc/cc2cc
git add python-core-mvp/src/cccore/query_loop.py python-core-mvp/tests/test_query_loop.py
git commit -m "feat(loop): add multi-turn provider tool execution" -m "Why:
- the query loop is the core execution path that makes the Python port source-faithful
- without it the package cannot translate model-issued tool calls into local work

What:
- add the query loop implementation
- execute provider-issued tool calls against the local registry
- append tool results back into message history
- add tests for plain-text and tool-using turns

How:
- keep the loop synchronous and deterministic for the MVP
- centralize tool execution error handling inside the loop
- return a QueryLoopResult with the full history and final message

Verification:
- cd python-core-mvp && pytest tests/test_query_loop.py -q"
```

## Task 6: Add Session State, Transcripts, And The Query Engine

**Files:**
- Create: `python-core-mvp/src/cccore/runtime/app_state.py`
- Create: `python-core-mvp/src/cccore/runtime/transcript.py`
- Create: `python-core-mvp/src/cccore/query_engine.py`
- Test: `python-core-mvp/tests/test_query_engine.py`

- [ ] **Step 1: Write the failing query engine tests**

```python
# python-core-mvp/tests/test_query_engine.py
from pathlib import Path

from cccore.config import RuntimeConfig
from cccore.messages import make_assistant_message
from cccore.models import ProviderTurn
from cccore.query_engine import QueryEngine


class FakeProvider:
    name = "fake"

    def complete(self, *, system_prompt, messages, tools, model):
        return ProviderTurn(
            assistant_message=make_assistant_message("Engine reply"),
            stop_reason="end_turn",
        )


def test_query_engine_returns_text_and_persists_transcript(tmp_path: Path) -> None:
    config = RuntimeConfig(
        cwd=tmp_path,
        provider_name="fake",
        model="test-model",
        permission_mode="default",
        transcript_dir=tmp_path / ".cccore" / "transcripts",
        max_turns=4,
        max_output_tokens=256,
        anthropic_api_key=None,
    )
    engine = QueryEngine(
        config=config,
        provider=FakeProvider(),
        tools=[],
        system_prompt="You are a test engine",
    )

    result = engine.submit("Say hello")

    assert result == "Engine reply"
    assert len(engine.state.messages) == 2
    transcript_text = engine.state.transcript_path.read_text(encoding="utf-8")
    assert '"role": "user"' in transcript_text
    assert '"role": "assistant"' in transcript_text
```

- [ ] **Step 2: Run the query engine tests to verify they fail**

Run:

```bash
cd /tmp/cc/cc2cc/python-core-mvp
pytest tests/test_query_engine.py -q
```

Expected:

```text
E   ModuleNotFoundError: No module named 'cccore.query_engine'
```

- [ ] **Step 3: Implement session state and transcript-aware orchestration**

```python
# python-core-mvp/src/cccore/runtime/app_state.py
from __future__ import annotations

from dataclasses import dataclass, field
from pathlib import Path
from uuid import uuid4

from cccore.config import RuntimeConfig
from cccore.messages import Message


@dataclass(slots=True)
class SessionState:
    config: RuntimeConfig
    transcript_path: Path
    messages: list[Message] = field(default_factory=list)


def create_session_state(config: RuntimeConfig) -> SessionState:
    config.transcript_dir.mkdir(parents=True, exist_ok=True)
    transcript_path = config.transcript_dir / f"session-{uuid4().hex}.jsonl"
    return SessionState(config=config, transcript_path=transcript_path)
```

```python
# python-core-mvp/src/cccore/runtime/transcript.py
from __future__ import annotations

import json
from pathlib import Path
from typing import Iterable

from cccore.messages import Message


def append_transcript_messages(path: Path, messages: Iterable[Message]) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    with path.open("a", encoding="utf-8") as handle:
        for message in messages:
            handle.write(json.dumps(message.to_record(), ensure_ascii=False) + "\n")
```

```python
# python-core-mvp/src/cccore/query_engine.py
from __future__ import annotations

from collections.abc import Sequence

from cccore.config import RuntimeConfig
from cccore.messages import Message, make_user_message
from cccore.models import ModelProvider, ToolDefinition
from cccore.query_loop import run_query_loop
from cccore.runtime.app_state import SessionState, create_session_state
from cccore.runtime.transcript import append_transcript_messages


DEFAULT_SYSTEM_PROMPT = (
    "You are a Python translation of a headless coding agent. "
    "Use tools when needed and return concise final answers."
)


class QueryEngine:
    def __init__(
        self,
        *,
        config: RuntimeConfig,
        provider: ModelProvider,
        tools: Sequence[ToolDefinition],
        system_prompt: str = DEFAULT_SYSTEM_PROMPT,
    ) -> None:
        self.config = config
        self.provider = provider
        self.tools = list(tools)
        self.system_prompt = system_prompt
        self.state: SessionState = create_session_state(config)

    def submit(self, prompt: str) -> str:
        user_message = make_user_message(prompt)
        self.state.messages.append(user_message)
        append_transcript_messages(self.state.transcript_path, [user_message])

        result = run_query_loop(
            provider=self.provider,
            system_prompt=self.system_prompt,
            initial_messages=self.state.messages,
            tools=self.tools,
            cwd=self.config.cwd,
            permission_mode=self.config.permission_mode,
            model=self.config.model,
            max_turns=self.config.max_turns,
        )

        new_messages = result.history[len(self.state.messages) :]
        self.state.messages = result.history
        append_transcript_messages(self.state.transcript_path, new_messages)
        return result.final_message.content
```

- [ ] **Step 4: Run the query engine tests to verify they pass**

Run:

```bash
cd /tmp/cc/cc2cc/python-core-mvp
pytest tests/test_query_engine.py -q
```

Expected:

```text
1 passed
```

- [ ] **Step 5: Commit the session engine**

```bash
cd /tmp/cc/cc2cc
git add python-core-mvp/src/cccore/runtime/app_state.py python-core-mvp/src/cccore/runtime/transcript.py python-core-mvp/src/cccore/query_engine.py python-core-mvp/tests/test_query_engine.py
git commit -m "feat(engine): add session state and transcript-aware query engine" -m "Why:
- the session orchestrator is the source-level counterpart to QueryEngine.ts
- transcript persistence is required to make the MVP observable and debuggable

What:
- add minimal session state creation
- add JSONL transcript persistence
- add the QueryEngine submit path on top of the query loop
- add a session-level test that verifies transcript output

How:
- isolate transcript writing in a dedicated runtime module
- keep QueryEngine focused on session lifecycle and prompt submission
- append only new messages after each query loop run

Verification:
- cd python-core-mvp && pytest tests/test_query_engine.py -q"
```

## Task 7: Add The Anthropic Provider And Headless CLI

**Files:**
- Create: `python-core-mvp/src/cccore/providers/anthropic.py`
- Create: `python-core-mvp/src/cccore/cli.py`
- Modify: `python-core-mvp/README.md`
- Test: `python-core-mvp/tests/test_anthropic_provider.py`
- Test: `python-core-mvp/tests/test_cli.py`

- [ ] **Step 1: Write the failing provider and CLI tests**

```python
# python-core-mvp/tests/test_anthropic_provider.py
from cccore.messages import Message, ToolCall
from cccore.models import ToolDefinition
from cccore.providers.anthropic import build_anthropic_request, parse_anthropic_response


def _dummy_tool() -> ToolDefinition:
    return ToolDefinition(
        name="file_write",
        description="Create or overwrite a text file",
        input_schema={"type": "object", "properties": {"path": {"type": "string"}}},
        readonly=False,
        requires_confirmation=True,
        enabled=True,
        run=lambda arguments, context: None,  # type: ignore[arg-type]
    )


def test_build_anthropic_request_maps_tool_schema() -> None:
    body = build_anthropic_request(
        system_prompt="System prompt",
        messages=[Message(role="user", content="Write a file")],
        tools=[_dummy_tool()],
        model="claude-test-model",
        max_output_tokens=256,
    )

    assert body["system"] == "System prompt"
    assert body["model"] == "claude-test-model"
    assert body["tools"][0]["name"] == "file_write"


def test_parse_anthropic_response_extracts_tool_calls() -> None:
    response = {
        "stop_reason": "tool_use",
        "content": [
            {"type": "text", "text": "Writing the file"},
            {
                "type": "tool_use",
                "id": "call_1",
                "name": "file_write",
                "input": {"path": "out.txt", "content": "hello"},
            },
        ],
    }

    turn = parse_anthropic_response(response)

    assert turn.assistant_message.content == "Writing the file"
    assert turn.assistant_message.tool_calls == [
        ToolCall(
            id="call_1",
            name="file_write",
            arguments={"path": "out.txt", "content": "hello"},
        )
    ]
```

```python
# python-core-mvp/tests/test_cli.py
from pathlib import Path

from cccore.cli import main
from cccore.messages import make_assistant_message
from cccore.models import ProviderTurn


class FakeProvider:
    name = "fake"

    def complete(self, *, system_prompt, messages, tools, model):
        return ProviderTurn(
            assistant_message=make_assistant_message("CLI reply"),
            stop_reason="end_turn",
        )


def test_cli_run_command_prints_engine_reply(capsys, monkeypatch, tmp_path: Path) -> None:
    monkeypatch.setattr("cccore.cli.build_provider", lambda config: FakeProvider())
    exit_code = main(
        [
            "run",
            "Say hello",
            "--cwd",
            str(tmp_path),
            "--provider",
            "anthropic",
            "--model",
            "test-model",
            "--permission-mode",
            "default",
        ]
    )

    captured = capsys.readouterr()

    assert exit_code == 0
    assert captured.out.strip() == "CLI reply"
```

- [ ] **Step 2: Run the provider and CLI tests to verify they fail**

Run:

```bash
cd /tmp/cc/cc2cc/python-core-mvp
pytest tests/test_anthropic_provider.py tests/test_cli.py -q
```

Expected:

```text
E   ModuleNotFoundError: No module named 'cccore.providers.anthropic'
```

- [ ] **Step 3: Implement the Anthropic provider and CLI wiring**

```python
# python-core-mvp/src/cccore/providers/anthropic.py
from __future__ import annotations

from collections.abc import Sequence
from typing import Any

import httpx

from cccore.messages import Message, ToolCall, make_assistant_message
from cccore.models import ProviderTurn, ToolDefinition


ANTHROPIC_API_URL = "https://api.anthropic.com/v1/messages"
ANTHROPIC_VERSION = "2023-06-01"


def build_anthropic_request(
    *,
    system_prompt: str,
    messages: Sequence[Message],
    tools: Sequence[ToolDefinition],
    model: str,
    max_output_tokens: int,
) -> dict[str, Any]:
    body_messages: list[dict[str, Any]] = []
    for message in messages:
        if message.role == "user":
            body_messages.append(
                {"role": "user", "content": [{"type": "text", "text": message.content}]}
            )
        elif message.role == "assistant":
            content: list[dict[str, Any]] = []
            if message.content:
                content.append({"type": "text", "text": message.content})
            for tool_call in message.tool_calls:
                content.append(
                    {
                        "type": "tool_use",
                        "id": tool_call.id,
                        "name": tool_call.name,
                        "input": tool_call.arguments,
                    }
                )
            body_messages.append({"role": "assistant", "content": content})
        elif message.role == "tool":
            body_messages.append(
                {
                    "role": "user",
                    "content": [
                        {
                            "type": "tool_result",
                            "tool_use_id": message.tool_call_id,
                            "content": message.content,
                            "is_error": message.is_error,
                        }
                    ],
                }
            )

    return {
        "model": model,
        "system": system_prompt,
        "max_tokens": max_output_tokens,
        "messages": body_messages,
        "tools": [
            {
                "name": tool.name,
                "description": tool.description,
                "input_schema": tool.input_schema,
            }
            for tool in tools
        ],
    }


def parse_anthropic_response(payload: dict[str, Any]) -> ProviderTurn:
    text_parts: list[str] = []
    tool_calls: list[ToolCall] = []

    for block in payload.get("content", []):
        if block["type"] == "text":
            text_parts.append(block["text"])
        elif block["type"] == "tool_use":
            tool_calls.append(
                ToolCall(
                    id=block["id"],
                    name=block["name"],
                    arguments=block["input"],
                )
            )

    return ProviderTurn(
        assistant_message=make_assistant_message(
            "\n".join(text_parts).strip(),
            tool_calls=tool_calls,
        ),
        stop_reason=payload.get("stop_reason"),
    )


class AnthropicProvider:
    name = "anthropic"

    def __init__(self, *, api_key: str, max_output_tokens: int) -> None:
        self.api_key = api_key
        self.max_output_tokens = max_output_tokens

    def complete(
        self,
        *,
        system_prompt: str,
        messages: Sequence[Message],
        tools: Sequence[ToolDefinition],
        model: str | None,
    ) -> ProviderTurn:
        if not model:
            raise ValueError("Anthropic provider requires a model name")

        body = build_anthropic_request(
            system_prompt=system_prompt,
            messages=messages,
            tools=tools,
            model=model,
            max_output_tokens=self.max_output_tokens,
        )
        response = httpx.post(
            ANTHROPIC_API_URL,
            headers={
                "x-api-key": self.api_key,
                "anthropic-version": ANTHROPIC_VERSION,
                "content-type": "application/json",
            },
            json=body,
            timeout=60.0,
        )
        response.raise_for_status()
        return parse_anthropic_response(response.json())
```

```python
# python-core-mvp/src/cccore/cli.py
from __future__ import annotations

import argparse
from pathlib import Path
from typing import Sequence

from cccore.commands import build_default_commands
from cccore.config import RuntimeConfig
from cccore.providers.anthropic import AnthropicProvider
from cccore.query_engine import QueryEngine
from cccore.tools import build_default_tools


def build_provider(config: RuntimeConfig):
    if config.provider_name != "anthropic":
        raise ValueError(f"Unsupported provider: {config.provider_name}")
    if not config.anthropic_api_key:
        raise ValueError("ANTHROPIC_API_KEY is required for the anthropic provider")
    return AnthropicProvider(
        api_key=config.anthropic_api_key,
        max_output_tokens=config.max_output_tokens,
    )


def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(prog="cccore")
    subparsers = parser.add_subparsers(dest="command", required=True)

    commands = {command.name: command for command in build_default_commands()}

    run_parser = subparsers.add_parser("run", help=commands["run"].description)
    run_parser.add_argument("prompt")
    run_parser.add_argument("--cwd", default=str(Path.cwd()))
    run_parser.add_argument("--provider", default="anthropic")
    run_parser.add_argument("--model", default=None)
    run_parser.add_argument(
        "--permission-mode",
        default="default",
        choices=["default", "accept_edits", "bypass_permissions"],
    )

    subparsers.add_parser("tools", help=commands["tools"].description)
    subparsers.add_parser("transcript", help=commands["transcript"].description)
    return parser


def main(argv: Sequence[str] | None = None) -> int:
    parser = build_parser()
    args = parser.parse_args(argv)

    if args.command == "tools":
        for tool in build_default_tools():
            print(tool.name)
        return 0

    if args.command == "transcript":
        config = RuntimeConfig.from_env()
        print(config.transcript_dir)
        return 0

    config = RuntimeConfig.from_env(cwd=Path(args.cwd))
    config.provider_name = args.provider
    config.model = args.model or config.model
    config.permission_mode = args.permission_mode

    provider = build_provider(config)
    engine = QueryEngine(
        config=config,
        provider=provider,
        tools=build_default_tools(),
    )
    print(engine.submit(args.prompt))
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

~~~~markdown
# python-core-mvp/README.md
# cccore

`cccore` is a Python MVP that translates the core headless execution architecture
from Claude Code into a smaller, testable implementation.

## Status

This package intentionally targets the headless core engine only:

- CLI bootstrap
- command and tool registries
- query engine
- multi-turn query loop
- local tool execution
- transcript persistence

## Install

```bash
cd /tmp/cc/cc2cc/python-core-mvp
python -m venv .venv
. .venv/bin/activate
pip install -e .[dev]
```

## Run

```bash
export ANTHROPIC_API_KEY=your-key
export CCCORE_MODEL=your-model
cccore run "Summarize the repository"
```

## CLI Commands

- `cccore run "<prompt>"` executes a prompt through the query engine
- `cccore tools` lists the built-in local tools
- `cccore transcript` prints the transcript directory

## Development

```bash
cd /tmp/cc/cc2cc/python-core-mvp
. .venv/bin/activate
pytest -q
```
~~~~

- [ ] **Step 4: Run the provider and CLI tests to verify they pass**

Run:

```bash
cd /tmp/cc/cc2cc/python-core-mvp
pytest tests/test_anthropic_provider.py tests/test_cli.py -q
```

Expected:

```text
3 passed
```

- [ ] **Step 5: Run the full test suite and commit the MVP**

Run:

```bash
cd /tmp/cc/cc2cc/python-core-mvp
pytest -q
```

Expected:

```text
20 passed
```

Commit:

```bash
cd /tmp/cc/cc2cc
git add python-core-mvp/README.md python-core-mvp/src/cccore/providers/anthropic.py python-core-mvp/src/cccore/cli.py python-core-mvp/tests/test_anthropic_provider.py python-core-mvp/tests/test_cli.py
git commit -m "feat(cli): add anthropic provider and headless entrypoint" -m "Why:
- the MVP needs a real provider implementation and a usable CLI to complete the headless flow
- this step turns the translated engine into an executable package

What:
- add Anthropic request and response mapping
- add the Anthropic provider implementation
- add the headless CLI with run, tools, and transcript commands
- update the README with installation and usage instructions
- add provider mapping and CLI tests

How:
- keep provider mapping explicit through helper functions
- build the CLI on argparse and the existing registries
- wire QueryEngine into the run command without introducing UI-specific state

Verification:
- cd python-core-mvp && pytest tests/test_anthropic_provider.py tests/test_cli.py -q
- cd python-core-mvp && pytest -q"
```

## Self-Review

Spec coverage checked against [2026-03-31-python-core-mvp-design.md](/tmp/cc/cc2cc/docs/superpowers/specs/2026-03-31-python-core-mvp-design.md):

- project bootstrap and isolated package layout: covered by Task 1
- shared message-driven runtime model: covered by Task 2
- command and tool registries: covered by Task 3
- minimal permission model: covered by Task 3
- four core local tools: covered by Task 4
- multi-turn query loop: covered by Task 5
- session state and transcript persistence: covered by Task 6
- Anthropic provider abstraction and headless CLI: covered by Task 7
- testing strategy and end-to-end CLI behavior: covered by Tasks 4 through 7

Placeholder scan:

- no deferred implementation markers remain
- every task includes exact paths, commands, and code

Type consistency:

- `ToolDefinition`, `ToolContext`, `ProviderTurn`, `QueryLoopResult`, and `RuntimeConfig` are defined before later tasks use them
- `file_edit` uses `path`, `old`, and `new` consistently across tests and implementation
- `QueryEngine.submit()` returns a final string consistently across engine and CLI tasks
