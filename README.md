# dsh-skill-pack

A shareable set of **11 workflow skills** for the [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (`dsh`). One plugin, one `FileSystemSkillProvider`, eleven skills.

The pack is a single-package `dsh` plugin (the same shape as a Hermes / DSH community bundle). Installing it registers a filesystem skill provider that points at the `skills/` directory bundled inside this package — **added alongside** your own project and user skills, never replacing them.

## Requirements

- `dsh` >= **0.1.0-rc.6**

This package declares **no** `@deepseek-ai/*` dependency. Like other DSH bundles, the official packages (`@deepseek-ai/dsh-skill-filesystem`, etc.) are injected by your profile's closure at boot time, so the plugin resolves them from the profile's own `node_modules`.

## The 11 skills

| Skill | What it does |
| --- | --- |
| `dsh-mode-routing` | DSH session mode (agent preset) selection and routing — the four built-in presets (`standard` / `code` / `minimal` / `cordis`) plus user-built presets. |
| `handoff` | Compact the current conversation into a handoff document for another agent to pick up. |
| `triage` | Move issues and external PRs through a triage state machine — categorise, verify, grill, and write agent-ready briefs. |
| `to-spec` | Turn the current conversation into a spec and publish it to the project issue tracker. |
| `to-tickets` | Break a plan, spec, or conversation into tracer-bullet tickets, each declaring its blocking edges. |
| `wayfinder` | Plan a huge chunk of work as a shared map of decision tickets on the issue tracker. |
| `wait-what` | Stop — that last message did not land; re-pitch it. |
| `teach` | Teach the user a new skill or concept, within the workspace. |
| `ask-matt` | Ask which skill or flow fits your situation — a router over the shared skills set plus the user's own skills. |
| `overnight-execution` | ⚠️ Requires prior approval. Unattended overnight coding while you sleep or step away for hours. |
| `full-throttle` | Explicit escalation protocol (`$full-throttle` only): deeper reasoning, multi-agent/background parallelism with explicit gears, optional cross-family blind review. |

## Install

Choose one of the three install forms. Replace `web` with whichever profile you use (`web`, `headless`, `tui`, `acp`, …).

### npm registry

```sh
dsh plugin --profile web add @jeremy9682/dsh-skill-pack
```

### Git

```sh
dsh plugin --profile web add github:jeremy9682/dsh-skill-pack
```

### Local path (file:)

```sh
dsh plugin --profile web add file:/path/to/dsh-skill-pack
```

`dsh plugin` forwards its arguments to `pnpm` inside the profile directory and then reconciles the profile's bundle list, so any spec `pnpm add` accepts works here.

## Verify

```sh
dsh --profile web --dump-config | grep -A 2 dsh-skill-pack
```

## Uninstall

```sh
dsh plugin --profile web remove @jeremy9682/dsh-skill-pack
```

`dsh plugin` forwards to `pnpm`, so `remove` is `pnpm remove` (alias `pnpm rm`): it drops the dependency from the profile's `package.json` and `node_modules`, and `dsh` then removes the bundle from the profile's layer list. If your `dsh` build lacks `remove` forwarding, delete the dependency from the profile's `package.json` and re-run `dsh plugin --profile web install`.

## How it works

`index.mjs` is a Cordis function plugin (`inject: ['skills']`). Its `apply` registers a `FileSystemSkillProvider` over this package's `skills/` directory:

```js
ctx.skills.registerProvider((control) =>
  new FileSystemSkillProvider(ctx, control, {
    providerName: 'dsh-skill-pack',
    customSkillDirs: [skillDir],
  }),
)
```

Because `includeDefaultRoots` stays at its default (`true`), the shared skills join the provider's discovery ranks as a custom root (rank 300) — below project roots and above user roots — so your own skills keep working alongside them.

## License

MIT — see [LICENSE](LICENSE).
