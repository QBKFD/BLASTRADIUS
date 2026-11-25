# Stage 1 plan: toy system and observability

Planning only. Nothing in this file is built yet. The server is not set up; Docker is
not run before it is. Choices here are logged in DECISIONS.md (D-066 onward) as
proposed, before results. Done tests are in PLAN.md, stage 1.

## Work order

On the Mac (no Docker; code and unit tests only):

1. `common`: settings, telemetry setup, metrics helpers, admin server helper.
2. `checker` with unit tests.
3. `mock_llm` with unit tests.
4. `gateway` with unit tests (Redis faked with fakeredis, HTTP faked with respx).
5. `loadgen` with unit tests (schedule determinism, open-loop timing logic).
6. Prompt set: generator script and the committed prompt file.
7. Dockerfile, `compose.yaml`, Prometheus, Jaeger and Toxiproxy config files.
8. Scripts: smoke test, checker CPU calibration, capacity test, baseline noise check.

On the server (after it is set up, see "Server setup"):

9. `docker compose up -d`, port check, smoke test.
10. Checker CPU work calibration (target about 5 ms per request).
11. Capacity test; set the load rate; log it.
12. Baseline noise check; fix steal threshold, scrape interval and timeouts; log them.
13. Memory check against the 3 GB budget; adjust `mem_limit` values; log them.

One commit per step, each with ruff and unit tests passing.

## Code layout

```
src/blastradius/
  common/        settings, telemetry, metrics, admin server, request ids
  gateway/       app.py, clients (redis, checker, llm), retry policy
  checker/       app.py, scoring, policy loading, admin routes
  mock_llm/      app.py, fault modes, latency model, admin routes
  loadgen/       schedule, runner, request log writer
  scripts/       calibrate_checker.py, capacity_test.py, baseline_noise.py
deploy/
  Dockerfile
  compose.yaml
  prometheus.yml
  jaeger.yaml
  toxiproxy.json
  checker/policy.yaml
  checker/policy_block_all.yaml   (used only by fault F6, stage 2)
data/prompts.jsonl                generated, committed
tests/unit/...                    run on the Mac
tests/integration/...             marked `integration`, run on the server
```

One Python package and one image for all four Python containers (gateway, checker,
mock_llm, loadgen); the container command selects the service (D-066). The image
also contains `stress-ng` (D-067), used for fault F4 in the checker container.

Planned dependencies (added when stage 1 starts): fastapi, uvicorn, httpx, redis,
prometheus-client, pyyaml, opentelemetry-sdk, opentelemetry-exporter-otlp-proto-http,
opentelemetry-instrumentation-fastapi, opentelemetry-instrumentation-httpx,
opentelemetry-instrumentation-redis; dev: fakeredis, respx, pytest-asyncio. Versions
locked in `uv.lock`.

## Services

All three services: one uvicorn process, one worker (D-071). `GET /healthz` checks
the process only, not dependencies, so Docker health does not flap during faults.
`/healthz` and `/metrics` are excluded from tracing and from request metrics.

Admin endpoints run as a second, uninstrumented HTTP server in the same process on a
separate port (D-068). They are not traced, not counted in metrics, and not routed
through Toxiproxy.

### gateway (port 8000)

`POST /v1/chat`, body `{"client_id": str, "prompt": str}`, header `X-Request-Id`.

Steps per request:
1. Redis: `INCR rl:<client_id>:<minute>` plus `EXPIRE` (pipeline). Limit 1000 per
   client per minute (D-076), so it never triggers at the planned load. Timeout
   50 ms. Redis error or timeout: 503 `{"error": "unavailable"}`. Over the limit: 429.
2. Checker: `POST /classify`, timeout 1 s. Verdict "block": 403
   `{"blocked": true}`. Connection error, timeout or 500: the same 403 body (fail
   closed).
3. mock_llm: `POST /v1/messages`, timeout 5 s per attempt. On 429: up to 2 retries,
   backoff `0.2 s x 2^attempt` with seeded jitter (plus or minus 20%), or Retry-After
   if given, capped at 1 s. Any other failure, or 429 after the last retry: 502
   `{"error": "upstream"}`.
4. 200 `{"answer": str, "usage": {...}}`.

