# Redline

Redline is a [directed-workflow agent skill](https://github.com/TGPSKI/directed-workflows) that finds the safe context ceiling for a
vLLM-served model on a fixed GPU. It climbs `--max-model-len` and
`--max-num-batched-tokens` one variable at a time, clean-rebooting between rungs and
recording the real KV-cache and concurrency signal from vLLM boot logs.

The workflow is the program. The agent is the runtime.

## Quick Start

Clone this repo alongside the model-tuning workspace or make it available to your agent:

```bash
git clone https://github.com/TGPSKI/redline.git
```

Invoke the workflow by pointing your agent at the skill:

```text
@redline/SKILL.md

Which model and GPU profile are we climbing?
The ladder file should be ./ladder.yml unless you find an existing one.
```

Redline expects a known-good low-context boot command. It does not discover the starting
floor; it climbs from one.

## How It Works

The workflow persists state in a ladder file: `ladder.yml` following `assets/rung-result.schema.yml`.

Each rung records:

- the model identity and exact `vllm serve` parameters
- the GPU and allocator environment
- whether boot succeeded, OOMed, or failed for another reason
- vLLM's reported KV-cache capacity and maximum concurrency
- smoke-test status for near-ceiling prefill behavior

The router in `SKILL.md` reads the last rung and
dispatches to the next phase:

| Phase | Purpose |
|---|---|
| `references/01-run-rung.md` | Compute and run the next single-axis tuning attempt |
| `references/02-record-ceiling.md` | Classify a failed rung and write the ceiling summary |
| `references/03-smoke-test.md` | Verify a successful boot under near-ceiling request load |

## Repository Layout

```text
redline/
├── SKILL.md                         # Top-level skill router
├── references/                      # Top-level phase files
│   ├── 01-run-rung.md
│   ├── 02-record-ceiling.md
│   └── 03-smoke-test.md
├── assets/                          # Top-level schema and example assets
│   ├── rung-result.schema.yml
│   └── example-ladder.yml
├── AGENTS.md                        # Contributor and agent guidance
├── README.md                        # You are here
└── LICENSE
```

## Design Rules

- Do not trust model-card context claims as final. Treat them as targets to test toward.
- Raise `max_model_len` and `max_num_batched_tokens` independently.
- Classify OOM site before calling a failure a real ceiling.
- Keep `ladder.yml` append-only for boot outcomes.
- Treat `Maximum concurrency` as the useful serving signal, not just raw context length.

## License

[GNU General Public License v3.0](./LICENSE)

## Author

Tyler Pate ([@TGPSKI](https://github.com/TGPSKI)), 2026
