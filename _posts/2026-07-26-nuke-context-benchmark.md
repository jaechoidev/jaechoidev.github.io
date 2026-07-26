---
title: 'Does grounding still pay? A controlled benchmark of the nuke-context plugin'
date: 2026-07-26
permalink: /posts/2026/07/nuke-context-benchmark/
tags:
  - project, nuke, llm, benchmark, claude
---

**Status: working paper from a partial run — the Opus sweep was stopped at 418
of 760 runs to conserve quota; every completed run is graded. Methods are
settled; intervals are wider than designed.**

## 1. Summary

This is a study of whether the [`nuke-context`](https://github.com/jaechoidev/nuke-agent-context)
Claude Code plugin; a pure-content bundle of skills, version-pinned API
tables, community-distilled field guides, and worked examples for Foundry Nuke
tool development, actually improves an agent's Nuke work, measured against a
live, licensed Nuke 17, with paired statistics, on tasks the plugin has never
shipped as examples.

The design is a 2×2 factorial: plugin present/absent crossed with live-Nuke
execution feedback present/absent, five trials per task per arm over a frozen
set of 38 tasks spanning Nuke Python, BlinkScript, and the NDK (C++). Grading
is a deterministic gate pyramid (deliverable exists → syntax/compile →
behavioural assertions executed inside the running Nuke), with an
invented-API count as the secondary metric and a blinded LLM judge for the
quality dimensions no gate can assert. Analysis is paired at the task level:
bootstrap CIs on per-task pass-rate differences, McNemar's exact test on
task-majority outcomes, pass^3 for reliability, and compute parity (tokens,
turns, wall-clock) so a win cannot silently be "more compute."

One of this document's stated purposes is to make it possible to conclude,
without embarrassment, that the plugin is *not* worth its cost, the study is
built so that either answer is publishable. A previous, cruder ablation found
a large positive effect; this study exists because that result had known
confounds and because base models have since improved. The other purpose is to
record honestly what building a trustworthy measurement cost: the harness was
wrong in six materially different ways before it was right, and each defect
would have produced a confident, false number if published when first seen.

## 2. Motivation and prior work

### 2.1 The earlier result

An 8-case ablation run in July 2026 tested the same grounding hypothesis
against the NDK's most obscure surface (deep ops, scanline caching, runtime
knobs, custom readers, temporal access), with 3 runs per case per arm on a Sonnet-class model, graded objectively by
the compiler. Headline: compile rate **38% → 79% (2.1×)** with grounding,
and **−70% invented API references** (20 → 6). The baseline's fabrications
were the motivating failure mode in the flesh: `GeometryList::compute_normals`,
`DeepPlane::exists`, `Input_Channel_knob`, `#include "DDImage/Lock.h"`, every
one plausible, the right shape, the right neighbourhood, and nonexistent.

That document's more important half was methodological: *"the result is only
trustworthy because the measurement was wrong three times first."* Its grader
initially measured class names instead of method calls (scoring
hallucination-ridden files "100% clean"); a permission denial was misread as
timeouts and nearly published as a plugin-arm collapse; and the confound was
only found because the summary arithmetic refused to reconcile against its own
detail. That theme (the eval itself is the least-tested code in the room)
recurs throughout the present study (§7).

### 2.2 Why re-test

That study's own caveats are the current design's requirements:

- **n=3, 8 cases, one model**, directional, not a benchmark.
- **The comparison was end-to-end**, plugin conflated with a set-up project
  (CLAUDE.md, generated index); it credited the whole package, not the
  grounding mechanism.
- **Compile-only grading**, "compiles" and "no invented API" are necessary,
  not sufficient; nothing checked that the tool *worked*.
- **No environment isolation**; both arms inherited the operator's global
  CLAUDE.md, other plugins, and MCP servers.

Two further reasons: the plugin has since been rebuilt as a self-contained
Claude Code plugin (skills + refs + references + examples, no setup step), so
the deliverable under test is different; and frontier base models have
improved enough that the central empirical question has genuinely changed from
"does grounding help?" to "does grounding *still* help, and where?"

## 3. What the plugin is

The subject under test is the `nuke-context` plugin directory at
`nuke-agent-context/plugins/nuke-context`, passed to `claude -p` via
`--plugin-dir`. It is pure content; no hooks, nothing executable at install
time. Its parts, as of the pinned commit recorded in every run:

**Six skills** (progressive-disclosure instructions the agent loads on
demand):

| Skill | Role |
| --- | --- |
| `nuke-tool-structure` | The paradigm document: pull-based per-scanline evaluation, the Op contract, thin-shell/pure-core structure, test-first workflow, a verification ladder. Read before writing any Nuke code. |
| `nuke-api-lookup` | "Never write a Nuke API you have not looked up", routes the agent into the version-pinned symbol tables and real headers/docs before it writes a signature. |
| `nuke-python-model` | The live node graph, knob/animation model, callbacks and deferred evaluation, gizmo/group authoring. |
| `nuke-ndk-model` | The DD::Image Op contract, thread-safe `engine()`, Interest caching, channels/bbox/format, plugin registration. |
| `nuke-blink-model` | The kernel/GPU per-pixel paradigm, access patterns (point/ranged/random), param/local/define/init/process, image specs. |
| `nuke-performance` | Choosing Python vs Blink vs NDK by granularity, scanline request/engine discipline, bbox/channel hygiene, hashing/caching correctness, Blink GPU cost. |

**Version-pinned API references** (`refs/`), machine-extracted from real local
Nuke installs by maintainer-run tooling, one directory per supported version
(`nuke-15.2`, `nuke-16.1`, `nuke-17.0`), each containing: `python_symbols.tsv`
(~6,100 rows for 17.0), `python_methods.txt` (~540 signatures),
`symbol_map.tsv` (NDK: 523 symbols for 16.1/17.0, 456 for 15.2, with doc
URLs), `blink_symbols.tsv` (~70 rows), plus index files and page-anchor maps
into Foundry's dev guide, Python guide, and Blink guide
(`devguide_map.tsv` ~230 rows, `pyguide_map.tsv` ~180, `blinkguide_map.tsv`).
`VERSIONS.md` records build provenance per directory; notably 16.1 and 17.0
ship byte-identical DDImage APIs (same index hash), and both directories are
kept so version pinning stays explicit.

**Five community field guides** (`references/`): `ndk.md`, `blink.md`,
`python.md`, `pyside-panels.md`, `tool-architecture.md`, ~440 lines total of
distilled practitioner knowledge, built from the curated source list in
`SOURCES.md` (reproduced in §12.2) under the rule *"only extract facts the
model would otherwise get wrong; paraphrase, attribute with a link per claim,
never republish prose."*

