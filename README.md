# Agentic Loop Setup

A Claude skill that **actively configures a complete agentic loop for any project**. It interviews you, then generates the real files that let Claude drive itself — plan, build, check its own work, recover from token limits, and improve from every session.

Works in Claude Code, Cowork, and chat (with reduced mechanisms in chat).

## The idea

An agentic loop is the cycle **gather context → take action → verify work → repeat**, plus a persistence layer so lessons survive across sessions.

The one principle behind the whole skill: **Claude stops when work *looks* done.** Without a check it can actually run, "looks done" is the only signal and a human becomes the verification loop. Every setup this skill produces answers one question: *what check can Claude run, and what gates the loop on it?*

## What it generates

Run the skill and it interviews you, picks a loop architecture for your failure mode, routes models per role, then writes:

| File | Purpose |
|---|---|
| `CLAUDE.md` | Auto-loaded memory: build/test commands, conventions, the loop contract, and the end-of-session learning pass |
| `PROGRESS.md` | The loop's working memory — done / failed-and-why / next — and the durable state that makes resume safe |
| `LOOP.md` | Operating instructions: chosen pattern, the check, session-start prompt, stop condition, model routing, budget |
| `scripts/verify.sh` | A runnable pass/fail check (created with you if none exists) |
| `scripts/auto_resume.sh` | Unattended token-exhaustion recovery — waits out a usage-limit reset, then continues from `PROGRESS.md` |
| `.claude/settings.json` | Permission allow/deny list so Claude runs autonomously but can't do anything irreversible |
| `.claude/agents/*.md` | The subagent roster (below) |

## The subagent roster

Each runs in a **fresh, scoped context**, and no agent both produces work and judges it.

- **`plan-critic`** — adversarial critic that runs *after the plan, before any code*. Surfaces unstated assumptions, dismissed alternatives, and edge cases, then returns a verdict (PROCEED / PROCEED WITH CHANGES / RECONSIDER). Replaces the human *plan-approval* gate.
- **`reviewer`** — adversarial correctness reviewer that runs *after implementation* in a context that never saw the code being written. Checks domain correctness, UX, and security — not approach, not style. Replaces the human *merge-approval* gate.
- **`explorer`** — read-only scout that answers one investigation question and returns a compact `path:line` digest, protecting the driver's context on long runs.
- **`verifier`** — mechanical check-runner that reports `PASS` or a triaged `FAIL` list and never fixes, keeping the pass/fail signal honest.

Together with the permission allow/deny list, these remove the gates that normally stall an autonomous run.

## Smart model routing

Loops multiply cost — the same prompt may run hundreds of times — so the skill scores complexity once per role and assigns the cheapest capable tier (Haiku / Sonnet / Opus) with an escalation rule. Critic and reviewer get Sonnet+, explorer and verifier get Haiku, the lead gets Opus or Sonnet by scope. See `references/model-routing.md`.

## Token-limit auto-resume

Claude Code does not yet auto-resume after a subscription usage-limit reset (open feature requests [#36320](https://github.com/anthropics/claude-code/issues/36320), [#35744](https://github.com/anthropics/claude-code/issues/35744), [#18980](https://github.com/anthropics/claude-code/issues/18980)) — it just prints the reset time and stops. `scripts/auto_resume.sh` wraps the loop: it detects the limit message, parses the reset time (or falls back to the rolling window), sleeps, then relaunches a session that continues from `PROGRESS.md`. Durable state is what makes "wait and continue automatically" safe. In Cowork, the equivalent is a scheduled task that fires after the reset window.

## Install

**Claude Code / Cowork (as a skill):**
1. Download the packaged skill: [`agentic-loop-setup.skill`](./agentic-loop-setup.skill) (or zip the `agentic-loop-setup/` folder yourself).
2. In Cowork, open the `.skill` file and choose **Save skill**. In Claude Code, place the `agentic-loop-setup/` folder under your skills directory.
3. Invoke it by describing what you want to loop — e.g. *"set up my project for agentic coding"* or *"make Claude keep working until it's done."*

**Or clone and copy the folder directly:**
```bash
git clone https://github.com/jersilb1400/agentic-loop-setup.git
cp -r agentic-loop-setup/agentic-loop-setup ~/.claude/skills/
```

## Repo contents

```
agentic-loop-setup/        the skill itself
  SKILL.md                 the wizard
  references/              patterns, mechanisms, model-routing, prompts
  assets/                  file + subagent templates
Agentic_Loop_Playbook.pdf            the playbook this skill was distilled from
README.md
```

## Background

Built on Anthropic engineering practice — *Building Effective Agents*, the multi-agent research system, and Claude Code best practices. The PDF playbook in this repo is the long-form guide the skill was distilled from.
