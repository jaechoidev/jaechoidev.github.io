---
title: "Ollama vs MLX on an M2 Max: six local coding models, and why my last benchmark was wrong"
date: 2026-08-19
permalink: /posts/2026/08/local-llm-ollama-vs-mlx/
tags:
  - local-llm
  - benchmark
  - mlx
  - ollama
  - apple-silicon
  - qwen
  - llama
---

**In short:** the highest-return change on this machine was which tag I pulled.
`qwen3.6:35b-mlx` decodes at 71.7 tok/s where `qwen3.6:35b` manages 47.4, same
weights and same Ollama build, because Ollama's Apple Silicon MLX backend
engages only for models published in MLX format and leaves GGUF models on
llama.cpp. Everything else was smaller than expected: mlx-lm beats Ollama 0.23.2
on four of five models but by 5 to 45%, upgrading Ollama bought memory rather
than speed (DeepSeek dropped from 17.7GB to 13.0GB), and prefill rather than
decode is what makes a model unusable, with Llama 3.3 70B taking 355 seconds to
read a 15k-token prompt before emitting a single token. Written quality was
mediocre across the board: 52 blind-graded outputs averaged 2.92 of 5, and only
4 of 13 code reviews noticed a reverted security fix sitting in the diff.

## 1. Summary

Six local coding models on a 96GB M2 Max, measured across three runtimes: Ollama
0.23.2, Ollama 0.32.14, and mlx-lm 0.31.3. Every engine is timed the same way,
with prompt prefill separated from generation, because a single tokens-per-second
figure hides the number that actually decides whether a model is usable.

Four things this run establishes:

1. **Prefill, not decode, decides whether a local model is usable.** Llama 3.3
   70B waits nearly six minutes before its first token on a 15k-token code
   review. Its decode rate of 4.7 tok/s does not tell you that, and neither did
   any decode rate on its own.
2. **The artefact format matters more than the tool.** Ollama's Apple Silicon
   MLX backend does not reach existing GGUF models; it only engages for models
   published in MLX format. Pull the right tag and the same weights on the same
   Ollama build decode **51% faster**, and 38% faster than stock mlx-lm.
3. **Quality is low across the board.** 52 blind-graded outputs average 2.92 of
   5. Only 4 of 13 review answers found a reverted security fix sitting in the
   diff, and two models invented a race condition in a module with no
   concurrency.
4. **Most tool-calling failures are the runtime's fault, not the model's.**
   Llama 3.3 emits valid tool calls that Ollama parses and mlx-lm discards,
   because no shipped parser matches its format.

A year of model progress bought throughput and agent-compatibility rather than
better code review: Qwen3.6 35B-A3B reaches its first token in less than half
the time Qwen3-Coder 30B-A3B takes and calls tools natively, but scores 3.75
against its predecessor's 4.00.

Section 7 lists the six defects I found in this harness while running it. A
benchmark harness is the least-tested code in the room and reports to one
decimal place whether or not it is right, so the failures are worth as much as
the numbers.

## 2. Hardware and software

| | |
|---|---|
| Machine | Apple M2 Max, 12 cores (8P+4E), 96GB unified memory |
| Arm A | Ollama 0.23.2, `OLLAMA_FLASH_ATTENTION=1`, `OLLAMA_KV_CACHE_TYPE=q8_0` |
| Arm B | Ollama 0.32.14, same flags |
| Arm C | mlx-lm 0.31.3, `mlx_lm.server`, one model resident at a time |
| Models | 5 from 2024-25, plus Qwen3.6 35B-A3B (August 2026) in two formats |
| Context | 32768 tokens for every model, forced via Modelfile |
| Sampling | temperature 0.2, `max_tokens` 1024 |
| Target repo | ComfyUI at `e35348aa` |

Three arms rather than two, because "Ollama vs MLX" is no longer one question.
Ollama on Apple Silicon is now built on MLX, so an Ollama column has to say
which version it is. Arm B is labelled by version rather than by engine for that
reason: as section 8.4 shows, the backend change reaches fewer models than the
announcement suggests, and for the five GGUF models arm B is still llama.cpp.

