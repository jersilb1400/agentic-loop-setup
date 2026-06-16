# Environment Mechanisms Reference

What exists where, and exact syntax. Use this to decide which files to generate
in Step 3.

## Capability table

| Mechanism | Claude Code CLI | Cowork | Claude chat |
|---|---|---|---|
| CLAUDE.md auto-loaded | Yes | Yes (project folder) | Via project instructions |
| Plan mode | Yes | No (use "don't write code yet" prompts) | No |
| /goal conditions | Yes | No | No |
| Hooks (deterministic) | Yes (.claude/settings.json) | No | No |
| Subagents | Yes (.claude/agents/) | Yes (Agent tool) | No |
| Headless `claude -p` | Yes | No (use scheduled tasks instead) | No |
| Worktrees / parallel sessions | Yes | Multiple sessions | No |
| Checkpoints /rewind | Yes | No | No |
| Skills | Yes | Yes | Yes (capabilities) |
| Persistent files (PROGRESS.md etc.) | Yes | Yes | Project knowledge only |
| Auto-resume after usage-limit reset | Wrapper script (not built in) | Scheduled task after reset | Manual |

When the environment lacks a mechanism, substitute the file-based equivalent:
no hooks → put the rule in CLAUDE.md *and* the verification script; no /goal →
end every working prompt with "do not stop until <check> passes; show the
output"; no headless mode → scheduled tasks or supervised batches.

## Syntax quick reference (Claude Code)

**Hook (lint after every edit)** — ask Claude: "Write a hook that runs eslint
after every file edit" or edit `.claude/settings.json` directly.

**Subagent definition** — `.claude/agents/reviewer.md`:
```markdown
---
name: reviewer
description: Reviews diffs against the plan for correctness gaps
tools: Read, Grep, Glob, Bash
---
You are an independent reviewer. Check the diff against PLAN.md:
every requirement implemented, edge cases tested, nothing out of scope
changed. Report gaps that affect correctness, not style preferences.
```

**Headless invocation** (always pass an explicit model — see
`references/model-routing.md` for tier selection):
```bash
claude -p "prompt" --model claude-sonnet-4-6 --output-format json
claude --permission-mode auto -p "fix all lint errors" --model claude-haiku-4-5
```
Interactive sessions: recommend `/model` for the tier; subagent definitions:
set `model:` in the frontmatter (e.g. `model: haiku` for a verifier that just
reads check output, `model: sonnet` for reviewers).

**Fan-out loop**:
```bash
for item in $(cat tasks.txt); do
  claude -p "Process $item. Run the check. Return OK or FAIL." \
    --allowedTools "Edit,Bash(git commit *)"
done
```
Test on 2–3 items first; refine the prompt; then run at scale.

**Goal condition**: `/goal all tests in tests/ pass and lint is clean`

## Usage-limit auto-resume (continue after running out of tokens)

Claude Code does NOT auto-resume after a subscription usage-limit reset — it
prints the reset time and stops (open requests anthropics/claude-code #36320,
#35744, #18980). For unattended loops, wrap the run so a limit pauses rather
than ends it. Durable PROGRESS.md is the precondition: the resumed session
rebuilds its place from disk, so nothing is lost across the gap.

**Claude Code — wrapper script** (`scripts/auto_resume.sh`, template in
`assets/auto_resume.sh.template`). Shape:
```bash
while :; do
  out="$(claude --continue -p "$PROMPT" --model "$MODEL" 2>&1)"
  if grep -qiE 'usage limit|rate limit|limit reached|429|resets? at' <<<"$out"; then
    wait_for_reset "$out"   # parse reset time / retry-after, else fall back to ~5h, then sleep
    continue
  fi
  grep -qF "$DONE_MARKER" <<<"$out" && break   # loop prints DONE_MARKER only when fully complete
done
```
Detection: match the limit message in stdout; for API 429s honor `retry-after`.
Reset timing: parse an ISO timestamp or "resets at <time>" if present, otherwise
sleep the rolling window (~5h) + a small margin. Always cap total resume cycles.

**Cowork — scheduled task.** No headless mode, so instead of sleeping a process,
create a scheduled task that fires after the reset window and re-runs the
session prompt against PROGRESS.md. Re-arm it until PROGRESS.md shows complete.

**Chat — manual.** No background waiting; the user re-sends "continue from
PROGRESS.md" after the reset.

Pair this with the existing token/iteration budget: the budget caps a single
window's spend; auto-resume spans windows. Both write state to PROGRESS.md on
stop so the next launch is a clean cold start.

## Autonomy levels and required gates

| Level | Gate required | Extras |
|---|---|---|
| Supervised | Check in the prompt | Plan approval before implementing |
| Semi-attended | /goal or Stop hook | Permission allowlist, evidence in output |
| Unattended | Stop hook + adversarial review | auto mode, --allowedTools, token budget, auto_resume.sh wrapper, isolation (worktree/VM) |

## Safety rails (non-negotiable for unattended)

- Explicit token and iteration budget.
- `--allowedTools` scoping on every headless call.
- Untrusted input never reaches a high-privilege agent.
- Isolation: worktrees or VMs so parallel loops can't collide.
- Usage-limit auto-resume wrapper so a token-limit hit pauses and continues
  instead of silently ending the run.
