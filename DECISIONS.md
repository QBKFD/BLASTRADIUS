# Decisions

Every design choice, when it was made, and whether it was made before or after seeing
any results. "Results" means any measurement from the system: baseline noise, dev
runs, test runs or RCA scores.

Source:
- **owner**: set in the project brief.
- **proposed**: chosen while writing the plan; needs owner review.

Owner review on 2026-11-25 (before results): every proposed decision D-001 to D-043 is
accepted, except those marked **superseded** or **amended** below; the changes are
D-044 to D-059. Superseded entries stay in the log unchanged apart from that mark.
Later entries marked **proposed** still need review.

## Infrastructure

### D-001 Docker Compose only
- 2026-11-25, before results, owner.
- No Kubernetes. One compose file per system under test.

### D-002 Single host, load generator on the same host
- 2026-11-25, before results, owner.
- netcup VPS Lite 2 G12.5s. The load generator competes with the system for CPU.
  Mitigation: `cpus` caps per container (D-034), CPU steal and baseline variance
  measured and reported (D-028).

### D-003 No Grafana
- 2026-11-25, before results, owner.
- Data is read through the Prometheus and Jaeger APIs. UIs only for manual inspection.

### D-004 All ports on 127.0.0.1
- 2026-11-25, before results, owner.
- UIs over SSH tunnel. Checked with `ss -tlnp` in the stage 1 done test.

### D-005 Python 3.12, uv, FastAPI, pytest, ruff, gitleaks
- 2026-11-25, before results, owner.
- `.python-version` pins 3.12. `requires-python = ">=3.12,<3.13"`.

### D-006 Build backend hatchling, src layout, no runtime dependencies yet
- 2026-11-25, before results, proposed.
- Dependencies are added in the stage that needs them. pytest uses `pythonpath = src`
  so tests run without an editable install.

### D-007 Pre-commit hook as a plain git hook
- 2026-11-25, before results, proposed.
- `.githooks/pre-commit` runs `gitleaks git --pre-commit --staged`. Enabled per clone
  with `git config core.hooksPath .githooks`. It refuses the commit if gitleaks is not
  installed.
- Alternative: the pre-commit framework. Not chosen because it needs an extra tool and
  its gitleaks hook builds gitleaks with Go. Can switch later if ruff should also run
  on commit.

### D-008 Secrets only in `.env`
- 2026-11-25, before results, owner.
- `.env` and `.env.*` are git-ignored, except `.env.example` with empty values.

## System under test (stage 1)

### D-009 Topology is a chain: gateway -> checker -> mock_llm, gateway -> redis
- 2026-11-25, before results, proposed.
- **Superseded by D-044.**
- Read literally from the brief. The checker proxies allowed prompts to mock_llm.
- Alternative: gateway calls checker, then calls mock_llm itself (fan-out). Gives a
  different propagation pattern (LLM errors would not pass through the checker).

### D-010 Gateway fails closed on checker failure, with the same 403 as a policy block
- 2026-11-25, before results, proposed.
- **Superseded by D-045.**
- Connection error, timeout or 500 from the checker leads to 403 "blocked" with the
  same body as a policy block. The block reason is not a metric label. Gateway client
  spans to the checker carry the error. A 502 from the checker (upstream LLM error)
  is passed on as 502.
- This is what makes F5 misleading at the metric level and visible in traces.

### D-011 Gateway uses Redis for per-client rate-limit counters; Redis down gives 503
- 2026-11-25, before results, proposed.
- Redis client timeout 50 ms. No fallback (fail closed). No response cache, to keep
  Redis behaviour simple.

### D-012 Checker retries only 429 from mock_llm
- 2026-11-25, before results, proposed.
- **Superseded by D-046.**
- Up to 2 retries, exponential backoff with seeded jitter, Retry-After honoured up to
  1 s. 503 is not retried. Gives F3 its propagating behaviour and keeps F1 simple.

