# EVM RPC DoS Probing

Probe an EVM JSON-RPC endpoint for denial-of-service weaknesses: missing or weak rate limits,
expensive methods with no caps, and resource-exhaustion vectors. Public RPC endpoints (chain-run,
infra-provider, or dApp-bundled) are a classic Immunefi category — taking down a chain's public RPC
or exhausting its nodes is a valid, often high-severity finding when the endpoint is in scope.

## When to Use

- A bug bounty scope includes an RPC endpoint, node API, or "JSON-RPC" / "RPC" target (L1/L2 chains,
  Cronos, Sei EVM, BSC nodes, public endpoints).
- A dApp bundles its own RPC endpoint (check its rate limiting and method allowlist).
- After a `chain-fork-diff-audit` finds removed DoS guards (tx pool limits, log-range caps, RPC
  allowlists) — this skill verifies them LIVE.

## When NOT to Use

- Endpoints NOT in scope. Never flood a third-party endpoint (Infura, Alchemy, public community
  RPCs) that is not the bounty target — that is an attack on someone else's infrastructure.
- Mainnet endpoints of a chain whose program scope is contracts-only — check scope first.
- GraphQL endpoints — use `graphql-audit` (batching/complexity DoS there).

## Rules of Engagement (read first)

1. **Scope check is mandatory**: confirm the RPC URL / domain is explicitly in scope before any
   probing. If scope is ambiguous, ask / treat as out of scope.
2. **Start gentle, escalate slowly**: begin with single requests and timing measurements. Only
   increase load stepwise, and stop the moment you observe degradation.
3. **Never take down a production endpoint**: the goal is to MEASURE the limit and PROVE the
   resource cost, not to sustain an outage. Use the smallest load that demonstrates the issue,
   prefer testnet/devnet endpoints when the program provides them, and keep concurrent connections
   low (start at 1–5, cap at ~20 unless the program explicitly allows load testing).
4. **No destructive methods**: never send state-changing transactions, never spam the mempool with
   real transactions, never call admin/debug methods that mutate node state. Read-only methods only.
5. **Record everything**: method, params, request count, concurrency, latency, error codes, and
   timestamps — the report needs exact reproduction steps and measured impact.

## Phase 1: Fingerprint the Endpoint

```bash
# What client + version? (may be disabled — a disabled web3_clientVersion is itself a note)
curl -s -X POST <rpc-url> -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"web3_clientVersion","params":[],"id":1}'
# Chain ID and block height (baseline for timing)
curl -s -X POST <rpc-url> -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}'
# Which namespaces are exposed? Probe one method per namespace and record which answer.
```

Probe method availability (one request each, read-only): `eth_*`, `net_*`, `web3_*`,
`debug_*`, `trace_*`, `txpool_*`, `admin_*`, `engine_*`, `erigon_*`, `ots_*`, `eth_subscribe`.
A node exposing `debug_*` / `trace_*` / `admin_*` / `txpool_*` to the public internet is a
finding in itself (information disclosure + expensive-method surface) — record it before any
load test.

## Phase 2: Rate-Limit Measurement

1. Send sequential identical requests (e.g. `eth_blockNumber`) and measure latency + HTTP status.
2. Ramp concurrency stepwise: 1 → 5 → 10 → 20 parallel clients, short bursts (5–10s each).
3. Record: requests/second sustained before 429 / 503 / connection reset / timeout; whether the
   limit is per-IP, per-method, or global; whether limits reset (Retry-After header, window).
4. Test bypasses (gently, one at a time): different `User-Agent`, `X-Forwarded-For` header,
   HTTP vs WebSocket transport, batching many calls into ONE request (see Phase 3).
5. Finding threshold: no rate limit at all on a public endpoint, or a limit so high (e.g. >100
   req/s/IP sustained with no degradation) that a single client can degrade the node, is reportable.

## Phase 3: Expensive-Method Probing (one request at a time, measure cost)

For each method below, send ONE request with moderately expensive parameters, measure response
time and size, and stop if the node slows. Do NOT loop these.

