---
tags: [工具陷阱, 索引, AI]
---

# 工具陷阱索引

**使用规则**：会话开始时先读本索引。使用对应工具前，加载对应陷阱文件，同会话不重复加载。发现新陷阱时，写入对应分类文件（不存在则新建），并在本索引补一行。

## 工具与陷阱文件对照

| 使用场景 | 加载文件 |
|---|---|
| 用 Write 连续写多个文件 | [[write]] |
| 用 Edit 修改文件 | [[edit]] |
| 用 Bash 执行命令、处理路径 | [[bash]] |
| Python 脚本、`python -c` 内联 | [[python]] |
| 子Agent 审查、批量读文件 | [[subagent]] |

## 陷阱速查

### Write #工具
- [[write]] **连续调用静默失败**：连写 3 个以上文件时只有第一个真正落盘。每次 Write 后用 `ls`/`cat` 验证；批量写分批进行（每批 1-2 个）；不信任返回的 "ok" 就是成功。

### Edit #工具
- [[edit]] **三参数缺一不可**：必须同时提供 `file path`、`old string`、`new string`；`old string` 需与文件逐字符匹配（含缩进）。调用前先 Read 确认内容；匹配失败重新 Read 再试，不盲目重试。

### Bash #工具
- [[bash]] **python 与 python3 差异**：Windows 用 `python`，Linux/macOS 用 `python3`；不确定先 `python3 --version` 确认。
- [[bash]] **`/tmp/` 路径不稳定**：Windows 下 `/tmp/` 映射不可靠，临时文件用 `$PWD` 或 `$HOME/.claude/cache`；任何临时路径报 `FileNotFoundError`，第一反应检查跨平台兼容性，勿重试同路径。

### Python #工具
- [[python]] **subprocess 路径与编码**：优先 Bash 直调外部命令；确需 subprocess 时指定 `encoding='utf-8', errors='replace'`；路径用 raw string 或正斜杠。
- [[python]] **中文输出乱码**：转换类任务的验证不依赖终端输出，写文件后读取；需要中文输出时 `chcp 65001`。
- [[python]] **内联脚本 JSON 常量**：dict 字面量用 `None/True/False`，`json.dumps()` 序列化为 `null/true/false`；构造完先 `echo` 验证再传下游。

### 子Agent #审查
- [[subagent]] **报告行号不可信**：Critical/Major 问题用 `grep -n` 或 `git blame` 手动确认行号；prompt 写明"基于 HEAD 当前工作树"；单次批量审查不超过 5-10 个文件；涉及事务/oom/安全的问题必须定位原始代码再下结论。

## 更新约定

发现新陷阱时按此流程更新：
1. 写入对应分类文件（如 `bash.md`），追加新的陷阱小节
2. 在本索引的对应工具分类下补一行速查
3. 分类文件不存在时，新建并补进"工具与陷阱文件对照"表
