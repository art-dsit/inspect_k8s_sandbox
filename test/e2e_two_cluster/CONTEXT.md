# E2E Two-Cluster Test for PR #177

## What this is

An E2E test that reproduces the exact scenario fixed by [PR #177](https://github.com/UKGovernmentBEIS/inspect_k8s_sandbox/pull/177): running an Inspect eval inside a Kubernetes pod that has **both** in-cluster config and a mounted kubeconfig pointing at a different cluster.

## The bug (PR #177)

In `src/k8s_sandbox/_kubernetes_api.py`, the `_Config._load()` method (line 117 on `main`) tries `load_incluster_config()` first, then falls back to `load_kube_config()`. When running inside a pod with a mounted kubeconfig (METR's deployment pattern), in-cluster config wins and k8s-sandbox targets the **runner** cluster instead of the intended **sandbox** cluster. PR #177 flips the order: kubeconfig first, in-cluster fallback.

## Test design

Two minikube clusters:

- **runner** — minimal cluster, deliberately NOT set up for sandbox workloads (no Cilium, no gVisor, no nfs-csi StorageClass). Runs a pod containing the test eval.
- **sandbox** — fully provisioned (runc RuntimeClass, nfs-csi StorageClass, Cilium). This is where sandbox pods should land.

The test pod in `runner` has:
1. In-cluster config (automatic from service account) → points at runner cluster
2. Mounted kubeconfig (via Secret + `KUBECONFIG` env var) → points at sandbox cluster

**With the PR #177 fix:** `load_kube_config()` succeeds first → eval targets sandbox cluster → succeeds.
**Without the fix:** `load_incluster_config()` succeeds first → Python client talks to runner, Helm uses KUBECONFIG and talks to sandbox → pods not found → fails.

## Files in this directory

### Already created

- `values.yaml` — Helm values override. Uses `runtimeClassName: runc` so we don't need gVisor on either cluster. The test's purpose is config loading, not runtime isolation.
- `test_eval.py` — Self-contained eval script that runs inside the pod. Contains:
  - Inline `MockToolCallModel` (same pattern as `test/k8s_sandbox/inspect_integration/testing_utils/mock_model.py`)
  - Asserts `Config.get_instance().in_cluster is False` (validates kubeconfig was preferred)
  - Runs a minimal Inspect eval: `bash("echo hello")` with `target="hello"` and `match()` scorer
  - Uses `K8sSandboxEnvironmentConfig(values=Path("/app/test/e2e_two_cluster/values.yaml"))` — no explicit context (uses KUBECONFIG default)

### Still need to create

- `Dockerfile` — Test runner image. Design:
  ```dockerfile
  FROM python:3.12-bookworm
  ARG HELM_VERSION=3.17.0
  RUN curl -fsSL "https://get.helm.sh/helm-v${HELM_VERSION}-linux-amd64.tar.gz" | tar xz \
      && mv linux-amd64/helm /usr/local/bin/helm && rm -rf linux-amd64
  COPY . /app
  WORKDIR /app
  RUN pip install --no-cache-dir .
  CMD ["python", "/app/test/e2e_two_cluster/test_eval.py"]
  ```
  Key: installs k8s_sandbox from local source (pulls in inspect-ai as dependency). Helm binary needed because k8s_sandbox shells out to helm.

- `run.sh` — Main orchestration script. Steps:
  1. **Prereqs check** — verify `minikube`, `docker`, `kubectl`, `cilium` are on PATH
  2. **Create runner cluster** — `minikube start -p runner --driver=docker --cni=bridge --container-runtime=containerd --memory=2g`
  3. **Create sandbox cluster** — `minikube start -p sandbox --driver=docker --cni=bridge --container-runtime=containerd --memory=4g` then apply runc RuntimeClass, nfs-csi StorageClass, install Cilium
  4. **Generate standalone kubeconfig for sandbox** — `kubectl config view --flatten --minify --context sandbox` then sed the server URL to use `$(minikube ip -p sandbox):8443` (Docker-network-reachable IP instead of localhost forwarded port)
  5. **Build test image** — `docker build -t e2e-runner:latest -f test/e2e_two_cluster/Dockerfile .` from repo root
  6. **Load image into runner** — `minikube -p runner image load e2e-runner:latest`
  7. **Create Secret in runner** — `kubectl --context runner create secret generic sandbox-kubeconfig --from-file=config=/tmp/sandbox-kubeconfig`
  8. **Run Job in runner** — apply Job manifest (heredoc) that mounts kubeconfig at `/home/appuser/.kube/config`, sets `KUBECONFIG` and `INSPECT_HELM_TIMEOUT=120` env vars, runs `python /app/test/e2e_two_cluster/test_eval.py`, uses `imagePullPolicy: Never`
  9. **Wait for Job** — poll for complete/failed, timeout 5 minutes
  10. **Print logs** and report pass/fail
  11. **Cleanup** — `minikube delete -p runner && minikube delete -p sandbox` (trap on EXIT)

## Key architecture decisions

- **No gVisor on either cluster** — irrelevant to what we're testing, values.yaml overrides to runc
- **Cilium on sandbox only** — required because the agent-env chart creates `CiliumNetworkPolicy` CRDs (`cilium.io/v2` in `src/k8s_sandbox/resources/helm/agent-env/templates/network-policy.yaml`)
- **`KUBECONFIG` env var** — set in the pod so both the Python kubernetes library and the Helm subprocess read the same config
- **No explicit `context` in K8sSandboxEnvironmentConfig** — the mounted kubeconfig has a single context which becomes the default. This matches METR's real deployment pattern.

## Networking

minikube `--driver=docker` runs each cluster as a Docker container. Both profiles share the `minikube` Docker network. Pods in runner reach sandbox's API server because pod traffic to non-cluster IPs is NATed through the node (runner Docker container), which is on the same Docker network as the sandbox container.

The generated kubeconfig rewrites the server URL from `https://127.0.0.1:<forwarded-port>` to `https://<sandbox-container-ip>:8443`.

**Potential issue:** If minikube creates separate Docker networks per profile, the containers won't be able to reach each other. Fix: either use `--network=<shared-name>` on both `minikube start` commands, or `docker network connect` after creation.

## Relevant source code

- `src/k8s_sandbox/_kubernetes_api.py:117` — `_Config._load()` method (the code PR #177 changes)
- `src/k8s_sandbox/_sandbox_environment.py:180` — `sample_init()` flow
- `src/k8s_sandbox/_sandbox_environment.py:511` — `_create_release()` passes `config.context` to Release
- `src/k8s_sandbox/_helm.py:170` — `Release.__init__()` stores context_name, calls `get_default_namespace(context_name)`
- `src/k8s_sandbox/_helm.py:606` — `_kubeconfig_context_args()` adds `--kube-context` to helm commands
- `src/k8s_sandbox/resources/helm/agent-env/templates/network-policy.yaml` — CiliumNetworkPolicy CRDs (why we need Cilium)
- `src/k8s_sandbox/resources/helm/agent-env/values.yaml:43` — default `runtimeClassName: gvisor` (why we override)
- `test/k8s_sandbox/inspect_integration/testing_utils/mock_model.py` — the mock model pattern test_eval.py is based on
- `test/k8s_sandbox/inspect_integration/testing_utils/utils.py` — `create_task()` and `run_and_verify_inspect_eval()` patterns
- `.devcontainer/post-create.sh` — existing minikube + Cilium setup (reference for cluster provisioning)