### D-013 Gateway timeout to checker 5 s
- 2026-11-25, before results, proposed.
- **Superseded by D-047.**
- Above the checker's worst-case retry time plus LLM latency, so F3 and F4 should not
  cause fail-closed blocks. Revisited after the capacity test (stage 1).

### D-014 Mock LLM implements a subset of the Anthropic Messages API
- 2026-11-25, before results, proposed.
- So that PROMPTWALL's Anthropic backend can point at it in stage 5 through a base URL
  setting, without a new backend. Needs a check that PROMPTWALL allows a base URL.

### D-015 Toxiproxy on all four edges, including loadgen -> gateway
- 2026-11-25, before results, proposed.
- **Amended by D-048** (edge list changes with the topology).
- No initial fault uses Toxiproxy. Kept in the path so later network faults need no
  topology change, and so baseline latency already includes the proxy hop.

### D-016 Metrics with prometheus_client, traces with the OTel SDK sent directly to Jaeger
- 2026-11-25, before results, proposed.
- No OTel collector, to save memory. Jaeger all-in-one receives OTLP.

### D-017 100% trace sampling
- 2026-11-25, before results, proposed.
- Load is low enough; RCA needs complete traces. Completeness checked per run.

### D-018 Prometheus scrape interval 5 s
- 2026-11-25, before results, proposed.
- 1 s gives finer data but adds CPU load on a shared host. Confirmed or changed after
  the baseline noise check, before any fault run.

### D-019 Jaeger in-memory storage with a trace cap; traces exported after each run
- 2026-11-25, before results, proposed.
- Trace completeness (at least 99% of loadgen requests) is checked per run.

### D-020 Admin and fault endpoints on separate ports, not traced, not in metrics
- 2026-11-25, before results, proposed.
- Prevents the injection call from appearing in the data given to RCA methods.

### D-021 No Redis exporter
- 2026-11-25, before results, proposed.
- Redis is visible through gateway client spans and `docker stats`. Saves a container.

### D-022 Load generator is open loop
- 2026-11-25, before results, proposed.
- Poisson arrivals at a fixed rate with a fixed seed, so slow responses do not reduce
  offered load (no coordinated omission). Rate set to 30 to 40% of the measured
  saturation rate in stage 1, fixed before any fault run. Fixed prompt set with about
  5% malicious prompts.

## Faults

### D-023 Initial fault catalog: 6 faults, two per difficulty level
- 2026-11-25, before results, owner (fault list); proposed (injection methods).
- See FAULTS.md.

### D-024 Ground truth from the injector only
- 2026-11-25, before results, owner.
- A run is excluded only if the injector's own verification fails. Never on symptoms
  or RCA output. Excluded runs are repeated, counted and reported.

### D-025 Redis down and checker crash use docker stop and docker kill, not Toxiproxy
- 2026-11-25, before results, proposed.
- A real outage of the component, so the ground truth component is the one that
  failed. Toxiproxy would model a network partition instead.

### D-026 CPU pressure through stress-ng inside the checker container
- 2026-11-25, before results, proposed.
- Limited by the checker's `cpus: 1.0` cap, so it does not load the whole host.
  stress-ng is in the checker image only for this purpose. The checker does a fixed
  amount of CPU work per request so the pressure has an effect.

### D-027 No config change signal given to RCA methods
- 2026-11-25, before results, proposed.
- The checker logs reloads but exposes no config version metric, and no deploy log is
  part of the RCA view. Keeps F6 silent. A later condition with a change log is
  possible.

## Experiment protocol

### D-028 CPU steal measured per run; sensitivity analysis on high-steal runs
- 2026-11-25, before results, owner (measure steal); proposed (analysis).
- `vmstat 1` on the host for every run. Threshold for exclusion in the sensitivity
  analysis set from the stage 1 baseline (for example the p95 of 1-min mean steal),
  before any fault run.

### D-029 Run timing
- 2026-11-25, before results, proposed.
- Settle 30 s, baseline 120 s plus seeded 0 to 30 s, fault 120 s, recovery 60 s,
  export after 10 s. About 6 min per run.

