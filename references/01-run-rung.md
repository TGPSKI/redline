---
name: 01-run-rung
parent: redline
---

# Phase: Run Rung

Compute the next rung's parameters from the ladder's history, run it (or hand the exact
command to the user to run), and append a structured result to `ladder.yml`.

## Inspect

Read `ladder.yml`. If it doesn't exist, this is rung 1 and there is no history to inspect --
skip to **Decide** and confirm the floor.

| Status | Action |
|--------|--------|
| Last rung `status: boot_ok`, no prior failed rung at a higher value for the same param | Climb that param: double it (see Decide table) |
| Last rung `status: boot_ok`, a *later* rung already failed at a higher value for this param | Don't re-climb past a known failure. Move to the other param, or to `references/02-record-ceiling.md` if both are maxed |
| Last rung `status: oom_*` | Stop climbing the param that was raised on this rung. The previous `boot_ok` rung is the confirmed ceiling for that param. Go to `references/02-record-ceiling.md` |
| Last rung `status: crash_other` | Not an OOM -- don't treat this as a ceiling signal. Diagnose the actual error first (it may be unrelated to context size entirely, e.g. the `TPYTORCH_CUDA_ALLOC_CONF` typo class of bug). Fix and rerun the *same* rung index's params before climbing further |
| `ladder.yml` exists but `smoke_test.attempted: false` on the last `boot_ok` rung | Don't climb yet. A boot success without a load test isn't a confirmed rung -- recommend `references/03-smoke-test.md` first, but allow the user to override and climb anyway if they explicitly say so |

## Decide

If this is rung 1, ask:

> "What's the known-good floor? (max-model-len, max-num-batched-tokens, gpu-memory-utilization)
> If you don't have one, I'd suggest starting at max-model-len=32768,
> max-num-batched-tokens=8192, gpu-memory-utilization=0.85 as a conservative floor."

Otherwise, compute the next rung automatically -- do not ask. The climb order is:

1. **Climb `max-model-len` first**, batched-tokens held at its last confirmed value.
   Double it: 32768 → 65536 → 131072 → 262144 (or the model card's stated ceiling,
   whichever is lower -- but verify the card's number before treating it as a hard stop;
   card-quoted ceilings and actual safetensors metadata have disagreed before on this
   exact family of model).
2. Once `max-model-len` OOMs or hits the card ceiling, **back off to the last `boot_ok`
   value** and **switch to climbing `max-num-batched-tokens`**, starting from its last
   confirmed value: 8192 → 16384 → 32768 → 65536, capped at the confirmed `max-model-len`.
3. Once both are maxed independently, optionally **nudge `gpu-memory-utilization`** up in
   +0.02 steps (0.85 → 0.87 → 0.90) and re-attempt the highest failed rung from steps 1-2
   to see if margin was the actual blocker rather than the dimension itself. Only do this
   if the user asks to push further after both axes are independently maxed -- it's not
   automatic.

Never raise `max-model-len` and `max-num-batched-tokens` together in the same rung. If the
last failure's `outcome.failure_site` doesn't clearly indicate which axis caused it,
ask the user rather than guess:

> "The last failure was [status] at [failure_site]. I can't tell from this alone whether
> max-model-len or max-num-batched-tokens caused it. Which one do you want to back off?"

## Generate

Produce the exact `vllm serve` command for this rung, using the last known-good command as
the base and changing only the one parameter under test. Echo it back to the user (or run
it directly if this agent has shell access to the serving host) along with what to watch for:

```bash
PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True \
vllm serve {model_path} \
  --served-model-name={served_model_name},{model_path} \
  --quantization {quantization} \
  --kv-cache-dtype {kv_cache_dtype} \
  --attention-backend {attention_backend} \
  --gpu-memory-utilization {gpu_memory_utilization} \
  --max-model-len {max_model_len} \
  --max-num-seqs {max_num_seqs} \
  --max-num-batched-tokens {max_num_batched_tokens} \
  {limit_mm_per_prompt_flag} \
  --enable-chunked-prefill \
  --enable-prefix-caching \
  --reasoning-parser {reasoning_parser} \
  --tool-call-parser {tool_call_parser} \
  --enable-auto-tool-choice \
  --host=0.0.0.0 \
  --port={port}
```

Tell the user what to watch for in the boot log, in priority order:

1. Does it boot at all, or OOM? If OOM, **where** -- `profile_run` (multimodal/dummy-batch
   sizing), `torch.compile`/Dynamo tracing, or the final `_initialize_kv_caches` /
   `marlin_gemm` runtime call. The failure site determines `outcome.status` below and which
   axis to back off.
2. If it boots: capture these four lines verbatim, they populate `signal` in the rung record:
   - `Model loading took X GiB`
   - `Available KV cache memory: X GiB`
   - `GPU KV cache size: N tokens`
   - `Maximum concurrency for N tokens per request: Wx`
3. **`Wx` is the real headroom signal, not `max-model-len` itself.** If `W` is close to
   1.0x, you're at the edge even though the boot succeeded -- flag this to the user as
   "boots, but no concurrency margin" rather than letting them read it as a clean pass.

After the user reports the outcome, fill in a rung entry conforming to
`assets/rung-result.schema.yml` and append it to `ladder.yml` (create the file with an
empty list if it doesn't exist). **Append-only** -- never edit a prior rung's recorded
outcome, even if it turns out to have been caused by something fixable (like an env var
typo). If a rung needs to be redone because the prior attempt was invalidated by a
config mistake (not a real ceiling), record the mistake in that rung's `notes` field and
add a fresh rung for the corrected retry, rather than overwriting.

Write to: `{ladder_file_path}` (default `./ladder.yml` in the current repo)

## After Generate

| Outcome | Next |
|---|---|
| `boot_ok`, smoke test not yet run | Recommend `references/03-smoke-test.md` before climbing again |
| `boot_ok`, smoke test already passed at this rung | Loop back to **Inspect** in this same file -- compute and run the next rung |
| `oom_*` or `crash_other` | Go to `references/02-record-ceiling.md` |
