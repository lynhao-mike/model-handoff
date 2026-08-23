# 交接包结构与填写规则

编制交接包或核对接手内容时读取本文件。交接包是跨模型协作的唯一权威输入，不是聊天摘要的替代品。

## 路径与写入规则

- 路径：`.handoff/<task-id>.md`。`task-id` 用稳定的任务标识（模块名 + 意图），不用日期或流水号。
- 写入时机：发出 `ROUTE_REQUEST` **之前**。
- 已存在同名文件时**追加**新一轮，用 `## Round N` 分隔，不覆盖历史轮次。失败与被推翻的判断保留可见。
- 任务同时受长期治理 Skill 管理时，本文件只**引用** `TASK_PROGRESS.md` / `PROJECT_STATE.md` 的对应条目，不复制其内容，不成为竞争性真源。

## 证据标签（硬要求）

每条内容必须带且只带一个标签：

| 标签 | 含义 | 要求 |
| --- | --- | --- |
| `FACT` | 已直接观察到的事实 | 必须可被接手模型独立复核 |
| `EVIDENCE` | 支撑某条 FACT 的具体位置 | **必须带 `文件:行`**，多处用逗号分隔 |
| `INFERENCE` | 由事实推出的判断 | 必须写出依据的 FACT 编号 |
| `HYPOTHESIS` | 尚无证据的可能解释 | 必须写出可证伪条件 |
| `QUESTION` | 未决问题 | 必须写出谁有权回答 |
| `DECISION` | 已作出的决定 | 必须写出作出者的能力层级 |

无标签内容一律视为 `HYPOTHESIS`。接手模型不得据 `INFERENCE`、`HYPOTHESIS` 作最终决策。

不要写成 "X 应该重构成 Domain Service"。要写成：`FACT` 当前逻辑位于何处 / `EVIDENCE` 对应文件:行 / `HYPOTHESIS` 可能职责过多，若 A 与 B 无共同变更原因则不成立 / `QUESTION` 是否应拆分，需 ARCHITECT 判断。

## 字段

四组，每组解决一个具体失效。字段不适用时写 `N/A`，不删除标题。

### 1. 身份与路由 — 防止接手模型不知道自己有没有决策权

```text
task-id:
current model / tier:
previous model / tier:
target model / tier:
current phase:
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

### 3. 分级证据 — 防止前置推断被当成事实

```text
FACT-1:
  EVIDENCE: <文件:行>
FACT-2:
  EVIDENCE: <文件:行>
INFERENCE-1:               （依据 FACT-x）
HYPOTHESIS-1:              （证伪条件）
DECISION-1:                （作出者层级）
relevant files:
relevant symbols:
dependencies:
```

### 4. 未决与风险 — 防止决策门被静默跳过

```text
QUESTION-1:                （谁有权回答）
risks:
evidence conflicts:        （不同来源或不同模型的结论冲突，本身即升级信号）
confidence:                （对当前结论的置信度，及不足之处）
```

## 编制时的自检

写完后逐项确认，任一不通过就补齐再发 `ROUTE_REQUEST`：

1. 每条 `EVIDENCE` 都有 `文件:行`，且行号来自实际读取而非记忆。
2. 没有把本层级禁止作出的判断伪装成 `INFERENCE` 或"建议"。
3. `required next action` 只有一项，且接手模型读完本文件就能开始，不需要追问。
4. `restrictions` 中用户的原始限定词未被概括掉。
5. 一个只看本文件、没有任何聊天历史的模型能独立开始工作。第 5 条不通过时，其余都不算通过。
