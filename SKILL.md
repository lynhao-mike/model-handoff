---
name: model-handoff
description: "Use when work must cross model boundaries in a host that cannot switch models automatically, so a human performs the switch. Covers judging whether a cross-model handoff is warranted and in which direction, writing a persisted handoff packet before requesting the switch, then asking the human to switch through a clickable confirmation that carries the packet path, and on takeover self-verifying model identity, reading the packet, and confirming or reporting missing context. Enforces capability tiers SCOUT, RESEARCHER, ORCHESTRATOR, ARCHITECT with graded FACT, EVIDENCE, INFERENCE, HYPOTHESIS labels so upstream guesses never become final architecture decisions. Trigger on 切换模型, 换模型, 切到某模型了, 换个模型接手, 跨模型交接, 交接包, 升级到 Opus, 交给更强的模型, ROUTE_ACK, model handoff, hand off to another model. Do not use for single-model tasks, mode switching, task strength routing, root-cause debugging, or completion verification."
---

# Model Handoff

## 行为合同

- **用户结果：** 跨模型协作时，前置模型的调查成果不丢失、不被当成结论；接手模型明确知道自己是谁、接的是什么、还缺什么；架构决策始终由被指定的最高能力层级作出。用户全程只需在模型选择器里切换，然后点一下确认项——不需要记忆或输入任何口令。
- **稳定信息真源（优先级固定）：** 用户明确指令 > 落盘交接包 > 仓库源码与测试 > 项目文档 > 模型推断。聊天历史不是权威真源。
- **默认允许动作：** 只读检索与分析；在 `.handoff/<task-id>.md` 写入或追加交接包；用 `ask_followup_question` 请求人工切换。
- **按需动作：** 修改代码、运行测试、Git 操作等仍按各自授权边界；本 Skill 的路由决定不授予这些动作。
- **永久不可能的动作：** 本环境没有切换模型的工具。模型只能请求，人工必须亲自在模型选择器里切换。禁止假设切换已发生。
- **停止条件：** 发出切换请求即由 `ask_followup_question` 结构性停止；`CONTEXT_INSUFFICIENT` 时停止并只列缺失项。

## 第一判断

收到请求后按顺序做三件事，不可跳过：

1. **读取自身身份。** 从 `environment_details` 的 `Current Mode` 块读 `<model>` 值。字段缺失时记为 `UNVERIFIED`。
2. **判断本轮角色。** 消息中含 `ROUTE_ACK` 或 `ROUTE_NACK`（无论是点击确认项还是手工输入）→ 接手侧，进入「接手流程」。否则 → 发起侧。
3. **判断是否交接及方向。** 按下表判定。

## 是否交接与方向

**按顺序判断，前面的命中即覆盖后面所有层级推理。**

| 序 | 情形 | 动作 |
| --- | --- | --- |
| 1 | 任务需要目标模型不具备的能力（图像/视觉输入、超长上下文、特定工具） | **按能力交接**，与层级高低无关。能力缺失不能用层级弥补。目标模型该项能力在 `references/model-map.md` 中未确认时，先问用户，不要假设 |
| 2 | 较低层级已实际尝试并失败 | **向上交接一级**。失败本身就是该层级不足的证据；不在同层反复重试 |
| 3 | 自身层级已足够完成任务 | **不交接**，直接做完。最高层级不给自己发切换请求 |
| 4 | 单文件局部修改、目标明确的 bug 修复、文案与配置、只读问答、纯解释 | **不交接**，交回 `skill-router` 的直办或轻量清单档 |
| 5 | 用户说「就用当前模型做完」 | **不交接**，在当前层级完成，并显式标注哪些结论未获决策授权 |
| 6 | 自身层级低于所需层级，且命中下方强制升级清单 | **向上交接** |
| 7 | 任务整体属于例行工作，自身层级明显过高 | **主动建议向下委派**（见下） |

跨模型交接要落盘、要人工切换、要新模型重新验证。对单点小任务使用它本身就是设计失败。

### 强制升级清单

自身层级低于 ARCHITECT 且命中任一项时必须交接，不得为省成本绕过：

- 架构变更、核心算法变更、跨模块重构；
- 领域模型或持久化结构变更、公共 API 变更；
- 身份、权限、隐私、安全相关改动；
- 不可逆操作，或存在重大 Trade-off；
- 前置模型之间结论冲突（冲突本身就是升级信号）；
- 关键证据不足或置信度不足。

### 例行工作与向下委派

四项**全部**成立才算例行：单步或机械重复、指令无歧义、不需要判断、预期输出确定。典型是批量取证与定位、原文通读摘要、格式转换、状态核对、按已知参数调用。

自身层级明显高于例行工作所需时**主动提出**委派，不要默默用高层级做完——但先算账：委派要落盘、要人工切换两次、还要切回来。取证量小或一次就能查完时自己做更省，此时不建议委派。

委派的交接包 `required next action` 只写取证范围并写明「不得下结论」。委派结果回到本层级后仍按 `INFERENCE` / `HYPOTHESIS` 对待，不因为是自己派出去的就升级为 `FACT`。

## 能力层级与权限

角色绑定能力层级，不绑定模型名。模型名到层级的映射见 `references/model-map.md`。

