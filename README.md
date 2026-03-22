# httpx_persistent_probe

`bin/httpx_persistent_probe` is a self-contained reproduction harness for investigating possible `httpx` issues when many same-host requests run concurrently on different fibers through one shared persistent session.

It:

- starts a local `WEBrick` server on `127.0.0.1`
- sends requests through one shared persistent `HTTPX` session for the probe phase
- collects a sequential non-persistent baseline first
- compares each concurrent probe response to the baseline for the same request shape
- logs both client-side and server-side events to JSONL

The server is designed to always return the correct deterministic response for the request it received. It does not intentionally fake mismatched responses.

## Why This Exists

The goal is to answer questions like:

- can a shared persistent `HTTPX` session used across many fibers mis-associate responses?
- do certain stress factors change the failure mode?

## Requirements

Required gems/libraries:

- `async`
- `httpx`
- `webrick`

The script itself uses only standard library code plus those gems.

## What It Sends

The probe uses three deterministic request templates:

- `probe_template_a`
- `probe_template_b`
- `probe_template_c`

Default case names:

- `email`
- `facebook`
- `group_location`
- `instagram`
- `soundcloud`
- `weather`

For each `(template_name, case_name)` pair, the script:

1. sends a sequential baseline request with a non-persistent session
2. records the normalized response digest and response fingerprint
3. later sends the same request shape concurrently during the persistent-session probe

Each request body is a small JSON object with:

- `template_name`
- `case_name`

By default, total request count is:

`3 request templates x 6 cases x iterations`

So `--iterations 100` produces `1800` probe requests.

## Stress Profiles

The default profile is `baseline`.

Available profiles:

- `baseline`
- `pool10`
- `cpu_pool10`
- `large_pool10`
- `stream_pool10`
- `cpu_stream_pool10`
- `cpu_stream_pool10_gzip`
- `cpu_stream_pool10_close`

High-level intent of each profile:

- `baseline`: no additional stress factors
- `pool10`: limits server handler concurrency to roughly 10 workers
- `cpu_pool10`: adds CPU-heavy response generation plus a small padded body
- `large_pool10`: adds larger deterministic response bodies
- `stream_pool10`: streams responses in multiple chunks with small delays
- `cpu_stream_pool10`: combines CPU burn, padding, and chunked streaming
- `cpu_stream_pool10_gzip`: adds gzip on top of the CPU+streaming profile
- `cpu_stream_pool10_close`: adds keep-alive connection churn on top of the CPU+streaming profile

Profiles only change timing, buffering, framing, compression, and connection lifecycle. They do not intentionally return incorrect payloads.

## Key Options

- `--profile NAME`
- `--cases a,b,c`
- `--iterations N`
- `--jsonl PATH`
  Overwrites the target file with the current run's JSONL output. When `--repeat > 1`, each run gets its own derived file such as `PATH.run-01.jsonl`, `PATH.run-02.jsonl`, and so on.
- `--httpx-debug-log PATH`
  Writes `httpx` debug output to PATH. When `--repeat > 1`, each run gets its own derived file such as `PATH.run-01.log` or `PATH.run-01.<ext>` using the same naming convention as `--jsonl`.
- `--httpx-debug-level N`
  Sets the `httpx` `debug_level` for both the baseline and persistent probe sessions. The default is `1`.
- `--worker-limit N`
- `--cpu-burn-ms MIN:MAX`
- `--response-padding-bytes N`
- `--chunk-count N`
- `--first-byte-delay-ms MIN:MAX`
- `--inter-chunk-delay-ms MIN:MAX`
- `--[no-]gzip`
- `--close-connection-rate FLOAT`
- `--cohort-size N`
- `--cohort-release-delay-ms MIN:MAX`

The profile supplies defaults for many of these, and individual flags can override them.

## Example Usage

Run a minimal sanity check:

```bash
bin/httpx_persistent_probe \
  --profile baseline \
  --iterations 1 \
  --jsonl tmp/httpx_persistent_probe.jsonl
```

Run the worker-limited profile that has produced response fingerprint mismatches:

```bash
bin/httpx_persistent_probe \
  --profile pool10 \
  --iterations 100 \
  --jsonl tmp/httpx_persistent_probe.jsonl
```

Run a heavier streaming profile:

```bash
bin/httpx_persistent_probe \
  --profile cpu_stream_pool10 \
  --iterations 100 \
  --jsonl tmp/httpx_persistent_probe.jsonl
```

Capture both JSONL events and `httpx` debug output:

```bash
bin/httpx_persistent_probe \
  --profile baseline \
  --iterations 30 \
  --client-burst-size 5 \
  --client-burst-gap-ms 0:0 \
  --jsonl tmp/httpx_persistent_probe.jsonl \
  --httpx-debug-log tmp/httpx_persistent_probe.log \
  --httpx-debug-level 2
```

Override individual stress factors:

```bash
bin/httpx_persistent_probe \
  --profile pool10 \
  --worker-limit 10 \
  --chunk-count 8 \
  --first-byte-delay-ms 5:20 \
  --inter-chunk-delay-ms 5:20 \
  --response-padding-bytes 65536 \
  --cohort-size 10 \
  --cohort-release-delay-ms 5:15 \
  --iterations 100 \
  --jsonl tmp/httpx_persistent_probe.jsonl
```

