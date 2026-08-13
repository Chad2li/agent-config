# Bash 执行陷阱

## 2. python 与 python3 命令差异

**现象**：使用 `python3` 报错无此命令。

**根因**：某些平台（如 Windows）python 命令为 `python`，而非 `python3`。

**正确做法**：
1. 执行前先确认平台：`python --version`
2. Windows 平台用 `python`，Linux/macos 用 `python3`
3. 不确定的场景先用 `python3` 尝试，失败再检查

---

## 3. 临时文件路径跨平台不一致

**现象**：`curl -o /tmp/file.json` 或 `open('/tmp/file.json')` 写入成功但读取时报 `FileNotFoundError`。

**根因**：Windows 环境下 `/tmp/` 等 Linux 路径映射不稳定，不同工具对这些路径的解析可能不一致。

**正确做法**：
1. 临时文件用 `$PWD`（当前工作目录）或 `$HOME/.claude/cache` 等明确的跨平台可访问路径
2. 避免在 Windows 环境使用 `/tmp/` 或 `/var/tmp` 等 Linux 专用路径
3. 跨工具传递文件路径时，统一用 `$HOME` 或项目目录下的绝对路径
4. **失败后立即识别**：任何临时路径报 `FileNotFoundError`，第一反应检查路径是否跨平台兼容，立刻改用 `$PWD` 或 `$HOME/.claude/cache`，不要重试同一路径。

---