### D-030 Seeded random run order, faults and controls interleaved
- 2026-11-25, before results, proposed.
- Spreads time-of-day and neighbour noise evenly over fault types.

### D-031 Run counts: dev 3 per fault plus 3 controls; test 20 per fault plus 20 controls
- 2026-11-25, before results, proposed.
- Test set about 14 h. Wilson CI half-width about 0.17 to 0.2 per fault type, about
  0.07 to 0.09 pooled.

### D-032 Opaque random run IDs
- 2026-11-25, before results, proposed.
- Directory names carry no fault information.

### D-033 Paired design
- 2026-11-25, before results, owner.
- Record once, score every method on the same recordings.

### D-034 Memory and CPU caps per container
- 2026-11-25, before results, owner (mem_limit, under about 3 GB); proposed (values).
- Planned values in PLAN.md, total 2496 MB. Revised after `docker stats` in stage 1.

## RCA task and evaluation

### D-035 Incident time given to methods is the true fault start
- 2026-11-25, before results, proposed.
- **Amended by D-049** (scope statement).
- The benchmark measures localization given detection. Control runs get a placebo
  incident time from the same distribution.
- Alternative: methods detect the incident themselves. Harder to score and mixes
  detection with localization.

### D-036 Methods are not given the fault catalog
- 2026-11-25, before results, proposed.
- Output uses a general vocabulary (FAULTS.md) that includes types never injected.

### D-037 Candidate set includes "none"
- 2026-11-25, before results, proposed.
- Allows false alarm measurement on control runs.

### D-038 Primary metric: top-1 component accuracy
- 2026-11-25, before results, proposed.
- **Superseded by D-051.**
- Top-3 component accuracy is reported as requested, but with 5 candidates chance is
  0.60, so it says little. Pair (component, fault type) accuracy is also reported.
  Chance levels printed next to every result.

### D-039 Statistics
- 2026-11-25, before results, owner (95% CIs, Wilson or bootstrap); proposed (details).
- **Superseded in part by D-052** (pairwise all-method tests dropped).
- Wilson for single proportions. McNemar exact for paired top-1 comparisons with Holm
  correction. Paired bootstrap (10,000 resamples, fixed seed) for differences and for
  non-proportion aggregates.

### D-040 Time to diagnosis = evidence window plus compute time
- 2026-11-25, before results, proposed.
- Recordings cut at incident time + 15, 30, 60, 120 s.

### D-041 LLM diagnoses run 3 times; sample 1 is scored
- 2026-11-25, before results, proposed.
- **Superseded by D-056.**
- Samples 2 and 3 only for agreement rates.

### D-042 Dev/test split and freeze
- 2026-11-25, before results, proposed.
- Tuning on dev runs only. Test set recorded after method code SHA, prompt hash, model
  ID and thresholds are logged here.

### D-043 RCA view includes Prometheus `up` and `docker stats`
- 2026-11-25, before results, proposed.
- **Superseded by D-050.**
- Realistic monitoring data, but it makes F2 and F5 easier (missing container, `up` =
  0). Option to also score an "application-only" view (traces and service metrics
  only) on the same recordings; open question.

## Owner review, 2026-11-25

All entries below: 2026-11-25, before results.

### D-044 Topology: the gateway orchestrates
- Owner. Supersedes D-009.
- gateway -> redis (rate-limit counter), gateway -> checker (verdict), then
  gateway -> mock_llm. The checker makes no outgoing calls. Matches PROMPTWALL, where
  the proxy runs its checks and then calls the LLM.

### D-045 Gateway failure handling in the new topology
- Proposed (follows from D-044). Supersedes D-010.
- Checker connection error, timeout or 500: 403 "blocked", same body as a policy
  block; reason not a metric label; the checker client span carries the error.
- Any mock_llm failure (after retries for 429): 502 to the client.

### D-046 The gateway retries 429 from mock_llm
- Proposed (follows from D-044). Supersedes D-012.
- Same policy as before, moved from checker to gateway: up to 2 retries, exponential
  backoff with seeded jitter, Retry-After honoured up to 1 s; 503 not retried.