| 层级 | 允许 | 禁止 |
| --- | --- | --- |
| SCOUT | 检索、读取、摘要、列候选 | 下结论、改代码 |
| RESEARCHER | 跨模块分析、调用链、风险发现、生成候选方案 | 最终架构决策 |
| ORCHESTRATOR | 任务分解、路由建议、编制交接包、发起升级 | 覆盖 ARCHITECT 的决策 |
| ARCHITECT | 架构判断、Trade-off、实施计划、高风险审查 | 把自身决策当作用户批准 |

自身身份为 `UNVERIFIED` 或不在映射表中时，按 ORCHESTRATOR 工作，且不得自称 ARCHITECT。

## 发起流程

顺序不可颠倒——**先落盘，再请求**。这样即使新模型完全失忆或未加载本 Skill，凭确认项里的路径也能恢复。

**第 1 步：落盘交接包。** 按 `references/handoff-packet.md` 写入 `.handoff/<task-id>.md`。已存在同名文件时追加新一轮，不覆盖历史。

**第 2 步：用 `ask_followup_question` 请求切换。** 这是默认且唯一的请求方式。工具调用本身就是停止点——不要在提问前后另写"我已停止"。

- `question`：一句话说明切到哪个模型、为什么、交接包在哪。模型名用**用户在模型选择器里看到的那个名字**。
- `follow_up`：三项，顺序固定。每项都是完整可直接发送的句子，用户点一下即可，不需要编辑：

```text
1. ROUTE_ACK: <模型名> — 已切换，请读取 .handoff/<task-id>.md 并接手
2. ROUTE_NACK: 无法切换到 <模型名>
3. 就用当前模型继续，并标注哪些结论未获决策授权
```

第 1 项必须自带模型名与交接包路径。理由：点击后这句话会成为新模型收到的第一条消息；即使新模型没有加载本 Skill、也没有任何历史，凭这一句仍能读文件接手。这是整条链路唯一的自愈点，不得省略路径。

不要在 `follow_up` 里放需要用户补全的占位符，也不要写成疑问句。

**第 3 步：** 工具返回前不做任何其它事，不预演接手模型会给出的结论，不把本层级禁止的判断写成"建议"。

收到第 2 项时：在当前层级完成能力内的部分，明确列出哪些结论因层级不足未作出，交还用户；不降低升级门槛。收到第 3 项时按用户意愿在当前层级完成并标注授权缺口。

用户手工输入 `ROUTE_ACK`、或直接说"切好了 / 已经换成 X 了"时同样进入接手流程，不要求他们改用固定格式。

## 接手流程

1. **IDENTITY_CHECK。** 比对自读 `<model>` 与 ACK 声称的目标。不一致 → 拒绝接管，说明实读身份，请人工重新切换，然后停止。身份为 `UNVERIFIED` → 可继续，但输出中必须标注「身份未自证」。
2. **CONTEXT_CHECK。** 读取 ACK 携带的交接包路径。ACK 未带路径时先查 `.handoff/`。核对任务目标、当前阶段、前置层级、事实与证据、未决问题、限制、决策授权。
3. **判定并输出。** 足够 → `TAKEOVER_CONFIRMED`，然后开始工作。有缺口 → `CONTEXT_INSUFFICIENT`，只列缺失项并停止，不用推测填补。

接管后：前置层级的 `INFERENCE` 与 `HYPOTHESIS` 一律按待验证处理。作架构决策前，独立复核关键 `FACT`；无标签内容降级为 `HYPOTHESIS`，不得据此决策。

## 与相邻 Skill 的边界

- `skill-router` 是上游：它选领域与强度，本 Skill 只管跨模型交接。
- `long-project-local-controller` 并行、不合并：本 Skill **不重新定义证据 Gate**。任务同时属长期治理时，交接包**引用**其 `TASK_PROGRESS.md` / `PROJECT_STATE.md`，不新建竞争性真源。
- `writing-plans` 是下游：ARCHITECT 决策后若需正式多阶段计划才调用。决策 ≠ 计划。
- `verification-before-completion` 是下游：完成声明的证据由它把关。交接 ≠ 完成验证。

## 未来自动 Router 的替换点

策略与执行解耦。未来宿主具备模型 API 自动切换能力时，只替换「发起流程」第 2 步与「接手流程」第 1 步的人工环节；能力层级、交接包、升级门槛、证据分级全部不变。

## 资源路由

| 什么时候读 | 路径 | 读它改变什么动作 |
| --- | --- | --- |
| 编制或核对交接包时 | `references/handoff-packet.md` | 决定写入哪些字段、每条内容打什么标签、`EVIDENCE` 必须带 `文件:行` |
| 身份自证、判断目标层级或确定确认项里写哪个模型名时 | `references/model-map.md` | 决定自身与目标模型属哪个能力层级、未知标识如何降级、用户可见的模型名怎么写 |

不默认加载两者。发起侧通常只需第一份，接手侧两份都需要。

## 交付

1. 第一句说明当前身份与本轮动作：发起、接手或不交接。
2. 发起侧：说明命中的升级条件和交接包路径，然后直接调用 `ask_followup_question`。不复述协议、不另写停止声明。
3. 接手侧：先报 `IDENTITY_CHECK` 与 `CONTEXT_CHECK` 结果，再报 `TAKEOVER_CONFIRMED` 或 `CONTEXT_INSUFFICIENT`。
4. 不交接时：一句话说明原因，然后正常处理请求。
5. 不罗列未触发的状态词，不为展示流程而铺陈已完成的中间步骤。
