# Validation Contract

Validation has three layers:

- This repository checks desired-state rendering and selected configuration contracts.
- The Argo apps repository checks environment composition and source wiring.
- Live kind workflows check controller convergence and runtime behavior.

No one layer replaces the others.

## Entry Points

The full suite requires a GNU/Linux environment, `kubectl`, and a running Docker daemon in addition to the tools installed by mise.

Install pinned tools and run the full platform suite:

```sh
mise install
mise run validate:all
```

Run one component through its stable mise task:

```sh
mise run validate:alloy
mise run validate:grafana
mise run validate:loki
mise run validate:tempo
```

All targeted tasks select components from the same locked PEP 723 script, [`scripts/validate.py`](../scripts/validate.py).

The direct script interface supports exact names and quoted wildcards:

```sh
uv run --locked --script scripts/validate.py --list
uv run --locked --script scripts/validate.py --list 'istio-*'
uv run --locked --script scripts/validate.py --verbose tempo
uv run --locked --script scripts/validate.py --keep-workdir tempo
```

With no selector, the script runs every component. An unmatched selector is an error.

## Static Suite

The suite covers these groups:

| Group | Main Contracts |
| --- | --- |
| Scripts | Shell syntax and behavioral tests for the hosts-file manager |
| Argo bootstrap/config | Kustomize rendering for empty-cluster bootstrap, repository Secrets, and steady-state configuration |
| Wrapper charts | Dependency-lock build, Helm rendering, selected linting, and chart-specific resource assertions |
| Mimir | Kustomize rendering, storage/TLS/gateway contracts, and native Mimir and Envoy configuration validation |
| Alloy | Helm rendering, mesh policy, OTLP, metrics, traces, logs, certificates, retry bounds, and native Alloy validation |
| Application | Kustomize rendering plus image and bounded OpenTelemetry environment checks |
| Plain manifests | Kustomize rendering for cert-manager configuration and both Gateway API components |

Tempo, Loki, and Grafana wrappers also run `helm lint`. Mimir, Tempo, Loki, Alloy, and Envoy use Docker-backed native validators where applicable.

Rendering proves local syntax and selected source contracts. It does not perform API-server admission, webhook execution, scheduling, image pulls, or live network handshakes.

## Wrapper Charts And Locks

Every wrapper must commit `Chart.lock`. Normal validation:

1. Copies the wrapper to `.validation/validate-<random>/<component>/chart`.
2. Excludes generated `charts/` directories.
3. Creates isolated Helm repository configuration and caches.
4. Runs `helm dependency build` against the committed lock.
5. Renders and validates the staged chart.

This process does not vendor downloaded archives or intentionally modify source charts.

When changing a wrapper dependency, refresh the lock deliberately:

```sh
helm dependency update platform/observability/tempo
mise run validate:tempo
mise run validate:all
```

Commit `Chart.lock`. Do not commit `platform/**/charts/`.

Grafana is an umbrella wrapper. Its primary dependency and its independently pinned PostgreSQL dependency remain in one chart and lock so resource-level waves can order database readiness before Grafana.

## Validation Workspaces

Each script invocation creates one ignored workspace under:

```text
.validation/validate-<random>/
```

Selected components share the workspace and isolated Helm cache. Artifacts may include staged charts, rendered YAML, extracted native configurations, and short-lived certificate fixtures used for parser validation.

Successful runs remove the workspace. `--keep-workdir`, test failures, and unhandled errors preserve it and print its path.

## Python Dependency Lock

The PEP 723 metadata and lock are:

```text
scripts/validate.py
scripts/validate.py.lock
```

Refresh the lock only when the script's dependency metadata changes:

```sh
uv lock --script scripts/validate.py
```

Keep `mise.toml` as thin orchestration. Add platform contract checks to `scripts/validate.py`, not as inline Python, Ruby, or large shell assertions in mise tasks.

## Argo Wiring Validation

The adjacent Argo repository owns roots, child ownership, source paths, releases, waves, projects, destinations, and sync policies.

After changing that wiring, run:

