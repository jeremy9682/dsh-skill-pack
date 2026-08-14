# 夜间无人值守计划 · <DATE> → <TOMORROW> morning

> 复制本文件到 `<REPO>/docs/plans/overnight/overnight-plan-<DATE>.md`，
> 填掉所有 `<...>` 占位符，删除本提示行后承担为正式计划。

## Context (T+0)

- Repo: `<ABSOLUTE_REPO_PATH>`
- Branch: `<base-branch>` = `<commit-sha>`
- Recent shipped this session: `<PR list or "none">`
- Test baseline: `<vitest N / pytest M / make test-fast K>`
- Untouched-area pinning (e.g. local path adaptation files): `<list or "n/a">`
- Live stack state: `<docker / staging URL / "static only">`
- Demo data state worth preserving: `<one-line summary>`

## ✅ 5 锁定决策（夜间不再问用户）

| # | 决策 | 选择 | 含义 |
|---|---|---|---|
| **D1** | 合并策略 | **<A/B/C>** | <e.g. `C` = codex 0 P-findings + tests green → auto-merge> |
| **D2** | Wave 范围纪律 | **<A/B>** | <e.g. `B` = strict scope, no expansion mid-night> |
| **D3** | 依赖 wave gate | **<A/B>** | <e.g. `B` = predecessor failure → skip dependents> |
| **D4** | P1 修复策略 | **<A/B/C>** | <e.g. `B` = best-effort Fix-First, >3 attempts revert> |
| **D5** | 可选 wave 时间盒 | **<A/B>** | <e.g. `A` = T+4h reached → skip optional waves> |

## 禁区（自带保护）

- 只动 `<REPO>`，不碰 `<BUSINESS_AREA_FORBIDDEN>`
- `git add` 用显式 path — 永不 `git add -A`
- `git push --force` 永禁到 main
- pre-commit hook wedge → `make pre-commit-skip SKIP=<id>` + 写 commit message

## 夜间 Waterfall（T+0 = <START_TIME_LOCAL>）

| 时段 | Round | 内容 |
|---|---|---|
| **T+0 → T+1h** | R1 | **Wave A** · <one-line scope> |
| **T+1h → T+2.5h** | R2 | **Wave B** · <one-line scope> |
| **T+2.5h → T+4h** | R3 | **Wave C** · <one-line scope> |
| **T+4h → T+5h** | R4 | (D5 gate) **Wave D/E** · <optional scope> |
| **T+5h → T+6h** | R5 | 全量 codex review + Fix-First 跨所有夜间 commits |
| **T+6h → T+7h** | R6 | memory write + 晨报 commit + 收官 |
| **T+7h → T+8h** | Buffer | rate limit recovery / 漏修 / 意外 |
| **T+8h** (`<HARD_STOP_TIME>`) | STOP | 等用户起床 |

**早停条件**（任一触发立即写晨报 + 停 wakeup chain）：

1. 所有 Round 完成
2. 2 个连续 Round vitest red 或 codex P0
3. 本地时间 ≥ `<USER_WAKE_HOUR>`
4. 任何需要人决定的问题（写 morning brief blockers + 停）
5. Anthropic rate limit 命中 3 次

## 9-step 执行模板（每个 wave 跑一遍）

```
1. 读 .overnight-heartbeat.json + 本计划文件 → 知道 current_wave/step
2. cd <REPO>; git checkout main && git pull --ff-only origin main
3. git checkout -b <wave_branch>
4. 实现 wave 范围内的改动（per Wave 详情 below）
5. 跑测试套件 — 必须 100% 绿（<test command>）
6. Agent({subagent_type:'codex:codex-rescue', ...}) 后台 review
7. 等结果；P1 按 D4 处理；P0 留人审；重测
8. git add <显式 path>; git commit; git push -u origin HEAD; gh pr create
9. 按 D1 处理 merge：codex 0 + 绿 → gh pr merge --squash --delete-branch + sync main
10. 写 heartbeat + status notes; ScheduleWakeup 下一 round
```

## Wave 详情

### Wave A · <name> (~<estimate>min)
- Branch: `<branch_name>`
- Scope:
  - <step 1>
  - <step 2>
- 验证：`<command>`
- 接受标准：`<acceptance criterion>`

### Wave B · <name> (~<estimate>min)
- Branch: `<branch_name>`
- Scope:
  - <...>

