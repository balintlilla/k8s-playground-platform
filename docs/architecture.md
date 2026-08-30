# Current Architecture

This document describes the platform that is present in the repository today. Kind is the implemented reference environment. GCP and AWS are planned provider tracks, not current deployment targets.

## Repository Boundaries

This repository owns Kubernetes desired state, local bootstrap support, and provider infrastructure code:

- `apps/` contains application-owned resources.
- `platform/` contains platform charts and manifests.
- `clusters/kind/` contains the kind topology and kind-only MetalLB template.
- `bootstrap/argocd/kind/` contains the resources needed before Argo CD can reconcile steady state.
- `mise.toml` and `scripts/` contain local lifecycle, validation, and recovery workflows.
- Future Terraform configuration will manage cloud clusters, IAM, networking, DNS, and provider services from a dedicated top-level area in this repository.

The adjacent [`k8s-playground-argocd-apps`](https://github.com/mblayman/k8s-playground-argocd-apps) repository owns Argo CD `Application` wiring. It selects components, source paths, destinations, projects, sync policies, and child-Application waves.

Argo-managed Helm components use wrapper charts in `platform/`. Each wrapper owns its dependency declaration, local values, and committed `Chart.lock`. The Argo apps repository points at one wrapper path rather than combining an upstream chart with a separate values source.

Provider infrastructure code will remain separate from Kubernetes desired state by directory, but both belong in this repository.

## Kind Reference Cluster

[`clusters/kind/cluster.yaml`](../clusters/kind/cluster.yaml) creates one control-plane node and two worker nodes. The cluster uses kind's default CNI. Istio CNI configures mesh traffic redirection but does not replace the cluster CNI.

MetalLB supplies kind `LoadBalancer` addresses in layer 2 mode. [`scripts/render-metallb-kind-config.sh`](../scripts/render-metallb-kind-config.sh) derives the address pool from the live Docker `kind` network. MetalLB remains outside Argo because the host Docker subnet is runtime state that Argo cannot discover safely.

## Logical Platform Zones

One physical cluster models three ownership zones:

| Zone | Current Components | Intended Boundary |
| --- | --- | --- |
| Shared platform | Argo CD, cert-manager, Gateway API CRDs, MetalLB bootstrap, MinIO, and object-storage provisioning | Capabilities available independently of application and observability workloads |
| Application | Istio, user and management gateways, Gateway API configuration, Alloy, and `k8s-playground-service` | Workloads, mesh identity, ingress, and workload-adjacent collectors |
| Observability | Mimir, Tempo, Loki, Grafana, and Grafana PostgreSQL | Central telemetry storage and query services |

Alloy runs in `observability-collectors`, but it belongs to the application zone. It should move with application workloads if the central backends move to another cluster.

MinIO is generic kind-local object storage. It is not part of the observability product boundary.

## Argo Ownership

Three roots reconcile independently:

| Root | Child Directory | Current Ownership |
| --- | --- | --- |
| `k8s-playground-kind-root` | `clusters/kind/apps` | Gateway API CRDs, Argo configuration and repositories, cert-manager, MinIO, object-storage provisioning, and the observability `AppProject` |
| `k8s-playground-kind-application-root` | `clusters/kind/application/apps` | Istio, both gateway data planes, Alloy, Gateway API configuration, and the application |
| `k8s-playground-kind-observability-root` | `clusters/kind/observability/apps` | Mimir, Tempo, Loki, and the Grafana umbrella chart |

The observability children use a restricted `AppProject` that permits the platform-config source and the `observability` destination namespace.

Child-Application waves provide coarse creation order inside one root. They do not wait for arbitrary resources in another child to become ready, and waves in separate roots do not globally interleave. Hard readiness dependencies belong inside one `Application`, where resource-level waves can gate later resources on earlier health.

The Argo apps repository owns the exact bands and guardrails in its [sync-wave contract](https://github.com/mblayman/k8s-playground-argocd-apps#sync-wave-contract).

## Bootstrap And Steady State

`mise run cluster:create` creates kind, bootstraps MetalLB, installs Argo CD, creates local-only credential Secrets, and applies all three roots. It waits for every expected root and child to become `Synced` and `Healthy` before it runs the application smoke test.

Argo CD is initially installed from pinned upstream manifests. The `argocd-config` child then owns its steady-state configuration. MetalLB and generated credentials remain bootstrap inputs outside Argo.

After bootstrap, each root reconciles independently with prune and self-heal. Application request handling does not depend on observability health. Exporters use bounded queues, timeouts, and retries when a telemetry backend is unavailable.

## Ingress And Traffic Flow

The platform has separate user and management gateway data planes.

User traffic follows this path:

```text
app.k8s-playground.test
  -> MetalLB address for istio-ingressgateway
  -> Gateway/k8s-playground-gateway
  -> TLS termination
  -> HTTPRoute/k8s-playground-service
  -> ClusterIP Service/k8s-playground-service
  -> application sidecar over Istio mTLS
  -> application container on port 8080
```

Management traffic uses a different `LoadBalancer` Service and Gateway:

```text
argocd.k8s-playground.test -> argocd-server:80
grafana.k8s-playground.test -> grafana:80
```

HTTP listeners redirect to HTTPS. cert-manager issues the user and management certificates from the cluster-local CA. The CA is intentionally not installed in the workstation trust store, so browser warnings are expected.

The application, Argo CD, and Grafana Services remain `ClusterIP`. Only the two Istio gateway Services receive MetalLB addresses.

## Identity And Authorization

Application and collector namespaces use revision-based Istio sidecar injection. `k8s-playground-service` has a dedicated ServiceAccount, which gives it the Istio principal:

```text
cluster.local/ns/k8s-playground-service/sa/k8s-playground-service
```

[`apps/k8s-playground-service/peerauthentication.yaml`](../apps/k8s-playground-service/peerauthentication.yaml) requires strict mTLS in the application namespace. [`apps/k8s-playground-service/authorizationpolicy.yaml`](../apps/k8s-playground-service/authorizationpolicy.yaml) permits only the ingress gateway ServiceAccount to reach application port `8080`. The policy does not currently restrict HTTP methods or paths.

Alloy's OTLP receiver uses a separate policy boundary. Istio strict mTLS protects port `4318`, and its `AuthorizationPolicy` permits only the application ServiceAccount to send metrics and traces.

Two certificate systems are involved:

- Istio issues workload certificates from Kubernetes namespace and ServiceAccount identity.
- cert-manager issues native client and server certificates for telemetry gateways, MinIO, public gateways, and PostgreSQL.

The native Alloy client certificate uses URI identity `spiffe://k8s-playground/collectors/alloy`. It is SPIFFE-shaped but is not an Istio workload certificate.

## Object Storage

The MinIO wrapper runs one persistent standalone server with an internal HTTPS Service. Its root credentials come from a bootstrap-created Secret.

[`platform/observability/object-storage-config/`](../platform/observability/object-storage-config/) provisions buckets, scoped users, and policies. Mimir, Tempo, and Loki each use dedicated credentials and verified HTTPS rather than MinIO root credentials. A reserved Pyroscope bucket does not mean that Pyroscope is installed.

## Observability Flow

Alloy runs one DaemonSet pod per node, including the control-plane node. Each instance limits discovery to its own node.

Node metrics flow through the local kubelet:

```text
kubelet /metrics and /metrics/cadvisor
  -> node-local Alloy
  -> native-mTLS Mimir gateway
  -> Mimir single-binary backend
  -> working PVC and MinIO
```

Application telemetry flows through the meshed OTLP receiver:

```text
k8s-playground-service OTLP/HTTP
  -> Alloy on port 4318 over Istio mTLS
  -> metrics through the Mimir gateway
  -> traces through the Tempo gateway
```

Container logs do not use application OTLP:

```text
node /var/log/pods
  -> node-local Alloy file tailer
  -> CRI parsing and Kubernetes labels
  -> native-mTLS Loki gateway
  -> Loki and MinIO
```

The Mimir, Tempo, and Loki Envoy gateways require the exact Alloy native certificate identity. They overwrite `X-Scope-OrgID` with `k8s-playground` and expose only required ingest routes. Raw backend Services use separate `*-internal` names.

The gateways prove the intended authenticated cross-zone contract. They do not provide exclusive network isolation in kind because kind's default CNI does not enforce the desired NetworkPolicy boundary. Raw internal Services remain cluster-reachable.

Grafana queries the internal Mimir, Tempo, and Loki Services within the observability zone. Datasource provisioning connects Mimir exemplars to Tempo traces and supports trace-to-log and log-to-trace navigation.

## Grafana Persistence

Grafana and PostgreSQL share one umbrella wrapper and one Argo `Application`. Resource-level waves create database certificates first, PostgreSQL second, and Grafana last.

Grafana uses password-authenticated mutual TLS with hostname verification. PostgreSQL stores Grafana metadata on its own persistent volume. The older Grafana PVC remains mounted but is not the active metadata database. Anonymous access and user sign-up are disabled.

## Portability Boundary

The intended portable contracts are:

- Application and platform desired state in `apps/` and `platform/`.
- Three logical ownership zones and independently reconciled Argo roots.
- Gateway API routing and internal `ClusterIP` services.
- Istio ServiceAccount-derived workload identity.
- OTLP, Prometheus remote write, and Loki push protocols.
- Verified object-storage transport with consumer-scoped access.
- Native TLS and authentication across the application-to-observability boundary.
- Application availability independent of observability health.

Provider profiles must replace or overlay these kind-specific details:

- kind topology and default networking.
- MetalLB and Docker-network address discovery.
- `.test` hostnames, `/etc/hosts`, and the local CA.
- MinIO and static S3-style credentials.
- Local storage classes and volume topology.
- Alloy kubelet certificates, pod-log host paths, and node-local positions.
- Same-cluster Service DNS standing in for private cross-cluster DNS.
- Same-cluster reachability of raw observability Services.

## Installed Components

Current desired state includes:

- Alloy.
- Argo CD.
- cert-manager.
- Gateway API CRDs and configuration.
- Grafana.
- Grafana's dedicated PostgreSQL database.
- Istio control plane and CNI.
- Istio user and management gateways.
- `k8s-playground-service`.
- Loki.
- Mimir.
- MinIO.
- Tempo.

The [roadmap](roadmap.md) owns planned capabilities and provider work.
