# 子 Agent 动态权限与分档方案

## 问题背景

Antigravity 框架在 IDE 启动时会扫描 `.agents/agents/` 目录下的 `.md` 文件，静态加载预设子 Agent。
该静态解析流程存在一个系统级 Bug：**即使文件 Frontmatter 中声明了 `enable_write_tools: true`，子 Agent 在实际派发后依然不具备 `write_to_file` 和 `run_command` 工具**，导致它们无法执行代码修改、编译脚本和 Git 提交。

## 验证过程

通过对照实验确认了该 Bug 的存在：

| 加载方式 | Agent 名称 | enable_write_tools | 实际是否拥有写工具 |
|---------|-----------|-------------------|-----------------|
| 静态加载（`.agents/agents/` 目录） | `agent-slg-dev-fast` | `true` | 否 |
| 动态注册（`define_subagent` API） | `agent-slg-dev-fast` | `true` | **是** |

结论：通过 `define_subagent` API 在运行时动态注册的 Agent，能够正确继承 `enable_write_tools` 参数，绕开静态解析 Bug。

## 解决方案

### 核心思路：物理隔离 + 动态加载

将需要写入权限的三个开发核心 Agent 从静态加载目录中移出，改为由技能脚本在运行时按需读取模板并动态注册。

### 目录结构

```
.agents/
├── agents/                      # IDE 静态加载目录（仅保留无需写入权限的 Agent）
│   ├── agent-config-knowledge-maintainer.md
│   ├── agent-config-table-expert.md
│   ├── agent-fairygui-editor-plugin.md
│   ├── agent-fairygui-plugin-dev.md
│   ├── agent-fairygui-ui.md
│   ├── agent-quality-checker.md
│   └── agent-slg-dev.md
└── agent_templates/             # 模板目录（IDE 不会扫描此目录）
    └── agent-slg-dev-base.md    # 唯一的公共基础模板
```

### 三卡合一（DRY 优化）

原本 `agent-slg-dev-fast.md`、`agent-slg-dev-mid.md`、`agent-slg-dev-ultra.md` 三个文件的系统提示词高度重合（铁律、定位方式、开发流程、完成规范完全一致），唯一区别仅在于 Frontmatter 中的 `model` 字段。
因此将三份文件合并为一份 `agent-slg-dev-base.md`，仅保留通用的提示词正文。模型分档在动态注册时通过参数注入实现。

### 动态加载流程

由 `parallel-tasks` 技能在每次派发任务前自动执行：

```
1. view_file 读取 agent_templates/agent-slg-dev-base.md
   └─ 提取 Frontmatter 之后的提示词正文

2. define_subagent 动态注册（按需，可并发）
   ├─ name: agent-slg-dev-ultra
   │   ├─ enable_write_tools: true
   │   └─ model: "Gemini 3.1 Pro (High)"
   ├─ name: agent-slg-dev-mid
   │   ├─ enable_write_tools: true
   │   └─ model: "Gemini 3.1 Pro (Low)"
   └─ name: agent-slg-dev-fast
       ├─ enable_write_tools: true
       └─ model: "Gemini 3.5 Flash (Medium)"

3. invoke_subagent 派发任务
   └─ TypeName 使用原名称（如 agent-slg-dev-ultra）
```

### 动态注册的模型继承机制（局限性）

经过交叉验证确认，Antigravity 内置的 `define_subagent` 工具**不支持**指定物理模型。
虽然传入 `model` 字段不会导致 API 报错，但底层实际会忽略该参数。任何通过动态注册衍生的子 Agent，都会**强制继承当前主控会话的模型算力**。
- 如果主控切到了 Claude Sonnet，所有子 Agent 都是 Claude Sonnet。
- 如果主控切到了 Gemini Pro High，所有子 Agent 都是 Gemini Pro High。

因此，环境 A（Antigravity 引擎）下，虽然完美解决了写权限和模板冗余问题，但**目前暂时牺牲了物理算力的分档降级能力**（任务仍然会有逻辑分工的差异，只是成本无法拆分）。未来如需严格控费降级，需考虑退化为使用命令行拉起独立 `agy` 进程（环境 B）的模式。

### 三级业务路由规范

| 档次 | Agent 名称 | 模型 | 适用任务类型 |
|-----|-----------|------|------------|
| Ultra（顶级） | `agent-slg-dev-ultra` | Gemini 3.1 Pro (High) | 底层框架、核心状态机、全局调试、大型系统集成 |
| Mid（中坚） | `agent-slg-dev-mid` | Gemini 3.1 Pro (Low) | 复杂业务逻辑、Controller、核心机制实现 |
| Fast（极速） | `agent-slg-dev-fast` | Gemini 3.5 Flash (Medium) | UI 面板配置、单点逻辑、配置表修改 |

## 方案优势

1. **免疫 Bug**：彻底绕开静态解析丢失写入权限的系统级缺陷，跨 IDE 重启稳定生效
2. **单模板维护**：修改项目开发规范只需编辑一份 `agent-slg-dev-base.md`，所有档次自动同步
3. **热更新生效**：模板修改后无需重启 IDE，下次 `/parallel-tasks` 派发时自动读取最新版本
4. **精准控费**：通过隐藏 model 参数实现算力分档，避免所有任务都消耗顶级模型额度
5. **命名无感**：派发和引用时依然使用原名称，对上层技能和用户完全透明
