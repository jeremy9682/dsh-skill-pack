---
name: dsh-dispatch
description: >
  Model and DSH-preset dispatch for implementation work: which coding agent,
  model, effort, reviewer, and DSH preset (standard/code/minimal/cordis) to use.
  Use when choosing Composer vs Grok 4.6 vs Codex vs Opus vs DSH, when the user
  asks 派给 dsh / 用哪个模型 / 第二梭, or before opening parallel sessions.
---

# dsh-dispatch — 模型分派（portable）

Canon: `~/Projects/agent-skill-advisor-layer/routing-policy.yaml`
Prose matrix: `~/Projects/agent-skill-advisor-layer/docs/model-dispatch-matrix.md`
Preset rules: sibling skill `dsh-mode-routing`. Quota gateway: LiteLLM, not this skill.

Product default is **Grok 4.6** (ACP `grok-4.6`, CLI `cursor-grok-4.6-high-fast`). 4.5 is explicit downgrade only.

| Task | Exec | Review | Forbid |
|---|---|---|---|
| mechanical / scan | Composer 2.5 Fast; 2nd shuttle Grok 4.6 Fast | dual-seal if gated | flagship writers |
| ordinary_bug_fix | Codex Terra or Claude Sonnet | independent Codex OK | Sol writes **and** reviews |
| standard_feature | Opus direction; Grok 4.6 or DSH Kimi K3 land | Sol | Composer as behavior author |
| judgment | Opus high careful; disputes Fable Max | cross-family | DSH as judgment seat |
| restricted_zone | Opus, no downshift | Sol xhigh; Codex producer → Fable | DSH self-GO; Composer final; PTC/`code` |
| arbitration | `claude-fable-5` max | this seat | producer family as arbitrator |
| parallel subtasks | Composer Fast ∥ Grok Fast; Cursor family ≤2 | after land | bulk new DSH Beijing 09–12 and 14–18 |
| retrieval | Luna / DeepSeek Flash / Gemini Flash | none | flagship writers |
| docs | Grok 4.6 or Sonnet or GLM | Sol if gated | Composer changing meaning |

DSH preset: `standard` default; `code` (PTC) only for >5-file mechanical batch; `minimal` for tiny template jobs; `cordis` only when editing DSH itself. Restricted zone = `standard` only. Need a different preset = **new session**.

ZCode/Cursor HTTP RPC for `127.0.0.1:3080` lives in the machine skills
`~/.agents/skills/dsh-dispatch` and `~/.cursor/skills/dsh-local-dispatch`.
This pack copy is the portable dispatch table, not a second policy YAML.
