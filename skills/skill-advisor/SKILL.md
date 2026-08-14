---
name: skill-advisor
description: Suggest high-cost or permissioned skills without executing them. Use for proactive routing before silently launching expensive, disruptive, or gated workflows (overnight runs, multi-session plans, shipping gates, fleet-wide changes). Suggest at most one skill with a one-sentence reason and wait for explicit approval. Never execute a suggested skill from a mere outcome request.
---

# skill-advisor

A small routing layer for valuable skills that should be suggested proactively but never executed without approval.

## Rule

Do not run the target skill automatically. Suggest at most one target skill, explain why in one sentence, then continue the main work unless the user approves.

A user describing a large desired outcome is not approval to start the target skill. Approval must explicitly say to run, start, enable, pair, set up, or launch that specific workflow after the suggestion is made.

If asked whether to execute one of the target skills directly, answer no unless the user has already explicitly approved that target workflow.

Use this format:

```text
This looks like a candidate for <skill> because <reason>. I can run it if you approve.
```

## Suggestion matrix

Personalize this table: add your own high-cost skills as rows and delete rows that do not apply. The rows below cover this pack's high-cost skills.

| Target skill | Suggest when | Do not suggest when |
| --- | --- | --- |
| `overnight-execution` | The user will be away for hours and wants unattended coding to continue. | The user is present, or interactive steering is expected. |
| `wayfinder` | The work is larger than one session can hold and needs a durable decision map across tickets. | The task fits in a single session. |
| `full-throttle` | The user explicitly invokes the protocol (`$full-throttle`) or asks for maximum-depth execution with multi-agent parallelism. | Colloquial "go hard" phrasing — the protocol is explicit-invocation only. |
| `to-spec` / `to-tickets` | A settled design should become a tracked spec, then tracer-bullet tickets, before implementation starts. | The conversation is still exploratory. |
| `<your own skill>` | Describe its trigger condition. | Describe when to stay silent. |

Keep the matrix short: routing value decays when every skill has a row.

## Personalization notes

- The `description` frontmatter is the model-visible surface of this skill; update it when you change the matrix.
- Skills in this pack are model-invocable; the advisor's job is gating the expensive ones, not re-listing every skill.

## Provenance

This is the portable, provider-neutral edition of the Skill Advisor pattern from [agent-skill-advisor-layer](https://github.com/jeremy9682/agent-skill-advisor-layer) (see its `docs/task-routing.md` and `docs/gate-policy.md` for the routing canon and gate semantics this pattern grew from). The upstream repository keeps the machine-specific matrix; this edition ships a template plus rows for this pack.
