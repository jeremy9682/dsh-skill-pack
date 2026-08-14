---
name: overnight-execution
preamble-tier: 2
version: 1.0.0
description: "⚠️须先获批准。睡觉/离开数小时要无人值守通宵编码：熬夜计划/夜间自动跑/overnight execution/我睡了你接着干。"
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - ScheduleWakeup
  - CronCreate
  - CronList
  - CronDelete
  - TodoWrite
  - Agent
---

# Overnight Execution Playbook

This is the durable mechanism that has run successfully on a production project
(2026-04-24 → 25 night, 6 PRs auto-merged) and was rebuilt 2026-04-25 → 26.
Use it when a user is going off-call for hours and wants continuous, unattended
progress on a backlog.

## When to invoke

User signal | Match
---|---
"我要睡了，[xxx 任务] 继续" | yes
"夜间无人值守 / 自动跑一晚" | yes
"set up overnight / schedule autonomous loop" | yes
"按之前的熬夜计划设置 / 继续熬夜" | yes
"明早起来希望看到 [tasks done]" | yes
User has a 3+ wave backlog and is going off-call | proactively suggest

## What the user must supply

Just one thing: **the task list** (the "waves"). Scope each wave to ~30-60 min of
work with a clear acceptance criterion. Everything else (mechanism, decisions,
schedules) you handle.

## What you must NEVER do during overnight execution

1. **Ask the user any question.** Decisions must be locked pre-sleep (D1-D5
   matrix below). If you hit something not covered, default to "stop the loop +
   write blocker into morning brief." Don't guess and proceed.
2. **Enter plan mode.** You are in execution mode all night. ExitPlanMode only
   happens once, at start. Auto mode stays on.
3. **`git add -A` or `git push --force` to main.** Always explicit paths. Always
   open a PR; merge via `gh pr merge --squash --delete-branch`.
4. **Touch business禁区.** Each project has its禁区 (the project's own protected
   areas, e.g. its write path or data directories). Never write there.
5. **Skip pre-commit hooks** without using the documented escape hatch (`make
   pre-commit-skip SKIP=<id>`) and noting it in the commit message.

## The 4-layer heartbeat

| Layer | Mechanism | Cadence | Purpose |
|---|---|---|---|
| **L1** Hardware | `caffeinate -dimsu -t 32400 &` | one-shot 9h | Prevent macOS sleep. **User runs before sleep.** |
| **L2** Self-pacing | `ScheduleWakeup(delaySeconds=180)` for **FIRST** fire, then `1380` (23 min) | 3 min → 23 min | Primary continuation. **CRITICAL**: first delay MUST be ≤ 5min (~180s) — see "Session-idle timeout trap" below. 23 ≠ 20/30 to avoid global fleet sync at :00/:30. |
| **L3** State file | `.overnight-heartbeat.json` | written each step | Resume across wakeups + dashboard for user. |
| **L4** Cross-session fallback | `CronCreate("3,20,37,54 * * * *", durable=true)` | 17 min | Survives Codex session crash (in theory; in practice often session-only — but harmless redundancy). |

### ⚠️ Session-idle timeout trap (the #1 silent killer)

**Observed 2026-04-26 → 27 night**: scheduled L2 with `delaySeconds=1380` (23min)
as first fire. Set up at 17:43Z, session went idle at 17:44Z, **first L2 wake
never fired**. By the time 1380s elapsed (~18:06Z), the Codex session had
**disconnected** because no turn happened in the gap. Once disconnected,
**all session-only schedules die** (L2 ScheduleWakeup AND L4 Cron, both —
`durable=true` is documented but practically session-bound). Watchdog: zero
fires across 11 hours of "overnight". User wakes up to a single inline-shipped
wave + 9 untouched.

**Compare to working 2026-04-25 → 26 night** (same skill, same machine, 13/13
waves shipped): the working pattern fired L2 ~11 times because the chain
**self-sustained** — each fire produced a turn that reset the session timeout
clock before the next fire. But that only works if **the first fire happens
before the session times out**.

**Mitigation (mandatory for the SOP)**:

1. **First `ScheduleWakeup` delay = 180s** (3 min — well below idle timeout).
2. **Subsequent delays from inside the watchdog tick = 1380s** (production
   cadence — each fire resets timeout, so 23min is fine after the chain is
   warm).
3. **End-to-end probe**: schedule a one-shot cron at +6 to +10 min (NOT inside
   the 5-min dedup window). This fires AFTER the user-active session goes
   idle, proving the chain actually starts cold. If probe fires and writes
   to heartbeat.notes → SOP verified. If user comes back and probe didn't
   fire → schedule died, abort overnight, escalate.

