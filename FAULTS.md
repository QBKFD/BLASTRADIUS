# Faults

Initial fault catalog for the toy system (stage 1 topology, see PLAN.md). Expected
symptoms are hypotheses written before any run. They are checked on dev runs in
stage 2; any difference is written here and marked "revised after dev runs".

RCA methods are not given this file.

Topology (D-044): the gateway orchestrates. Per request it increments a Redis
counter, asks the checker for a verdict, and if allowed calls mock_llm itself. The
checker makes no outgoing calls. Symptoms below follow from that.

## Summary

| ID | Name                 | Difficulty        | Component | Fault type       | Injection                        |
|----|----------------------|-------------------|-----------|------------------|----------------------------------|
| F1 | llm_outage_503       | easy              | mock_llm  | error_responses  | mock_llm admin: 503 on 100%      |
| F2 | redis_down           | easy              | redis     | unavailable      | `docker compose stop redis`      |
| F3 | llm_rate_limit_429   | propagating       | mock_llm  | rate_limited     | mock_llm admin: token bucket     |
| F4 | checker_cpu_pressure | propagating       | checker   | cpu_saturation   | `stress-ng` in checker container |
| F5 | checker_crash        | silent/misleading | checker   | unavailable      | `docker kill -s KILL checker`    |
| F6 | checker_bad_config   | silent/misleading | checker   | misconfiguration | block-all policy file + reload   |

Fault duration: 120 s for every fault (see PLAN.md, experiment protocol).

## Label vocabulary

Components: `gateway`, `checker`, `mock_llm`, `redis`, `none`.

Fault types: `unavailable`, `error_responses`, `rate_limited`, `cpu_saturation`,
`memory_saturation`, `added_latency`, `network_loss`, `misconfiguration`, `none`.

The vocabulary includes types that are never injected in the initial catalog
(`memory_saturation`, `added_latency`, `network_loss`), so a method cannot pick from
exactly the six injected pairs.

## Normal operation (reference)

- Load: open-loop Poisson at a fixed rate (set in stage 1). About 95% benign prompts,
  5% malicious.
- gateway: 200 for allowed prompts, 403 for blocked (about 5%), no 5xx. Its latency
  is Redis + checker + (for allowed prompts) mock_llm.
- checker: receives every request that passes the Redis step; verdict "allow" about
  95%, "block" about 5%; latency about the fixed CPU work (about 5 ms).
- mock_llm: receives about 95% of the gateway rate (allowed prompts only), latency
  from a lognormal (median about 300 ms, set in stage 1).
- redis: one INCR plus EXPIRE per gateway request.
- Errors: 5xx counts are errors; 403 blocks are not errors.

## Ground truth record

Written by the injector into `recordings/<run_id>/ground_truth.json`:

```json
{
  "run_id": "r_<random>",
  "fault_id": "F1",
  "fault_name": "llm_outage_503",
  "difficulty": "easy",
  "component": "mock_llm",
  "fault_type": "error_responses",
  "params": {"status": 503, "rate": 1.0},
  "inject_issued_at": "2026-01-01T00:00:00.000000Z",
  "inject_confirmed_at": "2026-01-01T00:00:00.120000Z",
  "revert_issued_at": "2026-01-01T00:02:00.130000Z",
  "revert_confirmed_at": "2026-01-01T00:02:01.900000Z",
  "injection_verified": true,
  "injector_git_sha": "<sha>"
}
```

- Scored fault window: `[inject_confirmed_at, revert_issued_at]`.
- Incident time given to RCA methods: `inject_confirmed_at`.
- Control runs: `fault_id`, `component` and `fault_type` are `none`; timestamps are the
  placebo incident time.
- `injection_verified` is based only on the injector's own check (listed per fault
  below), never on observed symptoms. A run with `false` is excluded and repeated;
  excluded runs are counted and reported.

## F1 llm_outage_503 (easy)

- **Injection:** `POST /admin/fault` on mock_llm admin port,
  `{"mode": "error", "status": 503, "rate": 1.0}`.
- **Verify:** `GET /admin/fault` returns the same mode.
- **Revert:** `POST /admin/reset`, then `GET /admin/fault` returns `none`.
- **Ground truth:** `mock_llm`, `error_responses`.
- **Expected symptoms:**
  - mock_llm: 503 on all requests; server latency drops (fails fast); request rate
    unchanged (503 is not retried).
  - gateway: mock_llm client spans end with 503; 502 to the client for about 95% of
    requests (all allowed prompts); blocked prompts still get 403; latency drops.
  - checker: unchanged (traffic, verdicts, latency).
  - redis: unchanged.
  - client: about 95% 5xx.
- **Misleading signals:** errors appear at gateway and mock_llm. A "component with
  most errors" or "entry point with errors" rule could pick gateway.

## F2 redis_down (easy)

- **Injection:** `docker compose stop redis`.
- **Verify:** container state `exited`; a Redis PING from the injector fails.
- **Revert:** `docker compose start redis`; PING succeeds; gateway health OK.
- **Ground truth:** `redis`, `unavailable`.
- **Expected symptoms:**
  - redis: no data (no exporter); full view only: its container disappears from
    running containers in `docker stats`.
  - gateway: 503 on every request; Redis client spans end with a connection error
    (through Toxiproxy this is likely a reset or EOF, not "refused"); latency drops to
    about the Redis client timeout (50 ms) or less.
  - checker: request rate falls to about 0 (the Redis step comes first).
  - mock_llm: request rate falls to about 0.
  - client: about 100% 503.
