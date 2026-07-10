---
name: start-task
description: >
  Execute a single task from the task decomposition table. Reads the spec file,
  explores referenced code, implements following project rules, then feeds back
  completion to the spec and task table. Triggers on: /start-task {ID}, user
  provides a task ID like MAP-C-007 and wants to start implementing it.
---

# /start-task — 单任务实现工作流

用法：`/start-task MAP-C-007`

---

## Step 1：定位任务

1. 根据任务 ID 前缀确定系统，在以下索引中查找任务拆解表：

   | 系统前缀 | 任务拆解表位置 |
   |---------|--------------|
   | MAP | `任务拆解/大地图/任务拆解表_大地图_20260514.md` |

2. 在任务拆解表中找到对应行，获取：
   - Spec 文件路径
   - 当前状态（若已是 100% 则提示用户确认是否重新实现）
   - 优先级和依赖关系（需求描述中是否注明前置任务）

3. 读取 spec 文件全文。

---

## Step 2：代码探索

根据 spec 的「关键现有代码」节，逐一读取相关文件。

**规则：**
- 只读 spec 中明确列出的文件，不做大范围扫描
- 如发现 spec 中的文件路径或类名有误，**直接修正 spec**，不中断流程
- 如发现 spec 遗漏了关键依赖文件，补充读取后继续

---

## Step 3：实现前确认

将以下信息呈现给用户，**等待确认后再动手写代码**：

```
任务：{需求ID} {需求标题}
优先级：{P0/P1} | 估时：{X}天

实现计划：
1. 修改 {文件A}：{做什么}
2. 新建 {文件B}：{做什么}
…

前置确认：
- [ ] 依赖任务 {ID} 已完成（如有）
- [ ] 相关配置表字段已查阅生成代码
- [ ] 通用性评估：该模块是否有复用价值？是否已提议将通用逻辑抽象为接口/基类？
```

用户回复「ok」或「可以」后进入实现。

---

## Step 4：实现

按 spec「实现要点」逐条执行。在实现前，**强制回顾并严格遵守当前项目的全局 AI 约定（如 `AGENTS.md`）**，严禁引入违反项目架构标准的设计。

**如果该任务属于特定领域（如 UI 拼包、配置表、战斗框架开发），必须先查阅并遵循该领域专属的专业 Skill（如 `slg-ui-edit`、`slg-config-edit` 等）。**

如果 Spec 中要求了单元测试，必须采用 TDD 方式（先写测试，后写实现）。

**强制检查点：**
- 异步操作使用 UniTask，不用 Coroutine/Task
- 新增 `MapUnitData` 派生类必须加 `[MapPoolable]` 特性
- Module 访问通过 ModuleContext，不用 GameContext
- 配置表字段名必须查阅 `Assets/Scripts/Configs/Gen/` 生成代码
- ViewExt 必须是 partial 类

**代码变动后：**
```
refresh_unity(compile="request") → read_console 确认无 error
```

---

## Step 5：验收自查

对照 spec「验收标准」逐条检查：
- 如果 Spec 包含单元测试要求，必须确保测试运行 Pass
- 未通过的项说明原因，视情况修复或标记为已知偏差
- 全部通过后进入 Step 6

---

## Step 6：代码 Review

唤起 `agent-quality-checker` 子 Agent（或由主控亲自执行），触发 `slg-code-review` skill，完成两阶段审查：
1. spec 符合性审查
2. 代码质量审查（SLG 项目规范，包含架构与通用性评估）

如果 Review 发现问题，必须完成修复。Review 彻底通过后才能进入 Step 7。

---

## Step 7：完成反馈（必须执行）

### 7a. 更新任务拆解表

找到对应行，更新以下字段：
- `状态` → `100%`
- `负责人` → 辛杰
- `预估完成时间` → 改为实际完成日期 `YYYY/M/D`

### 7b. 写回 Spec 文件

填写 spec 末尾的 `## 完成记录` 节：

```markdown
## 完成记录

> 完成时间：YYYY-MM-DD | 实际耗时：X天

### 实际偏差
（根据情况处理）
```

偏差处理规则：

| 情况 | 操作 |
|------|------|
| 与计划基本一致 | 删除 `### 实际偏差` 整节 |
| 小偏差（不影响其他任务） | 一句话说明 |
| 重大偏差（影响后续任务设计） | 说明原因，加 ⚠️，并在任务拆解表备注列注明 |
| 发现 spec 有错误 | 直接修正 spec 错误描述，不记录为偏差 |

---

## Step 8：AI 文档同步（自动触发）

任务完成且反馈写回后，若涉及核心逻辑或新机制，建议触发 `slg-doc-sync` skill：
- 分析本次任务的代码变更，更新关联的架构图、feature-map 或 system-overview。
- 如果本次任务沉淀了新的通用规则，可建议触发 `slg-rules-from-review` 更新工程约束。

---

## 约束

- **禁止跳过 Step 3 的确认**：未经确认不写代码
- **禁止跳过 Step 7**：完成反馈是工作流的一部分，不是可选项
- **禁止猜测字段名**：配置表字段必须查阅生成代码
- **策划案只读**：任何情况下不修改任何策划相关目录（如 `策划文档/`、`策划案/` 及其子目录）下的文件
