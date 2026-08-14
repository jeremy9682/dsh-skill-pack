---
name: dsh-mode-routing
description: >
  DSH 会话模式（agent preset）选型与路由规则。四套内置 preset：standard（标准）、code（PTC）、
  minimal（极简）、cordis（创造）。当用户问"用哪个模式""开什么模式""什么情况用 PTC/极简/标准/
  创造模式""DSH 模式怎么选"，或任务明显需要换 preset（大批量机械迁移、通宵任务、定制 DSH 自身、
  写插件/preset）时使用。核心事实：组装发生在会话边界、编排发生在会话内部；会话一旦有产出
  preset 即锁定（agent-preset-locked），需要不同 preset 就开新会话。本 skill 管"挂哪套工具"；
  若你还有流程编排类 skill（如 dev-workflow），两者叠加使用。
---

# dsh-mode-routing — DSH preset 选型与路由

> 事实源：deepseek-harness 仓库 `packages/preset/agent-presets/README.zh.md`、
> `apps/cli/config/agent-presets/*/`、`packages/bundle/web-app/cordis.patch.yml`。

## 一句话原则

**组装（composition）发生在会话边界，编排（orchestration）发生在会话内部。**
preset 在会话创建时挂载（早于第一个请求），会话一旦产出内容即锁定——网关在传输层执行
`agent-preset-locked`，中途换工具集会留下新组装无法执行的历史调用，破坏日志重建。
所以：**需要不同 preset = 开新会话**，而不是让 Agent"切模式"。只有空白会话可用
`recompose()` 切换，且切换以 `agent-preset/selected` 会话事件入日志。

## 三层概念，别混

1. **运行模式**：`dsh web`（Web UI）/ `dsh --profile headless "任务"`（一次性）/ `dsh plugin
   --profile <name> <pnpm 参数>`（插件管理）——启动时定。
2. **Agent preset（会话模式）**：standard / code(PTC) / minimal / cordis(创造)——新建会话时选。
3. **会话内状态**：plan mode、todo、goal、后台 job、skill 路由——会话进行中切换，不受锁定约束。

## 四个 preset 速查决策表

| preset (id) | 界面名 | 给谁用 | 典型任务 | 禁用/红线 |
|---|---|---|---|---|
| standard | 标准模式 | 默认主力，90% 场景 | Mode 0 探雷（只读侦察+plan mode）、research 汇总裁决、review 主脑直审、日常编码、主脑编排 | — |
| code | PTC 模式 | 大批量、机械、可一次打包的多步操作 | 跨 >5 文件的机械迁移/重命名、规则先行批量清洗、一次性验证脚本 | 禁区改动；需逐步审批/观察的交互调试；小任务（SDK 文本固定 token 开销不划算） |
| minimal | 极简模式 | 短平快、模板化、省 token、行为最可预测 | 跑脚本写文件、prompt A/B、大量重复小会话 | 长任务/通宵（无 compaction 会爆上下文）；需要 skills/plan/subagent/goal 的任务 |
| cordis | 创造模式 | 定制 DSH 本身 | 写/复制自定义 preset、开发插件、读改运行时 | 日常业务代码；cordis_mount 在活运行时执行模型写的 JS = shell 权限 |

**自定义 preset（用户自建，示例）**：除四套内置外，可按项目复制出自建 preset（例如面向某个具体
领域的「标准」与「批量」两档）。自建 preset 的 persona 可内置：skill 路由、锁定业务规则、禁区探雷
要求、执行席约定、Conventional Commits 提交纪律。任务提到某具体项目/领域的关键词时，建议用对应的
自建 preset（机械批量 → 其批量档）开会话，而不是裸 standard。

### 各 preset 机制要点（报原因时引用）

- **PTC（code）**：standard 全部行原封不动 + 一行 `tool-presentation`（`mode: code`）。模型只见
  `run_code` + 生成的 TypeScript SDK，一次程序跑多步；子调用仍过审批管道（tools/pre-execute
  门禁仍在），但中间结果 log-only、人类可读性差；`run_code` 每次状态清零。当前 Web 部署可用
  （web-app 组合包已挂 `code-runtime-worker-thread`）。`DSH_TOOLS_MODE` 未设置时进程默认 native，
  preset 的 `mode: code` 按 agent 覆盖。
- **极简（minimal）**：persona `complete: true`（系统提示词钉死一句，外部无法注入，运行时上下文
  快照关闭）+ 仅持久 bash 与 str_replace_editor + 无 compaction。提示词固定是特性也是限制。
- **创造（cordis）**：standard + `tool-cordis` 自修改工具集（inspect/define/run/mount）+ 两个
  内置 skill（editing-cordis-compositions、cordis-plugin-development）。信任级别 = shell。
- **内置四 preset 只读**（升级覆盖安装目录）；自定义 = 复制到 `$DSH_HOME/.agent-presets/<id>/`，
  或用创造模式让 Agent 生成。

## 硬规则（Agent 必须遵守）

- **禁区**（写路径 / 支付 / 记账 / 结算等不可逆区域）→ 永远标准模式，一步一步、每步可审；
  **禁止 PTC**。
- 任务需要换 preset → 不要"代劳"，开头一句话报"建议用 X 模式新开会话，原因：…"，并给出可直接
  粘贴的任务清单；用户开好新会话后在那边继续。
- DSH 的 `subagent`/`workflow` 派的是**本地 DSH 子代理，不是外部云执行席**——执行席按你自己的
  流程编排约定；DSH 当主脑/编排/分析/探雷/文档席。
- 通宵/长任务 = 标准模式 + goal（或 schedule 工具）+ 后台 job；**不用极简模式**。
- 建议 PTC 时同时提醒：把任务写成可校验清单（N 步、每步的断言），因 run_code 状态清零、
  中间结果只进日志。

## Agent 行为协议（每次任务开始时）

1. 判断任务形状 → 对照决策表选推荐 preset。
2. 当前会话 preset 不匹配 → 一句话建议 + 原因 + 任务清单；匹配 → 直接干。
3. 批量机械任务建议 PTC → 附可校验清单提醒。
4. 用户问"能不能自动切模式"→ 按本 skill 第一段回答：不能（有意设计），编排在会话内做。

## 与其他流程类 skill 的分工

流程编排类 skill（如 dev-workflow）管"流程怎么走"（research→implement→review→ship、
禁区探雷、场景确认门）；本 skill 管"DSH 会话挂哪套工具"（preset 选型）。两者叠加，不互相取代。
