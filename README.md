# JHarness Tools

**English** | [简体中文](#简体中文)

`jharness-tools` is the independently versioned, optional distribution for curated,
ready-to-use JHarness tool implementations.

The package targets Python 3.11 and newer and is fully typed. This initial scaffold
intentionally exports no preset tools and has no runtime dependencies. Tool families,
schemas, execution facts, and risk facts will be added only after their public
contracts are accepted.

## Install

```bash
uv add jharness-tools
```

```python
import jharness.tools
```

Installing the package does not discover, register, or activate tools. Applications
explicitly construct the presets they choose and supply them through JHarness kernel
contracts. Presets remain replaceable by application-defined implementations.

The dependency direction is one-way: this project may depend on `jharness-kernel` and
`jharness-toolkit` when implementations actually import them. Kernel, toolkit, and
providers never depend on `jharness-tools`, and applications remain fully constructible
with the kernel alone.

## Related Projects

- [JHarness specification](https://github.com/Ezio2000/jharness)
- [JHarness Python](https://github.com/Ezio2000/jharness-python)
- [Issue tracker](https://github.com/Ezio2000/jharness-tools/issues)

## License

MIT

## 简体中文

[English](#jharness-tools) | **简体中文**

`jharness-tools` 是独立版本管理、按需安装的 JHarness 通用预设工具发行包。

本包支持 Python 3.11 及以上版本，并提供完整类型标注。当前提交仅建立独立项目
脚手架，有意不导出任何预设工具，也没有运行时依赖。工具族、Schema、Execution
事实和 Risk 事实将在公共契约确认后再加入。

### 安装

```bash
uv add jharness-tools
```

```python
import jharness.tools
```

安装本包不会自动发现、注册或启用工具。应用需要显式构造选中的预设工具，并通过
JHarness Kernel 契约接入；所有预设实现都可以被应用自定义实现替换。

依赖方向保持单向：未来实现产生真实导入时，本项目可以依赖 `jharness-kernel` 和
`jharness-toolkit`；Kernel、Toolkit 和 Providers 永远不会依赖 `jharness-tools`。
用户仍然可以只使用 Kernel 构建完整应用。

### 相关项目

- [JHarness 规范](https://github.com/Ezio2000/jharness)
- [JHarness Python](https://github.com/Ezio2000/jharness-python)
- [问题跟踪](https://github.com/Ezio2000/jharness-tools/issues)

### 许可证

MIT
