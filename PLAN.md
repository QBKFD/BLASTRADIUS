# BLASTRADIUS plan

## Goal

Measure how accurately different root cause analysis (RCA) methods identify the root
cause of injected faults in a small AI microservice system, using traces and metrics.
Methods: a rule baseline, statistical methods, and LLMs. Every reported number is
measured, comes with a sample size and a 95% confidence interval, and is reported
whether it is good or bad.

## Scope: localisation, not detection

BLASTRADIUS measures **fault localisation**: given that an incident is known to have
started at a given time, which component (and which fault type) caused it. It does
not measure **detection**: whether or how fast a method notices that something is
wrong. RCA methods receive the true fault start time as the incident time (D-035,
D-049). This statement is repeated in the README with the results.

## Constraints

- Target host: netcup VPS Lite 2 G12.5s (4 shared x86 vCores, 8 GB RAM, 160 GB SSD,
  Linux). The load generator runs on the same host.
- Docker Compose only, no Kubernetes. `mem_limit` on every container. Whole stack
  under about 3 GB.
- No Grafana. Every published port is bound to 127.0.0.1. UIs (Jaeger, Prometheus,
  Toxiproxy API) are reached over an SSH tunnel. Nothing is exposed publicly.
- Python 3.12, uv, FastAPI, pytest, ruff, gitleaks.
- No real LLM API calls in stages 0 to 2. Secrets only in the git-ignored `.env`.
- LLM spend: hard cap of EUR 3 for the whole project, enforced in code. Every LLM
  prompt under 100k tokens.
- Shared CPU causes latency noise. CPU steal and baseline latency variance are
  measured and reported.

## Design rules

1. **Paired design.** Each experiment run records traces, metrics, host CPU steal and
   container stats once. Every RCA method is scored on the same recordings.
2. **Ground truth from the injector only.** Component, fault type, start and end time
   are written by the injector when it injects and reverts. Ground truth is never
   inferred from symptoms.
3. **Ground truth is kept out of RCA input.** A loader builds the RCA views of a
   recording. They exclude `ground_truth.json` and injector logs. Run IDs are random
   and carry no fault information. Admin and fault endpoints are not traced and not
   in metrics, so the injection call itself never appears in the data.
4. **Dev/test split.** Methods are developed and tuned only on dev runs. The test set
   is recorded after method code, prompts, model settings and thresholds are frozen
   and their hashes logged in DECISIONS.md.
5. **Analysis fixed in advance.** Metrics, the primary statistical test and sensitivity
   analyses are fixed in this file before the test set is recorded. Any later change is
   logged in DECISIONS.md as made after results.
6. **No run is dropped based on symptoms or RCA output.** A run is excluded only if
   the injector could not confirm the injection (criteria in FAULTS.md).

## System under test (stage 1)

The gateway orchestrates: it checks the prompt with the checker, then calls the LLM
itself. This matches PROMPTWALL, where the proxy runs its checks and then calls the
LLM (D-044).

```
                    +--tp--> redis
                    |
loadgen --tp--> gateway --tp--> checker
                    |
                    +--tp--> mock_llm

tp = Toxiproxy proxy (one per edge, 4 in total)
gateway, checker, mock_llm --OTLP--> jaeger (all-in-one)
prometheus scrapes /metrics on gateway, checker, mock_llm
```

Request flow in the gateway: Redis rate-limit counter, then checker, then (if allowed)
mock_llm.

- **gateway** (FastAPI): entry point `POST /v1/chat`.
  - Redis: per-client rate-limit counter. Redis unreachable gives 503 (client timeout
    50 ms).
  - Checker: `POST /classify`, timeout 1 s. Fail closed: connection error, timeout or
    500 gives the same 403 "blocked" response as a policy block. The block reason is
    not a metric label; the checker client span carries the error.
  - mock_llm: timeout 5 s per attempt. On 429 it retries up to 2 times with
    exponential backoff and seeded jitter, honouring Retry-After up to 1 s. 503 and
    other errors are not retried. Any LLM failure, including 429 after the last retry,
    gives 502 to the client.