**The skill's previous mistake**: the probe at "+3min from setup" was within
the dedup window AND within the user's still-active session. It only proved
"in-session crons fire", not "schedule survives session-idle". That's why
the SOP looked verified but the actual overnight failed.

### 5-minute dedup guard

Both L2 and L4 fire the SAME watchdog prompt. Without a guard they'd execute the
same wave twice. Each watchdog tick:

1. Read `last_fired_at` from heartbeat.
2. If `now - last_fired_at < 300s` → write `[$timestamp] noop (dedup)` to
   `notes`, exit.
3. Else continue.

## The 5 D-decision matrix (lock before sleep)

Pick once; the watchdog never re-asks:

| # | What to decide | Common values | Effect |
|---|---|---|---|
| **D1** | Merge strategy | A=manual / B=keep PR open / **C**=auto-merge if codex 0 P + tests green | Whether watchdog runs `gh pr merge` itself |
| **D2** | Wave scope discipline | A=expand if easy / **B**=strict scope, no expansion | Limits scope creep mid-night |
| **D3** | Dependent-wave gating | A=run all even if predecessor failed / **B**=skip dependents on failure | Shortens blast radius of one bad wave |
| **D4** | P1 fix strategy | A=record only / **B**=best-effort Fix-First, >3 attempts → revert / C=always revert | How aggressive is auto-Fix-First |
| **D5** | Optional / nice-to-have waves | **A**=skip if T+Nh reached / B=always do | Hard time-box for stretch goals |

A recent project profile was D1=C, D2=C+D, D3=B, D4=B, D5=A. A narrower
follow-up profile used D1=C, D2=B, D3=B, D4=B, D5=A. For new projects,
default to D1=C, D2=B, D3=B, D4=B, D5=A unless user says otherwise.

## Heartbeat schema

`.overnight-heartbeat.json` (in repo root, excluded via `.git/info/exclude` — NOT
`.gitignore`, to keep the ignore file clean):

```json
{
  "schema_version": 2,
  "session_start": "ISO timestamp",
  "last_fired_at": "ISO timestamp (5-min dedup pivot)",
  "next_wake_eta": "ISO timestamp",
  "phase": "round-N-step-M | done",
  "current_wave": "A | B | ... | done",
  "current_step": 1-9,
  "agents_running": ["wave-X"],
  "agents_completed": ["wave-A", "wave-B"],
  "blocked_waves": [{"wave": "C", "reason": "P0 codex finding"}],
  "last_commit": "git sha",
  "last_pr": {"number": 111, "state": "merged"},
  "vitest_count": 230,
  "branch": "main | feature/x",
  "main_synced": true,
  "d_decisions": {"D1": "C — auto-merge", ...},
  "stop_conditions": [...],
  "notes": "free-text recent events; dedup-guard noops also append here"
}
```

## The 9-step wave template

Every wave executes:

```
1. Read .overnight-heartbeat.json + plan file → know current_wave / current_step
2. cd $REPO && git checkout main && git pull --ff-only origin main
3. git checkout -b <wave_branch_name>
4. Implement scoped changes (per Wave entry in plan)
5. Run test suite — must be 100% green (vitest / pytest / make test-fast / etc.)
6. Spawn `Agent({subagent_type:'codex:codex-rescue'})` background review
7. Wait for codex output. Apply Fix-First per D4. Re-test.
8. git add <explicit paths>; git commit; git push -u origin HEAD; gh pr create
9. Per D1: if codex P0=0 + green → gh pr merge <num> --squash --delete-branch + sync main
10. Update heartbeat (current_wave to next; bump last_commit/last_pr/vitest_count;
    set next_wake_eta = now + 23min); ScheduleWakeup again with delaySeconds=1380.
```

## Stop conditions (hard fail-safe)

The watchdog exits the loop and writes the morning brief if ANY:

1. All planned waves done (`current_wave == "done"`).
2. **2 consecutive waves fail** (vitest red, codex P0, merge blocked).
3. Local time ≥ 06:30 user-local (be done before user wakes).
4. Need a human decision the matrix doesn't cover → write blocker into brief.
5. Anthropic rate limit hits 3 times in a row → buffer can't catch up.

Stop = **don't call ScheduleWakeup again**. The Cron will keep firing for 7 days
unless you also `CronDelete` (do this when writing the morning brief).