Context length is forced because Ollama's default is 4096, which silently
truncates a 15k-token diff. A model that never saw the input cannot be graded
on it.

## 3. Measurement

Both engines are driven over a streaming API by one harness and timed
identically:

- **`ttft_s`**: request sent to first content token. This is queueing plus
  prompt prefill.
- **`decode_s`**: first token to last token.
- **`decode_tps`**: `(out_tokens - 1) / decode_s`.
- **`prefill_tps`**: `prompt_tokens / ttft_s`.

Splitting prefill from decode is the single most important fix. A coding agent
sends 10-25k tokens of context before it wants a first token back, so a model
that decodes at 45 tok/s but prefills at 300 tok/s will feel slow in a way no
single "tokens per second" figure can express.

Memory is Metal-resident bytes from Ollama's `/api/ps`, and `phys_footprint`
of the `mlx_lm.server` process for arm C (section 7.2 explains why not RSS). `finish_reason` is recorded per row, so
truncation is visible rather than silent.

## 4. The prompts

Four prompts, each seeded from the same repo state:

1. **Code review** of a real 15k-token diff (`HEAD~5..HEAD`).
2. **Bug hunt** in one complete source file.
3. **Multi-file planning**, capped at 30 lines, from a file listing.
4. **Refactor proposal**: show the original, show the replacement.

### 4.1 Choosing the bug-hunt file

The target is chosen by rule rather than by hand: the hand-written file
that does real work (I/O, binary parsing, concurrency), fits whole inside the
context budget, and has the most control flow. On this ComfyUI revision that
selects `app/assets/services/metadata_extract.py`, a safetensors header parser.
The file is passed complete, not truncated, so a model that misses a defect
cannot blame a missing tail.

### 4.2 Ground truth

Before grading, I read that file and wrote down what is actually wrong with it,
along with the plausible-sounding things that are *not* wrong. The full list is
in `eval/ground-truth-2_bug.md`. Four real defects:

- The inner `try` around `ss_tag_frequency` catches only `JSONDecodeError`,
  while `tag_freq.values()` raises `AttributeError` on valid JSON that is not
  an object.
- Header values are assigned to `str`-typed fields with no `isinstance` check,
  so a dict in `source_url` reaches a `val[:2048]` slice and raises `TypeError`.
- `except OSError: pass` leaves `content_length` at 0 with no log, making an
  unreadable file indistinguishable from an empty one.
- `trained_words` is capped at 100 on two paths and uncapped on a third.

And five false positives that a model can reach for, each of which is wrong
because the code already handles it: the header length *is* bounds-checked, the
file handle *is* in a `with`, there is no concurrency to race, the 8-byte
length read has no off-by-one, and `os.stat_result` is always truthy.

This list is what separates "found a real bug" from "wrote a convincing
paragraph". Without it, grading a bug hunt is just rating prose.

## 5. Grading

Outputs are stripped of self-identifying strings, shuffled into a stable
pseudo-random order, and graded 1-5 against a published rubric
(`eval/rubric.md`) with a one-line justification recorded for every score.

The judge is Claude Opus 5. This is LLM-as-judge and carries the biases that
implies: it rewards well-structured prose, and it cannot be truly blind to
house style. The rubric, the ground truth, and every per-output justification
are published so the scores can be audited instead of trusted. Where the rubric
and a score disagree, the rubric is right and the score is a bug.

Two things are recorded but not scored: truncation at the token cap (a harness
limit, not a model failure) and tool-calling support, which is measured
separately.

## 6. Tool calling

Each model gets one probe per engine: a `tools` array plus a prompt that cannot
be answered without calling it. This is a gate, not a grade. A model that
cannot emit a tool call cannot drive a coding agent, whatever its prose scores.

## 7. What went wrong in this run

