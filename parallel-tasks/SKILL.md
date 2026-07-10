---
name: parallel-tasks
description: >
  Analyze remaining tasks from a task decomposition table, build a dependency graph,
  propose parallel execution waves, then dispatch multiple agents concurrently.
  Triggers on: /parallel-tasks {system-prefix} or /parallel-tasks {table-path}.
  Example: /parallel-tasks INT or /parallel-tasks E:/Project/任务拆解/通用/地图单元交互式/任务拆解表_交互式气泡_20260611.md
---

# /parallel-tasks — 并行任务编排与执行

用法：
- `/parallel-tasks INT` — 按系统前缀定位任务拆解表
- `/parallel-tasks <表路径>` — 直接指定任务拆解表文件

---

## Step 1：定位任务拆解表

1. 如果用户提供完整路径，直接读取。
2. 如果用户提供系统前缀，按以下索引查找：

   | 系统前缀 | 任务拆解表位置 |
   |---------|--------------|
   | MAP | `E:/Project/任务拆解/大地图/任务拆解表_大地图_20260514.md` |
   | PLT | `E:/Project/任务拆解/城建/任务拆解表_城建地块解锁_20260519.md` |
   | SCN | `E:/Project/任务拆解/场景/任务拆解表_场景切换_20260604.md` |
   | INT | `E:/Project/任务拆解/通用/地图单元交互式/任务拆解表_交互式气泡_20260611.md` |

3. 读取任务拆解表，提取所有状态非 `100%` 的任务行。
4. 如果所有任务已完成，提示用户并退出。

---

## Step 2：读取任务表分析依赖

对每个未完成任务：
1. 优先读取任务拆解表中的 `前置依赖` 列，直接提取依赖关系（如 `MAP-C-001, MAP-C-002`）。
2. （向后兼容）如果旧表没有该列，或者需要更精确的接口级确认，则回退/补充旧策略：读取对应 spec 文件，从「实现要点」和「关键现有代码」中推断隐式/显式依赖。
3. 构建有向无环依赖图（DAG）：
   ```
   {taskId} → [dependsOn: taskId[]]
   ```

---

## Step 3：并行分波编排

基于依赖图，将任务分为多个**执行波次（Wave）**：

**编排规则：**
- Wave 1：无前置依赖的任务（入度为 0 的节点）
- Wave N：所有前置依赖都在 Wave 1~(N-1) 中的任务
- 同一 Wave 内的任务互不依赖，可完全并行
- 单个 Wave 内任务数上限 = 4（受 Agent 并行能力限制）
- 超过 4 个同 Wave 任务时，拆为 Wave Na、Wave Nb 顺序批次

**编排输出格式：**

```
═══ 并行执行计划 ═══

Wave 1（并行）:
  ├─ INT-C-001 数据层接口与结构 (0.2d) [Ultra-tier: Gemini 1.5 Pro]
  └─ INT-C-004 集成点改造 (0.2d) [Fast-tier: Gemini Flash]

Wave 2（并行，依赖 Wave 1）:
  ├─ INT-C-002 控制器 (0.3d) ← 依赖 001 [Mid-tier]
  └─ INT-C-003 显示层 (0.5d) ← 依赖 001 [Mid-tier]

Wave 3（并行，依赖 Wave 2）:
  ├─ INT-C-005 世界地图集成 (0.3d) ← 依赖 002,003,004
  └─ INT-C-006 城建集成 (0.3d) ← 依赖 002,003,004

Wave 4（串行）:
  └─ INT-C-007 端到端验收 (0.2d) ← 依赖 005,006

总计：4 waves | 预估总耗时（串行）2.0d → 并行优化后 1.2d
```

---

## Step 4：用户确认

向用户展示并行编排方案，等待确认：

```
请确认执行方案：
1. 按上述分波执行（推荐）
2. 调整分波（说明要移动的任务）
3. 只执行指定 Wave（如：只执行 Wave 1）
4. 取消
```

用户可选择：
- **全部执行**：按 Wave 顺序逐波并行
- **执行到指定 Wave**：如"执行 Wave 1 和 2"
- **跳过某些任务**：如"跳过 008"
- **调整依赖**：如"002 不依赖 001，放 Wave 1"

