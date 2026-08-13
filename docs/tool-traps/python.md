# Python 陷阱

## 4. Python subprocess 调用外部命令时的路径与编码问题

**现象**：Python 调用外部命令（如 `git`）时报路径错误或编码异常。

**根因**：反斜杠路径在字符串中被转义破坏。Windows GBK 编码无法解码外部命令输出的 UTF-8 中文。

**正确做法**：
1. 尽量在指定路径下运行 Bash 调用外部命令，不经过 Python subprocess。
2. Python 调用 subprocess 时指定 `encoding='utf-8'`，errors='replace'。
3. 路径名 `raw_string_r'D:\path'` 或正斜杠。
4. 复杂批量操作可写临时 `.py` 文件执行。

---

## 5. Python 脚本向终端输出中文出现乱码

**现象**：Python 脚本输出中文到终端时出现乱码。

**根因**：某些平台终端默认编码与 Python stdout 默认编码不匹配（如 Windows GBK）。尝试通过 `sys.stdout.encoding` 修改也可能因 `readonly` 属性失败。

**正确做法**：
1. 转换类任务的验证不依赖终端输出，直接写文件后读取确认。
2. 需要中文输出时用 `chcp 65001` 切换终端编码页。

---

## 6. Python 内联脚本中 JSON null 误写

**现象**：用 `python -c "..."` 构造 JSON 字符串时报 `NameError: name 'null' is not defined`。

**根因**：JSON 的 `null` 在 Python 中是 `None`，`true` 和 `false` 对应 `True` 和 `False`。手写 dict 字面量时容易直接写 `null`。

**正确做法**：
1. Python dict 中用 `None`、`True`、`False`，通过 `json.dumps()` 序列化为 `null`、`true`、`false`。
2. 若要直接写 JSON 字面量，用字符串拼接而非 Python dict。
3. 构造完先 `echo "{\"msg\":\"JSON\"}"` 验证再传入下游命令。

---