Six defects, all mine, all caught during the run. Each would have produced a
confident number if it had gone unnoticed.

**8.1 The memory column read zero.** Ollama's `/api/ps` reports
`qwen3-coder-32k:latest`; `ollama create` registers `qwen3-coder-32k`. The
matcher compared them literally and never matched, so every row recorded
0.00 GB. Caught two minutes into the first arm because a column of zeroes is
obvious. A column of plausible-but-wrong numbers would not have been.

**8.2 The MLX memory metric undercounted by 3x.** `ps -o rss` reported
12.21 GB for Llama-3.3-70B, whose weights are 37 GB on disk. MLX mmaps its
weights and RSS does not count them. Replaced with macOS `phys_footprint`,
validated against a known 3 GB allocation, and re-measured: 37.02 GB against
36.98 GB on disk. A memory column can be wrong for one engine and right for
another, and nothing in the output says so.

**8.3 The reasoning model looked like an empty response.** Qwen3.6 returns its
chain of thought in `message.thinking` (Ollama) and `reasoning_content`
(mlx-lm), separate from `content`. The harness read only `content`, so the model
spent its entire 1024-token budget reasoning, returned no content, and was
recorded at **0.0 tok/s with an empty body**. All twelve Qwen3.6 runs were
invalid. They are quarantined in `invalid-thinking-bug/` rather than deleted,
and the model was re-run with reasoning disabled so it faces the same terms as
the five non-reasoning models.

This one is worth dwelling on. The failure was loud: a decode rate of exactly
0.0 across twelve runs. Had the model split its budget and emitted a *short*
answer, the harness would have reported a plausible rate over a truncated
answer, and the 2026 model would have looked mediocre for reasons that had
nothing to do with the model.

**8.4 The grading packet shredded its own contents.** Outputs were wrapped in
triple-backtick fences. The outputs contain 80 code blocks. The fences collided
and the packet structure broke. Now the wrapper is always longer than anything
inside it. Had this gone unnoticed I would have graded garbled text without
knowing, which is the quietest way for a benchmark to be wrong.

**8.5 The anonymous ids could collide.** Output ids hash
`(engine, model, prompt)`, so arm A and arm B rows for the same model map to the
same id. It never fired, because arm B was not graded, but the guard is now an
explicit error rather than a silent overwrite.

**8.6 A chained job waited on a sentinel that was never printed.** After
restarting the benchmark by hand, the follow-on arm waited for a "BENCH
COMPLETE" line that only the original wrapper emitted. It would have waited
forever. Nothing was lost, but a pipeline that hangs quietly is a pipeline that
silently drops an arm.

## 8. Results

### 8.1 Prefill, not decode, is what makes a local model feel slow

Every "tokens per second" figure you see quoted is a decode rate. It describes the part of the work that happens after the model has read
your code. On the 15k-token review prompt:

| Model | prefill | time to first token |
|---|---|---|
| DeepSeek-Coder-V2-Lite 16B | 1118 tok/s | 25.5s |
| Qwen3.6 35B-A3B | 800 tok/s | 20.9s |
| Qwen3-Coder 30B-A3B | 494 tok/s | 50.8s |
| Codestral 22B | 158 tok/s | 182.9s |
| Qwen2.5-Coder 32B | 103 tok/s | 173.7s |
| Llama 3.3 70B | 47 tok/s | 354.9s |

(Five models measured on arm A. Qwen3.6 is from arm B, since it cannot run on
Ollama 0.23.2 at all, which is itself part of the cost of not upgrading.)

Llama 3.3 sits for **nearly six minutes** before producing a first token. The
same model decodes at 4.7 tok/s, a number that sounds slow but survivable. The
six minutes is the number that decides whether you can use it, and averaging
prefill into decode hides it completely. A 24x spread separates the fastest
prefill from the slowest; the decode spread is 14x.

### 8.2 Ollama versus mlx-lm: no dominant winner

