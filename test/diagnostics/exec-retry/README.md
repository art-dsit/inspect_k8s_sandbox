# exec-retry diagnostic

E2E test for `K8sSandboxEnvironment.exec()` retry logic under real network
faults, using nftables to block the K8s API server.

## Prerequisites

- Linux with minikube running (Docker driver)
- `nft` available (nftables package)
- Passwordless `sudo`
- This repo cloned with `uv sync --extra dev`

Designed for AISI dev VMs. Will not work on macOS or in CI.

## What it tests

1. **Transient fault**: blocks the API server for 5s, then unblocks. With
   tenacity retry, exec should recover on the next attempt. Without retry
   logic, exec fails — this is expected on main before the retry PR lands.

2. **Sustained fault**: blocks the API server for the entire exec attempt.
   Expects `K8sError` (with or without retry logic).

3. **Recovery**: confirms the sandbox still works after faults are cleared.

## How it works

nftables REJECT rules on the OUTPUT chain cause immediate
`ConnectionRefusedError` for any TCP connection to the API server IP:port.
With urllib3's built-in retry disabled (see `_kubernetes_api.py`), this error
propagates immediately to the caller. Tenacity (if present) retries the exec;
without tenacity, the error surfaces as `K8sError`.

## Running

```bash
sudo -n true  # verify passwordless sudo works
uv run python test/diagnostics/exec-retry/run.py
```

Expected results:
- **With exec retry PR**: all 3 tests PASS
- **Without exec retry PR (main)**: transient FAIL, sustained PASS, recovery PASS