- **checker** (FastAPI): fake prompt classifier, returns a verdict and a score. Score
  is a deterministic function of the prompt text (keyword rules plus a hash-based
  score). Each request does a fixed amount of CPU work (target about 5 ms) so that CPU
  pressure has an effect. Policy (threshold, keyword lists) comes from a config file,
  reloaded through an admin endpoint. The checker makes no outgoing calls.
- **mock_llm** (FastAPI): subset of the Anthropic Messages API (`POST /v1/messages`),
  so PROMPTWALL can use it in stage 5 through a base URL setting. Latency from a seeded
  lognormal distribution. Admin endpoint sets fault modes: error status with a rate,
  rate limit (token bucket, 429 with Retry-After), extra latency.
- **redis**: plain Redis, no persistence, no exporter.
- **toxiproxy**: one proxy per edge (loadgen to gateway, gateway to redis, gateway to
  checker, gateway to mock_llm). No toxics in normal operation. No initial fault uses
  it; it is there for network faults added later.
- **jaeger** all-in-one: in-memory storage with a trace cap, OTLP receiver. 100%
  sampling.
- **prometheus**: scrape interval 5 s (revisited after the baseline noise check,
  before any fault run). Request counts, error counts, latency histograms per service
  and route, client-side latency per outgoing edge, checker verdict counts.
- **loadgen**: open-loop Poisson arrivals at a fixed rate with a fixed seed, fixed
  prompt set with a known benign/malicious mix (target about 5% blocked in normal
  operation). Writes one line per request (send time, latency, status).

Admin endpoints use a separate port per service, bound to 127.0.0.1, not traced, not
in metrics, not routed through Toxiproxy.

Planned resource limits (revised after the stage 1 measurement):

| Container  | mem_limit | cpus |
|------------|-----------|------|
| gateway    | 256m      | 1.0  |
| checker    | 256m      | 1.0  |
| mock_llm   | 256m      | 0.5  |
| redis      | 128m      | 0.5  |
| toxiproxy  | 64m       | 0.5  |
| jaeger     | 768m      | 1.0  |
| prometheus | 512m      | 0.5  |
| loadgen    | 256m      | 0.5  |
| **total**  | **2496m** |      |

`cpus` values are caps, not reservations; their sum may exceed 4.

## Experiment protocol (stages 2, 4, 5)

One run:

1. **Reset.** All containers running and healthy, mock_llm and checker admin state
   reset, Redis flushed, Toxiproxy toxics cleared. Load starts. Settle 30 s (not
   scored).
2. **Baseline.** 120 s plus a seeded random 0 to 30 s, so the fault does not start at a
   fixed offset.
3. **Fault.** 120 s. The injector writes ground truth with issue and confirm
   timestamps.
4. **Recovery.** 60 s after the revert is confirmed.
5. **Export.** Wait 10 s for span export, then export traces, metrics and side logs.

About 6 minutes per run including reset and export. Recorded for the whole run:
`vmstat 1` on the host (CPU steal), `docker stats` every 2 s, loadgen per-request log.

**Control runs** have no injection. They get a placebo incident time drawn the same
way as a real fault start.

**Schedule.** Faults and controls are interleaved in a seeded random order, so
time-of-day and neighbour noise spread evenly over fault types.

**Run counts.** Dev: 3 per fault plus 3 controls (21 runs). Test: 20 per fault plus
20 controls (140 runs, about 14 h).

All timestamps are UTC from the single host clock, so there is no clock skew between
components.

### Recording layout

```
recordings/<run_id>/
  manifest.json       git SHA, image digests, config hashes, seeds, load rate
  incident.json       incident time given to RCA methods
  traces.jsonl.gz     Jaeger export for the run window
  metrics.*           Prometheus range-query export (format decided in stage 2)
  host_vmstat.csv     vmstat 1 s, including steal
  docker_stats.csv    per-container CPU and memory, 2 s
  requests.csv.gz     loadgen per-request log
  ground_truth.json   injector output; in no RCA view
  injector.log        in no RCA view
```

### RCA views

Every method is scored on two views of the same recording (D-050):

- **full**: traces, all Prometheus series including `up`, `docker_stats.csv`,
  `host_vmstat.csv`, loadgen requests.
- **app-only**: traces, service metrics from gateway, checker and mock_llm, loadgen
  requests. No `up`, no scrape metadata, no `docker_stats.csv`, no `host_vmstat.csv`.
  A killed service still shows as series that stop; that is application-level data.

