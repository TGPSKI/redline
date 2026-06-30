---
name: 03-smoke-test
parent: redline
---

# Phase: Smoke Test

A rung reported `boot_ok`, but a clean boot only confirms the KV cache *budget* exists --
not that prefill at near-ceiling length actually completes, that chunked-prefill correctly
slices a long prompt into the configured batch size, or that the reasoning-channel split
still works correctly under a different context length than it was tested at.

## Inspect

| Status | Action |
|---|---|
| `smoke_test.attempted: false` on the rung under test | Run this phase |
| `smoke_test.attempted: true`, `prefill_completed: true`, `reasoning_channel_split_verified: true` | Already smoke-tested. Nothing to do -- return to the router and climb |
| `smoke_test.attempted: true`, but either flag is `false` | The boot was confirmed but the smoke test caught a real problem at this rung. Treat this the same as a failed rung -- do not climb past it. Route to `references/02-record-ceiling.md` with this rung as the boundary, noting the smoke-test failure (not a boot OOM) as the cause |

## Decide

Ask the user to choose a near-ceiling test payload, since synthetic length alone doesn't
exercise the same code paths as real content:

> "For the long-context smoke test, do you want to (a) concatenate a real document you
> have on hand to approximately {max_model_len - 2000} tokens, or (b) use a synthetic
> repeated-text filler? Real content is the better test -- filler can compress unrealistically
> well under prefix caching and hide a real problem."

Confirm the reasoning-channel check method: if served through a client that renders
`reasoning_content` separately (e.g. OpenWebUI's collapsible dropdown), visual confirmation
is sufficient. If served via raw API, the user should `curl` the endpoint and check the
response JSON has `reasoning_content` and `content` as distinct fields rather than one
combined blob -- do not assume the split works just because it worked at a different
context length on a prior rung.

## Generate

Produce the test request and the verification checklist. Write the JSON body to a file so
real long-context payloads with quotes or newlines do not break shell quoting:

```bash
payload_file=/tmp/redline-smoke-payload.txt
# Put the selected near-ceiling prompt in $payload_file before sending the request.

python3 - <<'PY' > /tmp/redline-smoke-request.json
import json
from pathlib import Path

payload = Path("/tmp/redline-smoke-payload.txt").read_text()
print(json.dumps({
    "model": "{served_model_name}",
    "messages": [{"role": "user", "content": payload}],
}))
PY

curl -s localhost:{port}/v1/chat/completions \
  -H "Content-Type: application/json" \
  --data-binary @/tmp/redline-smoke-request.json \
  | jq '.choices[0].message | {has_reasoning_content: has("reasoning_content"), content_preview: .content[0:200]}'
```

While the request is in flight, watch the server log for:

1. The `loggers.py` throughput line -- confirm `Avg prompt throughput` is nonzero and the
   request doesn't stall (a stall under chunked prefill at this batch size is a real signal
   distinct from an OOM, and won't show up from a clean boot alone).
2. Any `jit_monitor.py` "Triton kernel JIT compilation during inference" warnings --
   expected on the first request at a new shape, not expected on repeat requests at the
   same shape. If they repeat every request, warmup isn't covering this shape/config and
   it's worth noting even if not a hard failure.
3. Confirm no OOM during the actual generation, not just during boot -- decode-time KV
   cache growth at a long prompt can OOM even when the boot-time profiling pass didn't.

Fill in the `smoke_test` block on the rung record in `ladder.yml`:

```yaml
smoke_test:
  attempted: true
  long_context_tokens: {actual token count sent}
  prefill_completed: true|false
  reasoning_channel_split_verified: true|false
  notes: "{anything observed -- stalls, JIT warnings, repeated reasoning leakage, etc.}"
```

Write to: `{ladder_file_path}`, updating the existing rung entry's `smoke_test` block
in place (this is the one field allowed to be filled in after the fact on an existing
rung, since `boot_ok` + untested is a legitimately incomplete state, not a past outcome
being revised).

## Next Phase

Return to the router's Progress Detection table -- the rung is now either a confirmed
climbable result (both smoke-test flags true) or a discovered ceiling
(`references/02-record-ceiling.md`, citing the smoke-test failure as cause).
