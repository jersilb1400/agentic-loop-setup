# Prompt Template Library

Copy, fill brackets, paste. Inline these into LOOP.md when generating files.

## Interview-to-spec (before any large feature)

```
I want to build [brief description]. Interview me in detail using the
AskUserQuestion tool. Ask about technical implementation, UI/UX, edge cases,
concerns, and tradeoffs. Don't ask obvious questions — dig into the hard parts
I might not have considered. Keep interviewing until we've covered everything,
then write a complete spec to SPEC.md.
```
Then start a FRESH session to execute the spec.

## Session starter (the basic loop)

```
Read CLAUDE.md, LOOP.md, and PROGRESS.md. Pick up the next item in PROGRESS.md.
Explore the relevant files first and propose a plan — don't write code yet.
After I approve: implement, run [verification command] after each change, and
fix failures. Show me the check output, not just a summary.
```

## Orchestrator starter

```
You are an expert Orchestrator Agent. User task: [TASK].
1. Decompose into the smallest possible independent sub-tasks.
2. For each sub-task define: exact specialized role; focused instructions;
   relevant input data only; required JSON output schema with a concise
   summary (≤400 tokens).
3. Output a clear execution plan first. I will run sub-agents in parallel
   and return their summaries.
4. Then synthesize into one cohesive final deliverable.
Prioritize structured outputs and minimal context per agent.
```

## Adversarial review

```
Use a subagent to review the diff against PLAN.md. Check that every
requirement is implemented, the listed edge cases have tests, and nothing
outside the task's scope changed. Report gaps that affect correctness or
the stated requirements — not style preferences.
```

## End-of-session learning pass

```
Before we finish: (1) Append to CLAUDE.md any repeatable lesson from this
session — commands, gotchas, conventions — phrased as a short imperative
rule. (2) Delete any CLAUDE.md rule this session proved unnecessary.
(3) Update PROGRESS.md: completed items, failed approaches and why they
failed, and the exact next step for a fresh session to pick up.
```

## Loop-until-done (no /goal available)

```
Work through PROGRESS.md item by item. For each item: implement, run
[verification command], and only mark the item done when the check passes —
paste the passing output under the item. Do not stop while unblocked items
remain. If blocked, record exactly what's blocking in PROGRESS.md and move
to the next item.
```

## Fan-out shell script (Claude Code)

```bash
# 1) Generate the list:  claude -p "List every file needing [change], one per line" > tasks.txt
# 2) Test on 3 items, refine, then:
for item in $(head -3 tasks.txt); do
  claude -p "Apply [change] to $item. Run [check]. Return OK or FAIL with reason." \
    --model claude-haiku-4-5 \
    --allowedTools "Edit,Bash(git commit *)"
done
# Score one representative item first (references/model-routing.md): mechanical
# transforms → haiku; judgment per item → sonnet. Items that FAIL retry once on
# the next tier up before being flagged for human review.
```

## Two-correction reset

```
We've gone back and forth on this twice — clearing context. [In the fresh
session:] [Original task], with these constraints learned from the last
attempt: [lesson 1], [lesson 2]. Run [check] before reporting done.
```

## Resume after a usage-limit reset (cold start from PROGRESS.md)

Used by `scripts/auto_resume.sh` and by hand after a limit resets. Works as a
cold start because all state lives in PROGRESS.md.

```
Read CLAUDE.md, LOOP.md, and PROGRESS.md. Continue the next unfinished item in
PROGRESS.md — do not restart completed work. Run [verification command] after
each change and show its output. Keep PROGRESS.md current as you go. When every
item is checked off and the full check passes, print exactly: ALL ITEMS COMPLETE.
If you hit the usage limit again, leave PROGRESS.md mid-task with the exact next
step and stop — do not mark anything done that the check has not verified.
```
