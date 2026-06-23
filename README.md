# Joingonka Throughput Repro

Minimal provider repro for OpenAI-compatible chat throughput.

This is not the production eval harness and does not include gold labels,
campaign data, user data, or API keys. `payloads.jsonl` contains 1000 sanitized
public Reddit raw posts sampled outside the gold eval set. The prompt is a
hardcoded generic routing prompt, separate from the production lead-judge
prompt.

## Run

```bash
cd provider-repros/joingonka-throughput-repro
GONKA_API_KEY='...' node repro.mjs
```

Defaults:

```text
BASE_URL=https://gate.joingonka.ai/v1
MODEL=MiniMaxAI/MiniMax-M2.7
TARGET_RPS=2
TIMEOUT_MS=90000
MAX_TOKENS=1024
LIMIT=1000
PAYLOADS=./payloads.jsonl
OUT_DIR=./repro-runs/joingonka-throughput
```

Override example:

```bash
GONKA_API_KEY='...' \
BASE_URL='https://gate.joingonka.ai/v1' \
MODEL='MiniMaxAI/MiniMax-M2.7' \
TARGET_RPS=2 \
TIMEOUT_MS=90000 \
node repro.mjs
```

Model-mix example:

```bash
GONKA_API_KEY='...' \
MODELS='MiniMaxAI/MiniMax-M2.7,Qwen/Qwen3-235B-A22B-Instruct-2507-FP8,moonshotai/Kimi-K2.6' \
TARGET_RPS=2 \
node repro.mjs
```

Multiple keys can be provided with `GONKA_API_KEYS`, separated by commas,
spaces, semicolons, or newlines.

## Pass Criteria

Each request is usable only when all of these are true:

- HTTP status is `200`.
- Response body is valid OpenAI-compatible JSON.
- `choices[0].message.content` parses as JSON.
- Parsed content has `decision`.
- `decision` is one of `workplace`, `technical`, `go_to_market`, `other`.
- Parsed content has `confidence`.
- `confidence` is one of `low`, `medium`, `high`.
- Parsed content has a non-empty string `reason`.

Failures are classified as:

- `rate_limit`: HTTP `429`, or rate-limit/upstream-slot language in the body.
- `timeout`: local client timeout.
- `server_error`: HTTP `5xx`.
- `http_error`: other non-200 HTTP response.
- `invalid_response_json`: HTTP 200 but the OpenAI-compatible response body is
  not JSON.
- `malformed_output`: HTTP 200 and valid response body, but model content is
  missing required fields or is not valid JSON.
- `transport_error`: local fetch/network error.

## Outputs

The script writes:

```text
repro-runs/joingonka-throughput/attempts.jsonl
repro-runs/joingonka-throughput/summary.json
```

`summary.json` includes:

- sent / ok / failed
- failures by status
- failures by failure kind
- failures by model when `MODELS` is used
- all-attempt latency p50/p95/p99/max
- usable-200 latency p50/p95/p99/max
- actual start rate
- end-to-end completion rate

No API key is written to any output.