## Output

The script prints a compact stdout summary and can also write structured JSONL events.

Client-side JSONL events:

- `baseline_result`
- `probe_start`
- `probe_result`
- `probe_error`
- `summary`

Server-side JSONL events:

- `server_request_received`
- `server_response_started`
- `server_response_finished`

Important client-side fields:

- `script_run_id`
- `session_object_id`
- `async_task_object_id`
- `async_task_fiber_object_id`
- `fiber_object_id`
- `httpx_request_object_id`
- `request_label`
- `request_template`
- `case_name`
- `iteration`
- `status`
- `duration_ms`
- `peer_address`
- `response_fingerprint`
- `expected_response_fingerprint`
- `normalized_response_sha256`
- `normalized_response_prefix`
- `expected_request_key`
- `error_class`
- `error_message`
- `error_response_body`
- `matches_baseline`

Important server-side fields:

- `server_request_id`
- `server_connection_id`
- `server_worker_slot`
- `server_profile`
- `server_response_fingerprint`
- `server_response_sha256`
- `server_response_bytes`
- `server_wire_bytes`
- `server_chunk_count`
- `server_cpu_burn_ms`
- `server_first_byte_delay_ms`
- `server_inter_chunk_delay_ms`
- `server_closed_connection`

The `summary` event also includes:

- `failure_reasons`
- `error_breakdown`
- `unique_async_task_id_count`
- `unique_async_task_fiber_id_count`
- `unique_fiber_id_count`
- `unique_session_id_count`
- `unique_server_connection_id_count`
- `unique_server_worker_slot_count`
- `replayed_request_label_count`
- `replayed_request_total`
- `replayed_requests`
- `mismatch_count`
- `fingerprint_mismatch_count`
- `baseline_mismatch_count`
- `error_count`
- `duplicate_digest_count`
- `exit_code`

The JSONL `summary` event keeps the raw diagnostic counters. Stdout is more compact and emphasizes the fields that usually vary run to run:

- `probe_requests=actual/expected`
- `baseline_requests=actual/expected`
- `concurrency=N` when the task/fiber counts agree, otherwise `concurrency=mixed`
- `shared_session=yes|no`
- `server_connections=N`
- `mismatches=TOTAL` with subtype counts when non-zero
- `errors=N`
- `replays=LABELS(+EXTRA_REQUESTS)` when the server saw repeated probe request labels
- `log_file=PATH` so the stdout summary tells you which JSONL file belongs to that run
- `httpx_log_file=PATH` when `--httpx-debug-log` is enabled for that run

## Exit Codes

- `0`: all probe responses matched their baseline and no errors were recorded
- non-zero: one or more baseline errors, probe errors, response mismatches, or concurrency-shape checks failed

When the run exits non-zero, stdout includes:

- a `summary ...` line
- conditional detail lines only when an invariant is violated, such as mismatched request counts or mixed concurrency counts
- a `failure_reasons: ...` line when relevant
- an `error_breakdown:` section when errors were recorded

When `--repeat > 1`, there is no combined JSONL file. Each run writes its own derived JSONL file and, if enabled, its own derived `httpx` debug log file. The corresponding stdout summary line includes `log_file=...` and `httpx_log_file=...` paths for that run.

## How To Interpret A Mismatch

The strongest signal is a `probe_error` or `probe_result` where:

- `matches_baseline` is false, or
- `response_fingerprint` does not equal `expected_response_fingerprint`

In current output:

- `mismatch_count` is the total mismatch evidence count
- `fingerprint_mismatch_count` counts `ProbeError` events where the returned body identifies itself as belonging to a different request
- `baseline_mismatch_count` counts successful `probe_result` events that still failed the baseline SHA comparison
- `replayed_request_label_count` counts probe request labels that the server saw more than once in a single run
- `replayed_requests` lists those repeated labels and the server connection ids they arrived on

If that happens, compare the client-side event with nearby `server_request_received` and `server_response_finished` events for the same time window and connection.

If the server events show it emitted the correct deterministic body for request A, but the client event for request B reports request A’s fingerprint, that is strong evidence of client-side response misassociation rather than a server bug.

## Known Useful Failure Modes

In practice, this harness has surfaced several distinct classes of failures under stress:

- response fingerprint mismatches
- `IOError: stream closed in another thread`
- `HTTPX::Parser::Error "wrong head line format"`
- `HTTPX::ReadTimeoutError "Timed out after 60 seconds"`

Even when a run does not produce a fingerprint mismatch, those other failures can still be useful because they show instability in the same shared persistent-session + multi-fiber test shape.

## Suggested Workflow

1. Start with `--profile baseline --iterations 1`.
2. Increase to `--iterations 10`.
3. Try `--profile pool10 --iterations 100`.
4. If needed, move on to `cpu_pool10`, `large_pool10`, `stream_pool10`, `cpu_stream_pool10`, `cpu_stream_pool10_gzip`, and `cpu_stream_pool10_close`.
5. Always keep `--jsonl` enabled when collecting artifacts for analysis or sharing.
