# 交接包结构与填写规则

编制交接包或核对接手内容时读取本文件。交接包是跨模型协作的唯一权威输入，不是聊天摘要的替代品。同一文件承担三截职责：路由字段、可决策 briefing、可选 READ_LIST。不另建第三个真源。

## 路径与写入规则

- 路径：`.handoff/<task-id>.md`。`task-id` 用稳定的任务标识（模块名 + 意图），不用日期或流水号。
- 写入时机：发出 `ROUTE_REQUEST` **之前**。
- 已存在同名文件时**追加**新一轮，用 `## Round N` 分隔，不覆盖历史轮次。失败与被推翻的判断保留可见。
- 任务同时受长期治理 Skill 管理时，本文件只**引用** `TASK_PROGRESS.md` / `PROJECT_STATE.md` 的对应条目，不复制其内容，不成为竞争性真源。
- 向上决策的包必须带可决策 briefing。只有路径没有原文的包不得交给 ARCHITECT。
- 向下采集的包必须带封闭 READ_LIST。采集层不得扩大清单，也不得把架构结论写进本轮。

## 证据标签（硬要求）

每条内容必须带且只带一个标签：

| 标签 | 含义 | 要求 |
| --- | --- | --- |
| `FACT` | 已直接观察到的事实 | 必须可被接手模型在 briefing 内核对，不要求打开源码 |
| `EVIDENCE` | 支撑某条 FACT 的具体位置与原文 | **必须带 `文件:行`，并内联足够决策的原文（crux，默认 ≤8 行）** |
| `INFERENCE` | 由事实推出的判断 | 必须写出依据的 FACT 编号 |
| `HYPOTHESIS` | 尚无证据的可能解释 | 必须写出可证伪条件 |
| `QUESTION` | 未决问题 | 必须写出谁有权回答 |
| `DECISION` | 已作出的决定 | 必须写出作出者的能力层级 |

无标签内容一律视为 `HYPOTHESIS`。接手模型不得据 `INFERENCE`、`HYPOTHESIS` 作最终决策。

不要写成 "X 应该重构成 Domain Service"。要写成：`FACT` 当前逻辑做什么 / `EVIDENCE` 文件:行 + 原文 / `HYPOTHESIS` 可能职责过多，若 A 与 B 无共同变更原因则不成立 / `QUESTION` 是否应拆分，需 ARCHITECT 判断。

只有 `文件:行`、没有原文的 `EVIDENCE` 等于邀请决策层重读，视为不可决策。

## 字段

五组，每组解决一个具体失效。字段不适用时写 `N/A`，不删除标题。

### 1. 身份与路由 — 防止接手模型不知道自己有没有决策权

```text
task-id:
current model / tier:
previous model / tier:
target model / tier:
current phase:
work mode:                 （COLLECT 或 DECIDE）
decision authority:        （本轮谁有权作最终决策）
```

### 2. 任务与边界 — 防止接手模型扩大范围或重做已完成工作

```text
task goal:                 （一句话，含验收标准）
non-goals:
completed work:            （已完成且无需重做的部分）
restrictions:              （用户明确的限制，逐字保留"只""禁止""不要"等词）
required next action:      （接手模型必须完成的唯一动作）
```

`required next action` 在 COLLECT 轮只写采集或填 READ_LIST，并写明「不得下结论」。在 DECIDE 轮只写要作出的那一项判断。

### 3. 可决策 briefing — 防止决策层顺着路径把仓库再读一遍

这是决策层的工作记忆。聊天历史和「相关文件列表」都不能替代它。

```text
briefing status:           （decision-complete / insufficient / not-started）
FACT-1:
  EVIDENCE: <文件:行>
  CRUX:
    <足以决策的原文，默认 ≤8 行；被截断时标明 +N 并列入 READ_LIST>
FACT-2:
  EVIDENCE: <文件:行>
  CRUX:
    <原文>
INFERENCE-1:               （依据 FACT-x）
HYPOTHESIS-1:              （证伪条件）
DECISION-1:                （作出者层级）
relevant files:            （索引，不能代替 CRUX）
relevant symbols:
dependencies:
```

`briefing status = decision-complete` 当且仅当：决策所需每条 `FACT` 都有带 crux 的 `EVIDENCE`；阻塞决策的问题已列为 `QUESTION`；冲突已显式写出；一个只读本文件、不打开源码的决策层模型能完成 `required next action`。

`relevant files` / `relevant symbols` 只做索引。没有对应 crux 的路径必须进入 READ_LIST，或把 briefing 标为 `insufficient`。禁止用「详见某文件」冒充已采集。

### 4. 未决与风险 — 防止决策门被静默跳过

```text
QUESTION-1:                （谁有权回答）
risks:
evidence conflicts:        （不同来源或不同模型的结论冲突，本身即升级信号）
confidence:                （对当前结论的置信度，及不足之处）
```

### 5. READ_LIST — 防止决策层自己补读，也防止采集层漫游

无缺口时整组写 `N/A`。有缺口时必须封闭、一次批完。

```text
READ_LIST status:          （open / filled / N/A）
collector target:          （默认 Gemini 3.7 Flash；其它 SCOUT 仅用户点名）
R1:
  path:
  span or symbol:
  unblocks:                （补上后能回答哪个决策问题）
  extract:                 （signature / crux<=8 / contract-field / caller-list / graft-ask / iwe-retrieve）
  result:                  （采集后填写：FACT + EVIDENCE + CRUX，或 NOT_FOUND）
```

选择采集层：默认 **Gemini 3.7 Flash**（海量文档/代码的阅读器，填 READ_LIST）。Gemini **本轮**不可用或已失败 → 用户点名后备 SCOUT，或等 Gemini 恢复后再次调用；不写进默认改派。禁止对同一采集目标连续重试 Gemini。禁止把 Sol、Sonnet 5、Grok 4.6、Opus 5 写成 `collector target`。

采集层只填 `result`，不新增条目，不改 `unblocks`，不作 `DECISION`。某条 `NOT_FOUND` 时如实写；指定路径不存在时可记录实际读到的邻近真实路径，仍不算新增条目。由决策层决定是否另开一轮清单，采集层不得改去通读仓库。代码类条目默认 Graft，理论语义默认 IWE，见 `references/collect-sources.md`。

## 编制时的自检

写完后逐项确认，任一不通过就补齐再发切换请求：

1. 每条 `EVIDENCE` 都有 `文件:行`，行号来自实际读取而非记忆，并且内联了足以决策的 crux。
2. 没有把本层级禁止作出的判断伪装成 `INFERENCE` 或"建议"。
3. `required next action` 只有一项，且接手模型读完本文件就能开始，不需要追问。
4. `restrictions` 中用户的原始限定词未被概括掉。
5. 一个只看本文件、没有任何聊天历史的模型能独立开始工作。第 5 条不通过时，其余都不算通过。
6. 若目标是 DECIDE：只读本文件就能完成那一项判断。否则把 `briefing status` 标为 `insufficient`，并给出封闭 READ_LIST——不要把指针清单交上去。
7. 若目标是 COLLECT：READ_LIST 每条都有 path 或可定位的 symbol、`unblocks`、`extract`；没有「再看看相关代码」这类开放范围。
