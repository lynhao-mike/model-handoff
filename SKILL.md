---
name: model-handoff
description: "Use when work must cross model boundaries in a host that cannot switch models automatically, so a human performs the switch. Cheap models collect a decision-complete briefing; expensive models consume that briefing and decide, and must not re-read source or docs. Covers judging handoff direction, writing a persisted packet before a switch request, then on takeover verifying the live model identity. Clicking a confirmation does not switch models; IDENTITY_CHECK failure must stop and wait for a real selector change unless the user explicitly continues on the current model. Collectors use Graft then IWE before raw file reads. Trigger on 切换模型, 换模型, 切到某模型了, 换个模型接手, 跨模型交接, 交接包, 升级到 Opus, 交给更强的模型, 读取清单, 向下取证, ROUTE_ACK, model handoff. Do not use for single-model tasks, mode switching, task strength routing, root-cause debugging, or completion verification."
---

# Model Handoff

## 行为合同

- **用户结果：** 低端模型采集并整理成一份可决策简报；中端/高端只消费这份简报做判断，不再重读代码和文档。前置推断不会被当成结论；架构决策始终由被指定的最高能力层级作出。用户必须先在模型选择器里换成目标模型，确认界面已显示该名称，再发送接手消息。
- **稳定信息真源（优先级固定）：** 用户明确指令 > 落盘交接包（含 briefing 与 READ_LIST）> 仓库源码与测试 > 项目文档 > 模型推断。聊天历史不是权威真源。未写入交接包的已读内容，对接手模型视为不存在。
- **默认允许动作：** 在 `.handoff/<task-id>.md` 写入或追加交接包；用 `ask_followup_question` **发起一次**切换请求。采集层按 Graft → IWE → 精确读文件取证。决策层默认只读交接包和本 Skill 的 reference。
- **按需动作：** 修改代码、运行测试、Git 操作等仍按各自授权边界；本 Skill 的路由决定不授予这些动作。
- **永久不可能的动作：** 本环境没有切换模型的工具。确认项、`ROUTE_ACK` 文本、聊天回复都不能改 `<model>`。禁止假设切换已发生。决策层禁止用「独立复核」或「取证量小」为理由打开源码、Graft 或 IWE。
- **停止条件：** 发出切换请求即由 `ask_followup_question` 结构性停止；`SWITCH_NOT_APPLIED` 时停止且不得再发确认项；`CONTEXT_INSUFFICIENT` 时停止并只列缺失字段；`BRIEFING_INSUFFICIENT` 时只写封闭 READ_LIST 并请求向下，不自行补读。

## 第一判断

收到请求后按顺序做四件事，不可跳过：

1. **读取自身身份。** 从 `environment_details` 的 `Current Mode` 块读 `<model>` 值。字段缺失时记为 `UNVERIFIED`。
2. **判断本轮角色。** 消息中含 `ROUTE_ACK` 或 `ROUTE_NACK`（无论是点击确认项还是手工输入）→ 接手侧，进入「接手流程」。否则 → 发起侧。
3. **判断本轮工作模式。** 交接包 `required next action` 是采集、填 READ_LIST、或编制 briefing → **COLLECT**。是架构判断、Trade-off、实施计划、高风险审查、跨模块分析 → **DECIDE**。包不存在且自身已是决策层、任务又需要判断 → 仍算 DECIDE，但必须先走向下采集，不得自己读源码开干。
4. **判断是否交接及方向。** 按下表判定。

## 是否交接与方向

**按顺序判断，前面的命中即覆盖后面所有层级推理。**

