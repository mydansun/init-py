# init-py

A Claude Code skill that helps an agent scaffold an interactive `init.py` bring-up tool for a Docker Compose project — the kind that prompts the operator for every env var, writes `.env` (preserving comments via `python-dotenv set_key`), runs timeout-bounded reachability probes, and supports `check` / `reconfigure KEY` subcommands.

The single-file `uv run --script` shape means the operator only needs `uv` installed on the host — no project venv, no `requirements.txt`, no pre-step.

## What's in here

- [`SKILL.md`](SKILL.md) — the actual skill: structure, the `collect_field` skeleton, and the pitfalls that are easy to hit (manual `.env` writers, missing `quote_mode="auto"`, probes without timeouts, mTLS cert pair checks via `SubjectPublicKeyInfo`, etc.).

## Installation

```
npx skills add mydansun/init-py
```

Then ask Claude something like *"create an init.py for this compose project"* and it will load this skill.

## Where this comes from

Distilled from a ~840-LOC production `init.py` that bootstraps a Temporal-worker compose stack. The skill captures the patterns and anti-patterns; the reference file itself is internal.