---

## Step 5：并行执行

### 5a. Wave 基线准备

**Wave 1**：直接基于当前主分支开始。
**Wave N（N>1）**：必须在上一 Wave 合并完成后，基于最新主分支创建 worktree。这确保每个 agent 能引用前序 Wave 产出的接口/类。

### 5b. 获取可用模型池 (Model Discovery)

在派发任何任务前，主控 Session 必须通过命令行（如 `agy models`）或系统指令获取当前环境内**所有可使用的模型列表**。
根据返回的模型池，**主控需自主评估并挑选出三个档次的模型**（若资源有限可向下合并）：
1. **顶级算力（Ultra-tier）**：挑选参数量最大、推理能力最强的旗舰模型（如 Gemini Pro / Claude Sonnet 级别）。
2. **中坚算力（Mid-tier）**：挑选速度和智能较为均衡的模型（如参数量适中的通用模型）。
3. **极速算力（Fast-tier）**：挑选速度极快、成本极低的基础模型（如 Gemini Flash / Claude Haiku 级别）。

将这三个模型暂存在主控的上下文中，供后续调度使用。

### 5c. 分派 Agents (兼容 agy / claude 双引擎与三级模型协同)

主控 Session 需识别当前的运行环境，并根据任务的 `类型` 列动态路由到 5b 中挑选出的物理模型，以追求极致的性价比。

**三级模型路由规则：**
- **Ultra-tier (顶级)**：分配给 `底层框架`、`调试/验收` 等对架构理解和全局纠错要求极高的任务。
- **Mid-tier (中坚)**：分配给 `复杂业务逻辑`，用于常规 Controller 和机制的具体实现。
- **Fast-tier (极速)**：分配给 `配置`、`UI 视图`、`工具` 等周边和单点填表任务。

#### 环境 A：在 Antigravity (agy) 引擎下
对本波次的每个任务，调用内置的 `invoke_subagent` 工具执行并发派发：
- `Workspace`: `share`（自动利用底层类似 worktree 的隔离沙盒）。
- `TypeName`: 根据路由规则，分配到对应的类型如 `agent-slg-dev-ultra` / `agent-slg-dev-mid` / `agent-slg-dev-fast`。（若项目未细分该类别，则统一走 `agent-slg-dev`）。
- `Role`: **必须在展示名中明确包含所分配的具体模型名称**（例如：`INT-C-001 (Gemini 1.5 Pro)` 或 `UI 调整 (Gemini Flash)`），以便用户在 UI 会话列表中直观区分每个 Subagent 的算力级别。
- **运行方式**：利用工具的并发调用特性，直接并行派发。

#### 环境 B：在 Claude Code 引擎下
由于缺乏内置 Subagent API，主控需转变为“Shell 脚本调度者”角色：
1. 建立独立沙盒：对本波每个任务，执行 `git worktree add ../worktree-{taskId}`。
2. 并发拉起 CLI：进入各目录，根据三级路由规则追加对应的 `--model` 参数，并使用 `&` 放入后台执行：
   ```bash
   # 例：UI 任务分配给极速模型
   cd ../worktree-001 && claude -p "{Prompt内容}" --model claude-3-5-haiku-20241022 &
   # 例：底层框架分配给顶级模型
   cd ../worktree-002 && claude -p "{Prompt内容}" --model claude-3-5-sonnet-20241022 &
   wait
   ```
3. 执行 `wait` 等待所有进程结束后，进行后续合并。

**通用 Agent Prompt 模板（传给上述底层进程）：**

