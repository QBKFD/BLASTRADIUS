# BLASTRADIUS plan

## Goal

Measure how accurately different root cause analysis (RCA) methods identify the root
cause of injected faults in a small AI microservice system, using traces and metrics.
Methods: a rule baseline, statistical methods, and an LLM. Every reported number is
measured, comes with a sample size and a 95% confidence interval, and is reported
whether it is good or bad.

## Constraints

- Target host: netcup VPS Lite 2 G12.5s (4 shared x86 vCores, 8 GB RAM, 160 GB SSD,
  Linux). The load generator runs on the same host.
- Docker Compose only, no Kubernetes. `mem_limit` on every container. Whole stack
  under about 3 GB.
- No Grafana. Every published port is bound to 127.0.0.1. UIs (Jaeger, Prometheus,
  Toxiproxy API) are reached over an SSH tunnel. Nothing is exposed publicly.
- Python 3.12, uv, FastAPI, pytest, ruff, gitleaks.
- No real LLM API calls in stages 0 to 2. Secrets only in the git-ignored `.env`.
- Shared CPU causes latency noise. CPU steal and baseline latency variance are
  measured and reported.

## Design rules

1. **Paired design.** Each experiment run records traces, metrics, host CPU steal and
   container stats once. Every RCA method is scored on the same recordings.
2. **Ground truth from the injector only.** Component, fault type, start and end time
   are written by the injector when it injects and reverts. Ground truth is never
   inferred from symptoms.
3. **Ground truth is kept out of RCA input.** A loader builds the "RCA view" of a
   recording. It excludes `ground_truth.json` and injector logs. Run IDs are random
   and carry no fault information. Admin and fault endpoints are not traced and not
   in metrics, so the injection call itself never appears in the data.
4. **Dev/test split.** Methods are developed and tuned only on dev runs. The test set
   is recorded after method code, prompts and thresholds are frozen and their hashes
   logged in DECISIONS.md.
5. **Analysis fixed in advance.** Metrics, statistical tests and sensitivity analyses
   are fixed in this file before the test set is recorded. Any later change is logged
   in DECISIONS.md as made after results.
6. **No run is dropped based on symptoms or RCA output.** A run is excluded only if
   the injector could not confirm the injection (criteria in FAULTS.md).

## System under test (stage 1)

```
loadgen --tp--> gateway --tp--> checker --tp--> mock_llm
                   |
                   +----tp----> redis

tp = Toxiproxy proxy (one per edge, 4 in total)
gateway, checker, mock_llm --OTLP--> jaeger (all-in-one)
prometheus scrapes /metrics on gateway, checker, mock_llm
```

- **gateway** (FastAPI): entry point `POST /v1/chat`. Per-client rate-limit counter in
  Redis; if Redis is unreachable it returns 503. Calls the checker. Fail closed: if the
  checker cannot be reached, times out or returns 500, the gateway returns the same
  403 "blocked" response as a policy block. A 502 from the checker (upstream LLM
  failure) is passed on to the client as 502.
- **checker** (FastAPI): fake prompt classifier. Score is a deterministic function of
  the prompt text (keyword rules plus a hash-based score). Each request does a fixed
  amount of CPU work (target about 5 ms) so that CPU pressure has an effect. Policy
  (threshold, keyword lists) comes from a config file, reloaded through an admin
  endpoint. On "allow" it forwards the prompt to mock_llm and returns the answer.
  On 429 from mock_llm it retries up to 2 times with exponential backoff and seeded
  jitter, honouring Retry-After up to 1 s. Other LLM errors, and 429 after the last
  retry, are returned as 502.
- **mock_llm** (FastAPI): subset of the Anthropic Messages API (`POST /v1/messages`),
  so PROMPTWALL can use it in stage 5 through a base URL setting. Latency from a seeded
  lognormal distribution. Admin endpoint sets fault modes: error status with a rate,
  rate limit (token bucket, 429 with Retry-After), extra latency.
- **redis**: plain Redis, no persistence, no exporter.
- **toxiproxy**: one proxy per edge. No toxics in normal operation. No initial fault
  uses it; it is there for network faults added later.
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
  ground_truth.json   injector output; NOT in the RCA view
  injector.log        NOT in the RCA view