Comparing arm A against arm C, mlx-lm decodes faster on four of five models,
by 5% to 45%. DeepSeek-Coder-V2-Lite is the exception and it is not close:
67.3 tok/s on Ollama against 56.4 on mlx-lm, with prefill 1118 against 720.

That is the whole finding. Not "MLX is faster", which is what a single-number
table would have said. It depends on the model, and the one model where Ollama
wins is the one with the fastest prefill in the study.

### 8.3 Upgrading Ollama buys memory, not speed

Arm A to arm B is Ollama 0.23.2 to 0.32.14, same weights, same GGUF files:

| Model | decode | memory |
|---|---|---|
| DeepSeek-Coder-V2-Lite | 67.3 -> 78.7 (+17%) | 17.7 -> 13.0 GB |
| Qwen3-Coder 30B | 42.4 -> 45.7 (+8%) | 19.0 -> 18.9 GB |
| Codestral 22B | 17.3 -> 16.4 (-5%) | 18.2 -> 15.6 GB |
| Qwen2.5-Coder 32B | 10.8 -> 10.4 (-4%) | 24.7 -> 22.8 GB |
| Llama 3.3 70B | 4.7 -> 4.9 (+4%) | 48.2 -> 45.4 GB |

Every model got smaller, by 0.5% for Qwen3-Coder up to 27% for
DeepSeek-Coder-V2-Lite. Speed moved by single digits in both directions. On a
machine where the 70B costs 45 GB of 96, the memory saving is the upgrade's real
return.

### 8.4 The MLX backend only reaches MLX-format models

Ollama's announcement says Apple Silicon is "now built on top of MLX". Loading
each model and reading the server log says something more specific:

```
qwen3-coder-32k     (GGUF) -> ggml_metal_init ...        llama.cpp
qwen3.6-32k         (GGUF) -> ggml_metal_init ...        llama.cpp
qwen3.6-mlxfmt-32k  (MLX)  -> "MLX engine initialized"   MLX 0.32.0
```

Existing GGUF models keep running llama.cpp after the upgrade. The MLX engine
is reached only by pulling a model published in MLX format, which Ollama
distributes under separate `-mlx` tags. That makes arm B, for those five
models, a newer llama.cpp rather than an MLX comparison, and it is why the
speedups in 9.4 are small.

Pull the right tag and the difference is not small. Same weights, same Ollama
build, same afternoon:

| Runtime | decode | prefill |
|---|---|---|
| Ollama 0.32.14, GGUF | 47.4 tok/s | 800 tok/s |
| **Ollama 0.32.14, MLX format** | **71.7 tok/s** | 689 tok/s |
| mlx-lm 0.31.3, stock | 52.1 tok/s | 562 tok/s |

The MLX-format build decodes **51% faster than the GGUF build of the same
model**, and 38% faster than stock mlx-lm. This is the only comparison in the
study where the format varies and everything else is pinned, and it is the
single most actionable result here: the artefact you pull matters more than the
tool you run it with.

### 8.5 Quality is low, and mostly does not depend on the engine

Across 52 blind-graded outputs the mean is **2.92 of 5**. Two outputs scored 5.
Four scored 1.

| Model | best runtime | avg |
|---|---|---|
| Qwen3-Coder 30B-A3B | Ollama | 4.00 |
| Qwen3.6 35B-A3B | either | 3.75 |
| Llama 3.3 70B | Ollama | 3.50 |
| Qwen2.5-Coder 32B | either | 2.50 |
| DeepSeek-Coder-V2-Lite | mlx-lm | 2.50 |
| Codestral 22B | either | 2.25 |

On the review prompt, **only 4 of 13 outputs found the reverted security fix**,
and all four came from Qwen3-Coder or Qwen3.6. The misses fall into a tidy
pattern: both Qwen2.5-Coder runs answered "no substantive bugs found", and both
Codestral runs returned a file-by-file summary of what the diff did without
offering a single finding. Four outputs, two models, no findings, on a diff that
reintroduces a possible inline-rendering vector at four call sites.