The full view is primary. The app-only view is secondary and reported with the same
tables.

## RCA task definition

- **Input:** one RCA view of one recording and the incident time.
- **Incident time:** the true fault start (confirm timestamp). See "Scope" above.
- **Output:** a ranked list of candidates. Each candidate is a component from
  {gateway, checker, mock_llm, redis, none} and a fault type from the vocabulary in
  FAULTS.md. Methods are not given the fault catalog.

## Planned metrics

- **Primary: top-1 component accuracy** per method, overall and per fault type, with
  Wilson 95% CIs.
- **Secondary: top-1 and top-3 accuracy on the (component, fault type) pair**, same
  breakdown.
- **Top-3 component accuracy** is reported only in a secondary table, always next to
  the random-guess baseline (0.60 with 5 candidates).
- Random-guess baselines are printed next to every accuracy: top-1 component 0.20,
  pair top-1 1/33 and pair top-3 3/33 (4 components x 8 fault types plus "none").
- **False alarm rate** on control runs: share where top-1 is not "none", Wilson CI.
- **Time to diagnosis:**
  - compute time per diagnosis (wall clock, median and p95);
  - evidence window: recordings cut at incident time + {15, 30, 60, 120} s, accuracy
    at each cut. Time to diagnosis per run = smallest cut with correct top-1 plus
    compute time; runs never correct count as "> 120 s". Reported as median per
    method. For LLM methods only on the subset given in the budget section.
- **LLM cost per diagnosis:** input and output tokens (thinking tokens are output),
  cost from the provider's list price on a stated date with the model ID (batch
  price for test-set calls). Mean,
  median, total, and cost per correct diagnosis.
- **LLM run-to-run variation:** on the subset given in the budget section, each LLM
  diagnosis is run 3 times. Sample 1 is scored; samples 2 and 3 are used only to
  report agreement.
- **Noise:** per run, CPU steal mean and max, baseline p50/p95/p99 latency.
  Sensitivity analysis: all results repeated excluding runs whose mean steal during
  the fault window exceeds a threshold fixed after the stage 1 baseline check, before
  any fault run.

### Preregistered primary comparison (D-052)

- **Hypothesis family:** each method's top-1 component accuracy against the rule
  baseline.
- **Data:** all 120 test fault runs pooled over the 6 faults, full view, sample 1.
  Control runs are not part of this test.
- **Test:** McNemar exact test on paired top-1 correctness (method vs rule baseline),
  two-sided, alpha 0.05.
- **Multiple comparisons:** Holm correction over the methods in the family.
- **Family membership:** every method scored on all 120 fault runs. The list of
  methods is fixed in the freeze entry in DECISIONS.md before test recording. The
  random baseline is not in the family.
- **Effect size:** accuracy difference with a paired bootstrap 95% CI (10,000
  resamples of runs, fixed seed).
- **Everything else is descriptive:** per-fault results, the app-only view, the
  evidence-window cuts, LLM methods run on a subset, and the steal sensitivity
  analysis. They are reported with CIs but no significance claims.

Expected precision: with n = 20 per fault type, a Wilson CI around 0.8 is about
[0.58, 0.92]. Pooled over 120 fault runs the half-width is about 0.07 to 0.09.

### LLM models, settings and budget

Models (D-053): development on Claude Haiku 5.5 only. On the frozen test set: Claude
Haiku 5.5 and Claude Opus 5.5; optional Qwen3.8 Max through an OpenAI-compatible
backend.

Request settings (D-061), the same for both Anthropic models:
- `output_config.effort = "low"`. On Opus 5.5 thinking cannot be turned off and a
  thinking token budget (`budget_tokens`) is rejected by the API; effort is the only
  control for how much it thinks. Haiku 5.5 gets the same setting so the two models
  differ only in the model.
- `max_tokens = 8000` (thinking plus answer). Hard ceiling; revisited after the
  pre-freeze measurement.
- Thinking display left at the default (omitted); thinking tokens are still counted
  from `usage`.
- JSON output through structured outputs (`output_config.format`).
- No `temperature` or other sampling parameters (rejected by these models).
- A response with `stop_reason` `max_tokens` or `refusal`, or output that does not
  parse, counts as an incorrect diagnosis. It is not re-run. Such cases are counted
  and reported per model.