```
你正在执行任务 {taskId}。

## 任务信息
- 任务ID：{taskId}
- 标题：{title}
- Spec 文件：{specPath}
- 任务拆解表：{tablePath}

## 执行要求

1. 读取 spec 文件，理解任务目标和实现要点
2. 读取 spec 中「关键现有代码」列出的所有文件
3. 按 spec「实现要点」逐条实现
4. 实现完成后执行编译验证：bash .claude/verify-compile.sh
5. 对照 spec「验收标准」自查
6. 验收通过后，git commit 本次任务所有变更（提交信息格式见下方）

## Git Commit 规范（强制）
任务完成且编译通过后，必须立即 commit：
- 格式：`feat: {需求标题} [Agent]`
- 示例：`feat: IInteractProvider接口与InteractItem数据结构 [Agent]`
- 只 add 本任务新建/修改的文件，禁止 `git add -A`
- 如果编译未通过，不得 commit，报告错误等待主 session 处理

## 强制规范
- **【全局约定】**在动手前必须读取并严格遵守当前项目的全局 AI 约定（如 `AGENTS.md`）。
- **【专业技能引用】**如果是特定领域的开发任务（如 UI、配置表、战斗框架、TA等），必须先主动查阅并严格遵守项目中对应的专业 Skill（如 `slg-ui-edit`、`slg-config-edit`、`slg-battle-dev` 等）的工作流规范。
- **【通用性】**在实现中如识别到具备通用性的逻辑，必须抽象为接口或通用工具类，严禁与当前业务强耦合。
- **【单测驱动】**如果 Spec 要求了单元测试，必须为核心逻辑编写单测，并在提交前确保测试运行通过。
- 异步使用 UniTask，禁止 Coroutine/Task
- Module 访问通过 ModuleContext
- 配置表字段必须查阅 Configs/Gen/ 生成代码
- ViewExt 必须是 partial class
- 禁止 System.Linq，使用 LinqExt 或手动循环
- 禁止 UnityEngine.Debug，使用 LoggerService
- PooledList 替代 new List
- 500行上限，超过拆分 partial

## 产出要求
完成后报告：
1. 新建/修改了哪些文件（附 git commit hash）
2. 编译是否通过
3. 验收标准的通过情况
4. 是否有偏差需要记录
```

### 5c. 等待 Wave 完成

所有当前 Wave 的 agents 完成后：
1. 收集每个 agent 的执行结果和 commit hash
2. 检查是否有失败的任务
3. 汇总报告给用户

### 5d. 失败处理

| 情况 | 操作 |
|------|------|
| 编译错误 | 展示错误，询问用户是否手动修复后继续 |
| Agent 超时 | 标记任务为未完成，继续下一 Wave |
| 文件冲突 | 合并 worktree 时解决冲突（应极少发生，因同 Wave 无依赖） |

### 5e. Wave 间过渡（L2 验收）

当前 Wave 的所有 agent 均宣称完成后：
1. **统一 Code Review**：主 session 唤起 `agent-quality-checker` 子 Agent，触发 `slg-code-review` 技能，对本 Wave 产出的各个分支代码进行两阶段审查。审查不通过则打回修复。
2. **安全合并**：Review 全部通过后，将所有 worktree 分支合并到主分支（merge --no-ff 保留 commit 历史）。
3. **编译验证**：执行合并后的编译验证（如 `bash .claude/verify-compile.sh`）。
4. **接口衔接检查**：确认本 Wave 产出的类/接口能被下一 Wave 正确引用。
5. **单元测试检查**：如果本 Wave 产出了单测，执行测试并确认全部通过。
6. 向用户报告 Wave 完成状态（含各任务 commit hash），询问是否继续下一 Wave。

---

## Step 6：最终验收（L3）

所有 Wave 执行完毕后：

### 6a. 集成编译

主分支上执行最终编译验证，确认所有 Wave 合并后无冲突无 error。

### 6b. Play 模式验证（如有验收任务）

如果任务表中包含验收类任务（如 INT-C-007），由该任务的 agent 或主 session 执行 Play 模式端到端测试。

如果任务表无专门验收任务，主 session 执行快速冒烟测试：
1. Play 模式启动，确认不 crash
2. 核心流程走通（点击→响应→UI 显示正确）

### 6c. 完成汇总

1. **更新任务拆解表**：将所有完成的任务状态改为 `100%`
2. **更新 Spec 文件**：填写每个 spec 的「完成记录」节
3. **汇总报告**：

```
═══ 并行执行完成 ═══

Wave 1: ✅ INT-C-001 (abc1234), ✅ INT-C-004 (def5678)
Wave 2: ✅ INT-C-002 (ghi9012), ✅ INT-C-003 (jkl3456)
Wave 3: ✅ INT-C-005 (mno7890), ⚠️ INT-C-006（有偏差）
Wave 4: ✅ INT-C-007

完成：6/7 | 偏差：1 | 耗时：{实际时间}
Git commits：7 commits on branch feat/xxx
偏差详情：INT-C-006 城建场景 BuildBaseData 无 OnSelect 方法，改用 ...
```

