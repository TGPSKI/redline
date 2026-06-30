# Redline - Agent Instructions

## Project Purpose

Redline is a [directed-workflow agent skill](https://github.com/TGPSKI/directed-workflows) for finding the maximum safe vLLM context
configuration for one model on one fixed GPU. It records a ladder of real boot attempts
instead of guessing from model-card context claims or repeating ad hoc OOM experiments.

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
├── AGENTS.md                        # Agent instructions
├── README.md                        
└── LICENSE
```

## Working Principles

- Treat `ladder.yml` as the state machine. Do not infer progress from chat history.
- Append new boot attempts. Do not rewrite prior rung outcomes.
- Raise only one variable per rung: `max_model_len` or `max_num_batched_tokens`.
- A clean boot is not a confirmed ceiling result until the near-context smoke test has
  been handled or consciously deferred.
- Do not fabricate vLLM outcomes. The user or serving host must provide the boot log and
  signal lines.
- Keep reference files procedural. Put examples and schemas under `assets/`, not inline
  in every phase.

## Development Workflow

This repo has no compiled code. Validation is a review pass:

```bash
git diff --check
```

## Contributing Summary

Changes should preserve the directed-workflow spec: inspect state, decide from explicit
status tables, generate the next artifact. Avoid adding generic skill metadata or agent
platform files unless the repo explicitly adopts that platform.