| 序 | 情形 | 动作 |
| --- | --- | --- |
| 1 | 任务需要目标模型不具备的能力（图像/视觉输入、超长上下文、特定工具） | **按能力交接**，与层级高低无关。能力缺失不能用层级弥补。目标模型该项能力在 `references/model-map.md` 中未确认时，先问用户，不要假设 |
| 2 | 较低层级已实际尝试并失败 | **向上交接一级**。失败本身就是该层级不足的证据；不在同层反复重试。向上前必须把已读内容写成带 crux 的 briefing，禁止把空指针包交上去 |
| 3 | 单文件局部修改、目标明确的 bug 修复、文案与配置、只读问答、纯解释 | **不交接**，交回 `skill-router` 的直办或轻量清单档。这类请求不进入采集/决策协议 |
| 4 | 用户说「就用当前模型继续，并标注哪些结论未获决策授权」或同义「就用当前模型做完」 | **不交接**，在当前层级完成，并显式标注哪些结论未获决策授权。用户未同时授权读源码时，决策层仍不得打开源码 |
| 5 | 自身是决策层（ANALYST / ARCHITECT），剩余工作是判断，且 briefing 不可决策 | **必须向下采集**。写封闭 READ_LIST，默认目标 **Gemini 3.7 Flash**。禁止自己打开源码或项目文档补齐。最高层级也可以向下，这不是给自己发升级请求 |
| 6 | 自身层级已足够：采集层做采集，或决策层且 briefing 已可决策 | **不交接**，直接做完。最高层级不给自己发升级请求 |
| 7 | 自身层级低于所需层级，且命中下方强制升级清单 | **向上交接**。向上包必须可决策；否则先向下采集，再向上。禁止把只有路径、没有 crux 的包交给 ARCHITECT |
| 8 | 剩余工作是例行采集，自身层级明显高于采集层 | **必须向下**给 SCOUT（默认 Gemini），不是建议 |

跨模型交接要落盘、要人工切换。对单点小任务使用它本身就是设计失败。

### 强制升级清单

自身层级低于 ARCHITECT 且命中任一项时必须向上，不得为省成本绕过决策门：

- 架构变更、核心算法变更、跨模块重构；
- 领域模型或持久化结构变更、公共 API 变更；
- 身份、权限、隐私、安全相关改动；
- 不可逆操作，或存在重大 Trade-off；
- 前置模型之间结论冲突（冲突本身就是升级信号）；
- 关键证据不足或置信度不足。

证据不足时先向下采集，再向上决策；不要让 ARCHITECT 用重读来补证据。

### 采集与决策

一次跨模型任务拆成两个模块，而不是换了大脑的连续对话。

| 模式 | 谁做 | 允许 | 禁止 |
| --- | --- | --- | --- |
| COLLECT | 默认 Gemini 3.7 Flash（SCOUT）。其它 SCOUT 仅用户点名 | 按范围或 READ_LIST 用 Graft / IWE / 精确读文件抽取 briefing | 作架构决策、扩大清单、改代码；同一采集目标失败后原地重试；非 Gemini 模型接海量通读 |
| DECIDE | Sol / Sonnet 5 / Grok 4.6（ANALYST）；Opus 5（ARCHITECT） | 只读交接包做判断；发现缺口时写 READ_LIST 并向下给 Gemini | 打开源码或项目文档；用搜索 / Graft / IWE 来「复核」；ANALYST 做批量取证 |

**默认生命周期：** Gemini 抽包 → ANALYST 收包分析/编包 → 必要时 ARCHITECT 架构裁决。高端进场时，仓库应已经变成一份可决策简报。

**缺口回填环**（只在 DECIDE 发现 briefing 不可决策时进入，不是每次高端进场的默认路径）：

1. 决策层判断最终结论还缺哪类代码或文档信息；
2. 把缺口写成封闭 READ_LIST，一次批完；
3. 切换到采集层，只填清单；
4. 采集层把抽取结果写入同一交接包后请求切回；
5. 决策层只读更新后的 briefing 作判断，仍不打开源码。

人工切换有成本，所以 READ_LIST 必须一次列全，禁止每缺一个文件就乒乓一次，也禁止「就一个文件，我自己读更省」。

四项全部成立才算例行采集：单步或机械重复、指令无歧义、不需要判断、预期输出确定。典型是按清单取证、原文抽取、格式转换、状态核对。采集结果回到决策层后，其中的 `INFERENCE` / `HYPOTHESIS` 仍按待验证处理，不因为是自己派出去的就升级为 `FACT`。

## 能力层级与权限

角色绑定能力层级，不绑定模型名。模型名到层级的映射见 `references/model-map.md`。