```

## RCA task definition

- **Input:** the RCA view of one recording and the incident time.
- **Incident time:** the true fault start (confirm timestamp). The benchmark measures
  localization given detection, not detection. This is a stated limitation.
- **Output:** a ranked list of candidates. Each candidate is a component from
  {gateway, checker, mock_llm, redis, none} and a fault type from the vocabulary in
  FAULTS.md. Methods are not given the fault catalog.

## Planned metrics

- **Top-1 component accuracy** per method, overall and per fault type, with Wilson
  95% CIs. This is the primary metric.
- **Top-3 component accuracy**, same breakdown. With 5 candidates, chance is 0.20 for
  top-1 and 0.60 for top-3; chance levels are printed next to every result.
- **Top-1 and top-3 accuracy on the (component, fault type) pair**, same breakdown.
- **False alarm rate** on control runs: share where top-1 is not "none", Wilson CI.
- **Paired method comparison:** McNemar exact test on top-1 correctness for each pair
  of methods, Holm correction over pairs. Paired bootstrap 95% CI for the accuracy
  difference (10,000 resamples of runs, fixed seed).
- **Time to diagnosis:**
  - compute time per diagnosis (wall clock, median and p95);
  - evidence window: recordings cut at incident time + {15, 30, 60, 120} s, accuracy
    at each cut. Time to diagnosis per run = smallest cut with correct top-1 plus
    compute time; runs never correct count as "> 120 s". Reported as median per
    method.
- **LLM cost per diagnosis:** input and output tokens, USD from the provider's list
  price on a stated date with the model ID. Mean, median, total, and cost per correct
  diagnosis.
- **LLM run-to-run variation:** each LLM diagnosis is run 3 times. Sample 1 is scored;
  samples 2 and 3 are used only to report agreement.
- **Noise:** per run, CPU steal mean and max, baseline p50/p95/p99 latency.
  Sensitivity analysis: all results repeated excluding runs whose mean steal during
  the fault window exceeds a threshold fixed after the stage 1 baseline check, before
  any fault run.

Expected precision: with n = 20 per fault type, a Wilson CI around 0.8 is about
[0.58, 0.92]. Per-fault differences between methods will mostly not be significant.
Pooled over 120 fault runs the half-width is about 0.07 to 0.09.

## Stages

### Stage 0: plan

Deliverables: PLAN.md, DECISIONS.md, FAULTS.md, repo skeleton (pyproject for uv,
.gitignore, ruff and pytest config, gitleaks pre-commit hook, MIT LICENSE, README
stub).

Done when:
- the files above exist and contain no em dashes or marketing words;
- `uv sync && uv run ruff check . && uv run pytest` passes on a machine with uv;
- the pre-commit hook refuses a commit with a staged fake secret;
- commit "stage 0: plan" exists locally and is not pushed;
- the owner has reviewed the plan.

### Stage 1: toy system and observability

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
  trace contains spans from gateway, checker and mock_llm plus a Redis client span;
  every Prometheus target is up and request counters increase;
- capacity test done; load rate set to 30 to 40% of the saturation rate and logged in
  DECISIONS.md;
- baseline noise check: 10 min without faults, reporting
  - p50/p95/p99 per service (from spans) for the whole period and per 1-min window,
  - variance and coefficient of variation of p50/p95/p99 across the 10 windows,
  - CPU steal from `vmstat 1`: mean and max,
  - RAM per container from `docker stats`: mean and peak, total peak under 3 GB,
  - trace completeness: at least 99% of loadgen requests found in Jaeger;
- steal threshold for the sensitivity analysis fixed and logged;
- Prometheus scrape interval confirmed or changed, and logged;
- ruff clean, unit tests pass.

### Stage 2: fault injection and experiment runner

Deliverables:
- Injector with the 6 faults from FAULTS.md, each with inject, verify and revert.
- Ground truth writer (JSON schema in FAULTS.md).
- Runner with seeded schedule, reset, phases and export.
- Recorder: Jaeger export, Prometheus range export, vmstat, docker stats, loadgen log.
- Loader that builds the RCA view.
- Dev set: 21 runs.

Done when:
- each fault's injection verifies and its revert returns the stack to healthy within
  30 s, on all dev runs;
- every ground truth file validates against the schema and its timestamps are in
  order;
- a test shows the RCA view cannot read `ground_truth.json` or `injector.log`, and a
  search of RCA view files finds no fault IDs or fault names;
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
- LLM method: evidence summary built from the recording, fixed prompt, JSON output,
  hard spending cap, fake backend for tests.

Done when:
- every method runs on every dev run and returns valid output;
- unit tests on synthetic recordings pass;
- the LLM method passes with the fake backend; the spending cap refuses a call that
  would exceed it; tokens and cost are logged per call;
- freeze: method code git SHA, prompt hash, model ID and thresholds logged in
  DECISIONS.md before test recording starts.

### Stage 4: evaluation

Deliverables: test set (140 runs), diagnoses for every method, `results.json`,
`results.md`, charts.

Done when:
- one command regenerates every table and chart from recordings and diagnoses;
- every proportion is shown with n and CI, next to its chance level;
- paired tests and sensitivity analyses are computed exactly as planned here;
  anything else is labelled as exploratory and logged as made after results;
- cost and time-to-diagnosis tables exist.

### Stage 5: PROMPTWALL as second system under test

Known from the PROMPTWALL README (read 2026-11-25): one FastAPI process with
in-process input checks (secrets regex, PII regex, injection classifier), a YAML
policy, fail closed on check errors, a spending cap, Anthropic/Ollama/fake LLM
backends, SDK retries disabled.

Consequences: candidates are in-process checks (identified through spans) plus the LLM
backend, not containers; there is no Redis fault; a 429 is not retried.

Deliverables:
- Compose setup for PROMPTWALL plus mock_llm, jaeger, prometheus, loadgen.
- Instrumentation in PROMPTWALL: one span per check, policy decision and LLM call;
  Prometheus metrics.
- Fault mapping for PROMPTWALL in FAULTS.md (LLM 503, LLM 429, check crash, bad
  policy, CPU pressure; possibly spending cap exhaustion).
- Dev and test recordings; results reported separately from the toy system.

Done when:
- the same runner, recorder and RCA methods are used; method changes are limited to a
  topology description and are logged;
- run counts fixed before recording;
- no tuning on PROMPTWALL test runs;
- results tables generated as in stage 4.

### Stage 6: README

Deliverables: README with goal, setup, method descriptions, results tables, noise and
cost numbers, limitations, reproduction steps.

Done when:
- a script checks that every number in the README matches `results.json`;
- the reproduction steps work from a fresh clone on the server;
- `gitleaks git` over the full history reports nothing.

## Known limitations (stated before any results)

- Small system: 4 components. Localization is easier than in real systems, and top-3
  component accuracy is close to saturated by chance.
- The person who wrote the faults also writes the rule baseline. This favours rules.
  Rules are tuned on dev runs only.
- Faults are clean step changes with exact start times. Real incidents are messier.
- The incident time is given, so detection is not measured.
- Load generator and system under test share 4 shared vCores; neighbours on the host
  add CPU steal.
- LLM results depend on model version and date.

## Open questions (for owner review)

All "proposed" entries in DECISIONS.md need review. Specific questions:

1. Topology: chain (checker proxies to mock_llm) or fan-out (gateway calls both)?
   (D-009)
2. Should the RCA view include infrastructure signals (Prometheus `up`,
   `docker stats`), and should an application-only view also be scored? (D-043)
3. Is giving the true fault start as incident time acceptable? (D-035)
4. Top-3 component accuracy is close to chance with 5 candidates. Keep it as a
   secondary metric, or replace it with pair (component, fault type) top-3? (D-038)
5. Which LLM provider and models in stage 3, and what total budget cap?
6. Stage 5: may PROMPTWALL be changed (instrumentation, base URL for the Anthropic
   backend), on a branch of that repo?
7. Test set size: 20 per fault plus 20 controls (about 14 h) acceptable? (D-031)