| Method | Expensive parameters | What it proves |
|---|---|---|
| `eth_getLogs` | Wide block range (`fromBlock: 0` or 100k+ blocks), no address/topic filter | Unbounded log scan = disk/CPU exhaustion; check for range caps |
| `eth_getBlockByNumber` / `eth_getBlockByHash` | `full tx objects = true` on a full block | Large response amplification |
| `eth_call` | Contract call with huge return data / heavy computation; large `gas` field | Unbounded EVM execution via RPC |
| `eth_estimateGas` | Complex tx / large calldata | Same as above |
| `eth_getTransactionByBlockHashAndIndex` + loops | Sequential index scan | Enumeration cost |
| `debug_traceTransaction` / `debug_traceCall` | Any tx, `tracer: callTracer` / `prestateTracer` | Tracing is the most CPU-expensive RPC; public exposure is high-risk |
| `trace_filter` / `trace_block` | Wide block range | Full-trace generation cost |
| `eth_feeHistory` | Large `blockCount` (e.g. 1024) | CPU + memory |
| `eth_getProof` | Deep storage proof | Disk I/O |
| Batch JSON-RPC | Single HTTP request with 100–1000 calls (e.g. 500× `eth_blockNumber`) | Batching bypasses per-request limits; measures batch cap |
| `eth_subscribe` (WS) | Many subscriptions (`newHeads`, `logs` with broad filter) | Subscription exhaustion / memory |
| `eth_sendRawTransaction` | — | DO NOT USE for load testing (state-changing). Only note if the endpoint accepts unsigned/spam txs without any guard, and prove it with a single 0-value self-transfer on testnet if permitted |

For each: record baseline latency (cheap call) vs expensive-call latency, response size, and
whether the node enforces a cap (error message, truncated result, max range).

## Phase 4: Resource-Exhaustion Confirmation

A finding is VALID only if you can show measurable degradation or an uncapped cost:

- **Latency amplification**: expensive call takes >10× baseline, or >30s, with no cap.
- **Throughput collapse**: during a small burst (≤20 concurrent), baseline `eth_blockNumber`
  latency rises sharply or requests start timing out — the node is degraded by your (small) load.
- **Uncapped batch**: batch of 100+ calls accepted and executed with no limit.
- **Memory/disk**: `debug_trace*` on a heavy tx succeeds publicly (CPU exhaustion vector).
- **No rate limit + expensive method**: the combination is the finding — either alone may be Low,
  together they are Medium–High (or Critical if a single client can halt the public endpoint).

Always re-test after a cooldown to confirm the effect is reproducible and caused by your probes,
not background load. Include the cooldown re-test in the report.

## Phase 5: Validate Before Reporting

- Pass the 8-Question Gate (`triage-validation`): impact must be real (node degradation, endpoint
  outage, or resource exhaustion), not theoretical ("could be slow").
- Quantify: requests/second needed, latency numbers, response sizes, duration of effect.
- Severity guide: single-client public-RPC outage / node halt = High–Critical; expensive method
  with no cap but rate-limited = Medium; missing rate limit alone with cheap methods only = Low.
- Never submit a report based on a single timed-out request — timeouts happen for many reasons.
  Reproduce at least twice, with controls (baseline latency measured before/after).

## Output

Per finding: endpoint URL (in scope), method + exact params, request count/concurrency used,
baseline vs under-load latency, error codes observed, cooldown re-test result, CVSS 3.1, and
exact reproduction steps (curl commands). Also list: exposed namespaces, rate-limit behavior
(limit value or "none observed"), and batch cap (or "none observed").

## Tooling

- `curl` / `websocat` (or `wscat`) for single requests and WS subscriptions.
- Small shell/Python loop for stepwise concurrency ramps (keep it local, bounded, and stoppable).
- `cast` (Foundry) for `cast rpc` convenience calls.
- No dedicated load-testing tool (vegeta/k6) unless the program explicitly allows load testing —
  if used, cap the rate low and the duration short.
