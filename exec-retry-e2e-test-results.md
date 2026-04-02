# exec retry e2e test results

Investigation of [PR #176](https://github.com/UKGovernmentBEIS/inspect_k8s_sandbox/pull/176)
(METR's provider-level exec retry for transient K8s errors).

## What was tested

The e2e diagnostic (`test/diagnostics/exec-retry/run.py`) uses nftables on a
Linux host running minikube (Docker driver) to inject real network faults
during `K8sSandboxEnvironment.exec()` calls. It tests three scenarios:

1. **Transient fault**: start a long-running exec (`sleep 30 && echo hello`),
   wait 10s for the websocket to establish, block the K8s API server for 5s
   using nftables REJECT + `ss --kill` (to immediately kill the established
   connection), then unblock. Expect exec to retry and succeed.
2. **Sustained fault**: same setup but the block is never lifted. Expect
   `K8sError` after retries are exhausted (or immediately, without retry).
3. **Recovery**: after faults are cleared, a simple `exec(["echo", "recovered"])`
   should succeed, confirming no lasting damage.

The nftables rules target only the API server IP:port (extracted from
kubeconfig). REJECT on OUTPUT sends RST/ICMP for immediate errors; DROP on
INPUT silently discards server responses. `ss --kill` force-closes established
TCP connections so the websocket dies immediately rather than waiting for the
next send/recv.

## Configurations tested

Four configurations were tested, each in a separate git worktree with
`uv sync --extra dev`:

### Config 1: unmodified main (commit 18a8584)

No retry logic, no `_disable_urllib3_retry`, old execute.py behaviour (silently
swallows `BrokenPipeError`). Only the test files were copied in.

### Config 2: main + `_disable_urllib3_retry` + execute.py PodError change

The test files plus two source changes from this branch:
- `_kubernetes_api.py`: `_disable_urllib3_retry()` sets
  `Configuration.retries = False` to disable urllib3's built-in retry
- `_pod/execute.py`: `BrokenPipeError`/`ConnectionResetError` raises `PodError`
  instead of silently breaking out of the read loop

No tenacity retry.

### Config 3: METR's PR only (commit 70861b1)

Exactly commit 70861b1 from PR #176 with only the test files copied in. This
has tenacity retry on exec and the execute.py PodError change, but NOT
`_disable_urllib3_retry`. urllib3's built-in retry (3 attempts) is still active.

### Config 4: full branch (METR's PR + commit b4a53ec)

Everything: tenacity retry, execute.py change, `_disable_urllib3_retry`,
`NewConnectionError` added to `_TRANSIENT_TYPES`, and the e2e test.

## Results

| Config | transient (5s block) | sustained | recovery | notes |
|--------|---------------------|-----------|----------|-------|
| 1: unmodified main | **FAIL** (10.3s) | PASS (10.3s) | PASS | no retry; `WebSocketConnectionClosedException` → `K8sError` immediately |
| 2: main + urllib3 fix + execute.py | **FAIL** (10.3s) | PASS (10.3s) | PASS | same failure — execute.py change not in the code path (see below) |
| 3: METR PR only | **PASS** (45.9s) | PASS (16.2s) | PASS | tenacity retries websocket failure; urllib3 retries REST calls internally |
| 4: full branch | **PASS** (46.2s) | PASS (31.7s) | PASS | tenacity has full control; all 5 retry attempts used in sustained case |

## Analysis

### The retry works

METR's PR (config 3) fixes the transient fault scenario. On unmodified main
(config 1), a killed websocket immediately surfaces as `K8sError`. With the
tenacity retry, the exec retries after the block lifts and succeeds.

### The execute.py change is not exercised by this test

The `BrokenPipeError`/`ConnectionResetError` → `PodError` change in
`execute.py` is not in the code path triggered by `ss --kill` + nftables
REJECT. The error that actually fires is `WebSocketConnectionClosedException`
("Connection to remote host was lost"), which is a `websocket.WebSocketException`
subclass. This bypasses the `BrokenPipeError`/`ConnectionResetError` handler
entirely.

Configs 1 and 2 give identical results, confirming the execute.py change and
`_disable_urllib3_retry` make no difference without tenacity.

### urllib3 retry interacts with tenacity in a subtle way

On METR's PR (config 3, no `_disable_urllib3_retry`), the retry flow during a
fault is:

1. Websocket dies → `WebSocketConnectionClosedException` → tenacity retries
2. On retry, `_check_for_pod_restart()` makes a REST call to
   `read_namespaced_pod()`
3. urllib3 retries this REST call internally (3 attempts, ~1s apart, visible
   in logs as `urllib3.connectionpool Retrying...`)
4. If the block is still active, urllib3 exhausts its retries and raises
   `MaxRetryError`

`MaxRetryError` inherits from `urllib3.exceptions.RequestError` →
`PoolError` → `HTTPError`, none of which are in METR's `_TRANSIENT_TYPES`.
So tenacity treats `MaxRetryError` as a **permanent failure** and stops
retrying.

This means:

- **Config 3 (METR only), sustained test**: tenacity gets one retry after the
  initial websocket failure, the retry hits `MaxRetryError` from urllib3, and
  tenacity gives up. Total: ~16s (10s settle + ~4s urllib3 retry + ~2s
  tenacity backoff).
- **Config 4 (full branch), sustained test**: with `_disable_urllib3_retry`,
  `NewConnectionError` propagates immediately (it IS in `_TRANSIENT_TYPES`),
  so tenacity uses all 5 attempts. Total: ~32s.

For the **transient** test (5s block), this distinction doesn't matter — the
block lifts before urllib3 exhausts its retries, so the REST call on the next
tenacity attempt succeeds either way. But for longer transient faults (say
15-20s), METR's PR alone would likely fail because urllib3 would raise
`MaxRetryError` on the first retry and tenacity would give up. This hasn't been
tested yet.

### `_disable_urllib3_retry` trades off breadth for depth

Disabling urllib3 retry globally (`Configuration.retries = False`) gives
tenacity full control over exec retries, but it also removes urllib3 retry
for **all** kubernetes client calls. Other code paths that are NOT wrapped by
tenacity lose retry protection:

- `Pod.read_file()` / `Pod.write_file()` (websocket + `_check_for_pod_restart`)
- `Release.get_sandbox_pods()` (`list_namespaced_pod`)
- `Release._watch_for_scheduling_events()` (`list_namespaced_event`)

This is a tradeoff worth considering separately.

## How to reproduce

Requirements: Linux with minikube running (Docker driver), `nft` (nftables
package), passwordless sudo, this repo cloned.

```bash
# Verify prerequisites
sudo -n true
which nft
minikube status

# Install dependencies
uv sync --extra dev

# Run the diagnostic
uv run python test/diagnostics/exec-retry/run.py
```

To test a specific configuration, check out the relevant commit/branch and
copy `test/diagnostics/exec-retry/` into the worktree if it doesn't exist
there:

```bash
# Example: test METR's PR only
git worktree add /tmp/metr-test 70861b1
cp -r test/diagnostics/exec-retry /tmp/metr-test/test/diagnostics/exec-retry
cd /tmp/metr-test
uv sync --extra dev
uv run python test/diagnostics/exec-retry/run.py

# Clean up
git worktree remove /tmp/metr-test
```

## Open questions

- Should we test a longer transient fault (15-20s) to confirm the
  `MaxRetryError`-as-permanent-failure gap in METR's PR?
- Should `MaxRetryError` be added to `_TRANSIENT_TYPES` as an alternative to
  (or alongside) `_disable_urllib3_retry`?
- Is the execute.py `BrokenPipeError` → `PodError` change tested elsewhere,
  or does it need its own fault injection scenario (e.g. DROP on established
  connections rather than REJECT)?
