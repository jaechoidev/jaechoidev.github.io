---
title: 'Can a local model actually drive a coding agent?'
date: 2026-08-20
permalink: /posts/2026/08/local-llm-agent-loop/
tags:
  - project, llm, benchmark, local, agents, aider, opencode, mlx, ollama
---

## 1. Summary

Can a coding model running on your own machine actually fix a bug? Not describe
one, not write a review that sounds right: change the code so a failing test
suite goes green.

I gave six local models three broken Python repositories on a 96GB M2 Max and
ran `pytest` on the result. The grader has no opinions.

Given a harness that does not require native tool calls, **every model tested
fixed at least two of three real bugs, almost always in a single turn, for about
a thousand prompt tokens**. Local models on this machine can do this work.

Four findings, none of which a tokens-per-second table would have shown:

1. **The quality ranking inverts.** DeepSeek-Coder-V2-Lite 16B scored last of
   six on written quality and fixed every task on mlx-lm, in fourteen seconds,
   from an 8.6GB footprint. Writing a convincing code review and correctly
   changing one character are different skills.
2. **Tool calling excludes half the field.** Three of seven configurations
   cannot drive an agent that uses native tool calls, whatever their prose
   scores. They are not slow at it; they cannot do it.
3. **The scaffolding costs 47x to 106x the context** of the code being fixed.
   The same one-line fixes take about 1,000 prompt tokens through aider and
   200,000 to 300,000 through opencode. In wall clock the gap is 1.3x to 10x,
   because prefix caching absorbs most of the resend, and both numbers are
   reported here for that reason.
4. **Llama 3.3 70B calls tools correctly and still fixes nothing.** It clears
   the gate and then times out on all three tasks. At 47 tokens per second of
   prefill, a growing conversation never converges. This is what the previous
   post's prefill wall looks like when the measurement is a test suite instead
   of a stopwatch.

Section 7 is about the run that had to be thrown away, because an agent given an
isolated working directory reached the pristine task templates anyway and fixed
two of them. Every later run of those tasks would have started from correct
source and passed for free. Nothing crashed. That is the failure mode worth
worrying about.

## 2. Why tokens per second is the wrong question

Almost every local-LLM benchmark, including one I ran in May, reports tokens per
second and a quality score. Neither predicts whether a model can work as an
agent, because an agent does not answer one question. It runs a loop: read a
file, run the tests, decide, edit, run the tests again.

Three things change in a loop.

**The score stops being a matter of opinion.** Rating a code review means
judging prose. If the task is instead "make this failing suite pass", the grader
is `pytest` and no rubric is involved.

**Context is re-sent every turn.** A chat completion is stateless, so each turn
resends the whole conversation. Whether the server *re-prefills* it is a
separate question, and the answer is mostly no: both engines cache the prompt
prefix. Measuring tokens sent and seconds elapsed separately is the only way to
tell context volume apart from time actually spent.

**Tool calling becomes a hard gate.** Prose quality is irrelevant if the model
cannot emit a call the runtime will parse. That turns out to exclude half the
field, and for reasons that are often the runtime's fault rather than the
model's.

## 3. The tasks

Three small Python repositories, each with a failing pytest suite and exactly
one defect:

| task | defect | tests |
|---|---|---|
| `chunker` | off-by-one in a slice: `items[i:i + size - 1]` | 5, 2 failing |
| `config_loader` | `except FileNotFoundError` too narrow; malformed JSON and a directory path both escape | 4, 2 failing |
| `ledger` | wrong default: `row.get("amount", 1)` should be `0` | 4, 2 failing |

Each is fixable with a one-line change, verified by applying the reference fix
and watching the suite go green. Each suite also contains passing tests, so
deleting the failing assertions is not a winning strategy. The agent is told not
to modify anything under `tests/`, and a `git diff` afterwards checks whether it
did anyway.

Deliberately small. The point is not to see whether a local model can do hard
work. It is to see whether it can complete a loop at all, and what that costs.

They did not turn out equally hard. Across the thirteen aider configurations,
`ledger` was fixed by all thirteen, `chunker` by twelve, and `config_loader` by
seven. The two easy ones put the wrong token on the failing line, where the
defect is visible in the failure. `config_loader` requires knowing which
exceptions a directory path and malformed JSON actually raise, which is not
visible anywhere in the failing test.

## 4. Two harnesses, on purpose

**aider** parses search/replace blocks out of ordinary model output. It needs no
tool-calling support, so it can score every model on both runtimes.

**opencode** drives its loop through native tool calls. It structurally cannot
run the models that fail the tool gate.

Running both is the experiment, not redundancy. The gap between "can emit an
edit block" and "can drive a native tool loop" is the thing that decides whether
a local model is usable in a modern agent, and only running both makes that gap
visible.

