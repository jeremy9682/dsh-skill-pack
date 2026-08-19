# dsh-skill-pack

一套可分享的 **13 个工作流 skills**，面向 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（`dsh`）。一个插件、一个 `FileSystemSkillProvider`、十三个 skill。

本包是单包 `dsh` 插件（与 Hermes / DSH 社区 bundle 同一形态）。安装后注册一个文件系统 skill provider，指向包内自带的 `skills/` 目录——**叠加**在你自己的项目与用户 skills 之上，绝不替代它们。

## 要求

- `dsh` >= **0.1.0-rc.6**

本包**不声明**任何 `@deepseek-ai/*` 依赖。与其他 DSH bundle 一样，官方包（`@deepseek-ai/dsh-skill-filesystem` 等）由 profile 闭包在启动时注入，因此插件从 profile 自身的 `node_modules` 解析它们。

## 13 个 skill

| Skill | 一句话介绍 |
| --- | --- |
| `dsh-dispatch` | 任务类型 → 模型/effort/评审/DSH preset；产品默认 Grok 4.6。细节见 ADV 仓 `docs/model-dispatch-matrix.md`。 |
| `dsh-mode-routing` | DSH 会话模式（agent preset）选型与路由——四套内置 preset（`standard`/`code`/`minimal`/`cordis`）+ 用户自建 preset。 |
| `handoff` | 把当前对话压缩成交接文档，交给另一个 agent 接手。 |
| `triage` | 让 issue 与外部 PR 走过 triage 状态机——分类、验证、质询、产出 agent-ready brief。 |
| `to-spec` | 把当前对话变成一份 spec 并发布到项目 issue tracker。 |
| `to-tickets` | 把计划/spec/对话拆成 tracer-bullet tickets，每个都声明自己的阻塞边。 |
| `wayfinder` | 把超大块工作规划成 issue tracker 上的一张决策 ticket 共享地图。 |
| `wait-what` | 停——上一条消息没被理解，重述一遍。 |
| `teach` | 在工作区内教你一个新技能或概念。 |
| `ask-matt` | 问哪个 skill 或 flow 适合当前局面——共享 skills 集 + 用户自己 skills 之上的路由。 |
| `overnight-execution` | ⚠️ 须先获批准。睡觉/离开数小时的无人值守通宵编码。 |
| `full-throttle` | 显式升档协议（仅 `$full-throttle` 调用）：推理深度升档、多 agent/后台并行（须显式传档位）、可选跨家族盲审包。 |
| `skill-advisor` | 主动建议高成本/需许可的 skill（通宵任务、跨会话规划、发布门禁），一句话说明理由——未经明确批准绝不执行。 |

## 安装

三选一。把 `web` 换成你实际使用的 profile（`web`、`headless`、`tui`、`acp`……）。

### npm registry

```sh
dsh plugin --profile web add @jeremy9682/dsh-skill-pack
```

### Git

```sh
dsh plugin --profile web add github:jeremy9682/dsh-skill-pack
```

### 本地路径（file:）

```sh
dsh plugin --profile web add file:/path/to/dsh-skill-pack
```

`dsh plugin` 会把参数转发给 profile 目录里的 `pnpm`，随后重算该 profile 的 bundle 列表，因此任何 `pnpm add` 接受的 spec 都能用。

## 验证

```sh
dsh --profile web --dump-config | grep -A 2 dsh-skill-pack
```

## 卸载

```sh
dsh plugin --profile web remove @jeremy9682/dsh-skill-pack
```

`dsh plugin` 转发给 `pnpm`，`remove` 即 `pnpm remove`（别名 `pnpm rm`）：从 profile 的 `package.json` 与 `node_modules` 移除依赖，随后 `dsh` 把它从 profile 的层列表里去掉。若你的 `dsh` 构建不支持 `remove` 转发，就从 profile 的 `package.json` 里删掉该依赖再重跑 `dsh plugin --profile web install`。

## 工作原理

`index.mjs` 是一个 Cordis 函数插件（`inject: ['skills']`）。其 `apply` 注册一个指向本包 `skills/` 目录的 `FileSystemSkillProvider`：

```js
ctx.skills.registerProvider((control) =>
  new FileSystemSkillProvider(ctx, control, {
    providerName: 'dsh-skill-pack',
    customSkillDirs: [skillDir],
  }),
)
```

`includeDefaultRoots` 保持默认（`true`），共享 skills 以自定义根（rank 300）加入 provider 的发现顺序——位于项目根之下、用户根之上，因此你自己的 skills 照常工作。

## License

MIT — 见 [LICENSE](LICENSE)。
