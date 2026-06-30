---
name: redline
description: "Finds the maximum safe --max-model-len and --max-num-batched-tokens for a vLLM-served model on a fixed GPU, by climbing a doubling ladder one clean reboot at a time and recording the real KV-cache/concurrency signal from each boot rather than guessing. Use when asked to find max context, tune context length, or push a model to its context ceiling."
metadata:
  author: TGPSKI
  version: "1.0"
license: GPL-3.0
compatibility: "vLLM (V1 engine), bash, jq, a single-GPU local serve target. Assumes the model is already fetched (e.g. via vanity) and a known-good low-context boot command exists."
---

# Redline -- Router

This is a directed workflow for one specific, recurring tuning problem: a model boots fine
at a conservative `--max-model-len` / `--max-num-batched-tokens`, and you want to know the
real ceiling on this GPU without re-living a three-attempt OOM bisection every time.

The workflow climbs a doubling ladder, one parameter at a time, rebooting clean between
every rung and reading the actual signal vLLM reports (`Available KV cache memory`,
`GPU KV cache size`, `Maximum concurrency`) instead of trusting the model card's quoted
context ceiling or guessing the next number.

State lives in `ladder.yml` in the tuning repo -- not in this session's chat history.
Every rung this workflow runs gets appended there, win or lose. The ladder file *is* the
progress record: come back next week, point this workflow at the same model, and it picks
up from the last rung instead of re-discovering the OOM boundary from scratch.

## Prerequisites

- The model is already fetched and the path is known (vanity `registry.json` is the
  natural source if this repo is paired with a vanity library).
- You have a **known-good boot command** at some conservative `(max_model_len,
  max_num_batched_tokens)` pair -- this workflow does not discover that starting point,
  it only climbs from it. If you don't have one yet, get a clean boot manually first
  (start low: 32768 / 8192 is a reasonable floor on a single 24-32GB card).
- `jq` is available for parsing vLLM's structured log lines if you're scripting log capture.
- This is a **local, single-session tool**. It is not multi-phase in the router-pattern
  sense of spanning days/PRs -- but it behaves like one in that state persists across
  invocations via `ladder.yml`, so it follows the router shape rather than the
  single-file shape.
- `ladder.yml` schema at `assets/rung-result.schema.yml`, and a completed example at `assets/example-ladder.yml`

## Entry Point

Ask the user exactly one thing to start, if not already evident from the conversation:

> "Which model and which GPU profile are we climbing? (model key + ladder file path,
> default `./ladder.yml` in the current repo)"

Everything else -- the known-good floor, the GPU's total capacity, the model's quoted
context ceiling -- should be **inspected**, not asked, per the table below.

## Progress Detection

Detect state by reading `ladder.yml` (or the path the user gave), not by re-deriving it
from chat history or assuming a fresh start.

| Status | Action |
|--------|--------|
| `ladder.yml` does not exist for this model | This is rung 1. Confirm the known-good floor with the user, then go to `references/01-run-rung.md`. |
| `ladder.yml` exists, last rung `status: boot_ok` AND `smoke_test.attempted: false` | The boot ceiling may not be the *real* ceiling yet -- prefill under load can fail differently than boot. Go to `references/03-smoke-test.md` before climbing further or declaring victory. |
| `ladder.yml` exists, last rung `status: boot_ok` AND `smoke_test.attempted: true` | Climb. Go to `references/01-run-rung.md` to compute and run the next rung. |
| `ladder.yml` exists, last rung `status` starts with `oom_*` or `crash_*` | The ladder found its ceiling (or hit a non-context failure) at the last rung. Go to `references/02-record-ceiling.md` to confirm and close out, **not** to another climb -- do not retry the same failed rung blindly. |
| User explicitly asks to re-verify a prior rung | Go to `references/01-run-rung.md` in **replay mode** -- rerun the exact params from a named rung index, append as a new rung (never overwrite history). |

## Design Principles

1. **Ownership** -- The user runs the actual `vllm serve` command and pastes back the
   boot log (or this workflow's surrounding agent has shell access and runs it directly,
   if so equipped). Either way, this workflow's job is to compute the next rung's
   parameters, structure the result, and decide whether to climb again -- never to
   fabricate a boot outcome it didn't observe.
2. **Source of truth** -- `ladder.yml`, conforming to `assets/rung-result.schema.yml`, is
   the only place ceiling decisions get made from. The model card's quoted max context is
   a *target to test toward*, never a number to trust directly (see prior sessions on this
   exact model: the safetensors metadata and the card prose disagreed on parameter count,
   so card-quoted numbers get verified here too, not assumed).
3. **Ask less, infer more** -- The next rung's parameters are computed from the last
   rung's outcome (see the doubling/backoff table in `references/01-run-rung.md`), not
   asked of the user each time.
4. **One variable at a time** -- `max-model-len` and `max-num-batched-tokens` are climbed
   independently. Raising both at once on a failure makes it impossible to tell which one
   broke. This is the single biggest lesson encoded from the rungs that produced this
   workflow: three consecutive OOMs were eventually traced to three different causes
   (multimodal profiling, an env var typo, and batch size) -- conflating the variables
   would have hidden all three.

## Route to Phase

| Phase file | When |
|---|---|
| `references/01-run-rung.md` | Compute and execute the next rung; append result to `ladder.yml` |
| `references/02-record-ceiling.md` | Last rung failed -- confirm the ceiling, write the summary, stop climbing |
| `references/03-smoke-test.md` | Last rung booted clean but hasn't been load-tested at near-ceiling length |

## What This Workflow Does Not Do

- It does not tune `--gpu-memory-utilization` as a primary lever. That knob is held fixed
  through the climb and only adjusted as a last resort once `max-model-len` and
  `max-num-batched-tokens` are both maxed -- see `references/01-run-rung.md` step 4.
- It does not run the actual model-quality eval (e.g. a 15-case leather suite). Finding
  the ceiling and confirming the model is *good* at that ceiling are different jobs.
- It does not modify `vanity`'s `registry.json` or any served-model config. It only
  appends to `ladder.yml`. If a ceiling is found worth keeping as the default boot
  command, that's a manual follow-up, not something this workflow does automatically.