**33 worked examples** (`examples/` + `INDEX.md`): 14 BlinkScript kernels
(including 9 Book-of-Shaders-style pedagogical kernels, gradient, circle,
grid, matrix, random, noise, cellular, fbm, plus Dilate2D, RadialBlur,
Exposure, HorizontalMax, SDFRaymarcher), 11 NDK `.cpp` plugins plus a CMake
template (DeepGain, DeepPrune, EdgeDetect, Exposure, FrameBlend, Gradient,
MinimalReader, MixInputs, Premult, TimeOffset, Vignette), and 8 Python tools
(backdrop_selected, bake_expression, dockable_panel_stateful, gizmo_builder,
knob_linker, pyqt_panel, render_submitter_shape, select_downstream).

The examples tree matters to this study for an unhappy reason: it grew *after*
the benchmark task set was authored and came to overlap it, contaminating both
pilots (§7.5).

## 4. Study design

Full spec: `docs/superpowers/specs/2026-07-25-nuke-benchmark-design.md`;
methodology sources: `docs/research/2026-07-25-eval-methodology.md`.

### 4.1 Arms: 2×2 factorial

| arm | `--plugin-dir` | live-Nuke feedback |
| --- | --- | --- |
| `base` | no | no |
| `plugin` | yes | no |
| `base+fb` | no | yes |
| `plugin+fb` | yes | yes |

The **plugin delta** is exactly one CLI flag. No project CLAUDE.md, no setup
step, unlike the earlier ablation, this isolates the plugin itself. The **feedback
delta** is a copy of a small `nukebridge.py` script in the run cwd plus one
prompt paragraph telling the agent it may test Python/Blink in the running
Nuke 17 (and, for NDK tasks, the exact `clang++ -fsyntax-only` line). The
paragraph and script are identical across both feedback arms, so
execution-feedback gains cannot be misattributed to the plugin, a documented
ablation pitfall (arXiv:2603.22184).

**Trials:** 5 per task per arm (the Terminal-Bench standard), interleaved
round-robin across arms within each trial index so API-side drift over hours
does not correlate with arm. At the frozen 38-task set this is 38 × 4 × 5 =
**760 runs per subject model**.

**Model:** one pinned model per study; the runner records the requested model
in every record and `analyze.py` refuses to mix models in one results file.
Sweep support (`--model`, per-model results directories) exists so the same
study can be repeated per model; the currently running sweep is
`claude-opus-5`, with `claude-sonnet-5` as the other configured subject.

### 4.2 Hermetic isolation, empirically verified

The earlier ablation's silent flaw was environmental: both arms inherited the operator's
global config. Here every trial runs `claude -p` with `--setting-sources ""`
(no user/project/local settings), `--strict-mcp-config` with no MCP config (no
user MCP servers), `--no-session-persistence`, in a freshly created temp cwd
outside any project (so no CLAUDE.md discovery), under
`--permission-mode bypassPermissions`.

Isolation is not assumed, `preflight.py` proves it before any paid run with
**canary probes**: a real `claude -p` call per arm configuration asking the
model to report, as strict JSON, only what is *active* in its context
(skills, MCP tools, plugins, CLAUDE.md sources, memory files). The base-arm
probe must show none of the operator's config (canaries include the operator's
distinctive Co-Authored-By commit rule, the superpowers plugin, claude-mem)
and no nuke-context content; the plugin-arm probe *must* show the plugin's
skills (proving plugins load headless at all). A canary failure is a designed
abort: fix the environment, never the canaries. Preflight additionally pings
the Nuke bridge and asserts Nuke 17, verifies the NDK toolchain against a
known-good fixture, runs the task/example contamination guard (§7.5), and
re-validates every task's reference solution through the full grading
pipeline.

### 4.3 Analysis

Primary: the **plugin effect within each feedback mode**, paired at task
level, per-arm pass@1 with task-clustered SEs, per-task pass-rate
differences with bootstrap 95% CIs, **McNemar's exact test** on task-level
majority outcomes, and **pass^3** (unbiased, from 5 trials) as the
reliability metric. Secondary: the feedback effect and plugin×feedback
interaction, reported descriptively (the design is powered for the plugin
effect only). Judge scores get Wilcoxon signed-rank on per-task means, and
the order-swapped pairwise judge gets an exact binomial test over decided
pairs. Compute parity, tokens including cache read/creation, turns,
wall-clock, notional cost, is reported per arm. `analyze.py` enforces a
bookkeeping invariant per arm (`completed + errored + timed_out == run
count`, no duplicate run ids, graded records have boolean outcomes): Pass
1.5's Bug 3 (§2.1), institutionalised.

Honesty about power, stated up front: at ~35 non-control tasks this design
detects roughly 15-25-point effects. A domain plugin worth its context cost
should show a large effect; deltas smaller than the minimum detectable effect
will be reported as CIs, not claimed.

## 5. The task set

38 tasks in per-task directories (`tasks/<layer>/<tier>/<task-id>/` with
`task.json`, `verify.py`, and a validated `reference/` solution), frozen
before the sweep: **17 Python** (3 control, 4 easy, 4 medium, 6 hard),
**12 BlinkScript** (3 easy, 3 medium, 6 hard), **9 NDK** (2 easy, 2 medium,
5 hard). Eleven of the hard-tier tasks form a distinct *architecture* series
(`*-arch-*`, §5.2). Task ideas were sourced from a 915-tool Nukepedia
manifest, rephrased in original words to reduce training-data contamination,
and checked against the plugin's shipped examples (§7.5); each `task.json`
records its source slug or `original`.

Representative spread:

- **Python easy → hard:** create a named Blur node (smoke); register a knob
  default; configure an EXR Write; align nodes on an axis; fan a node into
  per-layer Shuffles; a knobChanged callback mirroring a Blur's size into its
  label; expression-link Grades to a master gain; prefix-rewrite file paths
  with a change report; snapshot node layout to CSV and rebuild it.
- **Blink easy → hard:** invert with a mix parameter; posterize; box blur
  with a radius knob (ranged access); chromatic aberration by opposed channel
  shifts (random access); luma keyer; green despill with selectable
  algorithm; tetrahedral RGB colour warp; relighting from position/normal
  passes.
- **NDK easy → hard:** pixel-wise invert Iop with a mix knob; per-channel
  gain with AColor/master knobs; spatial maximum filter with bbox expansion;
  a mode pulldown with runtime knob enabling; a ModifyGeo displacing points
  along normals.