- **Misleading signals:** the only component returning errors is gateway. checker and
  mock_llm traffic drops could be read as a problem there. The correct answer needs
  the Redis client spans or the missing Redis container.

## F3 llm_rate_limit_429 (propagating)

- **Injection:** `POST /admin/fault` on mock_llm,
  `{"mode": "rate_limit", "rps": R, "retry_after_s": 1}`, where R is 30% of the
  normal mock_llm request rate (absolute value fixed after the stage 1 capacity test).
- **Verify:** `GET /admin/fault` returns the same mode and R.
- **Revert:** `POST /admin/reset`, read back `none`.
- **Ground truth:** `mock_llm`, `rate_limited`.
- **Expected symptoms:**
  - mock_llm: mix of 200 and 429; request rate goes up because of gateway retries
    (up to 3 attempts per prompt); latency of successful calls about normal.
  - gateway: latency up by backoff and Retry-After waits (up to about 2 s extra);
    part of the requests return 502 after the last retry; several mock_llm client
    spans per request, most ending in 429.
  - checker: unchanged.
  - redis: unchanged.
  - client: higher tail latency, partial 5xx.
- **Misleading signals:** the extra latency is backoff waiting inside gateway spans,
  between the mock_llm client spans, so exclusive-time analysis may blame the
  gateway. The 429 client spans point to mock_llm.

## F4 checker_cpu_pressure (propagating)

- **Injection:** `docker exec checker stress-ng --cpu 2 --timeout 120s`. The checker
  container has `cpus: 1.0`, so stress-ng competes with the checker inside the same
  CPU quota and does not load the rest of the host beyond one core.
- **Verify:** `docker top checker` lists stress-ng processes.
- **Revert:** stop stress-ng inside the container (`pkill stress-ng`), confirm none
  are left.
- **Ground truth:** `checker`, `cpu_saturation`.
- **Expected symptoms:**
  - checker: server latency up (the fixed CPU work per request takes about 3 times
    longer, plus CPU throttling and queueing); no errors; full view only:
    `docker stats` CPU near the 1.0 limit.
  - gateway: latency up (waits on the checker). If checker latency exceeds the 1 s
    gateway timeout, the gateway fails closed and returns 403 for those requests.
    At the planned load this should affect at most the tail; dev runs will show.
  - mock_llm: request rate about unchanged (slightly lower if some requests are
    blocked by fail closed), latency normal.
  - redis: unchanged.
  - client: higher latency, few or no errors, possibly a small rise in 403.
- **Misleading signals:** gateway latency rises too. Any fail-closed blocks look like
  F5 or F6 on a small scale. One extra busy core may raise host CPU steal and slow
  other containers a little.

## F5 checker_crash (silent/misleading)

- **Injection:** `docker kill -s KILL checker`. Restart policy is `no` during
  experiments, so the container stays down until revert.
- **Verify:** container state `exited`, exit code 137.
- **Revert:** `docker start checker`, wait for its health check.
- **Ground truth:** `checker`, `unavailable`.
- **Expected symptoms:**
  - checker: no spans, its metric series stop; full view only: Prometheus
    `up{job="checker"}` is 0 and the container disappears from running containers in
    `docker stats`.
  - gateway: checker client spans end with a connection error; fail closed, so every
    request gets 403 "blocked"; no 5xx; latency drops (no LLM call).
  - mock_llm: request rate falls to about 0.
  - redis: unchanged.
  - client: about 100% 403, no errors.
- **Misleading signals:** from the client side it looks the same as F6 or as a wave of
  malicious prompts. Error-rate rules see nothing. mock_llm traffic drop could be read
  as a mock_llm problem. Distinguishing signals: gateway client span errors, no
  checker spans, checker series stop (and `up` = 0 in the full view).

## F6 checker_bad_config (silent/misleading)

- **Injection:** atomically replace the mounted checker policy file with
  `policy_block_all.yaml` (threshold set to 0.0 instead of the normal value, so every
  prompt scores at or above it), then `POST /admin/reload` on the checker admin port.
- **Verify:** `GET /admin/config` returns the hash of the block-all policy.
- **Revert:** restore the normal policy file, reload, read back its hash.
- **Ground truth:** `checker`, `misconfiguration`.
- **Expected symptoms:**
  - checker: up and serving; verdict "block" on 100% of prompts; no errors; latency
    unchanged; spans carry the verdict.
  - gateway: 403 on every request; no errors; latency drops (no LLM call).
  - mock_llm: request rate falls to about 0.
  - redis: unchanged.
  - client: about 100% 403, no errors.
- **Misleading signals:** no errors anywhere. mock_llm traffic drop could be read as a
  mock_llm problem. Client view is the same as F5. Distinguishing signals: checker up
  and answering, checker verdict counter for "block" rising, checker spans present
  with verdict "block".
- **Note:** no config version metric or change log is given to RCA methods in the
  initial setup (see DECISIONS.md D-027).