Test-set calls (D-060): every test-set LLM call to Anthropic goes through the Message
Batches API (50% discount on all tokens). Dev prompt development uses the normal
Messages API. A batch request that fails for infrastructure reasons (errored,
expired) is resubmitted once; this is not a re-run of a model answer.

Budget enforcement (D-054, D-055, D-062):
- Hard cap EUR 3 for the whole project, all providers together.
- Spend ledger `llm_spend/ledger.jsonl`, committed to git, append-only. Code refuses
  to call any LLM if the ledger file is missing.
- Before each call: count input tokens; refuse if 100k or more. Reserve the worst case
  (input tokens x input price + `max_tokens` x output price, at batch price for batch
  requests); refuse if spent + reserved would exceed the cap. After the call, record
  actual usage (input, output, thinking tokens) and cost.
- Batches are submitted in chunks small enough that the whole chunk's worst-case
  reservation fits in the remaining budget. Reservations are replaced by actual cost
  when the chunk returns.
- Prices per model (standard and batch) in a config file with the date they were
  read. USD prices are converted with a fixed rate recorded in that file, with a 10%
  margin. Prices are re-read from the provider's pricing page at the freeze.
- SDK automatic retries disabled, so no billed call happens outside the ledger.

Cost estimate with list prices as read on 2026-11-25 (Haiku 5.5: USD 0.10 / 0.50
per million input / output tokens for prompts up to 100k tokens; Opus 5.5: USD 4 /
20; batch: half of that). Assumed per call: 8k input tokens, 1.5k output tokens
including thinking, at effort low. EUR figures treat USD as EUR; the exchange rate
and the 10% margin roughly cancel at current rates.

| Model     | Standard, USD per call | Batch, USD per call |
|-----------|------------------------|---------------------|
| Haiku 5.5 | about 0.0016           | about 0.0008        |
| Opus 5.5  | about 0.062            | about 0.031         |

Allocation (D-056 as amended by D-063):

| Item                                                                  | API      | Calls      | EUR (est.) |
|-----------------------------------------------------------------------|----------|------------|------------|
| Dev, Haiku 5.5, prompt development                                    | standard | up to 150  | 0.23       |
| Pre-freeze measurement, 5 dev calls per model (Haiku, Opus)           | batch    | 10         | 0.16       |
| Test, Haiku 5.5, all 140 runs, both views, sample 1, 120 s cut        | batch    | 280        | 0.22       |
| Test, Opus 5.5, subset, full view, sample 1, 120 s cut                | batch    | 60         | 1.86       |
| Test, Haiku 5.5, subset: samples 2 and 3, cuts 15/30/60 s, full view  | batch    | 300        | 0.23       |
| Reserve (Qwen3.8 Max only if it fits, otherwise unused)               |          |            | 0.30       |
| **Total**                                                             |          |            | **3.00**   |

- **Subset:** seeded, stratified, 8 runs per fault plus 12 controls (60 runs), drawn
  and logged before the freeze.
- **Opus 5.5 is descriptive only**, not in the primary test family (D-052).
- **Sensitivity:** Opus output tokens drive the total. With 8k input, the allocation
  holds if Opus averages up to about 2k output tokens per call; each extra 1k output
  tokens costs about EUR 0.60 over 60 calls.
- **Order of LLM jobs on the test set:** Haiku sample 1 on all runs (primary family),
  Opus subset, Haiku subset extras, Qwen. If the cap stops a job, what was finished
  is reported and the rest is reported as not run.

### Pre-freeze cost measurement (D-064)

Before the freeze, 5 dev runs are diagnosed by each test-set model (Haiku 5.5 and
Opus 5.5; Qwen3.8 Max too if it is to be used) with the final prompt and settings,
through the same API path as the test set. Recorded per call: input, output and
thinking tokens, stop reason, cost. The allocation table is recomputed from the
measured means (and the maximum output, to check `max_tokens`).

If the recomputed plan exceeds EUR 3, work stops and the owner is told, with the
measured numbers. The subset size, effort, `max_tokens` and prompt are not changed to
make it fit without the owner's decision.

## Stages