| 层级 | 默认模型 | 允许 | 禁止 |
| --- | --- | --- | --- |
| SCOUT | **Gemini 3.7 Flash**（默认且可再次调用）。其它 SCOUT 仅用户点名 | 按 READ_LIST 用 Graft / IWE / 精确读文件抽 crux | 下结论、改代码、扩大 READ_LIST；非默认阅读器接海量通读 |
| ANALYST | Sol、Sonnet 5、Grok 4.6（同级） | 收压缩包：路由、深度分析、编包、缺口判断、发起向上或向下交接 | 覆盖 ARCHITECT 的决策；批量读取业务源码或项目文档 |
| ARCHITECT | Opus 5 | 架构判断、Trade-off、实施计划、高风险审查 | 把自身决策当作用户批准；打开源码做调查或复核 |

### 默认采集目标

向下采集时，确认项里的目标模型默认写 **Gemini 3.7 Flash**。Gemini 是海量文档/代码的默认阅读器，**单轮失败不等于本任务禁读 Gemini**。

改派规则：

1. Gemini **本轮**不可用或已实际失败 → 用户点名后备 SCOUT，或等 Gemini 恢复后再次调用。**不写进默认改派。** 禁止对同一采集目标连续重试 Gemini；也禁止因此让 ANALYST / ARCHITECT 接手整份海量通读。
2. 任何非 Gemini 的 SCOUT（仅当用户点名）接手后，若剩余工作仍是海量文档/源码阅读（判断原则：多数未填条目需要通读长文件，且 Graft/IWE 给不出 crux）→ **必须再切回 Gemini 3.7 Flash**，不得自己读完。用户未禁止再次使用 Gemini 时，默认再次调用。
3. 需要写 READ_LIST、编包、深度分析、判断是否升级，但不做架构结论 → 留在 ANALYST（Sol / Sonnet 5 / Grok 4.6）。读文件仍向下给 Gemini。中端接收的是压缩后的交接包，不是原始文档堆。
4. 架构结论、Trade-off、实施计划 → Opus 5；它只消费 briefing，缺口只写 READ_LIST 给 Gemini。

自身身份为 `UNVERIFIED` 或不在映射表中时，按 ANALYST 工作，且不得自称 ARCHITECT。

决策层允许读取的只有：当前交接包、`references/model-map.md`、`references/handoff-packet.md`。禁止对仓库做任何调查：`read_file`、搜索、列目录、Graft、IWE，以及其它会把仓库内容拉进上下文的工具。本 Skill 的决策层读预算覆盖仓库级「先查 Graft / 先读源码 / 先查文档」习惯；DECIDE 轮不得为遵守侦察规则而打开仓库。身份核对和字段核对不是调查。

ANALYST 在**向上编包**（不是 DECIDE）时，允许一次只读刷新 git 合同字段：`git status --short`、`git branch -vv`、`git log --oneline -5`。这只校正 briefing 里的 HEAD/脏文件，不算业务调查。命令无输出时不得当成工作树干净，必须拆成独立命令再取一次。

## 采集取证顺序

谁读：默认 Gemini。其它 SCOUT 仅用户点名。ANALYST / ARCHITECT 不读仓库。

怎么读：COLLECT 默认按索引取证，避免整文件通读。细则见 `references/collect-sources.md`。

1. **代码、定义、调用方、影响面：** Graft。优先已接入的 Graft MCP；MCP 不可用或返回未知工具时改 CLI：`graft map`、`graft ask "<问>" --source`、`graft skeleton <file>`、`graft callers <symbol>`、`graft grep "<literal>"`。
2. **理论、规则语义、案例关系、设计决策：** IWE `retrieve`，上限 max-documents 6、max-tokens 6000、max-document-tokens 1800。IWE 默认只读；写操作需用户显式授权且先 dry-run。
3. **仍缺精确行号或 Graft 标明 +N more lines：** 只 `read_file` 该 span。
4. **无索引文件（含多数 Markdown）：** 直接读文件。对 Markdown 跑 skeleton 得到「no definitions」不是错误，不要因此扩大清单。
5. **READ_LIST 路径不存在：** 写 `NOT_FOUND`。可记录实际发现的邻近真实路径，不把装饰性文件名当真，不新增 READ_LIST 条目。

