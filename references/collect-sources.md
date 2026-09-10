# 采集取证源

COLLECT 写 READ_LIST 或填 `result` 时读取。DECIDE 禁止使用本文件去打开仓库。

## 谁读

- **默认阅读器：Gemini 3.7 Flash。** 海量文档和代码由它按封闭 READ_LIST 抽 crux，压成交接包。单轮失败不等于本任务禁读 Gemini。
- **其它 SCOUT（含 Terra）仅用户点名。** 不写进默认改派。点名接手后若剩余仍是海量阅读（多数未填条目需通读长文件，且 Graft/IWE 给不出 crux）→ 停止自读，把同一封闭 READ_LIST 交回 Gemini。
- ANALYST（Sol / Sonnet 5 / Grok 4.6）与 ARCHITECT（Opus 5）不读本文件去打开仓库。

## 顺序

先走全局搜索梯子：代码 → Graft MCP；理论/设计决策 → IWE MCP；未命中、未索引或 MCP 不可用再自己精确搜。本文件只补充交接采集的例外。

1. 代码、定义、调用方、影响面 → Graft MCP（穷举用 `graft_find_all` / CLI `graft grep`，不要用 ranked ask 冒充完整列表）
2. 理论、规则语义、案例关系、设计决策 → IWE MCP；导航页不算证据
3. 仍缺精确行号，或 Graft 标明 `+N more lines` → 只读该 span
4. 无索引文件（含多数 Markdown）→ 直接读文件。对 Markdown 跑 skeleton 得到「no definitions」不是错误
5. 路径不存在 → `NOT_FOUND`，可注明实际发现的邻近真实路径，不新增 READ_LIST 条目

MCP 不可用或返回未知工具时立即改 CLI，不要对同一 MCP 名重试。采集结束可在 briefing 记累计 token 节省；不因此扩大清单。

## Git 合同字段

允许的只读命令：`git status --short`、`git branch -vv`、`git log --oneline -20`、`git diff --stat`、`git stash list`、`git worktree list`、`git rev-parse HEAD`。

- 必须记录：分支、tracking、HEAD、脏文件路径、stash、worktree。
- 命令无输出 ≠ 工作树干净。把 `status`、`diff --stat`、`stash list` 拆成独立命令再取一次。
- Windows `cmd.exe` 下不要用 Unix `&&` 把多条 git 命令串成一条还假设都能看到输出。
- ANALYST 向上编包前可再跑一轮这些命令，覆盖采集时的 HEAD 快照。

## READ_LIST extract 取值

```text
signature        Graft skeleton 或 spans
crux<=8          默认；截断标 +N more lines
contract-field   git / 授权文档的状态、白名单、VERIFY
caller-list      graft callers
graft-ask        ranked nodes + covers + crux
iwe-retrieve     命中文档 key + 限额内正文
NOT_FOUND        路径或符号不存在
```

采集层只填 `result`，格式 `FACT` + `EVIDENCE: 文件:行` + `CRUX`。禁止扩大清单、禁止改 `unblocks`、禁止 `DECISION`。