```sh
mise run validate:all
```

from `../k8s-playground-argocd-apps`.

Its root validator checks the exact three kind roots and unique child ownership. Targeted validators check component source paths and render the referenced platform configuration.

The sibling suite currently runs `helm dependency update` for some wrappers directly against the platform working tree. After running it, inspect this repository for unintended `Chart.lock` changes. Ignored `charts/` directories may also be populated.

## Fresh-Cluster Acceptance

Run:

```sh
mise run cluster:create
```

A successful run combines local lifecycle code with published Argo-managed desired state and proves that they can:

- Create the kind topology.
- Install and configure MetalLB.
- Install Argo CD and create bootstrap Secrets.
- Apply all three roots.
- Reach `Synced` and `Healthy` for the expected roots and children.
- Roll out Istio, Istio CNI, and both gateway data planes.
- Obtain gateway addresses and accepted Gateway API status.
- Route one HTTPS request through the user gateway to the application.

The smoke test expects:

```text
Howdy from k8s-playground-service
```

It uses `curl --insecure`, so it does not prove workstation trust in the local CA. The hosts-file check is advisory during cluster creation.

The local `mise.toml`, scripts, kind configuration, and bootstrap resources come from the working tree. The root Applications track the Argo repository's `HEAD`, most platform children track `main`, and externally sourced children may pin release revisions. A fresh cluster can therefore test local bootstrap edits, but it does not test unpushed changes to Argo-managed desired state.

## Automated Live Checks

These commands encode live pass/fail behavior:

| Command | Evidence |
| --- | --- |
| `mise run gateway:smoke-test` | Positive HTTPS application route and exact response |
| `mise run istio:wait` | Istio CRDs, control plane, CNI, ingress gateway, and ingress address readiness |
| `mise run gateway:wait` | User certificate, Gateway, and application route status |
| `mise run management-gateway:wait` | Management gateway, certificate, and Argo/Grafana route status |
| `mise run argocd:bootstrap-roots` | Expected roots and children become `Synced` and `Healthy` |
| `mise run argocd:sync-app --app grafana` | One requested sync completes and becomes healthy |
| `mise run kind:repair-mesh` | Near-expiry mesh certificates are repaired and the gateway path recovers |
| `mise run mimir:renew-ingest-certificates` | Mimir and Alloy certificate material changes and returns `Ready` |
| `mise run cluster:hosts:check` | Managed host mappings match live gateway addresses |

Status commands are diagnostic displays, not behavioral tests.

## Manual Runtime Checks

The repository has previously exercised these contracts, but no reusable command automates them all:

- Plaintext and unauthorized meshed clients are rejected by application policy.
- Mimir, Tempo, and Loki gateways reject missing or wrong client identities and unsupported routes.
- Gateway-controlled tenant selection cannot be overridden by callers.
- Metrics, traces, and logs arrive once, remain tenant-scoped, and correlate in Grafana.
- Object-backed telemetry and PostgreSQL state survive pod replacement.
- Grafana uses PostgreSQL over the intended mutual TLS identity.
- Application requests remain healthy during an observability outage.
- Alloy retries and buffers within configured bounds and resumes export after recovery.
- Scoped MinIO users cannot access another consumer's buckets.

Treat these as live exercises until a dedicated verification workflow encodes them.

## Limits Of Static Validation

Static validation cannot prove:

- Kubernetes admission, defaulting, quotas, or CRD compatibility.
- Scheduling, CNI setup, PVC binding, probes, or controller convergence.
- LoadBalancer routing, DNS, Istio interception, or real TLS handshakes.
- Exclusive isolation of raw observability Services in kind.
- cert-manager issuance, hot reload, or uninterrupted traffic during rotation.
- Credential correctness and least-privilege enforcement in live services.
- End-to-end telemetry correctness, persistence, or Grafana queries.
- Negative authorization behavior.
- Availability during dependency failure.
- Image provenance or vulnerability status.

Interpret the layers narrowly: platform validation checks source and configuration contracts, Argo validation checks environment wiring, and live workflows check selected runtime paths.
