# JHarness Tools

**English** | [简体中文](#简体中文)

`jharness-tools` is the independently versioned, optional distribution for curated,
ready-to-use JHarness tool implementations.

The package targets Python 3.11 and newer and is fully typed. It provides five
workspace-scoped filesystem tools, one bounded Bash runner, one durable structured
question tool, and four Host-mediated Child Agent lifecycle tools:

| Python type | Tool name | Purpose |
| --- | --- | --- |
| `ReadTool` | `Read` | Read a bounded range from one UTF-8 text file. |
| `GlobTool` | `Glob` | Find files with a relative glob pattern. |
| `GrepTool` | `Grep` | Search UTF-8 text files with a regular expression. |
| `EditTool` | `Edit` | CAS-guarded exact text replacement in an existing file. |
| `WriteTool` | `Write` | CAS-guarded creation or complete replacement of a file. |
| `BashTool` | `Bash` | Run one bounded, non-interactive foreground Bash command. |
| `AskQuestionTool` | `AskQuestion` | Suspend durably for structured user input. |
| `AgentTool` | `Agent` | Start a Child Agent in foreground or background mode. |
| `AgentGetTool` | `AgentGet` | Read the latest Agent snapshot without blocking. |
| `AgentWaitTool` | `AgentWait` | Suspend durably until a background Agent finishes. |
| `AgentCancelTool` | `AgentCancel` | Request idempotent Agent cancellation. |

## Install

```bash
uv add jharness-tools
```

Installing the package does not discover, register, or activate tools. Applications
explicitly construct the presets they choose and supply them through JHarness kernel
contracts. Presets remain replaceable by application-defined implementations.

```python
from pathlib import Path

from jharness.tools import (
    AskQuestionTool,
    BashTool,
    EditTool,
    GlobTool,
    GrepTool,
    ReadTool,
    WriteTool,
)

root = Path.cwd()
presets = (
    ReadTool(root),
    GlobTool(root),
    GrepTool(root),
    EditTool(root),
    WriteTool(root),
    BashTool(root),
    AskQuestionTool(),
)
```

Applications using `jharness-toolkit` can pass that tuple directly to `ToolRegistry`.
The tools distribution uses the public `jharness-kernel` contracts and the `regex`
engine for time-bounded searches; it does not require a particular registry
implementation.

## Contracts

All five filesystem tools accept relative or absolute paths only inside the
Host-configured workspace root. Open file and directory handles are revalidated.
Read-only traversal does not follow symbolic-link directories or Windows reparse
points; mutation paths reject symbolic links and reparse points in every path
component. The root, output bounds, pattern bounds, search work budgets, timeout, and
excluded directory names are constructor configuration and are never model-controlled.

`Read`, `Glob`, and `Grep` declare:

```text
concurrency = parallel
read_only   = true
idempotent  = true
filesystem = read
destructive = false
```

`Edit` and `Write` conservatively declare:

```text
concurrency = serial
read_only   = false
idempotent  = false
filesystem = write
destructive = true
requires_approval = true
```

`Bash` also executes serially and declares the broad effects that arbitrary commands
may have:

```text
concurrency = serial
read_only   = false
idempotent  = false
filesystem = write
network = unrestricted
subprocess = true
destructive = true
requires_approval = true
```

`AskQuestion` is a serial durable interaction barrier with no operating-system effects:

```text
concurrency = serial
read_only   = true
idempotent  = true
filesystem = none
network = none
subprocess = false
destructive = false
requires_approval = false
```

`Agent` is serial and may delegate any effects permitted by the Host-derived Child
Runtime, so its standard filesystem, network, subprocess, destructive, and approval
risk fields intentionally remain unspecified. `AgentGet` and `AgentWait` are read-only
and idempotent. `AgentCancel` is serial, idempotent, and destructive because it stops
Host-owned work, but has no direct filesystem, network, or subprocess effect.

`requires_approval` is a public risk fact, not an enforcement switch. A Host that wants
confirmation must configure an approval policy before registering mutation or Bash
tools. Without a policy, Runtime can execute them normally.

Normal path, encoding, pattern, and I/O problems return stable model-visible failures.
The default search exclusions cover VCS metadata, virtual environments, dependency
trees, bytecode, and common tool caches. Hosts can replace the exclusion set explicitly.
Searches default to a 10-second, 100,000-entry work budget; exceeding a Host-configured
time, entry, or byte budget is an explicit failure rather than an unbounded scan.

### Agent

The first Agent contract deliberately lets the model choose only the task and execution
mode:

```json
{
  "description": "Inspect authentication",
  "prompt": "Inspect the authentication flow and report concrete security findings.",
  "background": false
}
```

Applications inject an `AgentBackend` into all four tools. The backend owns Agent ids,
authorization, idempotent creation, the queue and store, Child supervision, completion
notifications, and cancellation. It must derive the Child Runtime from the parent
configuration under Host policy. Kernel does not expose Runtime configuration through
`ToolContext`, clone Runtime values, launch work after `ToolAccepted`, or maintain an
Agent registry.

The backend creates a fresh Child Run with `parent_run_id`, `parent_tool_call_id`, and
`run_kind="agent"`; it does not copy the parent conversation history. Model, tools,
approval policy, workspace controls, and bounded remaining limits are inherited or
narrowed by the Host. The first version exposes no model-, tool-, profile-, permission-,
or budget-selection arguments to the model.

```python
from jharness.tools import AgentCancelTool, AgentGetTool, AgentTool, AgentWaitTool

# agent_backend is the application's AgentBackend implementation.
agent_tools = (
    AgentTool(agent_backend),
    AgentGetTool(agent_backend),
    AgentWaitTool(agent_backend),
    AgentCancelTool(agent_backend),
)
```

`background=true` returns `ToolAccepted` with a stable `TaskRef`, after which the parent
continues and can use `AgentGet`, `AgentWait`, or `AgentCancel`. Foreground `Agent` and a
non-terminal `AgentWait` return `ToolWaiting` plus a durable `Suspension`. Hosts extract
the wait and resume the parent with a terminal immutable snapshot:

```python
from jharness.kernel import Runtime, ToolChoice
from jharness.toolkit import ToolRegistry
from jharness.tools.agent import extract_agent_wait, resume_agent

# Foreground Agent and AgentWait must be the only tool call in their assistant turn.
runtime = Runtime(
    model=model,
    tools=ToolRegistry(agent_tools),
    tool_choice=ToolChoice(allow_parallel_tool_calls=False),
)

wait = extract_agent_wait(suspended_checkpoint)
assert wait.agent_id == completed_snapshot.agent_id
invocation = resume_agent(runtime, suspended_checkpoint, completed_snapshot)
```

`AgentBackend.wait_or_get` must atomically return a terminal snapshot or register a
durable waiter so completion cannot be lost between tool execution and checkpoint
commit. `AgentCancel` may return a running snapshot with
`cancellation_requested=true`; only a later `cancelled` snapshot proves the Child
reached a safe Host-defined cancellation point. Kernel 0.1.x has active-tool
`cancel_tool` and durable suspension, but no run-level cancel API or `Cancelled` state.

### AskQuestion

`AskQuestion` accepts a bounded declarative questionnaire. Hosts may enable any subset
of `confirm`, `single_choice`, `multiple_choice`, `text`, `number`, `date`, `scale`, and
`ranking`; the generated input schema exposes only those enabled interaction kinds.
Question ids and option values are stable machine identifiers, separate from their
display text. Extracted requests retain the enabled kinds and all Host-configured size
limits so a UI can render and revalidate the exact durable contract. A `default` is a UI
hint, not an implicit answer; required questions still need an explicit response.
For choice questions, `allow_custom=true` adds one free-form value slot.

```json
{
  "questions": [
    {
      "id": "database",
      "kind": "single_choice",
      "prompt": "Which database should the application use?",
      "options": [
        {"value": "postgres", "label": "PostgreSQL"},
        {"value": "sqlite", "label": "SQLite"}
      ],
      "allow_custom": true
    },
    {
      "id": "retries",
      "kind": "number",
      "prompt": "How many retries are allowed?",
      "integer_only": true,
      "minimum": 0,
      "maximum": 10
    }
  ]
}
```

The tool does not read stdin or retain a live UI callback. It returns `ToolWaiting` with
a stable request id and a Host-only `Suspension`; both survive checkpoint wire
round-trips. The Host extracts the request, displays it using any UI, constructs a
validated `QuestionResponse`, and calls `resume_question`. The answer is committed as a
canonical JSON `Message.external` before model work resumes.

```python
from jharness.kernel import Runtime, ToolChoice
from jharness.toolkit import ToolRegistry
from jharness.tools.interaction import (
    AskQuestionTool,
    QuestionResponse,
    extract_question_request,
    resume_question,
)

# AskQuestion must be the only tool call in its assistant turn with Kernel 0.1.x.
runtime = Runtime(
    model=model,
    tools=ToolRegistry((AskQuestionTool(),)),
    tool_choice=ToolChoice(allow_parallel_tool_calls=False),
)

request = extract_question_request(suspended_checkpoint)
response = QuestionResponse.answered(
    request.request_id,
    "response-1",
    {"database": "postgres", "retries": 3},
)
invocation = resume_question(runtime, suspended_checkpoint, response)
```

`QuestionResponse.cancelled` represents an explicit refusal or dismissal. Merely
closing an application should leave the checkpoint suspended so another process can
resume it. Hosts using Kernel 0.1.x must configure
`ToolChoice(allow_parallel_tool_calls=False)`: `serial` controls execution batching but
does not prevent a model from emitting another tool call in the same turn. Kernel
validates that setting, so a non-conforming provider response fails before any tool
starts.

### Read

```json
{"file_path": "src/app.py", "offset": 1, "limit": 200}
```

Lines are one-based. `Read` accepts strict UTF-8 with an optional UTF-8 BOM, rejects
NUL-containing binary content, defaults to 200 lines, allows at most 2,000 lines per
call, and refuses files larger than 4 MiB by default. Successful results include the
returned range, `next_offset`, `truncated`, and the SHA-256 digest of the complete raw
file. The digest is independent of the requested range and includes any UTF-8 BOM and
original line endings. It appears both in structured output and at the start of the
model-visible text as `SHA-256 (raw file bytes): <digest>`.

### Edit

```json
{
  "file_path": "src/app.py",
  "old_string": "before",
  "new_string": "after",
  "replace_all": false,
  "expected_sha256": "<sha256 returned by Read>"
}
```

`Edit` only changes an existing UTF-8 text file. The digest is required, so a stale
read cannot silently overwrite newer content. `old_string` must be non-empty and must
match exactly once unless `replace_all` is true. Newline sequences are matched in the
same normalized form returned by `Read`, while untouched mixed line endings and an
existing UTF-8 BOM remain byte-for-byte intact. `Edit` never creates files or parent
directories and never returns unbounded file content or diffs.

### Write

```json
{
  "file_path": "src/new.py",
  "content": "print('hello')\n",
  "expected_sha256": null
}
```

Set `expected_sha256` to `null` for create-only behavior; the operation fails if the
path already exists. To replace an existing file completely, pass the SHA-256 returned
by `Read`; the operation fails if the file is missing or changed. Parent directories
must already exist. `Write` emits exact UTF-8 bytes, permits empty files, and rejects
NUL-containing or over-limit content.

Both mutation tools serialize cooperating in-process calls, recheck content and file
identity immediately before commit, write through a same-directory temporary file, and
commit atomically. Create-only commits never overwrite a concurrently created target.
Cross-process overwrite detection is optimistic because portable filesystems do not
offer an atomic content-compare-and-replace operation; Hosts should not allow an
untrusted process to mutate the same workspace concurrently. Atomic replacement keeps
the existing POSIX permission mode, but ownership, ACLs, and extended attributes are
not part of the portable contract.

### Bash

```json
{"command": "uv run pytest -q", "working_directory": "."}
```

`command` is required; `working_directory` defaults to `.` and must resolve to a
directory inside the configured workspace. Each call starts the Host-selected Bash as
`bash --noprofile --norc -c <command>`, closes stdin, and captures stdout and stderr
independently. It does not provide a PTY, interactive input, background-task API, or
persistent shell state between calls. The default Host limits are 32,768 command
characters, 120 seconds, and 128 KiB for each output stream. Even after a stream reaches
its retained-byte limit, it is drained to avoid blocking the child. Results report the
raw bytes observed during the bounded capture and separate flags for byte-limit
truncation or an incomplete stream closed during cleanup.

Successful structured output contains `status`, `exit_code`, `stdout`, `stderr`,
`duration_ms`, `stdout_bytes`, `stderr_bytes`, `stdout_truncated`, and
`stderr_truncated`. A nonzero exit code is still a `ToolSuccess`, so the model can inspect
the command's diagnostics. A timeout is `command_timeout` with `status="timeout"`, a
null exit code, and any bounded partial output captured before cleanup. Other stable
Bash failures are `invalid_command`, `invalid_working_directory`,
`command_spawn_failed`, `command_execution_failed`, and `cancelled`;
working-directory validation can additionally return `path_outside_workspace`,
`path_not_found`, `not_a_directory`, or `filesystem_error`.

The Bash path, environment, limits, and termination grace period are Host constructor
configuration, not model input. With `environment=None`, the child inherits the Host
process environment; pass an explicit mapping to replace it completely when commands
must not receive ambient credentials. On timeout or cancellation, the runner terminates
the managed POSIX process group or Windows Job/fallback tree, reaps the launched Bash,
and gives both output pipes a bounded drain window before closing any stream that is
still inherited elsewhere.

The workspace check applies only to the initial working directory. It is not a sandbox:
the command can change directory, access paths outside the workspace, start subprocesses,
and use any network or other capability allowed by the operating system. Applications
running untrusted commands must provide their own container or OS sandbox,
least-privilege environment, and approval policy. In particular, a process that
deliberately detaches from its POSIX process group can outlive the tool; process-group
cleanup is not a substitute for OS-level containment. Windows Job assignment occurs
immediately after process creation but is not atomic; if assignment is unavailable, the
tree-aware fallback is best-effort and only targets a Bash root still known to be alive.

### Glob

```json
{"pattern": "**/*.py", "path": "src", "limit": 100}
```

Patterns are relative to `path`, use `/` as their separator, cannot contain `..`, and
never choose a new filesystem root. Results contain files only, use workspace-relative
`/`-separated paths, and are sorted deterministically. The default result limit is 100
and the configurable schema maximum defaults to 1,000. Patterns default to at most
4,096 characters and 256 path components.

### Grep

```json
{
  "pattern": "class .*Tool",
  "path": "src",
  "glob": "*.py",
  "output_mode": "content",
  "case_insensitive": false,
  "context": 2,
  "limit": 100
}
```

`output_mode` is one of `files_with_matches` (the default), `content`, or `count`.
Files and content are returned in deterministic path and line order. Binary files,
invalid UTF-8, unreadable files, and files over 4 MiB are skipped and reported through
the structured result counters. Content lines are capped at 2,000 characters by
default and report `line_truncated`. In `content` mode, `limit` counts matching lines;
requested context is attached without consuming that quota. In the other modes it
counts matching files; `count` reports matching lines per file, not match occurrences.
A basename-only `glob`, such as `*.py`, filters files at any depth, while `Glob`
patterns remain anchored to their requested `path`. Invalid regular expressions and
searches exceeding the Host-configured per-line timeout (0.1 seconds by default) return
explicit failures. The default aggregate read budget is 64 MiB.

Stable failure codes include path and filesystem errors, `unsafe_path`, `binary_file`,
`binary_content`, `invalid_utf8`, `file_too_large`, `content_too_large`, `stale_file`,
`old_string_not_found`, `old_string_not_unique`, `no_changes`,
`invalid_glob_pattern`, `invalid_regex`, `regex_timeout`, `search_timeout`,
`search_budget_exceeded`, and `cancelled`.

The dependency direction is one-way: `jharness-tools` depends on the public Kernel API;
Kernel, Toolkit, and Providers never depend on this project.

## Related Projects

- [JHarness specification](https://github.com/Ezio2000/jharness)
- [JHarness Python](https://github.com/Ezio2000/jharness-python)
- [Issue tracker](https://github.com/Ezio2000/jharness-tools/issues)

## License

MIT

## 简体中文

[English](#jharness-tools) | **简体中文**

`jharness-tools` 是独立版本管理、按需安装的 JHarness 通用预设工具发行包。

本包支持 Python 3.11 及以上版本，并提供完整类型标注。它提供五个限定在工作区内的
文件系统工具、一个有界 Bash 执行工具、一个支持持久化暂停的结构化提问工具，以及
四个由 Host 驱动的 Child Agent 生命周期工具：

| Python 类型 | 工具名 | 用途 |
| --- | --- | --- |
| `ReadTool` | `Read` | 分段读取 UTF-8 文本文件。 |
| `GlobTool` | `Glob` | 使用相对 Glob 模式查找文件。 |
| `GrepTool` | `Grep` | 使用正则表达式搜索 UTF-8 文本。 |
| `EditTool` | `Edit` | 使用 CAS 摘要保护，精确替换现有文件中的文本。 |
| `WriteTool` | `Write` | 使用 CAS 摘要保护，创建或完整覆写文件。 |
| `BashTool` | `Bash` | 运行单个有界、非交互式的前台 Bash 命令。 |
| `AskQuestionTool` | `AskQuestion` | 持久化暂停并等待结构化用户输入。 |
| `AgentTool` | `Agent` | 以前台或后台模式启动 Child Agent。 |
| `AgentGetTool` | `AgentGet` | 非阻塞读取最新 Agent 快照。 |
| `AgentWaitTool` | `AgentWait` | 持久化暂停，直到后台 Agent 结束。 |
| `AgentCancelTool` | `AgentCancel` | 幂等请求取消 Agent。 |

### 安装

```bash
uv add jharness-tools
```

安装本包不会自动发现、注册或启用工具。应用需要显式构造选中的预设工具，并通过
JHarness Kernel 契约接入；所有预设实现都可以被应用自定义实现替换。

```python
from pathlib import Path

from jharness.tools import (
    AskQuestionTool,
    BashTool,
    EditTool,
    GlobTool,
    GrepTool,
    ReadTool,
    WriteTool,
)

root = Path.cwd()
presets = (
    ReadTool(root),
    GlobTool(root),
    GrepTool(root),
    EditTool(root),
    WriteTool(root),
    BashTool(root),
    AskQuestionTool(),
)
```

`Read`、`Glob`、`Grep` 声明为 `parallel + read_only + idempotent`，文件系统风险为
只读且非破坏性。`Edit`、`Write` 声明为 `serial + non-read-only + non-idempotent`，
文件系统风险为写入、破坏性且需要审批。`requires_approval` 只是公开风险事实，不会
自动阻止执行；需要写入确认的 Host 必须显式配置审批策略。`Bash` 同样串行且非幂等，
并声明 `filesystem=write`、`network=unrestricted`、`subprocess=true`、
`destructive=true` 和 `requires_approval=true`。

`AskQuestion` 声明为 `serial + read_only + idempotent`，不访问文件系统、网络或子进程，
并明确声明 `destructive=false`、`requires_approval=false`。它是用户交互，不复用副作用
工具的审批语义。

`Agent` 串行执行，并可能委派 Child Runtime 获准执行的任意效果，因此它不会虚构固定的
文件系统、网络、子进程、破坏性或审批风险。`AgentGet`、`AgentWait` 只读且幂等；
`AgentCancel` 串行、幂等且具有终止 Host 工作的破坏性，但不直接访问文件系统、网络或
子进程。

工作区根目录、输出上限、模式上限、搜索预算、超时和排除目录均由 Host 在构造时
配置，不会暴露为模型参数。遍历和读取都会基于已打开的句柄复核真实路径，不跟随
符号链接目录或 Windows 重解析点；写入路径会拒绝任意路径组件中的符号链接和重解析点。

### Agent

第一版只允许模型决定任务内容和执行模式：

```json
{
  "description": "检查认证模块",
  "prompt": "检查认证流程，给出具体安全问题及对应代码位置。",
  "background": false
}
```

应用需要把同一个 `AgentBackend` 注入四个工具。Backend 负责 Agent ID、鉴权、幂等创建、
队列和状态存储、Child 监督、完成通知及取消，并根据 Host 策略从父运行配置派生 Child
Runtime。Kernel 不会通过 `ToolContext` 暴露父 Runtime 配置，不会克隆 Runtime，也不会在
`ToolAccepted` 后自动启动任务或维护 Agent 注册表。

Backend 为 Child 创建新的 Run，并设置 `parent_run_id`、`parent_tool_call_id` 和
`run_kind="agent"`；它不会复制父对话历史。模型、工具、审批策略、工作区控制和剩余预算
由 Host 继承或收窄。第一版不向模型开放模型选择、工具选择、profile、权限或预算参数。

`background=true` 返回带稳定 `TaskRef` 的 `ToolAccepted`，父 Run 随即继续，并可调用
`AgentGet`、`AgentWait` 或 `AgentCancel`。前台 `Agent` 和尚未结束的 `AgentWait` 返回
`ToolWaiting` 与持久化 `Suspension`；Host 使用 `extract_agent_wait` 提取等待请求，再用
终态 `AgentSnapshot` 调用 `resume_agent` 恢复父 Run。

Kernel 0.1.x 下，前台 `Agent` 与 `AgentWait` 必须配置
`ToolChoice(allow_parallel_tool_calls=False)`，保证它们是 assistant turn 中唯一的工具调用。
`AgentBackend.wait_or_get` 必须原子地返回终态或登记持久化 waiter，避免工具执行与
checkpoint 提交之间丢失完成通知。`AgentCancel` 可以先返回
`cancellation_requested=true` 的运行中快照；只有后续 `cancelled` 快照才能证明 Child 已到达
Host 定义的安全取消点。Kernel 当前只有活跃工具取消和持久化暂停，没有 run-level cancel
API 或 `Cancelled` 状态。

### AskQuestion

`AskQuestion` 使用声明式问题数组，支持 `confirm`、`single_choice`、
`multiple_choice`、`text`、`number`、`date`、`scale` 和 `ranking`。Host 可以在构造
工具时只启用自身 UI 真正支持的类型；生成的模型输入 Schema 只会暴露这些类型。问题
`id` 和选项 `value` 是稳定机器标识，与展示文字分离。

提取后的请求会保留已启用类型以及 Host 配置的全部大小上限，UI 因而可以按同一个持久化
契约渲染和复核。`default` 只是 UI 提示，不会自动成为答案；必答题仍需显式提交。
选择题设置 `allow_custom=true` 时会增加一个自由填写值槽位。

工具不会读取 stdin，也不会让 Runtime 常驻等待 UI callback。它会返回带稳定请求 ID 的
`ToolWaiting` 和 Host-only `Suspension`，因此问题可以随 checkpoint 序列化并在另一进程
恢复。Host 使用 `extract_question_request` 提取题目，构造经过类型和范围校验的
`QuestionResponse`，再调用 `resume_question`；答案会以 canonical JSON 的
`Message.external` 先行提交，然后模型才继续执行。

Kernel 0.1.x 下，接入 `AskQuestion` 的 Runtime 必须配置
`ToolChoice(allow_parallel_tool_calls=False)`，保证提问是该 assistant turn 的唯一工具调用。
`serial` 只控制执行批次，不能单独阻止模型同轮返回其他调用。Kernel 会校验这个设置，
因此不遵守约束的 Provider 响应会在任何工具启动前失败。用户只是关闭应用时应保留暂停
checkpoint；明确拒绝或关闭题卡时使用 `QuestionResponse.cancelled` 恢复，让模型重新规划。

`Read` 使用从 1 开始的行号，严格读取 UTF-8/UTF-8 BOM 文本，并拒绝二进制内容和
超限文件。成功结果还会返回完整磁盘原始字节的 SHA-256，不受分页影响，并计入 BOM
与原始换行。该摘要同时出现在结构化结果和模型可见文本首行，格式为
`SHA-256 (raw file bytes): <digest>`。`Edit` 必须携带该摘要，只修改现有文件；`old_string` 默认必须唯一，
`replace_all=true` 时才替换全部匹配。它会保留 BOM 和未触及区域的混合换行。
`Write` 以 `expected_sha256=null` 表示“仅创建”，以 `Read` 返回的摘要表示“条件覆写”；
两种操作都不会隐式创建父目录，且拒绝 NUL 和超限内容。

两个写工具会串行协调进程内调用，在提交前复核摘要和文件身份，通过同目录临时文件
原子提交；仅创建操作不会覆盖并发出现的目标。由于跨平台文件系统没有按内容原子
比较并替换能力，非协作外部进程下的覆写检测属于乐观 CAS，Host 不应让不可信进程
同时修改同一工作区。POSIX 权限位会保留，但所有权、ACL 和扩展属性不属于跨平台契约。

### Bash

```json
{"command": "uv run pytest -q", "working_directory": "."}
```

`command` 为必填项；`working_directory` 默认为 `.`，且必须解析为配置工作区内的目录。
每次调用都会独立启动 Host 选定的 `bash --noprofile --norc -c <command>`，关闭 stdin，
并分别捕获 stdout 和 stderr。它不提供 PTY、交互输入、后台任务 API，也不会跨调用保留
shell 状态。默认限制为 32,768 个命令字符、120 秒，以及 stdout/stderr 各 128 KiB；
达到保留上限后仍会继续排空管道。结果中的字节数是有界捕获期间实际观察到的原始字节；
若超过字节上限，或清理时必须关闭仍未结束的流，对应的截断标志都会设为 `true`。

成功的结构化结果包含 `status`、`exit_code`、`stdout`、`stderr`、`duration_ms`、
`stdout_bytes`、`stderr_bytes`、`stdout_truncated` 和 `stderr_truncated`。非零退出码仍是
`ToolSuccess`，方便模型读取诊断信息。超时返回 `command_timeout`，其中
`status="timeout"`、`exit_code=null`，并保留清理前捕获到的有界部分输出。其他稳定失败码
包括 `invalid_command`、`invalid_working_directory`、`command_spawn_failed`、
`command_execution_failed` 和 `cancelled`；工作目录校验还可能返回
`path_outside_workspace`、`path_not_found`、`not_a_directory` 或 `filesystem_error`。

Bash 路径、环境、各项上限和终止宽限期均由 Host 构造器配置，不是模型参数。
`environment=None` 会继承 Host 进程环境；如需避免向命令暴露环境中的凭据，应传入显式
映射来完整替换环境。超时或取消时，runner 会终止受管的 POSIX 进程组或 Windows
Job/fallback 进程树，回收已启动的 Bash，并给两条输出管道一个有界排空窗口；仍被其他
进程继承的流会被主动关闭。

工作区检查只约束初始工作目录，并不构成沙箱。命令仍可切换目录、访问工作区外路径、
启动子进程，以及使用操作系统允许的网络和其他能力。运行不可信命令的应用必须自行提供
容器或 OS 沙箱、最小权限环境和审批策略。尤其是，主动脱离 POSIX 进程组的进程仍可能
存活；进程组清理不能替代操作系统级隔离。Windows Job 会在进程创建后立即绑定，但该
操作并非原子；若绑定不可用，进程树 fallback 只会处理仍确认存活的 Bash 根进程，属于
尽力清理而非强隔离保证。

`Glob` 只接受相对于搜索目录的模式，返回稳定排序的工作区相对文件路径。
`Grep` 支持 `files_with_matches`、`content` 和 `count` 三种模式，以及文件 Glob 过滤、
大小写忽略和上下文行。`content` 模式的 `limit` 统计匹配行，上下文不占配额；其他
模式统计匹配文件。正则搜索具有 Host 配置的逐行超时（默认 0.1 秒）。所有输出均
有界；搜索还默认限制为 10 秒、100,000 个目录项和 64 MiB 累计读取量。无效正则、
超时、预算耗尽、路径和 I/O 错误不会与“无匹配”混淆。`count` 统计每个文件中的
匹配行数，而不是同一行内的匹配次数。

运行时依赖 `jharness-kernel` 公共契约和提供超时保护的 `regex` 引擎；应用可以按需
使用 `jharness-toolkit` 注册这些工具。依赖方向保持单向，Kernel、Toolkit 和
Providers 永远不会依赖 `jharness-tools`。

### 相关项目

- [JHarness 规范](https://github.com/Ezio2000/jharness)
- [JHarness Python](https://github.com/Ezio2000/jharness-python)
- [问题跟踪](https://github.com/Ezio2000/jharness-tools/issues)

### 许可证

MIT