### (continue for each wave)

## 🫀 Heartbeat / 防中断（4 层防护）

| 层 | 机制 | 实施 |
|---|---|---|
| **L1 硬件** | `caffeinate -dimsu -t 32400 &` | **用户睡前手动**（新终端 tab；早上 `kill %1`） |
| **L2 Claude 自 pacing** | `ScheduleWakeup` | 每 **23 min** (`delaySeconds=1380`) 自唤醒；prompt 是完整自包含 watchdog（见下） |
| **L3 文件心跳** | `.overnight-heartbeat.json` | repo root（`.git/info/exclude` 排除） |
| **L4 外部 Cron 兜底** | `CronCreate("3,20,37,54 * * * *", durable=true, recurring=true)` | 跨 session 触发；prompt 同 L2 但 STEP 4 不再 ScheduleWakeup |

**5-min 噪声门**：每次唤醒读 `last_fired_at`，距上次 < 5 min 直接 noop 返回。

## Heartbeat schema

```json
{
  "schema_version": 2,
  "session_start": "<ISO>",
  "last_fired_at": "<ISO>",
  "next_wake_eta": "<ISO>",
  "phase": "round-N-step-M | done",
  "current_wave": "A | B | ... | done",
  "current_step": 1-9,
  "agents_running": ["wave-X"],
  "agents_completed": ["wave-A"],
  "blocked_waves": [],
  "last_commit": "<sha>",
  "last_pr": {"number": <N>, "state": "open | merged | closed"},
  "vitest_count": <N>,
  "branch": "main | feature/x",
  "main_synced": true,
  "d_decisions": {"D1": "...", "D2": "...", "D3": "...", "D4": "...", "D5": "..."},
  "stop_conditions": [...],
  "notes": "<append-only event log; dedup-guard noops also append here>"
}
```

## Watchdog wakeup prompt（L2 + L4 共用，verbatim 自包含）

```
[OVERNIGHT WATCHDOG · L<N> tick]

You are mid-overnight on <PROJECT>. Plan: <PLAN_PATH>.
Heartbeat: <REPO>/.overnight-heartbeat.json. 5 D-decisions locked.
DO NOT ask user. DO NOT enter plan mode.

STEP 1 · DEDUP (5-min noop):
  - Read heartbeat. If last_fired_at < 5 min ago → append "[$ts] L<N> noop (dedup)"
    to notes, stop.

STEP 2 · STOP CHECK:
  - Local time ≥ <USER_WAKE_HOUR> OR current_wave=="done" OR
    blocked_waves.length≥2 OR rate-limit hit ≥3 times → write morning brief at
    <REPO>/docs/plans/overnight/morning-brief-<TOMORROW>.md, commit, push,
    CronDelete <L4 id>, stop. Do NOT schedule another wakeup.

STEP 3 · DO ONE WAVE (9-step template).

STEP 4 · SCHEDULE NEXT (L2 only — L4 doesn't reschedule):
  - ScheduleWakeup(delaySeconds=1380, prompt=this-prompt-verbatim).

NEVER push --force. NEVER touch <BUSINESS_AREA_FORBIDDEN>. NEVER `git add -A`.
Auto mode.
```

## 晨报（`docs/plans/overnight/morning-brief-<TOMORROW>.md`）

完成所有 round 或触发停机后**立即**写：

```markdown
# Morning Brief · <TOMORROW>

## 夜间成果
- Wave A: ✅ / ❌ summary
- Wave B: ...

## PR 状态表
| PR | Merge SHA | 内容 |
| #<N> | `<sha>` | <one-line> |

## 测试矩阵
- vitest: <N> (was <M>)
- test-fast: ...

## 决策守则审计 (实际 vs 计划)
| # | Plan | Actual | Notes |
| D1 | C | C | 0 P-findings on all PRs |
| ... |

## 待你早上 ACK
- [ ] ...

## 下一步建议
- ...
```

## 用户睡前需要做的 1 件事

```bash
# 新终端 tab — 防 macOS 睡眠
caffeinate -dimsu -t 32400 &

# 早上起床后
kill %1   # 或 Ctrl+C

# 一眼看全貌
cat <REPO>/.overnight-heartbeat.json
git -C <REPO> log main --oneline -20
cat <REPO>/docs/plans/overnight/morning-brief-<TOMORROW>.md
```