Both are pointed at the same engines over the same OpenAI-compatible path so
that harness wiring is not a confounding variable.

## 5. Measuring the tax

Every request passes through a counting reverse proxy (`lib/proxy.py`) that sits
between the harness and the inference server. It records, per task: how many
turns the loop took, how many prompt tokens were sent in total across those
turns, how many completion tokens came back, and the slowest single turn.

Trusting the harness to report its own token usage would have meant trusting two
different implementations to agree. The proxy also injects
`stream_options: {include_usage: true}` into streaming OpenAI-style requests, so
the counts come from the server rather than from a characters-divided-by-four
estimate. That changes nothing about what the model sees.

The number this exists to produce is the **context tax**: total prompt tokens
across the whole loop, against the roughly 1.5k tokens of source the task
actually contains.

A caution about what that number is not. Tokens sent are not tokens prefilled.
In the largest run here, 4.0M tokens were sent in 1,060 seconds, an apparent
3,769 tok/s against a model whose measured prefill rate is about 460 tok/s.
Prefilling all of it would have taken two and a half hours; it took eighteen
minutes, and the server log carries 208 prompt-cache entries. Prefix caching
absorbs most of the resend.

So the token column measures how much context the harness moves, and the
wall-clock column measures what it costs. Both are reported, because they differ
by an order of magnitude and quoting only the first would overstate the case.

An early calibration run makes the point before any real model was involved.
The same task, the same tiny model, two harnesses:

| harness | turns | prompt tokens |
|---|---|---|
| aider | 1 | 1,035 |
| opencode | 2 | 11,231 |

Same repository, same defect. The difference is system prompt and tool
definitions, resent every turn. That is not a model property at all, and it is
invisible to every benchmark that measures tokens per second.

## 6. Results

57 executed runs across four harness/engine combinations, plus 21 recorded
skips. No run modified the test files it was told to leave alone. Every token
count is server-reported rather than estimated.

### 6.1 Given the right harness, local models do the work

Under aider, every one of the thirteen model/engine configurations landed at
least two of three verified fixes. Thirty-eight of the thirty-nine task runs
took a single turn:

| Model | aider/ollama | aider/mlx |
|---|---|---|
| Qwen3-Coder 30B-A3B | 3/3 | 3/3 |
| Qwen3.6 35B-A3B | 3/3 | 3/3 |
| DeepSeek-V2-Lite 16B | 2/3 | **3/3** |
| Codestral 22B | 2/3 | 2/3 |
| Qwen2.5-Coder 32B | 2/3 | 2/3 |
| Llama 3.3 70B | 2/3 | 2/3 |

One turn, in all but one run. Roughly 1,000 prompt tokens. Thirteen to 175
seconds for all three tasks. A local model on this machine can read a failing
test suite, locate the defect, and write a fix that makes the suite pass.

### 6.2 The prose ranking does not predict this

I had previously graded these same models on code review, bug hunting, planning
and refactoring, scoring 52 outputs blind against a written rubric and ranking
them 4.00 down to 2.00. That ranking does not survive contact with `pytest`:

| Model | prose score | tasks fixed (aider) |
|---|---|---|
| Qwen3-Coder 30B-A3B | 4.00 | 3/3 |
| Llama 3.3 70B | 3.50 | 2/3 |
| Qwen2.5-Coder 32B | 2.50 | 2/3 |
| Codestral 22B | 2.25 | 2/3 |
| **DeepSeek-V2-Lite 16B** | **2.00** | **3/3 on mlx-lm** |

DeepSeek-Coder-V2-Lite scored last of six on written quality and fixed every
task on mlx-lm, faster than anything else in the study: three tasks in fourteen
seconds. It is a 16B model in 8.6GB.

The two measurements reward different things. Writing a review that identifies a
subtle security regression is a different skill from changing one character in a
slice expression, and a model can be good at the second while being mediocre at
the first. If what you want is an agent that fixes failing tests, the rubric
score is the wrong number to shop on.

### 6.3 Tool calling is the gate, and it excludes half the field

Under opencode, which drives its loop through native tool calls, three of seven
model configurations could not participate at all. Not "performed poorly":
could not be run. DeepSeek-V2-Lite, Codestral and Qwen2.5-Coder are recorded as
skipped.

They fail in three distinct ways, and only one is the model's doing. Ollama
returns HTTP 400 and "does not support tools" for Codestral, whose mlx-community
repository ships no chat template at all. Qwen2.5-Coder emits a correct call
wrapped in `<tools>` tags where mlx-lm expects `<tool_call>`, so a valid call is
discarded by the parser. DeepSeek-V2-Lite answers in prose. Llama 3.3 emits
`{"type":"function","name":...}` that Ollama parses happily and mlx-lm drops,
because none of mlx-lm's eleven shipped parsers matches a delimiter-free JSON
format. The documented `tool_parser_type` override cannot fix that: the
mechanism segments calls out of the token stream using delimiter strings, and
Llama 3.3 does not emit any.