- Differs from PROMPTWALL, which disables SDK retries; noted for stage 5.

### D-047 Gateway timeouts
- Proposed. Supersedes D-013.
- Checker 1 s (the checker only classifies). mock_llm 5 s per attempt. Revisited
  after the stage 1 capacity test, before any fault run.
- With 1 s, F4 may cause some fail-closed blocks in the tail; expected symptoms in
  FAULTS.md say so.

### D-048 Toxiproxy edges
- Proposed. Amends D-015.
- loadgen -> gateway, gateway -> redis, gateway -> checker, gateway -> mock_llm.

### D-049 Scope statement: localisation, not detection
- Owner. Amends D-035.
- PLAN.md states it now; the README states it with the results (stage 6 done test).

### D-050 Two RCA views scored on the same recordings
- Owner (both views); proposed (which is primary). Supersedes D-043.
- full: everything including Prometheus `up`, docker stats, vmstat. app-only:
  traces, service metrics, loadgen requests. Definitions in PLAN.md.
- The full view is used for the primary comparison; app-only is descriptive.

### D-051 Metrics
- Owner. Supersedes D-038.
- Primary: top-1 component accuracy. Secondary: top-1 and top-3 on the (component,
  fault type) pair. Top-3 component accuracy only in a secondary table next to its
  random-guess baseline.

### D-052 Preregistered primary comparison
- Owner. Supersedes D-039 in part.
- McNemar exact test, each method vs the rule baseline, top-1 component correctness,
  pooled over all 120 test fault runs, full view. Holm correction over the methods in
  the family. Per-fault results are descriptive only. Details in PLAN.md.
- Kept from D-039: Wilson CIs for single proportions; paired bootstrap CI for the
  accuracy difference, reported as effect size.
- Proposed detail: family = every method scored on all 120 fault runs, fixed at
  freeze; a method run only on a subset is descriptive.

### D-053 LLM models
- Owner.
- Development: Claude Haiku 5.5 only. Frozen test set: Claude Haiku 5.5 and Claude
  Opus 5.5; optional Qwen3.8 Max through an OpenAI-compatible backend.

### D-054 Hard LLM budget cap EUR 3 for the whole project, enforced in code
- Owner (cap); proposed (mechanism).
- Committed append-only ledger `llm_spend/ledger.jsonl`; calls refused if it is
  missing. Worst-case cost reserved before each call; refused if spent + reserved
  would exceed EUR 3. Prices and USD to EUR rate (with 10% margin) in a dated config
  file. SDK automatic retries disabled.

### D-055 Every LLM prompt under 100k tokens
- Owner.
- Input tokens counted before each call; 100k or more is refused. Also keeps Haiku 5.5
  in its lower price band (its price is higher above 100k tokens).

### D-056 LLM budget allocation
- Proposed, **pending owner confirmation**. Supersedes D-041.
- Estimate (list prices read 2026-11-25, 8k input and 1.5k output tokens per call):
  Opus 5.5 on all 140 runs x 2 views is about USD 17, far over the cap.
- Proposal: Haiku 5.5 on all test runs, both views, sample 1. Opus 5.5 on a seeded
  stratified subset of 30 runs (4 per fault, 6 controls), full view, sample 1. Haiku
  agreement samples and evidence-window cuts only on that subset. Qwen3.8 Max only
  if the remaining budget covers the subset. Estimated total about EUR 2.90.
- Consequence: Opus 5.5 is not in the primary test family (D-052).
- Estimates are replaced by measured tokens from dev runs before the freeze.

### D-057 Python 3.12 from uv; lock file committed
- Owner (uv installs Python); proposed (commit `uv.lock`).
- Python is installed by uv into its own directory, not system-wide.

### D-058 PROMPTWALL changes in stage 5 only, on a branch
- Owner.
- Instrumentation and any base URL setting go on a branch of the PROMPTWALL repo,
  merged only if its test suite passes. Nothing changes there before stage 5.

### D-059 Test set size confirmed
- Owner. Confirms D-031.
- 20 runs per fault plus 20 controls.
