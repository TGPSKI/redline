---
name: 02-record-ceiling
parent: redline
---

# Phase: Record Ceiling

A rung failed. Before treating that as the final answer, confirm it's a real ceiling
rather than an artifact of a fixable mistake, then write the summary.

## Inspect

Read the failed rung's full record from `ladder.yml`, specifically `outcome.failure_site`,
`outcome.oom_requested_bytes`, `outcome.oom_free_bytes`, and `env.pytorch_cuda_alloc_conf`.

| `outcome.failure_site` pattern | Classification | Action |
|---|---|---|
| `profile_run -> embed_multimodal` or mentions a vision-model file / vision tower | Not a context-length ceiling at all -- this is a multimodal profiling cost. If `limit_mm_per_prompt` wasn't set to zero out image/video, that's the real fix, not backing off context. Confirm the flag was actually present in this rung's `params.limit_mm_per_prompt`; if it was empty/unset, this rung is invalid -- note it and don't count it as a ceiling | Mark this rung's `notes` as "invalid -- multimodal not disabled" and rerun with the flag set, as a new rung, before concluding anything about the real ceiling |
| `env.pytorch_cuda_alloc_conf` is not exactly `expandable_segments:True` (e.g. empty, or a value with a typo'd var name) | The allocator flag never actually applied. This rung's OOM may be an artifact, not a real ceiling | Mark this rung's `notes` as "invalid -- alloc_conf not applied", fix the env var, rerun the same params as a new rung before concluding the ceiling |
| `marlin_gemm` or other GEMM kernel call during compile/profiling, `oom_requested_bytes` scales with `max_num_batched_tokens` | Real batch-size ceiling | This is a genuine signal. Continue below |
| OOM during `_initialize_kv_caches` with no kernel-specific frame, `oom_free_bytes` very small relative to total | Real context-length ceiling -- KV cache budget exhausted | This is a genuine signal. Continue below |
| Anything not matching the above (import errors, config errors, missing files) | Not a ceiling signal at all | Stop the ladder. This is a bug, not a tuning result. Fix it outside this workflow |

If the rung is invalid per the table, route back to `references/01-run-rung.md` to redo it
correctly as a new rung -- do not write a ceiling summary from an invalid rung.

## Decide

If the failure is a confirmed real ceiling (last two rows above), present the user with
the boundary and ask whether to stop here or push the margin lever:

> "Ceiling found: `{param}` boots clean at `{last_good_value}`, fails at `{failed_value}`
> ({failure_classification}). The last clean boot reported {max_concurrency}x concurrency
> headroom. Stop here, or try nudging `--gpu-memory-utilization` up from
> `{gpu_memory_utilization}` to see if margin (not the dimension itself) was the limiter?"

Do not automatically retry with higher utilization -- this is explicitly a user decision
per the router's "What This Workflow Does Not Do" section, since pushing utilization has
its own risk (less headroom for the OS/other GPU processes already observed sharing this
card).

## Generate

Write the human-readable ceiling summary to `ladder-summary.md`, next to
`{ladder_file_path}`. Keep `{ladder_file_path}` machine-readable: append or update only
rung records there, never markdown prose. The summary should cover:

- The confirmed safe `(max_model_len, max_num_batched_tokens, gpu_memory_utilization)`
  triple -- the last `boot_ok` AND smoke-tested rung, not just the last `boot_ok` rung.
- The first failing rung for each axis, with its classification from the Inspect table.
- The `max_concurrency` reported at the confirmed safe rung -- this is the number that
  actually matters for serving capacity, not the raw token count.
- Any invalidated rungs and why, so a future read of `ladder.yml` doesn't mistake an
  artifact OOM for a real boundary.

Example markdown summary block:

```markdown
## Ceiling: {model_key} on {gpu}

Confirmed safe boot: max-model-len={X}, max-num-batched-tokens={Y}, gpu-mem-util={Z}
Reported concurrency at this rung: {W}x for {X} tokens/request
KV cache budget: {kv_cache_tokens} tokens

max-model-len ceiling: fails above {X_fail} ({classification})
max-num-batched-tokens ceiling: fails above {Y_fail} ({classification})

Invalidated rungs: {list, with reasons} -- excluded from the above.
```

Write to: `ladder-summary.md`, appending the summary block above. Only append to
`{ladder_file_path}` when recording a new rung record.

## PR Checkpoint

If this tuning repo is version-controlled and PRs are how results get shared with future
sessions or teammates:

**Title**: `Context ceiling: {model_key} on {gpu} -- max-model-len={X}, batched-tokens={Y}`

**Files to include**:
- `ladder.yml` (full rung history, including invalidated rungs)
- `ladder-summary.md` (the human-readable ceiling)