Of the four that cleared the gate, three succeeded:

| Model | opencode/ollama | opencode/mlx |
|---|---|---|
| Qwen3-Coder 30B-A3B | 3/3 | 3/3 |
| Qwen3.6 35B-A3B | 3/3 | 3/3 |
| Qwen3.6 (MLX format) | 3/3 | - |
| Llama 3.3 70B | **0/3** | no tools |

Llama 3.3 is the interesting failure. It emits tool calls that Ollama parses
correctly, so it clears the gate, and it still fixed nothing: five to ten turns
per task, and it hit the ten-minute ceiling on all three. At 47 tokens per second
of prefill, a loop that needs several turns of growing context does not converge
in any tolerable time. That is what "too slow" means once the measurement is a
test suite rather than a stopwatch.

### 6.4 The context tax is real, and smaller in seconds than in tokens

Same models, same three one-line fixes, two harnesses:

| Model | aider tokens | opencode tokens | ratio | aider wall | opencode wall | ratio |
|---|---|---|---|---|---|---|
| Qwen3-Coder (ollama) | 2,935 | 312,414 | 106x | 23s | 241s | 10x |
| Qwen3.6 (ollama) | 3,102 | 207,238 | 67x | 112s | 247s | 2.2x |
| Qwen3.6 (MLX fmt) | 4,625 | 216,097 | 47x | 111s | 146s | 1.3x |

Two orders of magnitude more context to accomplish identical work. The
scaffolding is the payload: a system prompt, a tool schema for every available
tool, and a conversation that grows with each turn, all resent on every request.

The wall-clock column is the honest counterweight. If those tokens were being
prefilled from cold, the qwen3-coder run would have taken over an hour; it took
four minutes. Prefix caching absorbs most of the resend, so the tax is paid in
context window and in per-turn latency rather than in raw prefill compute. Both
numbers belong in the table. Quoting only the token ratio would be the more
dramatic and less true version.

Note the ordering of the wall-clock ratios: the faster the model's prefill, the
less the loop costs it. Qwen3.6 in MLX format, the fastest-prefilling
configuration I have measured on this machine, pays a 1.3x wall-clock penalty
for a 47x context penalty.

### 6.5 One configuration thrashed, for two separate reasons

Qwen3-Coder on opencode plus mlx-lm took 207 turns and 4.0M tokens for three
tasks, against 28 turns and 312k for the same model on Ollama. It passed all
three, so the outcome column looks fine. The transcripts do not.

On `chunker`, 71 of its bash calls were rejected by opencode with
`SchemaError(Missing key at ["description"])`. The model kept omitting a
required argument, kept getting the same error, and kept retrying the same
malformed call. On `ledger` it ran 119 turns with no schema errors at all and
was still going when the ten-minute timeout cut it off; the tests passed because
an earlier turn had already written the fix.

Neither happened on Ollama, with the same model and the same harness. Nor did it
happen to Qwen3.6 on mlx-lm. The difference is confined to one model on one
runtime, which points at the 4-bit MLX conversion rather than at mlx-lm or at
the model. The same conversion produced two other malfunctions in my earlier
speed and quality runs: repetition loops that filled the token budget with one
bullet repeated fifteen times, and code "reproductions" containing syntax errors
absent from the source file, both reproduced across independent runs. Three
distinct malfunctions in one artefact is enough to stop calling it coincidence.

I do not have a mechanism beyond that, and I am not going to invent one.

## 7. What went wrong: the agent escaped the sandbox

The first tier-2 attempt is void, and the way it failed is more interesting than
the numbers it would have produced.

Each task runs in a fresh temporary directory: the template is copied out of
`agent-tasks/`, `git init`-ed, and the harness runs the agent there with
`subprocess.run(cwd=workdir)`. That looked like isolation.

Partway through the opencode runs I checked the templates by hand and found
`agent-tasks/ledger/src/ledger.py` reading `row.get("amount", 0)`. The correct
value. The pristine template had been fixed. A second check minutes later found
`chunker` fixed too, altered between one check and the next while the run was
still going.

The agent's own transcript shows how:

    cd agent-tasks/ledger && python -m pytest tests/

A **relative** path. From the temp workdir that command cannot resolve, so
opencode's bash tool was not running in the directory the harness handed it. The
likeliest cause is that `opencode run` attached to a server process left over
from an earlier invocation, carrying that older process's working directory. The
agent went looking for a failing test suite, found the real one, and fixed it,
which from its point of view was the assigned task performed correctly.

