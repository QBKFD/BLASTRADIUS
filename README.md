# BLASTRADIUS

A benchmark for root cause analysis (RCA) on a small AI microservice system. Known
faults are injected, ground truth is logged by the injector, and RCA methods (rule
baseline, statistical, LLM) are scored on the same recordings of traces and metrics.

**Status:** stage 0 (plan). Nothing is built or measured yet. No results.

- [PLAN.md](PLAN.md): stages, deliverables, done tests, planned metrics
- [FAULTS.md](FAULTS.md): fault catalog, ground truth format, expected symptoms
- [DECISIONS.md](DECISIONS.md): design decisions, made before or after results

## Development

Requires [uv](https://docs.astral.sh/uv/) and
[gitleaks](https://github.com/gitleaks/gitleaks).

```sh
uv sync
uv run ruff check .
uv run ruff format --check .
uv run pytest
git config core.hooksPath .githooks   # once per clone: gitleaks pre-commit hook
```

Secrets go in `.env` (git-ignored); see `.env.example`.

## Licence

MIT, see [LICENSE](LICENSE).
