# Loop Patterns Reference

Source: Anthropic's "Building Effective Agents," the multi-agent research system
write-up, and Claude Code best practices.

## Core principles (apply to every design)

1. **Simplicity.** The most successful implementations use simple, composable
   patterns, not frameworks. Start with one loop; add agents only when one
   demonstrably isn't enough.
2. **Transparency.** Show planning steps explicitly so the user can correct the
   plan before it becomes wrong work.
3. **Agent–computer interface.** Document and test tools as carefully as a
   human-facing API. Most agent failures are tool-documentation failures.
4. **Context is the constraint.** Performance degrades as context fills. Every
   pattern below exists partly to protect the context window.

## The five composable workflow patterns

| Pattern | How it works | Best for |
|---|---|---|
| Prompt chaining | Each step's output feeds the next; optional gates between | Fixed sequential subtasks |
| Routing | Classifier directs each input to a specialized handler | Heterogeneous inputs, triage |
| Parallelization | Independent subtasks run simultaneously; aggregate or vote | Speed; multiple perspectives |
| Orchestrator–workers | Lead agent decomposes dynamically, delegates to spawned workers | Unpredictable scope: multi-file changes, research |
| Evaluator–optimizer | One generates, a second evaluates against criteria; repeat | Iterative refinement with clear criteria |

## The six dynamic-workflow patterns (failure-mode driven)

| Failure mode | Pattern |
|---|---|
| Agentic laziness (partial → "done") | Loop-until-done with /goal or Stop hook |
| Self-preferential bias | Adversarial verification, fresh context |
| Goal drift across turns | Fan-out-and-synthesize: durable orchestrator, short-lived workers |
| Hard to score / taste-based | Tournament (pairwise comparison, never absolute scores) |
| Unknown amount of work | Loop-until-done over a generated task list |
| Heterogeneous work items | Classify-and-act → dedupe → fix or escalate |

## Delegation rules (from Anthropic's research system)

Their orchestrator–worker system beat single-agent Opus by 90.2% on research
evals. The lessons:

- **Think like your agents.** Simulate a step with only the context the worker
  will have. If you can't do it with that context, neither can Claude.
- **Scale effort to complexity.** Early versions spawned 50 subagents for simple
  queries. Tell the orchestrator explicitly how many workers and tool calls
  different task types deserve.
- **Teach the orchestrator to delegate.** Every worker spec needs: exact role,
  focused instructions, only the relevant input, and a required output schema
  including a concise summary (≤400 tokens). Vague delegation produces
  duplicated and missing work.
- Workers return summaries, not raw dumps — the orchestrator's context is the
  scarce resource.

## Writer/Reviewer split

Run implementation and review in separate sessions/contexts. A fresh context
improves review because Claude won't favor code it just wrote. Variants:
- A implements → B reviews diff → A addresses feedback.
- B writes tests → A writes code to pass them.

## Verification escalation ladder

1. **In the prompt**: "run the tests after implementing and fix failures" — any task, zero setup.
2. **/goal condition**: separate evaluator re-checks after every turn; prevents soft completion.
3. **Stop hook**: deterministic script gates the end of turn; for unattended runs.
4. **Adversarial subagent**: fresh context reviews diff against plan; final sign-off on long runs.

Always demand evidence over assertions: test output, the command and its result,
or a screenshot — never just "done."