DECIDE 禁止走本顺序。

## 发起流程

顺序不可颠倒——**先落盘，再请求**。这样即使新模型完全失忆或未加载本 Skill，凭接手消息里的路径也能恢复。

向上决策前：本轮已读内容必须写成带 crux 的 briefing，否则视为不可决策，先补采集。向下采集前：READ_LIST 必须封闭、一次批完，每条写明补上后能回答哪个决策问题。

**第 1 步：落盘交接包。** 按 `references/handoff-packet.md` 写入 `.handoff/<task-id>.md`。已存在同名文件时追加新一轮，不覆盖历史。

**第 2 步：用 `ask_followup_question` 请求切换。** 每个目标模型只发起一次。工具调用本身就是停止点——不要在提问前后另写「我已停止」。

- `question` 必须写清操作顺序：先在模型选择器切换到目标名称，确认界面已显示该名称，再点第一项或发新消息。带上交接包路径。
- `follow_up`：三项，顺序固定。每项都是完整可直接发送的句子：

```text
1. ROUTE_ACK: <模型名> — 已切换，请读取 .handoff/<task-id>.md 并接手
2. ROUTE_NACK: 无法切换到 <模型名>
3. 就用当前模型继续，并标注哪些结论未获决策授权
```

第 1 项必须自带模型名与交接包路径，供**已经完成选择器切换**之后的模型作为第一条消息。它不会替用户改模型。

不要在 `follow_up` 里放需要用户补全的占位符，也不要写成疑问句。

**第 3 步：** 工具返回前不做任何其它事，不预演接手模型会给出的结论，不把本层级禁止的判断写成「建议」。

收到第 2 项时：在当前层级完成能力内的部分，明确列出哪些结论因层级不足未作出，交还用户；不降低升级门槛。决策层即使留下继续，也不得改为自己读源码。收到第 3 项时按用户意愿在当前层级完成并标注授权缺口。

用户手工输入 `ROUTE_ACK`、或直接说「切好了 / 已经换成 X 了」时同样进入接手流程，不要求他们改用固定格式。

## 接手流程

先做 IDENTITY_CHECK，再做任何采集或决策。

1. **IDENTITY_CHECK。** 用 `references/model-map.md` 比对自读 `<model>` 与 ACK 声称的目标。
   - **一致** → 继续 CONTEXT_CHECK。
   - **不一致** → `SWITCH_NOT_APPLIED`。报告实读身份与目标身份。说明：确认项不会切换模型。请用户先在选择器改成目标名称，确认界面已显示该名称，再发一条新消息（可粘贴原来的 `ROUTE_ACK` 句）。然后 **停止**：不得再调用 `ask_followup_question`，不得采集，不得决策，不得编包。
   - **唯一例外：** 同一条消息明确是第 3 项「就用当前模型继续，并标注哪些结论未获决策授权」→ 不接管目标角色，按当前层级做完并标注未获授权的结论。
   - 身份为 `UNVERIFIED` 且 ACK 目标可解析 → 可继续，但输出必须标注「身份未自证」，且不得自称 ARCHITECT。
2. **CONTEXT_CHECK。** 读取 ACK 携带的交接包路径。ACK 未带路径时先查 `.handoff/`。核对任务目标、当前阶段、前置层级、限制、决策授权、`required next action`。缺这些字段 → `CONTEXT_INSUFFICIENT`，只列缺失字段并停止，不用推测填补，也不打开源码补上下文。
3. **按工作模式接手。**
   - **COLLECT：** `TAKEOVER_CONFIRMED`。有 READ_LIST 则只填清单；没有则按任务目标采集并写成可决策 briefing。不得下架构结论。填完后走发起流程，请求切回决策层。
   - **DECIDE：** 做 BRIEFING_CHECK。可决策 → `TAKEOVER_CONFIRMED`，只凭 briefing 判断，打开源码属于协议违规。不可决策 → `BRIEFING_INSUFFICIENT`，只写封闭 READ_LIST，然后走发起流程向下；禁止自行补读。