**Why this is the dangerous kind of bug.** Nothing crashed. Two task templates
were left in a *passing* state, so every later run of those tasks would have
started from already-correct source, gone green, and been recorded as a success.
The benchmark would have reported that local models are excellent at fixing bugs
that were no longer there. A harness that fails loudly wastes an afternoon; a
harness that fails quietly gets published.

**The fix, and the fix's own bug.** The harness now fingerprints every template
file with sha256, holds `agent-tasks/` read-only for the duration of a run,
kills stray agent servers before each task, and re-checks the fingerprint after
every task, aborting rather than writing a plausible row.

That hardening promptly broke the benchmark in a new way. `shutil.copy2`
preserves permissions, so the read-only bit propagated into the working copies
and the agent could no longer write its fix. The best model in the study went
from 3/3 to 0/2, which reads exactly like a model regression and was entirely
mine. The working copy is now explicitly made writable, and both properties are
asserted before a run: the copy accepts writes, the template refuses them.

**Everything from the first attempt was discarded**, including the aider runs
that finished before the first opencode run started and were almost certainly
clean. Proving them clean would require knowing when each template changed, and
that evidence does not exist. When the clean re-run reproduced those aider
numbers to the token, it confirmed they had been fine. That is a reason to feel
better afterwards, not a reason to have kept them.

## 8. Threats to validity

**Three tasks, one defect each, one run apiece.** A 2/3 and a 3/3 differ by a
single task. Nothing here supports a fine-grained ranking; the honest resolution
is roughly "fixes most simple bugs" against "fails the one requiring inference
about exception types".

**The tasks are small on purpose, and that limits what they show.** Each defect
is one line, each repository is three files. A model that fixes all three has
not been shown capable of a refactor across a real codebase. The question asked
here is whether the loop closes at all, not how far it scales.

**`config_loader` may simply be a harder task rather than a discriminating
one.** It was missed by 6 of 13 aider configurations, against 1 for `chunker` and 0
for `ledger`. The fix requires knowing that a directory path raises `IsADirectoryError`
and malformed JSON raises `JSONDecodeError`, so widening to
`(OSError, json.JSONDecodeError)` catches both. That is knowledge, not
reasoning, and a model that lacks it fails for an uninteresting reason.

**The tool-capability skips are inherited, not re-measured.** Three models were
skipped under opencode on the strength of a separate tool-calling probe, which
sent each model a tool definition and a prompt unanswerable without calling it.
If a model
would have emitted a parseable call under opencode's specific schema despite
failing that probe, this study would not know.

**Timeouts are a harness parameter.** Ten minutes for three one-line fixes is
generous, but Llama 3.3's 0/3 is partly a statement about that ceiling. With an
hour it might have finished. It would still not be usable.

**opencode and aider are not being compared on merit.** They have different
jobs: one drives native tool loops, the other parses edit blocks. The token
ratio between them measures how much scaffolding each sends, not which is
better software.

**Prefix caching is uncontrolled.** Both engines cache prompt prefixes and
neither was configured to do so or prevented from doing so. The wall-clock
numbers include whatever caching happened to occur, which is realistic but not
isolated.

**One machine, one afternoon, no repetitions.** Every number is a single sample
on an uncontrolled thermal and cache state.

## 9. What this changes

Ranked on speed and written quality alone, my shortlist had been Qwen3-Coder and
Qwen3.6. Measuring against a test suite changes it:

- **Qwen3-Coder 30B-A3B** and **Qwen3.6 35B-A3B** are the only models that both
  drive a native tool loop and fix every task. Either one, in MLX format, is a
  working local coding agent on this machine.
- **DeepSeek-Coder-V2-Lite 16B** is the surprise. It cannot drive a native tool
  loop at all, so it is useless to opencode or Claude Code. Given aider, it fixed
  every task in fourteen seconds from an 8.6GB footprint, which makes it the best
  value in the study by a distance.
- **Llama 3.3 70B** is now clearly out. It costs 45GB, takes six minutes to read
  a large prompt, and fixed nothing in an agent loop despite calling tools
  correctly.

The larger point is that "which local model is best" has no answer without
naming the harness. The ranking by prose quality, the ranking by tool-calling
support, and the ranking by tests-passed disagree with each other, and the model
that wins the third is last in the first.

## 10. Reproducing this

The speed, memory and written-quality measurements referenced here are in a
[companion post](/posts/2026/08/local-llm-ollama-vs-mlx/), along with the
prefill numbers and the tool-calling probe.

`bench-agent.py` runs the loop, `lib/proxy.py` counts the tokens,
`agent-tasks/` holds the three tasks with their reference fixes, and
`report-tier2.py` regenerates `results-tier2.md` from the raw runs. Per-task
transcripts are under `runs-agent/`, including the ones that thrashed.

The discarded first attempt is kept in `runs-agent-INVALID-contaminated/` with a
note explaining what the agent did to the templates and why every row from that
attempt went in the bin.