### checker (port 8001, admin 19002)

`POST /classify`, body `{"prompt": str}`, returns `{"verdict": "allow" | "block",
"score": float}`.

- Score: weighted keyword matches from the policy plus a deterministic hash-based
  term in [0, 0.3) from the prompt text (D-073).
- Verdict: "block" if score >= `threshold` (normal policy 0.9; block-all policy 0.0).
- CPU work: a fixed number of SHA-256 iterations per request. The number is
  calibrated once on the server to about 5 ms, then fixed and logged.
- Admin: `POST /admin/reload` (re-read the policy file), `GET /admin/config`
  (returns the SHA-256 of the loaded policy file).

### mock_llm (port 8002, admin 19003)

`POST /v1/messages`: subset of the Anthropic Messages API (D-014). Accepts `model`,
`max_tokens`, `messages`. Returns a message object (`id`, `type`, `role`, `content`
with one text block, `model`, `stop_reason`, `usage` with `input_tokens` and
`output_tokens`). Errors use the Anthropic error shape
(`{"type": "error", "error": {"type": ..., "message": ...}}`); 429 carries a
`retry-after` header.

- Latency: lognormal, median 300 ms, sigma 0.5, drawn from a generator seeded with
  (run seed, request id), so each request gets the same latency on every run (D-074).
- Answer text and output tokens: deterministic from the request id.
- Fault modes, set through admin `POST /admin/fault`:
  `{"mode": "none"}`,
  `{"mode": "error", "status": int, "rate": float}`,
  `{"mode": "rate_limit", "rps": float, "retry_after_s": int}` (token bucket),
  `{"mode": "latency", "extra_ms": int}` (not in the initial catalog).
  `GET /admin/fault` reads the mode back; `POST /admin/reset` sets "none".

## Metrics (Prometheus, D-069)

Server-side only, plus the checker verdict counter. Outgoing-call behaviour (Redis,
checker and LLM calls from the gateway, retries, connection errors) is visible only
in spans. This keeps a clear split between what metric-only and trace-based methods
can see.

| Metric                                  | Type      | Labels                     |
|-----------------------------------------|-----------|----------------------------|
| `http_server_requests_total`            | counter   | `service`, `route`, `status` |
| `http_server_request_duration_seconds`  | histogram | `service`, `route`         |
| `http_server_in_flight_requests`        | gauge     | `service`                  |
| `checker_verdicts_total`                | counter   | `verdict`                  |

Histogram buckets (seconds): 0.005, 0.01, 0.025, 0.05, 0.1, 0.2, 0.3, 0.5, 0.75, 1,
1.5, 2, 3, 5, 10. Standard process metrics from prometheus-client (CPU seconds,
resident memory) are kept.

Prometheus scrapes the services directly, not through Toxiproxy, every 5 s.
Retention 2 days and 1 GB.

## Traces (D-070)

- OpenTelemetry SDK, OTLP over HTTP to Jaeger, batch span processor with a 1 s
  schedule delay.
- Auto-instrumentation: FastAPI (server spans), httpx (client spans), redis (client
  spans). Standard semantic convention attributes.
- Manual attributes: gateway server span `gateway.decision`
  (`allowed`, `blocked`, `error`); a gateway span `llm.call` around the retry loop
  with `llm.attempts`; checker server span `checker.verdict`, `checker.score`;
  mock_llm server span `llm.output_tokens`.
- `service.name` is the only resource attribute that names the component. No
  attribute names a fault.
- Sampling 100%.

## Load generator (D-075)

- Open loop: arrival times from a Poisson process with a fixed seed; each request is
  sent at its planned time whether or not earlier ones have returned.
- 50 client ids; prompts drawn from `data/prompts.jsonl` with the same seed.
- In-flight cap 500; a request that would exceed it is not sent and is counted.
- Client timeout 15 s.
- Per-request log: `request_id`, `planned_ts`, `sent_ts`, `latency_ms`, `status`,
  `client_id`, `prompt_id`, `prompt_label`. `prompt_id` and `prompt_label` are kept
  for checks but are removed from both RCA views (stage 2 loader).
