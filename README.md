# JHarness Tools

**English** | [简体中文](#简体中文)

`jharness-tools` is the independently versioned, optional distribution for curated,
ready-to-use JHarness tool implementations.

The package targets Python 3.11 and newer and is fully typed. Its filesystem preset
family provides five workspace-scoped tools with model-visible names compatible with
common coding agents:

| Python type | Tool name | Purpose |
| --- | --- | --- |
| `ReadTool` | `Read` | Read a bounded range from one UTF-8 text file. |
| `GlobTool` | `Glob` | Find files with a relative glob pattern. |
| `GrepTool` | `Grep` | Search UTF-8 text files with a regular expression. |
| `EditTool` | `Edit` | CAS-guarded exact text replacement in an existing file. |
| `WriteTool` | `Write` | CAS-guarded creation or complete replacement of a file. |

## Install

```bash
uv add jharness-tools
```

Installing the package does not discover, register, or activate tools. Applications
explicitly construct the presets they choose and supply them through JHarness kernel
contracts. Presets remain replaceable by application-defined implementations.

```python
from pathlib import Path

from jharness.tools import EditTool, GlobTool, GrepTool, ReadTool, WriteTool

root = Path.cwd()
presets = (
    ReadTool(root),
    GlobTool(root),
    GrepTool(root),
    EditTool(root),
    WriteTool(root),
)
```

Applications using `jharness-toolkit` can pass that tuple directly to `ToolRegistry`.
The tools distribution uses the public `jharness-kernel` contracts and the `regex`
engine for time-bounded searches; it does not require a particular registry
implementation.

## Contracts

All five tools accept relative or absolute paths only inside the Host-configured
workspace root. Open file and directory handles are revalidated. Read-only traversal
does not follow symbolic-link directories or Windows reparse points; mutation paths
reject symbolic links and reparse points in every path component. The root, output
bounds, pattern bounds, search work budgets, timeout, and excluded directory names are
constructor configuration and are never model-controlled.

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

`requires_approval` is a public risk fact, not an enforcement switch. A Host that wants
write confirmation must configure an approval policy before registering mutation tools.
Without a policy, Runtime can execute them normally.

Normal path, encoding, pattern, and I/O problems return stable model-visible failures.
The default search exclusions cover VCS metadata, virtual environments, dependency
trees, bytecode, and common tool caches. Hosts can replace the exclusion set explicitly.
Searches default to a 10-second, 100,000-entry work budget; exceeding a Host-configured
time, entry, or byte budget is an explicit failure rather than an unbounded scan.

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

本包支持 Python 3.11 及以上版本，并提供完整类型标注。文件系统预设集提供五个限定
在工作区内的工具：

| Python 类型 | 工具名 | 用途 |
| --- | --- | --- |
| `ReadTool` | `Read` | 分段读取 UTF-8 文本文件。 |
| `GlobTool` | `Glob` | 使用相对 Glob 模式查找文件。 |
| `GrepTool` | `Grep` | 使用正则表达式搜索 UTF-8 文本。 |
| `EditTool` | `Edit` | 使用 CAS 摘要保护，精确替换现有文件中的文本。 |
| `WriteTool` | `Write` | 使用 CAS 摘要保护，创建或完整覆写文件。 |

### 安装

```bash
uv add jharness-tools
```

安装本包不会自动发现、注册或启用工具。应用需要显式构造选中的预设工具，并通过
JHarness Kernel 契约接入；所有预设实现都可以被应用自定义实现替换。

```python
from pathlib import Path

from jharness.tools import EditTool, GlobTool, GrepTool, ReadTool, WriteTool

root = Path.cwd()
presets = (
    ReadTool(root),
    GlobTool(root),
    GrepTool(root),
    EditTool(root),
    WriteTool(root),
)
```

`Read`、`Glob`、`Grep` 声明为 `parallel + read_only + idempotent`，文件系统风险为
只读且非破坏性。`Edit`、`Write` 声明为 `serial + non-read-only + non-idempotent`，
文件系统风险为写入、破坏性且需要审批。`requires_approval` 只是公开风险事实，不会
自动阻止执行；需要写入确认的 Host 必须显式配置审批策略。

工作区根目录、输出上限、模式上限、搜索预算、超时和排除目录均由 Host 在构造时
配置，不会暴露为模型参数。遍历和读取都会基于已打开的句柄复核真实路径，不跟随
符号链接目录或 Windows 重解析点；写入路径会拒绝任意路径组件中的符号链接和重解析点。

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
