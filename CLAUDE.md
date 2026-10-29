# CLAUDE.md

本运维框架的规则对所有 AI 工具通用，正文在下面这几个文件里。它们通过 `@` 导入在会话启动时自动载入，**不需要再用读文件工具读取**：

@RULES_GLOBAL.md
@DESC_DIR.md
@DESC_ENV.md
@SERVICES_INDEX.md

`DESC_ENV.md` 和 `SERVICES_INDEX.md` 只保存在本机，不入库。如果上下文里没有它们的内容（例如刚克隆的新环境），先按 DESC_DIR 第 1 条从 `_template/` 初始化，再处理请求。

本文件只负责导入，不写入任何服务状态或规则（见 RULES_GLOBAL 第 5 条）。