- Schedule lag (`sent_ts - planned_ts`) is reported; p99 lag must stay under 50 ms at
  the chosen rate, otherwise the load generator is the bottleneck.

## Prompt set (D-072)

Synthetic, generated from templates with a fixed seed, committed as
`data/prompts.jsonl` (about 2,000 prompts). About 5% "malicious" prompts contain
policy keywords; the rest are benign. No external dataset, so no licence question.
The generator and its seed are committed with the file.

## Infrastructure containers

- Image versions pinned to exact tags and digests in `compose.yaml` when it is
  written; recorded in each run manifest (D-081).
- **redis**: no persistence (`--save "" --appendonly no`), not published.
- **toxiproxy**: proxies created from `toxiproxy.json` at startup:

  | Proxy   | Listen | Upstream       |
  |---------|--------|----------------|
  | gateway | 18080  | gateway:8000   |
  | redis   | 16379  | redis:6379     |
  | checker | 18001  | checker:8001   |
  | llm     | 18002  | mock_llm:8002  |

  Services address each other through these proxies.
- **jaeger** (v2 all-in-one): memory storage, `max_traces` 50,000 (about 6 runs at
  20 requests per second; traces are exported after each run).
- **prometheus**: config as above.
- Restart policy `no` for every container (D-082), so a killed container stays down
  until the injector restarts it, and an unplanned crash is visible.

Published ports, all on 127.0.0.1 (D-077):

| Port  | What                                    |
|-------|-----------------------------------------|
| 18080 | gateway entry (through Toxiproxy)       |
| 16686 | Jaeger UI and query API                 |
| 9090  | Prometheus UI and API                   |
| 8474  | Toxiproxy API                           |
| 19002 | checker admin                           |
| 19003 | mock_llm admin                          |

Planned `mem_limit` and `cpus` values are in PLAN.md.

## Tests

Unit tests (Mac, no Docker):
- checker: scoring is deterministic; threshold boundary; policy reload changes the
  hash; block-all policy blocks every prompt in the prompt set.
- mock_llm: response shape matches the Messages API subset; each fault mode;
  token bucket rate; Retry-After header; latency deterministic per request id.
- gateway: each failure path gives the planned status (Redis down 503, checker
  timeout and connection error 403, checker 500 403, LLM 503 502, LLM 429 retried
  then 200 or 502); retry timing within the backoff bounds; no retry on 503.
- loadgen: same seed gives the same schedule and prompt sequence; open-loop timing
  does not wait on responses.
- admin routes are absent from the instrumented app.

Integration tests (server, marked `integration`): the smoke test from PLAN.md.

## Scripts

- **calibrate_checker.py**: runs the checker CPU work in the checker container with
  no load and reports the iteration count for a 5 ms median.
- **capacity_test.py**: step load 5, 10, 20, 40, 80 requests per second, 60 s per
  step, no faults (D-079). Saturation rate = the lowest step where gateway p95
  exceeds twice the 5 rps p95, or 5xx rate exceeds 1%, or loadgen p99 lag exceeds
  50 ms. The load rate is set to 50% of it, rounded down to a whole number of
  requests per second; the measured values are logged in DECISIONS.md (D-083).
- **baseline_noise.py**: 60 s warm-up (not scored), then 10 min at the chosen rate.
  Records `vmstat 1` with timestamps, `docker stats` every 2 s, the loadgen log and a
  Jaeger export. Writes `out/baseline/<timestamp>/report.md` and `summary.json` with
  the numbers listed in the PLAN.md done test. Proposed: 3 sessions at different
  times of day; one is required for the done test (D-078).

## Server setup (before step 9)

Owner or later session, on the server:
- Non-root user with SSH key login; SSH is the only open port (firewall).
- Docker Engine and the compose plugin from Docker's repository; user in the docker
  group.
- uv and gitleaks; `procps` for `vmstat`.
- Clone the repo; `git config core.hooksPath .githooks`.
- UIs through an SSH tunnel, for example
  `ssh -L 16686:127.0.0.1:16686 -L 9090:127.0.0.1:9090 <user>@<server>`.
- Note: Docker writes its own firewall rules for published ports, so the firewall
  alone does not protect them; binding every port to 127.0.0.1 does. Checked with
  `ss -tlnp`.
