---
name: agentic-loop-setup
description: >
  Active setup wizard that configures a complete agentic loop for any project.
  Interviews the user, then generates real files: CLAUDE.md, a verification
  check, LOOP.md, PROGRESS.md, an adversarial reviewer subagent
  (.claude/agents/reviewer.md), a permissions file (.claude/settings.json),
  and a learning system so Claude improves from every session. Includes smart
  model routing (Haiku/Sonnet/Opus) per loop role and unattended auto-resume
  that waits out a usage-limit reset and continues on its own.
  Use when the user wants to set up an agentic loop, make Claude work
  autonomously, have Claude "prompt itself", build a self-correcting workflow,
  run Claude in a loop, set up an orchestrator with subagents, automate a
  project, or says things like "set up my project for agentic coding", "make
  Claude keep working until it's done", "build a loop", "help Claude learn from
  every scenario", "keep running even if it hits the token limit", or asks
  about cost-efficient model selection for loops.
---

# Agentic Loop Setup

You are setting up an agentic loop: the cycle of **gather context → take action →
verify work → repeat**, plus the persistence layer that lets lessons survive
across sessions. The user wants their project configured so Claude can drive
itself — plan, build, check its own work, and improve over time.

This is an **active wizard**: don't just explain — interview, then create real
files in the user's project. The reason files matter: Claude's context dies with
the session, so any loop discipline that lives only in conversation is lost.
Everything important gets written to disk.

## The one principle behind everything

Claude stops when work *looks* done. Without a check it can run, "looks done" is
the only signal, and the user becomes the verification loop. Every setup you
produce must answer: **"What check can Claude run, and what gates the loop on
it?"** If you finish this wizard without a concrete pass/fail check, you have
failed the setup.

## Workflow

### Step 1 — Interview (use AskUserQuestion if available, otherwise ask in chat)

Keep it to 2 rounds max. Learn:

1. **The project**: language/stack, how it's built and tested, where it lives.
   If you have file access, explore the project first and infer what you can —
   only ask what you can't discover.
2. **The goal**: what kind of work should the loop do? (feature development,
   migration, research, content, data tasks…)
3. **The check**: what already produces a pass/fail signal? (test suite, build,
   linter, diff script, schema validator). If nothing exists, plan to create the
   simplest one — even a 10-line script that checks output shape beats nothing.
4. **Autonomy level**: supervised sessions, semi-attended (/goal + hooks), or
   fully unattended (headless `claude -p` loops). This decides how hard the
   verification gate must be.
5. **Environment**: Claude Code CLI, Cowork, or chat — this changes which
   mechanisms exist (see reference table below).

### Step 2 — Choose the loop architecture

Match the pattern to the dominant failure mode, not to ambition. Default to the
simplest thing: a single Explore → Plan → Implement → Verify loop. Only add
agents when one demonstrably isn't enough.

| Failure mode the user describes | Pattern |
|---|---|
| Stops early, declares partial work "done" | Loop-until-done: /goal condition or Stop hook |
| Misses its own bugs | Adversarial verification: fresh-context reviewer subagent |
| Loses the objective over long runs | Orchestrator–workers: durable lead, short-lived workers |
| Many similar items (files, records, tickets) | Fan-out: headless `claude -p` over a generated task list |
| Output quality is taste-based | Evaluator–optimizer or tournament (pairwise) |
| Mixed/heterogeneous incoming work | Classify-and-act routing |

For multi-agent designs, read `references/patterns.md` before generating files —
it has the delegation rules (exact role, scoped input, output schema) that make
orchestrators work, and the lessons from Anthropic's research system.

**Then route models.** Read `references/model-routing.md` and assign a model
tier to every role in the architecture (orchestrator, workers, fan-out items,
verifier, reviewer). Loops multiply cost: the same prompt may run hundreds of
times, so score complexity once per role and use the cheapest capable tier —
Haiku for mechanical work, Sonnet for scoped subtasks, Opus only where
decomposition quality or error risk demands it. Write the assignments and the
escalation rule into LOOP.md, and put `--model` flags in any generated scripts.

### Step 3 — Generate the files

Create these in the user's project (adapt names to what already exists — never
clobber an existing CLAUDE.md; append a marked section instead):

1. **CLAUDE.md** — short. Build/test commands, style rules that differ from
   defaults, architecture decisions, gotchas. Include the loop contract:
   the verification command, the two-correction rule, and the end-of-session
   learning pass. For every line ask: "would removing this cause mistakes?"
   Bloated files get ignored. Template: `assets/CLAUDE.md.template`

2. **Verification check** — a runnable script or documented command that
   returns pass/fail. If none exists, write `scripts/verify.sh` (or equivalent)
   now, with the user.

3. **PROGRESS.md** — the loop's working memory: done / failed-and-why / next.
   Template: `assets/PROGRESS.md.template`

4. **LOOP.md** — the loop's operating instructions: the chosen pattern, the
   exact prompts to start a session, the gate level, and the stop condition.
   Template: `assets/LOOP.md.template`