On the bug hunt, **two models invented a race condition** in a module with no
concurrency, one of them proposing `fcntl` locking. Both answers read as
competent. Without a written list of the plausible-but-wrong findings, both
would have been scored as hits. This is the strongest argument in the study for
writing ground truth before grading rather than after.

Engine changed quality by 0.00 for three of five models. The exception is
discussed in 8.6.

### 8.6 The one place format changed quality

Qwen3-Coder scored 4.00 on Ollama and 2.50 on mlx-lm, and the mechanism was
visible rather than statistical. On mlx-lm it looped fifteen near-identical
bullets until it hit the token cap, and on the refactor prompt it reproduced the
"original" code with syntax errors that are not in the file
(`isinstance(ttw, str)`, `st_meta.source_arn)`). The GGUF build did neither.

Because a 1.5-point gap on four samples is not evidence, I re-ran those four
prompts on mlx-lm. Both failure modes reproduced:

- **Repetition collapse.** The second run again produced one bullet repeated
  verbatim until the token cap, about fifteen times. The *content* differed (an
  `interpolate` bounds-check claim rather than division by zero), so this is not
  one unlucky sample; it is what the build does on a 15k-token review prompt.
  Both runs found the security regression correctly in their first bullet, then
  degenerated.
- **Verbatim code corruption.** The second run reproduced the "original" with
  the *same two* fabricated syntax errors at the same places: `isinstance(ttw,
  str)` for `tw`, and `meta.source_arn = st_meta.source_arn)`. Identical
  corruption across independent runs is a systematic defect, not sampling noise.

Speed reproduced closely (33.0, 52.9, 62.5, 50.1 tok/s against 31.5, 50.8,
60.7, 50.4), and the scores were unchanged at 3 and 1 for the two affected
prompts.

I am reporting this as a property of **this 4-bit MLX conversion of this
model**, not of MLX generally, and not of Qwen3-Coder generally. The GGUF build
of the same model, graded by the same rubric, scored 4.00 and did neither of
these things. Quantization schemes both called "4-bit" are not the same
arithmetic, and this is what that can cost.

### 8.7 Tool calling fails three ways, and two are the runtime's fault

A model that cannot emit a tool call cannot drive a coding agent, whatever it
scores on prose. Probing each model on each runtime with a `tools` array:

| Model | Ollama | mlx-lm |
|---|---|---|
| Qwen3-Coder 30B | native | native |
| Qwen3.6 35B (both formats) | native | native |
| Llama 3.3 70B | **native** | **text only** |
| Qwen2.5-Coder 32B | text only | text only |
| DeepSeek-Coder-V2-Lite | refused | prose |
| Codestral 22B | refused | prose |

Three distinct failures hide behind a yes/no column:

- **refused**: Ollama returns HTTP 400, "does not support tools". Codestral's
  mlx-community repo ships **no chat template at all**, so this is the model.
- **text only**: the model emits a correct call and the runtime fails to parse
  it. Llama 3.3 produced
  `{"type":"function","name":"read_file","parameters":{...}}` on mlx-lm and no
  shipped parser matched it, while Ollama parsed the same model fine.
  Qwen2.5-Coder wrapped a valid call in `<tools>` where mlx-lm expects
  `<tool_call>`.
- **prose**: the model answers in English, or writes Python that would read the
  file.

mlx-lm 0.31.3 infers a parser from the chat template and ships eleven of them.
Llama 3.3's template carries `<|python_tag|>` and `tool_call.name` but not
`<tool_call>`, so inference returns `None` and a capable model is reported as
incapable. Three of five failures on mlx-lm are one-line parser gaps, not model
limitations.

### 8.8 What a year of model progress actually bought

Qwen3.6 35B-A3B (August 2026) against Qwen3-Coder 30B-A3B (July 2025), same
architecture family, same 3B active parameters:

- **Quality: no gain.** 3.75 against 4.00. Qwen3.6 wrote the single best output
  in the study, a review that cites RFC 6266 and identifies that `filename` is
  invalid without a disposition type, but it did not beat the older model
  overall.