- **Controls (3, Python):** generic programming with no Nuke content,
  summarize a render-log CSV, expand a frame-range spec, increment a version
  token in a filename. If the plugin arm beats base *here*, the study has a
  confound, not a domain effect.

### 5.1 Grading pyramid per layer

| gate | python | blink | ndk |
| --- | --- | --- | --- |
| G0 exists | deliverable file present, non-empty | same | same |
| G1 valid | `ast.parse` | kernel compiles in the live Nuke (kernelName oracle, §6.2) | `clang++ -fsyntax-only -std=c++17` against the real Nuke 17 NDK headers + include-existence check |
| G2 behaviour | execute in live Nuke, then the task's `verify.py` assertions (graph state, knob values, computed pixels) | render over a test pattern via the bridge; `verify.py` asserts pixel values within tolerance | (compile-graded; runtime loading out of scope) |

Pass is binary per run: all applicable gates. A failed gate is a failed run
regardless of any judge opinion. NDK's pass is G1; a deliberate scope cut
(loading third-party binaries into the licensed session was excluded), and a
stated limit on what an NDK "pass" means (§8).

### 5.2 The architecture tier

The `*-arch-*` tasks (4 Python, 3 Blink, 4 NDK) encode a design idea: make
architecture *objectively* gradeable by pinning a module API surface in the
prompt, so that structure is verified by deterministic assertions rather than
judged by an LLM. `py-arch-01-graph-conform`, for example, requires a **pure
planner** `plan_conform(current, edits)` over plain data, exercised by the
verifier with node classes that do not exist, so any implementation that
consults Nuke inside the planner fails, plus a separate applier, with
idempotency ("running it a second time changes nothing"), ordered
conflict-vs-action semantics, and non-mutation of inputs all pinned and
asserted. The other Python arch tasks pin transactional expression batches
with cycle refusal, a dependency-aware render scheduler, and a self-healing
managed node chain. Blink arch tasks pin kernel class names, input order and
graded edge behaviour (e.g. `bl-arch-01-vector-push`: clamped out-of-bounds
sampling, exact identity at strength 0, verified at the borders); NDK arch
tasks pin the full source-Op contract (exact knob list, `_validate`
obligations, engine semantics) for a flat-to-deep promote, a strict
bbox/request/channel integer translate, an SDF raymarching generator, and a
Whitted-style raytracer. The raymarcher/raytracer pair is also a deliberate
*transfer* test: the plugin ships a Blink SDF raymarcher example but no NDK
one, so success in the NDK requires carrying the concept across layers, not
retrieving a shipped answer.

A further six **version-drift** tasks (targeting API surface that changed
across Nuke 15.2/16.1/17.0, where the plugin's per-version refs should matter
most) are being authored at the time of writing and are not yet in the tree;
they are out of scope for this document beyond this mention.

## 6. Measurement apparatus

### 6.1 The bridge to live Nuke