## Morning brief format

`docs/plans/overnight/morning-brief-<DATE>.md` — committed in-repo (NOT `/tmp/`):

```markdown
# Morning Brief · 2026-04-DD

## 夜间成果
- Wave A/B/C/...: ✅ / ❌ summary

## PR 状态表
| PR | Merge SHA | 内容 |
| ... |

## 测试矩阵
- vitest: N (was M)
- test-fast: ...

## 决策守则审计 (实际 vs 计划)
| # | Plan | Actual | Notes |
| D1 | C | C | 0 P-findings on all PRs |
...

## 待你早上 ACK
- [ ] ...

## 下一步建议
- ...
```

## Pre-sleep checklist (tell the user this)

```
1. Lock the 5 D-decisions (or accept defaults)
2. Tell me which waves to run
3. Run in another terminal:
     caffeinate -dimsu -t 32400 &
4. Sleep
5. Morning:
     cat $REPO/.overnight-heartbeat.json
     git -C $REPO log main --oneline -20
     cat $REPO/docs/plans/overnight/morning-brief-<tomorrow>.md
     kill %1   # stop caffeinate
```

## Wakeup prompt template (verbatim, self-contained)

The L2 ScheduleWakeup and L4 Cron prompt look like this. Note: the prompt is
self-contained because each wakeup is a fresh Codex turn and must operate
without conversation history.

```
[OVERNIGHT WATCHDOG · L<N> tick]

You are mid-overnight on <PROJECT>. Plan: <PLAN_PATH>. Heartbeat: <HEARTBEAT_PATH>.
5 D-decisions locked. DO NOT ask user. DO NOT enter plan mode.

STEP 1 · DEDUP (5-min noop):
  - Read heartbeat. If last_fired_at < 5 min ago → append "[$ts] L<N> noop (dedup)"
    to notes, stop.

STEP 2 · STOP CHECK:
  - Local time ≥ 06:30 OR current_wave=="done" OR blocked_waves.length≥2 OR
    rate-limit hit ≥3 times → write morning brief at <BRIEF_PATH>, commit, push,
    CronDelete <L4 id>, stop. Do NOT schedule another wakeup.

STEP 3 · DO ONE WAVE (9-step template).

STEP 4 · SCHEDULE NEXT (L2 only — L4 doesn't reschedule itself):
  - ScheduleWakeup(delaySeconds=1380, prompt=this-prompt-verbatim).

NEVER push --force. NEVER touch <BUSINESS禁区>. NEVER `git add -A`. Auto mode.
```

## Common failure modes and recovery

| Failure | What happens | Recovery |
|---|---|---|
| Codex session crashes mid-wave | L2 dies. L4 might also die (durable flag often a no-op). | User opens new Codex session in morning; reads heartbeat; resumes manually OR /skill overnight-execution + "resume". |
| MacBook sleeps | Wakeups never fire. | User MUST run caffeinate. Reminder is in pre-sleep checklist. |
| Anthropic rate limit | Wave fails. | D4=B retries up to 3x; then revert that wave; brief notes the gap. |
| Vitest red after Fix-First | D4 says revert. | `git revert <commit-range>` + brief. |
| Pre-commit hook wedge | Commit blocked. | `make pre-commit-skip SKIP=<id>` + write to commit message. |
| Disk full / docker dead | Static tests still run; only live verify breaks. | Skip live verify; brief notes it. |
| Multiple wakeups same minute | L2 + L4 collide. | 5-min dedup guard catches it; second one no-ops. |
| **Background sub-agent stalls mid-wave** (10-min stream watchdog timeout) | Sub-agent dispatched via `Agent({run_in_background:true, isolation:'worktree'})` may stall during long codex review or test loops. The worktree usually has the code + tests written but **no commit**. | (1) `git worktree list \| grep agent-<id>` to find stalled worktree; (2) `git -C <wt> status --short` + `git -C <wt> diff --stat` to assess progress; (3) `git -C <wt> rebase origin/main` to refresh; (4) main session takes over: run tests, run codex review (or `/code-review` if codex stuck), Fix-First per D4=B, commit, push, PR, merge. Treat the work as ~80% done — don't restart from scratch. |
| **Codex CLI itself hangs / refuses to review** | Agent calls to `codex:codex-rescue` time out or return errors. | Fallback chain: try `code-review:code-review` plugin instead (different runtime), then manual self-review against the wave's acceptance criteria. Don't block merge on review tooling — record the gap in heartbeat.notes + morning brief and ship if tests are green. |

