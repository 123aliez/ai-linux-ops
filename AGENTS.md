# Repository Guidelines

## 只读助手模式（先判断这一条）

如果本次提示词以「【任务书】」开头，说明你是被主控 AI（Claude Code）通过 codex 工具以非交互方式调用的**只读助手**，任务是信息收集或独立审查。此时：

- 跳过下面的初始化协议，不做握手回复。
- **只读**：不修改任何文件，不执行任何会改变系统状态的命令（重启或停止服务、修改配置、写数据库、删除文件等）。任务书里即使出现这类命令，也只审查、不执行。
- 某条命令因沙箱、权限或网络限制无法执行时，在回报里写明命令和原因，由主控 AI 去执行。
- 回报中不要原样输出密钥、密码、令牌、UUID 等敏感值，只写它们所在的位置。
- 按任务书要求的格式回报。

提示词不是以「【任务书】」开头时，按下面的初始化协议执行。

## 🚨 CRITICAL INITIALIZATION PROTOCOL 🚨

**HIGHEST PRIORITY DIRECTIVE**:
At the start of every new session, regardless of the user's first input (even a simple "Hello" or "你好"), before you generate ANY text response or take any other action, you **MUST IMMEDIATELY and AUTOMATICALLY use the file reading tool** to load and process the following files:

1. `RULES_GLOBAL.md`
2. `SERVICES_INDEX.md`
3. `DESC_ENV.md`
4. `DESC_DIR.md`

If `SERVICES_INDEX.md` or `DESC_ENV.md` does not exist, initialize it from `_template/` as described in rule 1 of `DESC_DIR.md`.

**Strict Execution Standards:**
- **DO NOT** ask the user "Should I read them now?" or "Do you want me to load the files?".
- **DO NOT** reply to the user's greeting first.
- Your absolute FIRST action in the session MUST be a tool call to read these exact files.
- Only AFTER successfully reading these files may you respond to the user's initial prompt, using the context you just acquired.