- **Speed: large gain.** 71.7 against 42.4 tok/s decode, and first token in
  20.9s against 50.8s on the review prompt, less than half the wait.
- **Capability: the real difference.** Native tool calling in both formats,
  256k context against 32k, and reasoning available when you want it.

The headline "new models are much better at coding" is not what this measures.
What improved is throughput and the ability to participate in an agent loop.
For a coding assistant those may matter more than a rubric score, which is what
the second post is about.

## 9. Threats to validity

**Quantization is confounded with engine.** Ollama serves GGUF (Q4_K_M for
these models); mlx-lm serves MLX 4-bit. Both are "4-bit" and neither is the
same arithmetic. A cross-engine speed difference is therefore an engine *and*
format difference, and a quality difference could be either. The arm B
comparison of `qwen3.6:35b` against `qwen3.6:35b-mlx` on one Ollama build is
the only place in this study where the format varies and everything else is
held fixed.

**One machine, one run, no repetitions.** Every number is a single sample.
Thermal state, background load, and page cache are uncontrolled. Differences
under roughly 10% should not be read as real.

**The judge is a language model.** See section 5. Scores are auditable, not
authoritative.

**Prompt length is not balanced.** Prompt 1 is 15-19k tokens; prompts 2-4 are
0.4-3.5k. Averaging "decode tok/s" across the four weights the short prompts
equally with the long one, which is why the per-prompt tables are the ones to
read.

**The generation cap binds unevenly.** `max_tokens` is 1024 for everyone, but a
model that would have written 1500 tokens is cut off while a terser one is not.
Truncation is reported per row rather than corrected for.

**Model builds differ across engines.** The Ollama and MLX artifacts come from
different publishers converting the same upstream weights. They are the same
model in name and parameter count, not bit-for-bit the same file.

**Neither engine was tuned.** Both run stock settings apart from the shared
32k context and two Ollama environment flags
(`OLLAMA_FLASH_ATTENTION=1`, `OLLAMA_KV_CACHE_TYPE=q8_0`). A
tuned deployment of either could look different.

## 10. What I would actually run

On this machine, for coding work, in this order:

1. **Qwen3.6 35B-A3B, MLX format, via Ollama.** 71.7 tok/s decode, 21GB, 21s to
   first token on a large review, native tool calling, 256k context. The fastest
   thing here that can also drive an agent. Pull `qwen3.6:35b-mlx`, not
   `qwen3.6:35b`.
2. **Qwen3-Coder 30B-A3B, GGUF, via Ollama.** The best quality score in the
   study at 4.00, 19GB, native tool calling. Slower to first token, and avoid
   the 4-bit MLX conversion for the reasons in 9.7.
3. **Nothing else.** Codestral 22B and DeepSeek-Coder-V2-Lite score 2.25 and
   2.00 and cannot call tools on either runtime. Qwen2.5-Coder 32B answers a
   15k-token review in 25 tokens. Llama 3.3 70B is the best of the old models on
   quality but costs 45GB and six minutes to first token.

The honest summary is that two of the seven configurations tested are worth
keeping, both from the same vendor, and the older five are superseded.

Whether either can actually drive an agent loop, as opposed to answering one
question well, is a different measurement. That is the next post.

## 11. Reproducing this

The harness is [`agentfit`](https://github.com/jaechoidev/agentfit):
`agentfit speed --engine ollama --repo <a git repo>` produces the speed and
memory tables, and `agentfit check --engine ollama` runs the tool-calling probe
and the format comparison in seconds rather than hours.

The grading rubric and the two ground-truth documents are in `docs/grading/`.
They matter more than the scores: without a written list of the
plausible-but-wrong findings, two models that invented a race condition in a
module with no concurrency would both have been scored as hits.

Run outputs are not checked in. They are regenerable, and a repository that is
mostly one afternoon's CSV files is an archive rather than a tool.
