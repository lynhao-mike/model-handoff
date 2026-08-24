# 示例交接包

本文件展示两个完整示例：一个采集完成后向上决策的包，一个决策层发现缺口后向下采集的包。实际使用时复制字段骨架，填入本轮内容。

## 示例 1：COLLECT → DECIDE（briefing 可决策）

```markdown
## Round 1

### 1. 身份与路由

task-id: liuyao-targeting-refactor
current model / tier: Gemini 3.7 Flash / SCOUT
previous model / tier: N/A
target model / tier: Opus 5 / ARCHITECT
current phase: 采集完成，请求架构决策
work mode: DECIDE
decision authority: ARCHITECT

### 2. 任务与边界

task goal: 判断 targeting 模块是否需要拆分，给出拆分方案或保持现状的理由
non-goals: 不实施重构，不修改测试，不迁移调用方
completed work: 已读取 targeting 相关代码、调用链、测试覆盖
restrictions: 只判断是否拆分，不扩展到其它模块重构
required next action: 作出拆分决策并说明理由

### 3. 可决策 briefing

briefing status: decision-complete

FACT-1: targeting 模块当前有三类职责
  EVIDENCE: liuyao/domain/targeting.py:15-120
  CRUX:
    ```python
    class TargetResolver:
        def resolve_subject(self, hexagram): ...  # 主体判定
        def resolve_role_binding(self, subject): ...  # 角色绑定
        def format_output(self, binding): ...  # 输出格式化
    ```

FACT-2: 三类职责的变更原因不同
  EVIDENCE: git log 分析 + tests/test_target*.py:1-50
  CRUX:
    - 主体判定：六爻规则变化触发（最近 6 个月 8 次提交）
    - 角色绑定：问题类型扩展触发（最近 6 个月 3 次提交）
    - 输出格式化：UI 需求触发（最近 6 个月 12 次提交，最高频）

FACT-3: 当前调用方有 4 处
  EVIDENCE: liuyao/application/analysis.py:45, liuyao/web/service.py:78, tests/:多处
  CRUX: 全部调用完整 resolve 流程，没有只调某一步的

INFERENCE-1: 输出格式化变更频率远高于其它两项（依据 FACT-2）
INFERENCE-2: 若拆分，格式化独立后可减少核心逻辑受 UI 变更影响

HYPOTHESIS-1: 拆成 TargetResolver（主体+角色）+ TargetFormatter（格式化）可降低变更传播。证伪条件：调用方必须同时持有两个实例，增加复杂度抵消收益
HYPOTHESIS-2: 保持现状也可行，若未来格式化需求稳定。证伪条件：格式化继续高频变更

QUESTION-1: 是否拆分？（需 ARCHITECT 权衡变更频率 vs 调用复杂度）
QUESTION-2: 若拆分，接口如何设计？（需 ARCHITECT 决定依赖方向）

relevant files: liuyao/domain/targeting.py, liuyao/application/analysis.py, liuyao/web/service.py
relevant symbols: TargetResolver, resolve_subject, resolve_role_binding, format_output
dependencies: 无外部依赖冲突

### 4. 未决与风险

risks: 拆分后调用方需同时注入两个依赖，增加复杂度
evidence conflicts: 无
confidence: high（已覆盖核心代码、调用链、变更历史）

### 5. READ_LIST

READ_LIST status: N/A
```

---

## 示例 2：DECIDE → COLLECT（briefing 不足，需向下填）

```markdown
## Round 1

### 1. 身份与路由

task-id: liuyao-targeting-refactor
current model / tier: Opus 5 / ARCHITECT
previous model / tier: Gemini 3.7 Flash / SCOUT
target model / tier: Gemini 3.7 Flash / SCOUT
current phase: briefing 不足，需补充证据
work mode: COLLECT
decision authority: ARCHITECT（本轮只采集，不决策）

### 2. 任务与边界

task goal: 补充 briefing 缺失的证据，使 ARCHITECT 能判断是否拆分
non-goals: 不作拆分决策，不扩大调查范围
completed work: 已有 targeting 三类职责、变更频率、调用方位置
restrictions: 只填 READ_LIST，不新增条目，不下架构结论
required next action: 按 READ_LIST 抽取原文并填入 `result`，完成后请求切回 Opus 5

### 3. 可决策 briefing

briefing status: insufficient

FACT-1: targeting 模块当前有三类职责（见 Round 1）
FACT-2: 三类职责的变更原因不同（见 Round 1）
FACT-3: 当前调用方有 4 处（见 Round 1）

INFERENCE-1: 输出格式化变更频率远高于其它两项
HYPOTHESIS-1: 拆成 TargetResolver + TargetFormatter 可降低变更传播
QUESTION-1: 是否拆分？（需 ARCHITECT，但缺拆分成本的定量证据）

relevant files: liuyao/domain/targeting.py, tests/test_target*.py
relevant symbols: TargetResolver
dependencies: 无

### 4. 未决与风险

risks: 拆分成本未知，可能抵消收益
evidence conflicts: 无
confidence: medium（缺拆分成本的实际证据）

### 5. READ_LIST

READ_LIST status: open
collector target: Gemini 3.7 Flash / SCOUT

R1:
  path: tests/test_targeting*.py
  span or symbol: 所有涉及 TargetResolver 的测试
  unblocks: 量化拆分后需要修改多少测试，决定拆分成本是否可接受
  extract: crux<=8（测试数量、典型测试结构）
  result: （采集层填写）

R2:
  path: liuyao/application/analysis.py, liuyao/web/service.py
  span or symbol: 调用 TargetResolver 的代码段
  unblocks: 确认调用模式是否允许低成本注入两个依赖
  extract: crux<=8（调用上下文、依赖注入方式）
  result: （采集层填写）

R3:
  path: liuyao/domain/targeting.py
  span or symbol: TargetResolver.__init__ 和依赖
  unblocks: 判断拆分是否会引入循环依赖
  extract: signature（构造函数签名和当前依赖）
  result: （采集层填写）
```

---

## 字段骨架（快速复制）

```markdown
## Round N

### 1. 身份与路由
task-id:
current model / tier:
previous model / tier:
target model / tier:
current phase:
work mode:
decision authority:

### 2. 任务与边界
task goal:
non-goals:
completed work:
restrictions:
required next action:

### 3. 可决策 briefing
briefing status:
FACT-1:
  EVIDENCE:
  CRUX:

INFERENCE-1:
HYPOTHESIS-1:
QUESTION-1:
relevant files:
relevant symbols:
dependencies:

### 4. 未决与风险
risks:
evidence conflicts:
confidence:

### 5. READ_LIST
READ_LIST status:
collector target:
R1:
  path:
  span or symbol:
  unblocks:
  extract:
  result:
```