briefing 可决策，当且仅当：决策所需每条 `FACT` 都有带 `文件:行` 且内联了足够原文的 `EVIDENCE`；阻塞决策的问题已列为 `QUESTION`；冲突已显式写出；一个只读本文件的决策层模型能完成 `required next action`。只有路径、摘要或「详见某文件」= 不可决策。git 合同字段必须带 HEAD；编包前允许 ANALYST 做一次只读刷新。

接管后：前置层级的 `INFERENCE` 与 `HYPOTHESIS` 一律按待验证处理。复核关键 `FACT` 只在 briefing 内进行——核对标签完整性、逻辑矛盾、截断标记、未决问题，不重新验证原文真实性。无标签内容降级为 `HYPOTHESIS`，不得据此决策。复核发现 briefing 逻辑撑不住决策时走 `BRIEFING_INSUFFICIENT` 并写 READ_LIST，不是打开源码重新验证。

## 与相邻 Skill 的边界

- `skill-router` 是上游：它选领域与强度，本 Skill 只管跨模型交接。
- `long-project-local-controller` 并行、不合并：本 Skill **不重新定义证据 Gate**。任务同时属长期治理时，交接包**引用**其 `TASK_PROGRESS.md` / `PROJECT_STATE.md`，不新建竞争性真源。
- `writing-plans` 是下游：ARCHITECT 决策后若需正式多阶段计划才调用。决策 ≠ 计划。
- `verification-before-completion` 是下游：完成声明的证据由它把关。交接 ≠ 完成验证。
- `meta-skills` 负责创建/改造 Skill。跨模型交接不安装外部 Skill OS，也不把工厂型元技能并入本流程。

## 未来自动 Router 的替换点

策略与执行解耦。未来宿主具备模型 API 自动切换能力时，只替换「发起流程」第 2 步与「接手流程」第 1 步的人工环节；能力层级、采集/决策分离、交接包、READ_LIST、升级门槛、证据分级全部不变。在那之前，确认项永远不能当作切换已完成的证据。

## 资源路由

| 什么时候读 | 路径 | 读它改变什么动作 |
| --- | --- | --- |
| 编制或核对接手包、判断 briefing 是否可决策、写 READ_LIST 时 | `references/handoff-packet.md` | 决定写入哪些字段、`EVIDENCE` 如何内联 crux、READ_LIST 每条写什么、怎样判定可决策 |
| 首次编制交接包、需要示例参考时 | `references/example-packets.md` | 提供完整示例：采集完成向上决策的包、决策层发现缺口向下采集的包，以及快速复制的字段骨架 |
| 身份自证、判断目标层级或确定确认项里写哪个模型名时 | `references/model-map.md` | 决定自身与目标模型属哪个能力层级、未知标识如何降级、用户可见的模型名怎么写 |
| COLLECT 取证、写 READ_LIST 的 extract 方式、Graft/IWE 不可用时 | `references/collect-sources.md` | 决定先 Graft 还是 IWE、CLI 回退、禁止对 Markdown 跑 skeleton、命令空输出如何处理 |

不默认加载 reference。发起侧通常读 handoff-packet；接手侧读 model-map，DECIDE 再读 handoff-packet。决策层读这些 reference 不等于获准读仓库。

## 交付

1. 第一句说明当前身份、本轮模式（COLLECT / DECIDE / 不交接 / SWITCH_NOT_APPLIED）和动作。
2. 发起侧：说明命中的交接条件、交接包路径、目标是采集还是决策，然后直接调用一次 `ask_followup_question`。不复述协议、不另写停止声明。
3. 接手侧：先报 `IDENTITY_CHECK`。失败则报 `SWITCH_NOT_APPLIED` 并停止。通过后再报 `CONTEXT_CHECK`，以及 `TAKEOVER_CONFIRMED`、`CONTEXT_INSUFFICIENT` 或 `BRIEFING_INSUFFICIENT`。后两者都立即停止调查；只有 `BRIEFING_INSUFFICIENT` 接着写 READ_LIST 并向下。
4. 不交接时：一句话说明原因，然后正常处理请求。
5. 不罗列未触发的状态词，不为展示流程而铺陈已完成的中间步骤。