### Stage 0: plan

Deliverables: PLAN.md, DECISIONS.md, FAULTS.md, repo skeleton (pyproject for uv,
uv.lock, .gitignore, ruff and pytest config, gitleaks pre-commit hook, MIT LICENSE,
README stub).

Done when:
- the files above exist and contain no em dashes or marketing words;
- `uv sync && uv run ruff check . && uv run ruff format --check . && uv run pytest`
  passes;
- the pre-commit hook refuses a commit with a staged fake secret;
- the owner has reviewed the plan; review changes are logged in DECISIONS.md;
- commits are pushed to origin main.

### Stage 1: toy system and observability

Detailed design and work order: [STAGE1.md](STAGE1.md).

Deliverables:
- gateway, checker, mock_llm services with unit tests; Dockerfiles; compose file with
  redis, toxiproxy, jaeger, prometheus, loadgen; `mem_limit` on every container.
- Loadgen with fixed seed and per-request log.
- Smoke test (pytest, marked `integration`).
- Capacity test script: finds the request rate where p95 latency or errors break down.
- Baseline noise check script and report in `out/baseline/`.

Done when:
- `docker compose up -d` on the server brings every container to healthy within 60 s;
- `ss -tlnp` shows every published port on 127.0.0.1 only;
- smoke test passes: requests return the expected mix of 200 and 403; at least one
  trace contains a gateway root span with child spans for Redis, the checker and
  mock_llm, and server spans from checker and mock_llm; every Prometheus target is up
  and request counters increase;
- capacity test done; load rate set to 50% of the measured saturation rate, rounded
  down to a whole number of requests per second; saturation rate, step results and
  load rate logged in DECISIONS.md (D-083);
- baseline noise check: 10 min without faults, reporting
  - p50/p95/p99 per service (from spans) for the whole period and per 1-min window,
  - variance and coefficient of variation of p50/p95/p99 across the 10 windows,
  - CPU steal from `vmstat 1`: mean and max,
  - RAM per container from `docker stats`: mean and peak, total peak under 3 GB,
  - trace completeness: at least 99% of loadgen requests found in Jaeger;
- steal threshold for the sensitivity analysis fixed and logged;
- Prometheus scrape interval and gateway timeouts confirmed or changed, and logged;
- ruff clean, unit tests pass.

### Stage 2: fault injection and experiment runner

Deliverables:
- Injector with the 6 faults from FAULTS.md, each with inject, verify and revert.
- Ground truth writer (JSON schema in FAULTS.md).
- Runner with seeded schedule, reset, phases and export.
- Recorder: Jaeger export, Prometheus range export, vmstat, docker stats, loadgen log.
- Loader that builds the full and app-only RCA views.
- Dev set: 21 runs.

Done when:
- each fault's injection verifies and its revert returns the stack to healthy within
  30 s, on all dev runs;
- every ground truth file validates against the schema and its timestamps are in
  order;
- a test shows neither RCA view can read `ground_truth.json` or `injector.log`, and a
  search of RCA view files finds no fault IDs or fault names;
- a test shows the app-only view contains no `up` series, no docker stats and no
  vmstat data;
- recording checks pass: trace completeness at least 99% (except requests that never
  reach a service), no Prometheus gaps longer than 2 scrapes except for a killed
  target, vmstat rows match run duration;
- observed symptoms on dev runs are compared with FAULTS.md; differences are written
  into FAULTS.md, marked "revised after dev runs";
- disk use per run measured.

No real LLM calls in this stage.

### Stage 3: RCA methods

Deliverables:
- Common interface: `diagnose(view, incident_time) -> Diagnosis` (ranked candidates,
  compute time, tokens, cost).
- Random baseline (seeded) for reference.
- Rule baseline: hand-written rules over metrics and spans.
- Statistical method(s). Starting point: per-component anomaly scores (z-scores of
  error rate, latency, request rate, span exclusive time against the pre-incident
  window), ranked with call-graph ordering. A second method (for example a random
  walk on the service graph) if time allows. Final list fixed before the test set.
- LLM method: evidence summary built from the recording (under 100k tokens, target
  under 8k), fixed prompt, JSON output. Backends: Anthropic (Haiku 5.5, Opus 5.5),
  OpenAI-compatible (Qwen3.8 Max, optional), fake (tests). Spend ledger and cap as in
  the budget section.