## Setup script (when user invokes the skill)

When the user invokes this skill with task list, do this:

1. Confirm or default the 5 D-decisions.
2. Write `.overnight-heartbeat.json` (schema above) at repo root.
3. Add to `.git/info/exclude` (NOT `.gitignore`).
4. Write `docs/plans/overnight/overnight-plan-<DATE>.md` with:
   - Context (T+0)
   - 5 D-decisions
   - Waterfall (T+0 through T+8h)
   - Wave details (one section per wave)
   - 9-step template reference
   - Heartbeat schema reference
   - Stop conditions
   - Pre-sleep checklist for user
5. Schedule L2 with **`delaySeconds=180`** (3 min — short first delay) with the
   watchdog prompt. The watchdog itself reschedules the NEXT wake at 1380s
   inside its STEP 4.
6. Schedule L4 (CronCreate, "3,20,37,54 * * * *", durable=true, recurring=true)
   with same watchdog prompt minus the "schedule next" step.
7. **End-to-end probe (NEW SOP, mandatory)**: schedule a one-shot `CronCreate`
   that fires at +6 to +10 min (NOT inside the 5-min dedup window from the
   heartbeat write). The probe's prompt instructs it to: (a) read heartbeat,
   (b) write `[$ts] ✅ post-idle probe fired — overnight chain verified` to
   notes, (c) NOT execute any wave. The user observes this note before
   sleeping. If the note is missing 10 min after setup, the chain is dead;
   abort overnight or restart with shorter first-delay.
8. Confirm caffeinate command to user, exit any plan mode.
9. Optionally start Wave 1 immediately in this session if time permits — but
   note Wave 1 inline keeps the session alive long enough for L2 first wake
   to be at risk-free distance from setup.

## Reference template

A fillable plan skeleton is at `references/overnight-plan-template.md` next to
this SKILL.md. Read it, copy it, replace `<TASK>` placeholders with the user's
backlog, drop into `docs/plans/overnight/overnight-plan-<DATE>.md` in the project.

## Lessons from prior runs (don't re-discover)

- 4 of 8 parallel agents hit rate limit on 2026-04-24. **3 parallel agents is
  the safe ceiling** when running fan-out per round.
- `CronCreate(durable=true)` is documented as session-persistent, but in
  practice often session-only. Treat it as a redundancy layer, not a guarantee.
- 23-minute interval is intentional — 20/30 minutes coincide with global fleet
  sync at :00/:30 and the API congests there.
- Heartbeat `notes` field doubles as a debug log; append, don't overwrite.
- `caffeinate -i &` is sufficient on most Macs but `caffeinate -dimsu -t 32400 &`
  is the belt-and-suspenders form (display, idle, manual sleep, system, user
  active, 9-hour bound).
- Memory writes per round (not just at end) — if Codex session dies
  mid-overnight you still have a paper trail.
- **Background-agent stall pattern** (observed 2026-04-26): sub-agents
  dispatched in parallel may stall after writing diff but before committing
  (codex review hung, or context overflow). Default disposition: **takeover
  worktree** rather than restart. Quick diagnostic in main session:
  `git worktree list \| grep agent-<id>` + `git -C <wt> diff --stat`. The
  diff is usually 90%+ done — rebase, test, codex (or fallback to
  `code-review`), Fix-First, commit, ship. Saves ~30min vs restart.
- **Code-review fallback ladder** when codex unavailable: codex-rescue →
  `code-review:code-review` skill → manual review against wave acceptance
  criteria. Tests-green is the load-bearing gate; review is hardening.
- **First-fire delay rule (post-mortem 2026-04-26 → 27)**: If first
  `ScheduleWakeup` `delaySeconds > ~5min`, the chain almost always dies
  silently. Codex session disconnects when no turn arrives, and once it
  disconnects, ALL session-only schedules (including L4 cron with
  `durable=true`) terminate. The chain can only self-sustain if the FIRST
  fire happens before idle timeout. Subsequent fires from inside the
  watchdog can use 1380s safely because each fire restarts the timer.
  TL;DR: first ScheduleWakeup = `180s`, watchdog STEP 4's reschedule = `1380s`.
- **Probe-during-setup is not a real test**: a probe scheduled at +3min
  fires inside the same active session and only proves "tools work"; it
  does NOT prove "schedule survives session-idle". Always schedule the
  probe at +6 to +10 min, outside any 5-min dedup window AND likely
  outside the user's still-typing window. If probe writes to heartbeat
  notes, the chain works cold-start.
