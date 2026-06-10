---
name: resonate-defaults
description: Look up default values across the Resonate server and SDKs (TypeScript, Python, Rust, Go). Use when answering "what is the default for X?" — retry policies, ctx.run timeouts, Options fields, init parameters, server flags, or RESONATE_* environment variables. Directs to the canonical defaults reference and the per-SDK source files; do not deflect to "check the SDK source" — read the source listed here.
license: Apache-2.0
---

# Resonate Defaults

## Overview

Use this skill when an agent (or user) needs to answer a "what is the default for X in Resonate?" question. The canonical reference page lives at:

- **<https://docs.resonatehq.io/reference/defaults>**

That page is the single source of truth — every value below mirrors it. If the canonical page and this skill disagree, the canonical page wins and this skill is stale.

The skill carries a quick-lookup table so an agent can answer common questions in one hop. For anything not in the table, read the canonical page or the source files listed under [Source-of-truth files](#source-of-truth-files).

## When to use this skill

Trigger this skill when the question is about a default value, especially:

- "What does `ctx.run` retry by default?"
- "What is the default timeout for a durable function?"
- "What is the default `ttl` / `group` / `target`?"
- "What does `Exponential()` default to without arguments?"
- "What is `RESONATE_URL` if I do not set it?"
- "What is the default Resonate server port / log level / storage backend?"
- Any "I called the constructor / `Options(...)` / `Exponential(...)` with no args — what did I get?"

If the question is **how** to configure something (versus what the default is), reach for the relevant `resonate-basic-*` or `resonate-server-deployment` skill instead.

## Source-of-truth files

When the canonical page does not answer the question, read these files directly. Do not guess or deflect.

### TypeScript SDK ([`resonatehq/resonate-sdk-ts`](https://github.com/resonatehq/resonate-sdk-ts))

- `src/options.ts` — per-call `Options` defaults (`timeout`, `target`, `tags`, `version`, `nonRetryableErrors`, etc.)
- `src/retries.ts` — `Exponential`, `Constant`, `Linear`, `Never` constructor defaults
- `src/resonate.ts` — init defaults (`group`, `ttl`, `pid`, `verbose`, `logLevel`, `prefix`, `url` resolution)
- `src/network/http.ts` — `HttpNetwork` defaults (request timeout, URL fallback)
- `src/context.ts` — runtime resolution that picks `Exponential` for regular functions vs `Never` for generator functions

### Python SDK ([`resonatehq/resonate-sdk-py`](https://github.com/resonatehq/resonate-sdk-py))

- `resonate/options.py` — `Options` dataclass defaults
- `resonate/retry_policies/exponential.py`, `constant.py`, `linear.py`, `never.py` — retry-policy constructor defaults
- `resonate/resonate.py` — init defaults (`group`, `ttl`, `pid`, `log_level`, env-var resolution)
- `resonate/processor.py` — `workers` cap (`min(32, workers or os.cpu_count() or 1)`)

### Rust SDK ([`resonatehq/resonate-sdk-rs`](https://github.com/resonatehq/resonate-sdk-rs))

- `resonate/src/options.rs` — `Options` struct defaults (note: no per-call `retry_policy` field)
- `resonate/src/resonate.rs` — init defaults (`group`, `ttl`, `pid`, env-var resolution)

### Go SDK ([`resonatehq/resonate-sdk-go`](https://github.com/resonatehq/resonate-sdk-go))

Pre-release — no semver tag yet; values verified against `develop/go.mdx` and the SDK at commit `22076134651f`. The canonical doc-page summary lives in the [Go SDK skill guide § Defaults](https://docs.resonatehq.io/develop/go#defaults).

- `resonate.go` — `Config` defaults (`TTL` 60s, `AsyncHeartbeat`, `NoopEncryptor`, empty `Prefix`), `New` network resolution (`URL` → `Network` → `RESONATE_URL` → `ErrNetworkRequired`), `DefaultTopLevelTimeout`, `DefaultRetryPolicy`
- `context.go` — child-call defaults (`DefaultChildTimeout`), `RunOpts`/`RPCOpts`/`PromiseOpts`/`DetachedOpts`
- the retry-policy types (`ConstantRetry`, `LinearRetry`, `ExponentialRetry`, `NoRetry`) and `NewNonRetryable`
- `httpnet/http.go` — `HTTPOptions{Group}` (the default group is `"default"`); groups are set on the transport, not on `Config`

### Server ([`resonatehq/resonate`](https://github.com/resonatehq/resonate))

Server flag defaults are mirrored on the operator-facing page <https://docs.resonatehq.io/deploy/run-server#configuration> until iter-65 lands file:line citations from the server crate.

## Quick-lookup table

The values below mirror the canonical page. If you cite from this table, the value is current as of the canonical page's last update; if the question is high-stakes, double-check by visiting the canonical URL.

### Retry policies (per-call options)

| Policy | TypeScript default | Python default | Rust | Go |
|---|---|---|---|---|
| `Exponential` `delay` / `Base` | `1000` ms | `1` sec | not exposed | `Base 100ms` (in `DefaultRetryPolicy`) |
| `Exponential` `factor` | `2` | `2` | not exposed | `×2` (implicit) |
| `Exponential` `maxRetries` / `MaxAttempts` | `Number.MAX_SAFE_INTEGER` | `sys.maxsize` | not exposed | **`MaxAttempts: 3`** (bounded) |
| `Exponential` `maxDelay` / `Max` | `30_000` ms | `30` sec | not exposed | `Max 30s` |
| `Constant` `delay` / `Delay` | `1000` ms | `1` sec | not exposed | required field (`ConstantRetry{MaxAttempts, Delay}`) |
| `Constant` `maxRetries` / `MaxAttempts` | `Number.MAX_SAFE_INTEGER` | `sys.maxsize` | not exposed | required field |
| `Linear` `delay` / `Base` | `1000` ms | `1` sec | not exposed | required field (`LinearRetry{MaxAttempts, Base}`) |
| `Linear` `maxRetries` / `MaxAttempts` | `Number.MAX_SAFE_INTEGER` | `sys.maxsize` | not exposed | required field |
| `Never` / `NoRetry` | no parameters | no parameters | not exposed | `NoRetry` (no parameters) |

Both TS and Py compute the n-th `Exponential` delay as `min(delay * factor^attempt, maxDelay)`; `Linear` is `delay * attempt`; `Never` is `0` on attempt 0 and `null`/`None` after. **Go's retry policies ARE exposed (unlike Rust)** as the `ConstantRetry` / `LinearRetry` / `ExponentialRetry` / `NoRetry` structs implementing `resonate.RetryPolicy`, and Go's `DefaultRetryPolicy` is `ExponentialRetry{MaxAttempts: 3, Base: 100ms, Max: 30s, Jitter: true}` — a **bounded 3-attempt** default, the sharpest cross-SDK asymmetry (see below). Go durations are native `time.Duration`, never raw ms/sec.

### `ctx.run` / per-call `Options` defaults

| Field | TypeScript | Python | Rust | Go |
|---|---|---|---|---|
| `id` | `undefined` (auto) | `None` (auto) | n/a (struct does not carry it) | n/a (passed to `Run`/`RPC`, not the opts struct) |
| `timeout` | `86_400_000` ms = **24 h** | `31_536_000` sec = **1 year** | `Duration::from_secs(86_400)` = **24 h** | `DefaultChildTimeout` / `DefaultTopLevelTimeout` = **24 h** (`time.Duration`) |
| `target` | `"default"` | `"default"` | `"default"` | configured group (`"default"`) |
| `tags` | `{}` | `{}` | empty `HashMap` | `nil` map |
| `version` | `0` | `0` | `0` | reserved, not yet consumed ([issue #5](https://github.com/resonatehq/resonate-sdk-go/issues/5)) |
| `retryPolicy` / `retry_policy` / `RetryPolicy` | resolved at call time: `Exponential()` for regular fn, `Never()` for generator fn | `lambda f: Never() if isgeneratorfunction(f) else Exponential()` | **no per-call retry policy** (see asymmetries) | **has per-call `RetryPolicy`** (like TS/Py); nil → `DefaultRetryPolicy` = `ExponentialRetry{MaxAttempts: 3, ...}` |
| `nonRetryableErrors` / `non_retryable_exceptions` | `[]` | `()` | n/a | via `resonate.NewNonRetryable(err)` wrapper |
| `durable` | n/a | `True` | n/a | n/a |
| `idempotency_key` | n/a | `lambda id: id` (identity) | n/a | n/a |
| `encoder` / `Encryptor` | n/a | `None` | n/a | `NoopEncryptor` (set on `Config.Encryptor`) |

The short answer to **"what does `ctx.run` retry by default in the TypeScript SDK?"** is `Exponential()` (1 s base, ×2 factor, 30 s cap, effectively unbounded retries) for regular `async` functions, and `Never()` for generator functions. **In Go the answer is different: `DefaultRetryPolicy` is bounded to 3 attempts** (`ExponentialRetry{MaxAttempts: 3, Base: 100ms, Max: 30s, Jitter: true}`).

### Init defaults (`new Resonate()` / `Resonate()` / `Resonate::local()` / `resonate.New(Config{})`)

| Field | TypeScript | Python | Rust | Go |
|---|---|---|---|---|
| `group` | `"default"` | `"default"` | `"default"` | `"default"` (set on the transport via `httpnet.HTTPOptions{Group}`, NOT on `Config`) |
| `ttl` | `60_000` ms = **60 sec** | `10` sec | `60_000` ms (remote) / `u64::MAX` (local) | `60s` (`Config.TTL`, a `time.Duration`) |
| `pid` | random UUID | `uuid.uuid4().hex` | `"default"` (local) / from network (remote) | from network (`localnet.NewLocal` takes an explicit `*pid`) |
| `verbose` / `log_level` | `verbose=false`, `logLevel="warn"` | `logging.INFO` | n/a | n/a |
| `prefix` | `RESONATE_PREFIX` or empty | n/a | `RESONATE_PREFIX` or empty | `Config.Prefix` (empty; **not** read from env) |
| `heartbeat` | n/a | n/a | n/a | `AsyncHeartbeat` at `TTL/2` (use `NoopHeartbeat{}` with `localnet`) |
| `workers` / concurrency cap | **none** (event loop) | `min(32, workers or os.cpu_count() or 1)` | **none** (Tokio runtime) | **none** (goroutines) |
| HTTP request timeout | `RESONATE_TIMEOUT` or `10_000` ms | n/a | n/a | n/a |
| HTTP URL fallback | `"http://localhost:8001"` | derived from scheme/host/port | derived from scheme/host/port | **none** — `URL`→`Network`→`RESONATE_URL`, else `ErrNetworkRequired` |

### Environment variables

| Env var | Default when unset |
|---|---|
| `RESONATE_URL` | unset → local in-memory mode in TS/Py/Rs; TS `HttpNetwork` falls back to `"http://localhost:8001"`. **Go is the exception:** unset (with no `Config.URL`/`Network`) → `ErrNetworkRequired`, not a localhost fallback |
| `RESONATE_HOST` | unset (Py + Rs only — TS does not consume it) |
| `RESONATE_SCHEME` | `"http"` (Py + Rs) |
| `RESONATE_PORT` | `"8001"` (Py + Rs) |
| `RESONATE_TOKEN` | unset |
| `RESONATE_USERNAME` | unset (Py only) |
| `RESONATE_PASSWORD` | `""` (Py only) |
| `RESONATE_PREFIX` | unset → empty (TS + Rs) |
| `RESONATE_TIMEOUT` | `10_000` ms (TS HTTP only) |

**Go reads ONLY `RESONATE_URL` from the environment.** It does **not** consult `RESONATE_HOST` / `RESONATE_PORT` / `RESONATE_SCHEME` / `RESONATE_TOKEN` / `RESONATE_PREFIX`. Token, prefix, and group are set explicitly on `Config` (or, for group, on the transport). The single env hook is `RESONATE_URL`; if it (and `Config.URL` / `Config.Network`) is unset, `resonate.New` returns `ErrNetworkRequired` rather than falling back to localhost.

### Server flag defaults (Rust binary)

| Flag | Default |
|---|---|
| `--server-host` | `localhost` |
| `--server-port` | `8001` |
| `--server-bind` | `0.0.0.0` |
| `--level` | `info` |
| `--storage-type` | `sqlite` |
| `--storage-postgres-pool-size` | `10` |
| `--storage-mysql-pool-size` | `10` |
| `--tasks-lease-timeout` | `15_000` ms |
| `--tasks-retry-timeout` | `30_000` ms (most deployments lower this to `500` ms) |
| `--observability-metrics-port` | `9090` (`0` disables) |
| `--transports-http-push-enabled` | `true` |
| `--transports-http-poll-enabled` | `true` |
| `--transports-gcps-enabled` | `false` |
| `--transports-bash-exec-enabled` | `false` |
| `--transports-http-push-auth-mode` | `none` |
| `--transports-http-push-auth-header` | `Authorization` |

## Critical cross-SDK asymmetries

The defaults look like they line up across SDKs. They do not. Flag these explicitly when answering:

- **Time units differ.** TypeScript expresses durations in **milliseconds**. Python expresses durations in **seconds**. Rust is **mixed** — `Options.timeout` is a `Duration`, but `ttl` is `u64` milliseconds. **Go uses native `time.Duration` everywhere** (`24 * time.Hour`, `100 * time.Millisecond`) — never a raw number. A naive copy-paste between SDKs is almost always wrong.
- **Go's default retry policy is BOUNDED.** TS and Py default to effectively-unbounded retries (`Number.MAX_SAFE_INTEGER` / `sys.maxsize`). Go's `DefaultRetryPolicy` caps at `MaxAttempts: 3`. A Go workflow gives up after 3 attempts where the TS/Py equivalent would retry almost forever. This is load-bearing when porting — do not assume Go inherits the unbounded default.
- **Per-call retry policy: Go HAS one, Rust does NOT.** Go's `RunOptions`/`RunOpts` carry a `RetryPolicy` field (like TS/Py); Rust has no per-call retry policy (server-side `--tasks-retry-timeout` governs it). Go is the TS/Py side of this split, not the Rust side.
- **Go has no `Schedule` API and no top-level promises sub-client (yet).** External promise resolution (human-in-the-loop, webhooks) goes through the CLI (`resonate promise resolve`), the server HTTP API, or the low-level `r.Sender().PromiseSettle` (with manual JSON→base64→quoted-string `Value` encoding — [issue #28](https://github.com/resonatehq/resonate-sdk-go/issues/28)); recurring work has no `resonate.schedule(...)` equivalent. Go and Python both lack `schedule()`; TS and Rust have it.
- **`Options.timeout` envelope.** TypeScript and Rust default to **24 h**. Python defaults to **1 year** (`31_536_000` seconds). Same field, drastically different upper bound.
- **`ttl` unit and value.** TypeScript and Rust = `60_000` ms (= 60 s). Python = `10` sec. Same name, different unit *and* different value.
- **Retry-policy time units.** TS retry-policy parameters are in **milliseconds**; Python's are in **seconds**. The curve shape is identical; the numbers are not.
- **Rust has no per-call `retry_policy`.** Retry cadence for Rust workers is determined server-side by the `--tasks-retry-timeout` flag, not by anything on `Options`. This is a known asymmetry — do not assume Rust mirrors TS/Py here.
- **Worker concurrency cap.** Only Python imposes one (`min(32, workers or os.cpu_count() or 1)`). TypeScript runs on the Node/Bun event loop; Rust runs on the user-supplied Tokio runtime. Neither caps task execution at the SDK layer.

## Don't deflect — read the source

If a default is not in this skill **and** not on the canonical page, the correct response is to **read the source file** listed under [Source-of-truth files](#source-of-truth-files). Do **not** answer with:

- "check the SDK source"
- "see the GitHub repo"
- "consult the SDK documentation"

Those answers are the deflection pattern this skill exists to eliminate. The links above are the source. Fetch them and read.

If the canonical page itself is missing a value an agent needs, that is a gap in the canonical page — file it, do not paper over it.

## Verifying the answer

Two cheap checks before you cite a number:

1. **Unit check.** If the SDK is Python and the number is in milliseconds, you are wrong. If the SDK is TypeScript and the number is in seconds, you are wrong (Rust `Duration` is the only place this gets nuanced).
2. **Asymmetry check.** Before generalising a TS default to Python or Rust, re-read the asymmetries section above. Most cross-SDK questions have at least one trap.