### 6d. 全局文档同步与知识沉淀

在所有 Wave 成功合并并验收通过后，统一调用一次 `slg-doc-sync` skill：
- 分析本次所有 Wave 带来的系统级变更。
- 更新相关的 `feature-map`、`system-overview`、`architecture-map` 及相关上下文文档。
- 如果在评审阶段提炼了新规则，调用 `slg-rules-from-review`。

### 6e. 清理临时 Worktree 和分支（强制）

所有任务验收通过、文档同步完成、用户确认没问题后，必须清理所有临时 agent worktree 和对应本地分支：

```bash
# 列出当前所有 worktree，确认要删除的目标
git worktree list

# 逐一删除 worktree 目录（--force 处理有未合并提交的情况，删前已确认内容已合并）
git worktree remove --force "<path>"

# 清理残留引用
git worktree prune

# 删除对应的本地临时分支
git branch -d worktree-agent-xxx  # 逐一删除，或批量：
git branch | grep worktree-agent | xargs git branch -d
```

**时机**：在用户确认任务结果没问题之后执行，不得在验收前清理。

---

## 三级验收体系

| 级别 | 时机 | 执行者 | 内容 |
|------|------|--------|------|
| **L1 Agent 自查** | 每个 agent 完成时（worktree 内） | agent 自身 | 编译通过 + spec 验收标准逐条 + git commit |
| **L2 Wave 合并验收** | 同 Wave 全部完成合并前后 | Quality Checker & 主 session | Code Review + 合并编译 + 接口衔接 + 单测运行 |
| **L3 端到端验收** | 所有 Wave 完成后 | 主 session / 验收任务 agent | Play 模式完整流程验证 |

---

## 约束

- **禁止跳过 Step 4 确认**：未经用户确认不派发 agents
- **禁止跨 Wave 并行**：必须上一 Wave 全部完成并合并后才开始下一 Wave
- **Worktree 清理**：所有任务验收通过、文档同步完成且用户确认没问题后，必须执行 Step 6e 清理全部临时 worktree 目录和对应本地分支，禁止验收前清理
- **Worktree 隔离**：同 Wave 的 agents 必须使用独立 worktree，防止文件冲突
- **编译守门**：每个 Wave 合并后必须通过编译验证才能继续
- **Git Commit 强制**：每个 agent 完成任务且编译通过后必须 commit，格式 `feat: {标题} [Agent]`
- **合并策略**：Wave 合并使用 `merge --no-ff`，保留每个任务的独立 commit 历史
- **策划案只读**：任何情况下不修改任何策划相关目录（如 `策划文档/`、`策划案/` 及其子目录）下的文件
- **spec 引用不猜测**：agent prompt 中必须包含实际 spec 路径，不使用占位符
- **单 Wave 上限 4 agents**：受并行 context 限制，超过需分批

---

## 高级用法

### 指定起始 Wave

```
/parallel-tasks INT --from-wave 3
```
跳过已完成的 Wave，直接从 Wave 3 开始。

### 排除任务

```
/parallel-tasks INT --skip 007,008
```
从编排中移除指定任务（如验收和文档类任务不需并行）。

### 仅规划不执行

```
/parallel-tasks INT --plan-only
```
只输出并行编排方案，不执行。用于 review 依赖关系是否正确。

---

## 依赖推断规则（降级策略）

当任务拆解表中没有 `前置依赖` 列，或 spec 中没有显式标注依赖时，按以下规则推断隐式依赖：

| 模式 | 推断依赖 |
|------|---------|
| spec B 的「关键现有代码」包含 spec A 要新建的文件 | B 依赖 A |
| spec B 的实现要点中使用 spec A 定义的接口/类 | B 依赖 A |
| spec B 的类型为「调试/验收」且涉及 A 的产出 | B 依赖 A |
| spec B 的优先级低于 A 且属同一子模块 | 不自动推断（需用户确认） |
| spec A 和 B 改动完全不同的文件 | 无依赖，可并行 |
