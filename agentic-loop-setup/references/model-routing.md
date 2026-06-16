# Smart Model Routing Reference

Route every piece of loop work to the cheapest model capable of doing it well.
In a loop this matters more than in chat: the same prompt may run hundreds of
times, so a tier mistake multiplies. Never spend Opus budget on Haiku work.

## Model tiers & pricing (2026)

| Tier | Model | Input $/1M | Output $/1M | Relative cost |
|---|---|---|---|---|
| Fast | `claude-haiku-4-5` | $1.00 | $5.00 | 1× |
| Balanced | `claude-sonnet-4-6` | $3.00 | $15.00 | 3× |
| Expert | `claude-opus-4-8` | $5.00 | $25.00 | 5× |

## Complexity scoring (run before assigning any loop role or task)

Score 0–10 on each factor, sum:

| Factor | Low (0–2) | Medium (3–6) | High (7–10) |
|---|---|---|---|
| Files / scope | 1–2 files, bounded | 3–10 files / a module | 10+, cross-cutting |
| Solution clarity | Known steps | Design decisions needed | Unknown, needs exploration |
| Creativity | None (format, template) | Adapt existing pattern | Novel, no prior art |
| Error risk | Easily reversible | Affects tests/build | Security, data loss, prod |
| Reasoning depth | Mechanical transform | Multi-step logic | Causal analysis, unknown root cause |

Bands: **0–14 → Haiku, 15–29 → Sonnet, 30–50 → Opus.**

Hard rules that bypass the rubric — always Haiku: format conversion, renames,
boilerplate from a clear template, single-doc summaries. Always Opus: security
analysis, architecture decisions with long-term consequences, debugging with no
error message or data-integrity stakes, anything where being wrong costs money.

Announce the routing in one line before work starts, e.g.
"→ Sonnet (score 22/50): multi-file refactor with existing pattern to follow."

## Routing inside loop architectures

| Loop role | Default tier | Why |
|---|---|---|
| Orchestrator / lead agent | Expert (or Balanced for modest scope) | Decomposition and synthesis quality bound the whole run — Anthropic's research system uses Opus lead + Sonnet workers |
| Workers (scoped, well-specified subtasks) | Balanced | Focused instructions + only relevant input = Sonnet territory |
| Fan-out items (mechanical per-file transforms) | Fast or Balanced | Score one representative item; the pilot run validates the choice |
| Verifier / grader running a deterministic check | Fast | Reading pass/fail output is mechanical |
| Adversarial reviewer (correctness judgment) | Balanced+ | Judgment work; never below the worker's tier |
| Plan critic (pre-implementation red team) | Balanced+ (Expert for high-stakes/irreversible designs) | Reasoning about alternatives and assumptions; getting direction wrong is the costliest error |
| Explorer / scout (read-only investigation) | Fast | Locate + summarize is mechanical; it exists to save the driver's context, not to judge |

In headless calls, pass the model explicitly:
```bash
claude -p "..." --model claude-haiku-4-5 --allowedTools "Edit"
```
In interactive Claude Code, recommend `/model` — routing is advisory; never
switch silently. In subagent definitions, set `model:` in the frontmatter.

## Escalation protocol (build into LOOP.md)

Move up one tier, and say so, when: (1) the first attempt fails validation,
(2) hidden complexity emerges, (3) confidence is low on an irreversible change,
(4) a task tracks >10K tokens with uncertain quality. In fan-out drivers,
implement this mechanically: items that FAIL on the Fast tier get retried once
on the next tier before being marked for human review.

## Budget tracking

Give every loop an explicit per-item and per-run token budget in LOOP.md.
For fan-outs, estimate: items × avg tokens × tier rate, and surface the number
to the user before the full run. The pilot (2–3 items) is also the cost
calibration: measure actual tokens per item there, then project.