Done when:
- every method runs on every dev run (LLM: Haiku 5.5 only) and returns valid output;
- unit tests on synthetic recordings pass;
- the LLM method passes with the fake backend; tests show the cap refuses a call
  that would exceed EUR 3, a prompt of 100k tokens or more is refused, a missing
  ledger file blocks all calls, and tokens and cost are logged per call;
- pre-freeze cost measurement done as specified (5 dev calls per model); the
  allocation table is recomputed from measured tokens; if it exceeds EUR 3, work
  stops for an owner decision;
- freeze: method code git SHA, prompt hash, model IDs and settings, thresholds, the
  primary test family and the LLM subset run IDs logged in DECISIONS.md before test
  recording starts.

### Stage 4: evaluation

Deliverables: test set (140 runs), diagnoses for every method on both views,
`results.json`, `results.md`, charts.

Done when:
- one command regenerates every table and chart from recordings and diagnoses;
- every proportion is shown with n and CI, next to its random-guess baseline;
- the primary comparison is computed exactly as preregistered; everything else is
  labelled descriptive; any deviation is logged as made after results;
- cost and time-to-diagnosis tables exist; total LLM spend from the ledger is
  reported.

### Stage 5: PROMPTWALL as second system under test

Known from the PROMPTWALL README (read 2026-11-25): one FastAPI process with
in-process input checks (secrets regex, PII regex, injection classifier), a YAML
policy, fail closed on check errors, a spending cap, Anthropic/Ollama/fake LLM
backends, SDK retries disabled.

Consequences: candidates are in-process checks (identified through spans) plus the LLM
backend, not containers; there is no Redis fault; a 429 is not retried, unlike the toy
gateway.

Changes to PROMPTWALL (D-058): made on a branch of the PROMPTWALL repo, merged only if
its test suite passes. Not started before stage 5.

Deliverables:
- Branch of PROMPTWALL with instrumentation (one span per check, policy decision and
  LLM call; Prometheus metrics) and, if not already supported, a base URL setting for
  the Anthropic backend.
- Compose setup for PROMPTWALL plus mock_llm, jaeger, prometheus, loadgen.
- Fault mapping for PROMPTWALL in FAULTS.md (LLM 503, LLM 429, check crash, bad
  policy, CPU pressure; possibly spending cap exhaustion).
- Dev and test recordings; results reported separately from the toy system.

Done when:
- the PROMPTWALL test suite passes on the branch before it is merged;
- the same runner, recorder and RCA methods are used; method changes are limited to a
  topology description and are logged;
- run counts and LLM budget use fixed before recording, within the EUR 3 project cap;
- no tuning on PROMPTWALL test runs;
- results tables generated as in stage 4.

### Stage 6: README

Deliverables: README with goal, scope (localisation, not detection), setup, method
descriptions, results tables, noise and cost numbers, limitations, reproduction
steps.

Done when:
- a script checks that every number in the README matches `results.json`;
- the README states that the benchmark measures localisation, not detection;
- the README limitations include every item in "Known limitations" below, including
  that both Anthropic models ran at effort low, not their best configuration (D-084);
- the reproduction steps work from a fresh clone on the server;
- `gitleaks git` over the full history reports nothing.

## Known limitations (stated before any results)

- Small system: 4 components. Localisation is easier than in real systems, and top-3
  component accuracy is close to saturated by chance.
- The person who wrote the faults also writes the rule baseline. This favours rules.
  Rules are tuned on dev runs only.
- Faults are clean step changes with exact start times. Real incidents are messier.
- The incident time is given, so detection is not measured.
- Load generator and system under test share 4 shared vCores; neighbours on the host
  add CPU steal.
- LLM results depend on model version and date. Opus 5.5 settings cannot be tuned on
  dev runs within the budget; both Anthropic models run at effort low, so results
  describe that setting, not the models' best.
- The EUR 3 cap limits LLM sample sizes; LLM results on the subset have wide CIs.

## Open questions (for owner review)

1. Stage 1 proposals in STAGE1.md still open: D-066 to D-073 (D-069 waiting on owner
   review of its full text), D-075 to D-077, D-080, D-081.