5. **`.claude/agents/` — the subagent roster.** Each runs in a **fresh,
   scoped context**. Together they remove the two human gates that normally
   stall an autonomous loop — *plan approval* (handled by the plan-critic) and
   *merge approval* (handled by the reviewer) — and they keep the driver's
   context from filling up (handled by the explorer). Generate all four for any
   loop meant to run unattended; for a quick supervised loop, the reviewer alone
   is the minimum. Set `model:` in each frontmatter per `references/model-routing.md`.

   - **`plan-critic.md`** — adversarial plan critic. Runs *after the plan, before
     any code*. Surfaces unstated assumptions, dismissed alternatives, edge
     cases, and a verdict (proceed / proceed-with-changes / reconsider). This is
     what lets the loop proceed without a human approving the plan — and what
     ensures all angles get considered up front. Template:
     `assets/plan-critic.md.template`
   - **`reviewer.md`** — adversarial correctness reviewer. Runs *after
     implementation* in a context that never saw the code being written, so it
     can't defend it. Checks domain correctness, UX, security, hardcoded values —
     **not** approach (that's the plan-critic's job) and not style. Customize per
     project: domain checks (API auth / data integrity / prompt injection), UX
     audience, stack-specific checks (React hook rules, SQL injection, missing
     migrations). Always outputs a numbered issue list or `LGTM — no issues found.`
     Template: `assets/reviewer.md.template`
   - **`explorer.md`** — read-only scout (no Edit/Write/Bash-mutate). Answers one
     focused investigation question and returns a compact `path:line` digest, so
     long autonomous runs don't exhaust the driver's context. Template:
     `assets/explorer.md.template`
   - **`verifier.md`** — mechanical check-runner. Runs the verification command,
     reports `PASS` or a triaged `FAIL` list, never fixes. Keeping run-the-check
     separate from fix-the-code keeps the pass/fail signal honest. Template:
     `assets/verifier.md.template`

   The division of labor is deliberate: plan-critic questions the *approach*,
   reviewer questions the *result*, verifier confirms the *check*, explorer
   feeds *context* — no single agent both produces work and judges it.

6. **`.claude/settings.json`** — the permissions file. This gates what Claude
   can do autonomously without asking. Tailor `allow` to the project's actual
   build/verify commands (don't list commands that don't exist). Always include
   these in `deny` regardless of project: `rm`, `git push`, any deploy command,
   and any outbound network calls (`curl`, `wget`). Template: `assets/settings.json.template`

   The principle: allow everything needed to build and verify; deny everything
   that is irreversible or external. For truly unattended runs, the allow-list
   is what removes permission prompts: pair it with `--permission-mode acceptEdits`
   (or `auto`) in the runner so Claude proceeds on allowed actions and still
   hard-stops on anything in `deny`. Never use `--dangerously-skip-permissions`
   for a loop — the deny-list is the safety net that makes autonomy survivable.

7. **`scripts/auto_resume.sh`** (generate whenever the loop runs unattended) —
   the token-exhaustion recovery wrapper. Claude Code does not yet auto-resume
   after a usage-limit reset (it only prints the reset time), so this wrapper
   runs the loop, and when it sees a usage/rate-limit message it parses the
   reset time, sleeps until then plus a margin, and relaunches a session that
   reads PROGRESS.md and continues. This is *why* PROGRESS.md and the DONE
   marker matter: durable state makes "wait and continue automatically" safe.
   In Cowork (no headless mode) substitute a scheduled task that fires after
   the reset window and continues from PROGRESS.md — see `references/mechanisms.md`.
   Template: `assets/auto_resume.sh.template`

8. **Optional, by environment**: hooks (lint-after-edit, protected paths),
   a fan-out shell script, or a /goal condition string. See
   `references/mechanisms.md` for what each environment supports and exact syntax.

### Step 4 — Wire the learning system

This is what makes the loop improve from every scenario:

- End every session with the learning pass (already in the CLAUDE.md template):
  append repeatable lessons as short imperative rules, delete rules the session
  proved unnecessary, update PROGRESS.md with failed approaches *and why*.
- The two-correction rule: corrected twice on the same issue → /clear, bank the
  lesson, restart with a better prompt. A clean session with a better prompt
  beats a long session with accumulated corrections.
- When a loop shape works twice, promote it to a skill or saved workflow so it
  becomes a one-line invocation.

### Step 5 — Dry run and hand off

Run one supervised pass of the loop on a small real task. Show evidence (test
output, diff), not assertions. Then give the user a short summary: the files
created, the prompt that starts the loop, and what to watch for in the first
week. Offer to scale up (parallel sessions, headless fan-out) only after the
supervised loop works.

## Guardrails to build into every setup

- Explicit token/iteration budget on anything that runs unattended.
- The worker never grades its own work — verification is a separate fresh
  context, always. The reviewer subagent exists precisely for this.
- Untrusted input (tickets, scraped content) never reaches a high-privilege
  actor agent.
- Scope unattended runs with `--allowedTools`; test fan-outs on 2–3 items
  before running at scale.
- Tell reviewers to flag only correctness gaps, not style — a reviewer asked to
  find gaps will always find some.
- `.claude/settings.json` is your safety net: if you're unsure whether an
  action should be autonomous, put it in `deny` and let the user promote it
  manually. It's easier to unlock than to undo.

## Reference files

- `references/patterns.md` — the 5 composable patterns + 6 dynamic-workflow
  patterns, delegation rules, and when to use which. Read when designing
  anything beyond a single-session loop.
- `references/mechanisms.md` — environment capability table (Code / Cowork /
  chat), exact syntax for /goal, hooks, subagents, headless mode, auto mode.
  Read in Step 3 before generating environment-specific files.
- `references/model-routing.md` — smart model routing: complexity scoring
  rubric, tier-per-loop-role table, escalation protocol, budget math. Read in
  Step 2 when assigning models, and whenever the user mentions cost or tokens.
- `references/prompts.md` — copy-paste prompt template library (interview-to-spec,
  orchestrator starter, adversarial review, end-of-session learning pass,
  fan-out loop script). Read in Step 3.
- `assets/` — file templates for CLAUDE.md, PROGRESS.md, LOOP.md, the subagent
  roster (plan-critic.md, reviewer.md, explorer.md, verifier.md), settings.json,
  and auto_resume.sh (unattended token-exhaustion recovery).