`harness/bridge.py` is a minimal TCP client for the
[kleer001/nuke-mcp](https://github.com/kleer001/nuke-mcp) addon socket
(newline-delimited JSON on port 54321), talking directly to the addon's server
inside a running, licensed GUI Nuke 17, deliberately *not* through the MCP
layer, so no MCP configuration exists in any arm. The exec bridge is a single
shared session: every caller holds a global `NUKE_LOCK`, and a full node-graph
cleanup (`nuke.delete` on all nodes) runs before and after each use. Because
failed Blink compiles pop modal dialogs in GUI Nuke that would wedge the
shared session, the harness installs a **dialog guard** inside Nuke's event
loop (a 250 ms QTimer that closes any visible QMessageBox/QErrorMessage)
whose liveness is verified by a tick counter, for reasons §7.2 makes painful.

Feedback-arm agent sessions hold `NUKE_LOCK` for their entire duration (the
agent mutates live graph state), as does all Python/Blink grading; only
no-feedback generation runs execute in parallel (default 3 workers). The
runner is resumable (last terminal record per run id wins), retries the *same*
run on rate-limit signatures with backoff, and kills the whole process group
on timeout so orphaned children cannot keep mutating Nuke after the lock is
released.

### 6.2 The Blink compile oracle

Compiling a Blink kernel programmatically is subtle in exactly the way that
produces false PASSes: setting the inline `kernelSource` knob is ignored (the
node keeps its cached kernel), `recompile` never raises, and `node.error()`
stays False on failure. The one reliable signal is the read-only `kernelName`
knob: a successful compile sets it to the declared kernel class; a failure
leaves the previous name. The oracle therefore loads the kernel from a file,
executes `reloadKernelSourceFile` + `recompile`, and requires `kernelName` to
equal a class name actually declared in the source, using `findall` over all
`kernel <Name>` matches, because a prose comment such as
`// box blur kernel with...` also matches the pattern (a first-match parse
failed on exactly that, §7).

### 6.3 NDK: syntax-only compile against real headers

`gate1_ndk` runs `clang++ -fsyntax-only -std=c++17` against the genuine Nuke
17.0v3 NDK headers. Invalid `#include "DDImage/*.h"` lines are detected first
(commented-out includes are inert) and blanked before compiling, a missing
header is a fatal error that would otherwise hide every later diagnostic,
then recorded as invented APIs in their own right.

### 6.4 Invented-API detection per layer, and its limits

The plugin's core historical claim is fabrication reduction, so the metric is
implemented per layer with the strongest available oracle, and its limits are
stated:

- **NDK; the compiler is the oracle.** Compile errors are classified by
  pattern (`no member named 'X' in 'Y'`, `unknown type name`, `use of
  undeclared identifier`, missing include, `no matching function for call`)
  into invented-API vs ordinary C++ mistakes; the fix for that study's first grading bug,
  ported. Limit: a fabricated API that happens to collide with a real
  signature is invisible; ordinary-C++ counts absorb some genuinely confused
  code.
- **Python; the runtime is the oracle.** During G2 execution inside Nuke,
  `module 'nuke' has no attribute 'X'` / `'Node' object has no attribute 'X'`
  in the failure details are harvested. Limit: only the *first* runtime error
  surfaces; fabrications behind an earlier failure go uncounted.
- **Blink; no oracle exists, so a conservative static scan.** Blink's
  compiler diagnostics never reach the harness (no error-text API on the
  node, nothing on stdout/stderr). Instead, a symbol table of 106 entries was
  extracted from Foundry's own Blink User Guide shipped inside the Nuke
  bundle (never from the plugin-under-test's refs), and kernel source is
  scanned for identifiers in API positions, called functions, declaration
  types, `eXxx` enums, and fabricated framework hooks (a defined non-special
  member function whose parameter list names an unknown type, the shape of an
  observed pilot fabrication `defineParams(ParamDefinitionContext&)`), that
  the docs never establish and the kernel never declares. The scan is
  recall-sacrificing by design: since a kernel that *compiles* cannot have
  called a nonexistent API, scan hits are only counted when the compile
  failed; on passing compiles the hits are surfaced under a diagnostic key
  the analysis never counts, so any value there is a known scanner false
  positive to investigate. Validation: zero false positives across all nine
  Blink reference kernels and all 21 Foundry example kernels; the pilot
  fabrication fixture returns exactly its two invented names. Limits, stated:
  real-but-undocumented Blink API (`rcp`, `atomicAdd`,
  `ImageRollingKernel`) would be flagged on a failing kernel, and a
  fabricated zero-argument hook is missed; both accepted under the rule
  that a false "invented" on correct code corrupts the metric worse than a
  miss.

### 6.5 The blinded judge

Quality dimensions that no gate can assert (API idiom, structure, domain
conventions) go to an LLM judge (`claude-opus-5`) under strict blinding: it
sees only the task prompt, an anchored 1-5 rubric per dimension, and the
deliverable *after* a stripping pass that removes comments naming the plugin
or its skills, any line mentioning the feedback bridge, arm tokens, and even
the run-id shape that leaks via hardcoded temp paths. It never sees the
transcript, the arm, or run metadata, and its own invocation never loads the
plugin. Each artifact is judged three times; the per-dimension median is
stored. Judge scores never affect pass/fail.

Calibration exposed a problem the literature predicts: the absolute rubric
**saturates**; the calibration run scored 4s and 5s on every dimension for
every artifact, separating nothing. The response was an additional
**order-swapped pairwise mode**: for each (task, trial), the base-arm and
plugin-arm artifacts are compared directly, judged in *both* presentation
orders, and a win is counted only when the orders agree; disagreement is
recorded as a tie with a position-bias flag, reported as a diagnostic on the
judge itself. Outcomes append to a separate file and are analysed with an
exact binomial test over decided pairs.

## 7. What went wrong and what it cost

This section is the study's second deliverable. Every item below is a defect
that existed in a "working" harness, was found before (or, in two cases,
*by*) spending significant sweep budget, and would have corrupted the
headline number if left in. The chronological record is the SDD ledger
(`.superpowers/sdd/2026-07-25-nuke-agent-benchmark/progress.md`).

### 7.1 The preflight that could pass for the wrong reasons

The isolation canaries went through three false-verdict vectors. First,
substring-matching the probe's *prose*: a thorough model (observed with Opus)
helpfully inspected `~/.claude` on disk and reported the exact canary strings
as *present but not loaded*, proving isolation while failing the check; and
conversely, an errored probe run would have its error message scanned,
passing vacuously. The probe was rewritten to forbid filesystem inspection
and demand structured JSON of *active context only*, matched against parsed
lists, with unparseable output treated as failure, never as "nothing active."
Second, vacuous-pass guards had to be added elsewhere too: an empty task list
made "all references pass" trivially true. Third, in the opposite direction,
the model-pinning check first required *every* billed model to match the
pinned one, but Claude Code bills auxiliary calls to Haiku regardless, so
the check rejected every real run and was relaxed to presence-not-exclusivity.

**Lesson:** a checker is code and fails like code, in both directions. Test
that it fails when it should *and* that it cannot pass vacuously; an
unparseable answer, an error message, or an empty input must each be a loud
failure, not a silent pass.

### 7.2 The dialog guard that installed successfully and did nothing

The Blink compile oracle requires GUI Nuke's modal error dialogs to be
auto-dismissed. The first guard installed cleanly and reported success, and
its timer callback then died with `NameError` on every tick, silently,
forever. Root cause: the nuke-mcp addon execs payloads with *separate*
globals/locals dicts, so a function defined in a payload cannot see the
payload's own top-level names at call time. The fix is a fully
self-contained callback (every import and name lives in its body, state
parked on `__main__`) plus a **liveness probe**: the harness asserts the
guard's tick counter is advancing, because "installed" is not "working." The
same split-namespace pitfall was then audited across every other payload the
harness sends into Nuke.

**Lesson:** in any remote-execution seam, verify liveness/effect, not
installation. A component that can fail silently on every invocation will.

### 7.3 The grader that penalised better-structured code

`verify.py` originally executed candidate solutions with a bare
`exec(open(SOLUTION_PATH).read())` in a namespace whose `__name__` was not
`"__main__"`. Any solution structured the professional way, work wrapped in
functions behind an `if __name__ == "__main__":` guard, executed nothing,
created nothing, and graded FAILED. The defect was invisible across the
harness's own reference solutions (written unguarded) and across most of the
Sonnet pilot, then surfaced immediately when the Opus pilot began: the
stronger model writes main-guarded scripts far more consistently, so the
grader systematically punished exactly the discipline the study hoped to
measure, and made the cross-model comparison meaningless. The fix
(`run_solution`) execs the source exactly once in a namespace with
`__name__ == "__main__"` and the correct `__file__`, never invokes any
`main()` a second time (double-executing would double side effects like node
creation), and returns the namespace so verifiers can reach defined
functions. Both pilots were archived as invalid.

**Lesson:** graders encode structural assumptions about solutions. The
methodology review had literally warned to "validate the grader against
reference solutions in more than one style"; and the harness still missed
it, because all its references shared one style. Anticipate that the
*stronger* subject diverges from your references first, so grader bias
correlates with subject quality: the most dangerous kind.

### 7.4 The metric that counted its own failure message as a fabrication

The Blink invented-API field was first populated from compile-failure text.
But the compile check's failure string is the harness's *own* prose,
`ERROR: kernel did not compile (kernelName=...)`, which says nothing about
hallucinated APIs, and bridge transport errors could land in the same field.
The metric was counting its own error messages as evidence of model
fabrication. Fixed in three steps: failure prose banned from the field;
bridge/transport failures now *raise*, marking the run `grade_pending` for
idempotent regrade instead of polluting any metric; and the field is fed only
by the doc-grounded static scan (§6.4), counted only on failed compiles. In
the same family: the first kernel-name parse took the *first* `kernel \w+`
match in the source, which a comment could shadow; a correct kernel with a
chatty header comment graded as failing to compile.

**Lesson:** every metric field must contain only the thing it claims to
count, and infrastructure failure must be structurally distinguishable from
subject failure; a metric that can absorb error prose will eventually
report the harness's bugs as the model's.

### 7.5 The contamination discovered mid-study

The task set was authored when the plugin shipped no `examples/` directory,
the ledger records that fact at authoring time. The plugin then gained
examples *while the study was in flight*, and the overlap was found only
after the Sonnet pilot completed: `py-hard-01-bake-expression` was
substantially `bake_expression.py`, `py-hard-03-auto-backdrop` was
`backdrop_selected.py` (same padded-bbox algorithm, including the margin
parameter), `ndk-hard-01-deep-zfilter` was `DeepPrune.cpp` with a different
threshold rule; and the user's proposed SDF-raymarcher task would have had
its literal answer shipped as `SDFRaymarcher.blink`. Both pilots' plugin-arm
numbers therefore contained an unknowable "shipped the answer" component,
concentrated in the hard tier where the apparent gain was largest; both are
archived with a DO-NOT-PUBLISH README. The remedy kept the tasks clean rather
than embracing memorisation: the three same-tool tasks were deleted, the
raymarcher/raytracer became NDK-only transfer tests, moderate
technique-overlap tasks were kept but tagged (`example-overlap-moderate`),
and a **permanent preflight guard** now diffs the task set against the
plugin's examples on every session, recall-biased term/stem matching whose
strong same-layer hits fail preflight unless a human has recorded an
adjudication (verdict + rationale) in a committed baseline file, with
`--allow-overlap` reserved for deliberately measuring example value.

**Lesson:** in an ablation, the subject under test is part of the
measurement configuration. If it can change mid-study; and a plugin under
active development will; the boundary between subject and task set must be
guarded by machinery, not memory. Freeze both sides, or automate the diff.

### 7.6 The resume machinery's stale-state traps

760-run sweeps on subscription billing *will* be interrupted, so resumability
is core machinery; and it had its own defects: a resumed or retried run
re-used its artifact directory, so a stale deliverable from a previous
attempt could be graded as if the new run produced it (fixed: artifacts are
cleared before each attempt); a torn write plus resume could duplicate a
runs.jsonl line (fixed: analysis dedupes by run id, last record wins); runs
that errored in a burned usage-limit window stayed terminally "done" (fixed:
`--retry-errored`); and a timeout initially killed only the direct child
while the CLI's grandchildren kept mutating Nuke after the lock was released
(fixed: process-group kill). None of these touches the science on a clean
run; all of them corrupt it quietly on the messy real runs.

**Lesson:** resume/retry paths are part of the measurement and deserve the
same adversarial testing as the graders. Anything that can survive from a
previous attempt eventually will.

## 8. Threats to validity

- **Contamination and the adjudication baseline.** The strong overlaps are
  removed, but the kept moderate-overlap tasks (e.g. box blur vs a shipped
  ranged-access dilate; knobChanged tasks vs a knob-linker example) mean the
  plugin arm still benefits from shipped *technique*, deliberately, since
  teaching technique is what a plugin is for, but the per-task breakdown
  should be read with the `example-overlap-moderate` tag in hand. The
  adjudication baseline also mutes reviewed rows permanently; a wrong
  adjudication stays wrong silently. And rephrasing notwithstanding, most
  non-`original` tasks descend from public Nukepedia tools that may be in
  training data for *both* arms.
- **Ceiling effects.** Frontier models may saturate the easy tiers, and the
  absolute judge rubric demonstrably saturates (§6.5). Both compress the
  measurable delta toward zero; per-tier breakdowns and the pairwise judge
  are the mitigations, not solutions.
- **Web access is enabled in both arms.** Nothing disables the CLI's built-in
  web tools, so the base arm can fetch Foundry documentation itself. The
  measured effect is therefore the plugin's *marginal* value over an agent
  that can already browse the docs, arguably the deployment-relevant
  question, but a different (and smaller) quantity than "grounding vs no
  grounding," and it adds network-dependent nondeterminism to both arms.
- **`bypassPermissions` means reachable-but-not-loaded.** The canaries prove
  the operator's config and the plugin are not *loaded* in the base arm; they
  cannot prove the agent never *reads* something it should not, since file
  permissions are bypassed. Nothing points the base arm at the plugin's repo,
  but the isolation is behavioural, not physical. (The flag itself is
  defensible for the same reason it was there: it reproduces "the agent is
  allowed to read what it needs," which is the realistic interactive state.)
- **One machine, one licence, one live Nuke session.** All feedback runs and
  all live grading share a single GUI Nuke process on one macOS/arm64
  machine. Node graphs are wiped between uses, but process-level state
  (caches, the Blink kernel cache, memory pressure) persists across the whole
  sweep, and a wedged Nuke serialises everything behind it. Results are from
  Nuke 17.0v3 on this platform, full stop.
- **Judge limits.** The judge is a Claude model evaluating Claude outputs;
  family self-preference largely cancels between arms, but plugin-arm *style*
  may resemble judge-preferred style in ways blinding cannot remove.
  Position bias in the pairwise mode is measured and controlled
  (order-swapping), not absent.
- **NDK passes are compile-passes.** A compiling, symbol-clean NDK plugin has
  not been loaded, instantiated, or rendered. The NDK column measures
  fabrication and API correctness, not working tools.
- **Task-set size and power.** ~35 non-control tasks × 5 trials detects
  effects of roughly 15-25 points; anything smaller is a CI, not a claim.
  Per-layer and per-tier cells are smaller still and are reported
  descriptively.

## 9. Results

**Status: partial run.** The Opus sweep was stopped by the operator at 418 of
760 runs to conserve quota, after the picture had stabilised. Every completed
run is graded (418/418, no errors, no timeouts, no pending grades). Depth is
3 trials per task on the no-feedback arms and 2-3 on the feedback arms, not
the planned 5, so intervals are wider than designed and pass^3 is unreliable
for the feedback arms (a task with fewer than 3 countable trials contributes a
structural zero). Numbers below are from `results/claude-opus-5/`, model
`claude-sonnet-5`'s sibling `claude-opus-5`, 35 non-control tasks plus 3
controls.

### 9.0 In brief

Section 9 is written from the 418 completed runs, and the picture is coherent
enough that stopping early cost us little.

The headline is a null result, honestly reported. Base 0.924, plugin 0.952 —
a paired difference of +2.9 points with a CI spanning zero and McNemar
p = 0.50, off just two discordant tasks out of 35. With feedback the plugin
is fractionally behind (0.976 vs 0.986).

Three findings are worth more than that headline.

**NDK is the one place grounding pays**: 0.852 → 1.000, +14.8 points, and
it's the only compile-graded layer, so a compiler rather than an author's
assertion is the judge. That same layer, and only that layer, separated in
both discarded pilots too — it's the most reproducible thing we found.

**Execution feedback beats the plugin.** Letting the agent test its own work
in Nuke moved base from 0.924 to 0.986, a bigger gain than the plugin
delivered, and it closes the NDK gap unaided. Stacking the plugin on top adds
nothing. That's the practical conclusion: when an agent can compile and run,
it finds its own API errors, and preloading the same facts becomes redundant
— which the literature anticipated.

**And the mechanism the plugin was built for has largely closed.** One
invented API in 418 runs. The original result predicted a large reduction
from a high baseline; the baseline isn't high anymore.

The cost framing is stated plainly below: 2.5× tokens and 2.4× turns for a
difference a paired test cannot separate from noise.

The study also records what it does not establish — 31 of 38 tasks never
failed, so a ceiling can't prove absence of an effect, and the version-drift
tasks remained the strongest untested route.

**Post-script: the Sonnet NDK sample is in**, aimed squarely at the one
signal that had held up. Six NDK tasks (four architecture-tier, two of the
new version-drift tasks), one trial per cell, 11 of 12 cells recorded: every
completed run passed in both arms, zero invented APIs anywhere. At one trial
per cell this is descriptive, not conclusive — but the NDK grounding effect
did not reproduce on a Sonnet-class model. Even the layer that separated on
Opus reads at ceiling, with or without the plugin.

### 9.1 Headline

| arm | pass@1 | 95% CI (clustered) | mean turns | mean tokens | mean $ (notional) |
|---|---|---|---|---|---|
| base | 0.924 | [0.848, 1.000] | 6.1 | 193k | 0.37 |
| plugin | 0.952 | [0.905, 1.000] | 14.7 | 482k | 0.69 |
| base+fb | 0.986 | [0.958, 1.000] | 17.5 | 658k | 0.88 |
| plugin+fb | 0.976 | [0.943, 1.000] | 21.7 | 831k | 1.07 |

Paired, within feedback mode:

| mode | mean delta pass@1 | bootstrap 95% CI | discordant b/c | McNemar exact p |
|---|---|---|---|---|
| no feedback | +0.029 | [-0.029, 0.105] | 0 / 2 | 0.50 |
| feedback | -0.010 | [-0.029, 0.000] | 0 / 0 | 1.00 |

The plugin's effect on task success is not statistically distinguishable from
zero in either mode. Only two tasks out of 35 ever discriminated between the
arms without feedback, and none did with feedback.

### 9.2 The one real signal: NDK

| layer | base | plugin | base+fb | plugin+fb |
|---|---|---|---|---|
| blink | 1.000 | 1.000 | 1.000 | 0.972 |
| ndk | 0.852 | **1.000** | 1.000 | 1.000 |
| python | 0.905 | 0.881 | 0.964 | 0.964 |

NDK is the only layer where grounding pays: +14.8 points, and it is also the
only compile-graded layer, so the result rests on a compiler rather than on an
assertion an author chose. This is consistent across every measurement taken
in this project: the same layer, and only that layer, separated in the
discarded pilots too. Blink and Python are at or near ceiling in every arm.

### 9.3 Execution feedback dominates grounding

Giving the agent a way to test its own work in the running Nuke moved the base
arm from 0.924 to 0.986. That is a larger improvement than the plugin
produced, from a much cheaper intervention, and it closes the NDK gap on its
own (base+fb reaches 1.000 without the plugin). Adding the plugin on top of
feedback produced no further gain (0.976 vs 0.986, delta -0.010).

The practical reading: when an agent can compile and run, it discovers its own
API errors, and pre-loading the same facts becomes redundant. The prior
literature anticipated exactly this (Section 12.1, the quantum-code ablation
in which execution feedback beat domain retrieval).

### 9.4 Invented APIs: the failure mode has largely closed

One invented API reference across 418 runs (a single base-arm instance);
zero in all three other arms. The hypothesis under re-test predicted a large
reduction from a high baseline. The baseline is no longer high. On this task
set a current model does not meaningfully fabricate Nuke APIs, with or without
grounding, which removes the mechanism the plugin was principally built to
address.

### 9.5 Cost

The plugin arm spent 2.5x the tokens and 2.4x the turns of base for +2.9
points that a paired test cannot separate from noise. With feedback the
comparison is 831k vs 658k tokens for -1.0 point. Costs shown are the CLI's
notional API-equivalent figures; the study ran on a Claude subscription, where
the binding constraint was the usage window rather than dollars.

### 9.6 Validity checks that passed

Controls (3 pure-Python tasks requiring no Nuke knowledge) sat at 1.000 in all
four arms, so no confound is inflating the plugin arm. The four tasks sharing
a technique with a shipped plugin example scored 1.000 in every arm, so
residual contamination is not driving the headline; the clean-task slice
(n=31) shows the same pattern as the whole (base 0.914, plugin 0.946,
base+fb 0.984, plugin+fb 0.973). Bookkeeping reconciles exactly: 418 records,
418 graded, zero unaccounted.

### 9.7 What this does not establish

The task set saturates: 31 of 38 tasks never failed in any arm, so roughly
four fifths of the compute carried no information. A ceiling cannot prove
absence of an effect; it can only fail to find one. The tasks that did
discriminate were all architecture-tier (multi-step state, idempotency,
transactional rollback), which suggests where a sharper task set should aim.
The version-drift tasks, built specifically to target facts a model's training
data gets wrong, were authored too late to be included here and remain the
strongest untested route to a real effect.

## 10. Cost and effort accounting

What a study of this shape actually costs, recorded so the next estimate is
grounded:

- **Runs.** 760 benchmark runs per subject model (38 tasks × 4 arms × 5
  trials), plus 2 paid canary probes per preflight, plus 3 judge calls per
  graded artifact and 2 per pairwise comparison, plus reference re-validation
  through the live pipeline each session. Two 60-run pilots (Sonnet, Opus;
  base vs plugin, 1 trial over the 30-task pilot set) were spent and then
  *discarded* as invalid (§7.3, §7.5); the price of the defects found by
  them.
- **Wall-clock and serialisation.** Per-run timeouts are 900 s (no-feedback)
  and 1500 s (feedback). No-feedback runs parallelise (3 workers); every
  feedback run and every Python/Blink grade serialises through the single
  Nuke session, so the live portion of a sweep is a queue, not a pool. A
  full 4-arm sweep is measured in days of wall-clock on one machine, not
  hours.
- **Billing model.** Runs authenticate with the operator's Claude
  subscription OAuth, not API credits: the recorded `total_cost_usd` is
  notional, and real throughput is governed by subscription usage windows,
  the harness exists to survive them (rate-limit backoff, resume,
  `--retry-errored` after a burned window). A sweep therefore also costs the
  operator's interactive Claude capacity while it runs.
- **Human/agent effort.** The harness is ~2,500 lines of Python across ten
  modules with ~170 tests, built and repaired over an intensive two-day
  session: a 14-task implementation plan, ~40 commits, per-task code review
  with recorded fix rounds, plus the six defect campaigns of §7, several of
  which (contamination guard, Blink symbol extraction and its adversarial
  false-positive hunt, the pairwise judge) were substantial sub-projects with
  their own reports. A fair summary of the ratio: the measurement apparatus
  consumed considerably more engineering than any plausible reading of the
  result will.
- **The recurring overhead nobody budgets:** every interruption (a wedged
  Nuke from a diagnostic tool, a usage window, a mid-study contamination
  finding) costs a preflight, an adjudication, or an archive-and-rerun.

Whether the plugin is worth *its* cost is the study's question; that the
*answer* cost this much is a result in itself, and it is one of the reasons
this report exists; the next "does context help?" question should be able
to reuse this harness rather than re-pay for it.

## 11. Open questions and future work

The sweep will inform these, but they are already visible:

- **Does grounding still pay when base models no longer fabricate?** Pass
  1.5's effect was driven by invented APIs on an earlier Sonnet-class model.
  If current models' fabrication rates on this task set are already low, the
  plugin's correctness value shrinks toward zero and its residual value, if
  any, lives in structure, conventions, and performance discipline. The
  study is designed to say so plainly, including "this may not be worth the
  cost."
- **Is version drift the remaining edge?** Symbol tables pinned per Nuke
  version are exactly the knowledge web search and training data get subtly
  wrong. The six in-authoring version-drift tasks target this; they are the
  natural Phase 2.
- **How to measure design quality when both rubric and pass rates saturate.**
  The absolute judge saturated before the study even started; if pass rates
  saturate too, the informative instruments left are the pairwise judge, the
  arch tier's pinned contracts, pass^3 reliability, and compute parity
  (does the plugin arm reach the same answer with fewer turns/tokens?).
  Building more discriminating *deterministic* structure probes, the
  arch-tier idea pushed further, seems more promising than better rubrics.
- **Has the plugin's value shifted from correctness to structure and
  performance?** Nothing in this harness measures runtime performance of the
  produced tools (scanline discipline, bbox hygiene, GPU cost) even though a
  full skill exists for it. A performance-graded tier would need golden-image
  and timing oracles, a different apparatus.
- **Feedback interaction.** If live-Nuke feedback lifts the base arm to the
  plugin arm's level, the plugin's value proposition becomes "cheaper than
  iterating against a licensed Nuke," which is an economic claim, not a
  capability claim, worth stating as such.
- **Reuse.** The harness is deliberately model-agnostic (`--model`, per-model
  results directories, uniformity asserted). Repeating the identical frozen
  study across model generations would measure how fast the grounding
  advantage decays, probably the most interesting long-run curve this
  apparatus can produce.

## 12. References

### 12.1 Methodology

- SWE-bench / SWE-bench Verified, grading by FAIL_TO_PASS/PASS_TO_PASS
  tests; human task validation. <https://openai.com/index/introducing-swe-bench-verified/>,
  <https://www.swebench.com/original.html>
- Terminal-Bench, container tasks + verifier scripts; 5 trials per
  task/agent as standard protocol; minimal-scaffold principle.
  <https://github.com/harbor-framework/terminal-bench>
- Aider polyglot benchmark, two-attempt self-correction protocol;
  edit-format compliance as a separate reported dimension.
  <https://aider.chat/docs/benchmarks.html>
- Chen et al. 2021, "Evaluating Large Language Models Trained on Code"
  (HumanEval); the unbiased pass@k estimator used here.
  <https://arxiv.org/abs/2107.03374>
- Anthropic, "Demystifying evals for AI agents", 20-50 tasks from real
  failures; reference solutions to prove solvability; outcome-state grading;
  pass@k vs pass^k; negative controls; read the transcripts.
  <https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents>
- τ-bench (Sierra), origin of pass^k as the reliability metric.
  <https://sierra.ai/blog/tau-bench-shaping-development-evaluation-agents>
- Evan Miller, "Adding Error Bars to Evals" (arXiv:2411.00640), paired
  differences, clustered SEs, trial resampling, power analysis; the source
  of this study's minimum-detectable-effect framing.
  <https://arxiv.org/abs/2411.00640>
- McNemar's test for paired binary outcomes (exact/binomial variant for few
  discordant pairs).
  <https://jameshoward.us/2024/12/17/mcnemars-test-the-hidden-gem-for-paired-binary-data>
- "Evaluation of LLMs Should Not Ignore Non-Determinism" (NAACL 2025),
  single runs do not represent capability.
  <https://aclanthology.org/2025.naacl-long.211.pdf>
- LLM-judge bias literature, position bias (order-swapping as control),
  verbosity bias, self-preference, blinding, human calibration:
  <https://arxiv.org/pdf/2410.21819>, <https://arxiv.org/pdf/2506.20274>,
  <https://mbrenndoerfer.com/writing/position-bias-in-llm-judges>,
  <https://wandb.ai/site/articles/exploring-llm-as-a-judge/>,
  <https://langfuse.com/docs/evaluation/evaluation-methods/llm-as-a-judge>
- Execution-feedback confound precedent in coding-agent ablations
  (arXiv:2603.22184), why both feedback arms receive identical feedback
  affordances. <https://arxiv.org/abs/2603.22184>
- Full literature review with the complete source list:
  `docs/research/2026-07-25-eval-methodology.md`.
- Prior internal ablation under re-test (2026-07-23), held locally.

### 12.2 Sources used to build the plugin

The plugin's own curation tracker (`nuke-agent-context/SOURCES.md`) governs
what external knowledge was distilled into the field guides and examples,
under the rule: extract only facts the model would otherwise get wrong,
paraphrase, attribute per claim, never republish prose. Reproduced here as
the plugin's bibliography:

**NDK (C++)**

- Erwan Leroy, NDK series: [Part 1: Intro & Compiling](https://erwanleroy.com/intro-to-writing-nuke-c-plugins-a-k-a-the-ndk-part-1-intro-compiling/)
  (per-platform build gotchas); [Part 2: Architecture, validate, knobs](https://erwanleroy.com/writing-nuke-c-plugins-a-k-a-the-ndk-part-2-architecture-the-validate-and-knobs-functions-and-first-simple-plugins/)
  (the Op lifecycle as practitioners explain it); [Part 3: engine and pixel_engine](https://erwanleroy.com/writing-nuke-c-plugins-a-k-a-the-ndk-part-3-engine-and-pixel-engine-functions/)
  (scanline engine model, threading pitfalls); [Part 4: Custom knobs, GL handles, table knob](https://erwanleroy.com/writing-nuke-c-plugins-ndk-part-4-custom-knobs-gl-handles-and-the-table-knob/)
  (barely-documented knob/GL surface); [Part 5: the PointGradient node](https://erwanleroy.com/writing-nuke-c-plugins-ndk-part-5-the-pointgradient-node/)
  (complete real-tool walkthrough).
- [Max van Leeuwen, Nuke NDK guide](https://maxvanleeuwen.com/project/nuke-ndk/): Windows/Linux build friction.
- [DeepC (charlesangus)](https://github.com/charlesangus/DeepC): real DeepOp source; informed the deep examples.
- [Vivek Reddy, Getting started with the NDK](https://www.vivekc.com/getting-started-with-the-nuke-ndk/); [Sujay Reddy, Beginner's guide](https://medium.com/@sujay_reddy/beginners-guide-to-building-a-nuke-c-plugin-2d787811c364): earlier-era setup guides, retained for VS specifics.

**BlinkScript**

- [Mads Hagbarth, blog/RnD](https://hagbarth.net/blog/) and [Point Rendering Engine introduction](https://hagbarth.net/nuke-point-rendering-engine-introduction/): the deepest public Blink performance knowledge.
- [Erwan Leroy, 3D lightning in BlinkScript](https://erwanleroy.com/making-3d-lightning-in-nuke-using-blinkscript/): advanced worked example.
- [Guillermo Algora, BlinkScript guide](https://guillermoalgora.com/blinkscript-guide.html): structured introduction.
- [Adrian Pueyo, gizmos](https://adrianpueyo.com/gizmos/): open-source kernels (apDespill, apMatte) used as example models.
- [lcrs/blinks](https://github.com/lcrs/blinks): assorted real-world kernels.

**Python**

- [Ben McEwan, Nuke Python posts](https://benmcewan.com/blog/category/nuke/python/): practical workflow scripts.
- [Nukepedia, code tutorials](https://www.nukepedia.com/knowledge/code-tutorials/gizmos/) and [Getting started with Nuke plugins](https://www.nukepedia.com/knowledge/general-tutorials/getting-started-with-nuke-plugins/): community knowledge base; plugin-loading facts. (The Nukepedia tool manifest is also where benchmark task *ideas* were sourced, independently of the plugin's distillation.)
- [Alexander Richter, Enhance Nuke workflows with Python](https://www.alexanderrichtertd.com/post/mastering-python-to-enhance-nuke-workflows).
- [Adrian Pueyo, KnobScripter](https://adrianpueyo.com/knobscripter/): exemplar of a polished Nuke Python tool.

**UI / PySide panels**

- [Foundry, Custom Panels (Python dev guide)](https://learn.foundry.com/nuke/developers/140/pythonreference/custom_panels.html): canonical `registerWidgetAsPanel` reference.
- [jedypod, dockable PySide GUI gist](https://gist.github.com/jedypod/0c2b89cf047b0bc5cebed109d707fa69): minimal dock-into-Nuke pattern.
- [stefanmuller/nuke_PySide_helper](https://github.com/stefanmuller/nuke_PySide_helper): widgets persisting values onto nodes.
- [shiningdesign/universal_tool_template.py](https://github.com/shiningdesign/universal_tool_template.py): PySide2/6 compatibility handling.
- [adrianpueyo/KnobScripter (source)](https://github.com/adrianpueyo/KnobScripter): heavy-UI exemplar incl. Nuke 16 / PySide6 migration history.
- [W_hotbox (melMass mirror)](https://github.com/melMass/W_hotbox) + [Nuke 16 update](https://github.com/georgeantonopoulos/W_Hotbox_Nuke16): Wouter Gilsing's UI tool; PySide2→6 migration diff.

**Complex-tool structure exemplars** (mined for patterns, not distilled
wholesale): [adrianpueyo/Stamps](https://github.com/adrianpueyo),
[ynput/ayon-nuke](https://github.com/ynput/ayon-nuke),
[PrismPipeline/Prism](https://github.com/PrismPipeline/Prism),
[masqu3rad3/tik_manager4](https://github.com/masqu3rad3/tik_manager4),
[aws-deadline/deadline-cloud-for-nuke](https://github.com/aws-deadline/deadline-cloud-for-nuke),
[mellowpictures/render_submission](https://github.com/mellowpictures/render_submission),
[gillesvink/NukeDeadlineSubmission](https://github.com/gillesvink/NukeDeadlineSubmission),
[pyblish/pyblish-nuke](https://github.com/pyblish/pyblish-nuke),
[openNuke/toolset](https://github.com/openNuke/toolset).

**Gizmo structure**

- [Keheka, Advanced Gizmo Building in Nuke](https://www.keheka.com/advanced-gizmo-building-in-nuke/): dynamic menus via Python, expression-driven controls.

**Official Foundry documentation** (primary sources for the refs tables and
the Blink grading symbol table; the harness's Blink table was extracted from
the docs shipped *inside the Nuke bundle*, never from the plugin's refs):

- [NDK dev guide, Op basics](https://learn.foundry.com/nuke/developers/17.0/ndkdevguide/advanced/opscommon.html)
- [NDK dev guide, Memory management](https://learn.foundry.com/nuke/developers/17.0/ndkdevguide/advanced/memorymanagement.html)
- [NDK dev guide, Deep ops](https://learn.foundry.com/nuke/developers/17.0/ndkdevguide/deep/deep.html)
- Blink Kernel API Reference, Blink User Guide, Python developer guide and
  reference (versioned learn.foundry.com URLs, per `refs/VERSIONS.md`).

Dropped from curation (video-only or paid, nothing distillable): fxphd,
Rebelway, Pluralsight, CGCircuit, ActionVFX, gatimedia, and the EOL'd
Ben McEwan "Python for Nuke 101" course.
