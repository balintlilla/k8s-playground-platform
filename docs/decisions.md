# Architecture Decisions

This is a concise record of decisions that still shape the repository. Git history preserves implementation chronology and superseded intermediate states.

## Kind Remains The Reference Environment

Status: Accepted.

Decision: Keep the complete kind lifecycle working while GCP and AWS are added as separate provider tracks.

Why: Kind provides a cheap, fast environment for learning the platform contracts before adding cloud infrastructure, identity, networking, and billing concerns.

Consequences: Provider work must preserve kind validation and lifecycle behavior. Cloud differences belong in provider infrastructure, Argo composition, or narrow values and overlays rather than forks of the full desired state.

## This Repository Owns Platform And Infrastructure

Status: Accepted.

Decision: This repository owns Kubernetes resources, platform wrapper charts, local bootstrap support, and Terraform cloud infrastructure. `k8s-playground-argocd-apps` owns Argo `Application` composition.

Why: Platform configuration and the infrastructure that hosts it are parts of the same playground. Keeping both here supports one end-to-end lifecycle without adding another repository. Argo composition remains separate because it selects which desired state a cluster reconciles.

Consequences: Do not store Argo `Application` definitions here. Keep cluster and provider infrastructure under `infrastructure/` rather than mixing it into `platform/`.

## Terraform Is The IaC Engine

Status: Accepted.

Decision: Use Terraform directly for cloud infrastructure. Do not introduce OpenTofu or Terragrunt at this stage.

Why: Terraform is the engine used by the official GCP documentation and has the broadest industry familiarity. The initial foundation and session roots do not need another orchestration layer, and mise can keep the Kubernetes and Argo lifecycle explicit.

Consequences: Pin Terraform through mise, commit `.terraform.lock.hcl` for each root, and store foundation and disposable session state separately. Avoid Terraform-specific hosted services unless a later requirement justifies them. Reconsider Terragrunt only if repeated environments, many state roots, or a nontrivial dependency graph make direct Terraform orchestration hard to maintain. Adopting OpenTofu later requires a separate state and compatibility review.

## Argo Follows A Proven Bootstrap Path

Status: Accepted.

Decision: Local automation creates kind, MetalLB, and Argo CD before Argo takes ownership of steady-state platform and application resources.

Why: An empty cluster cannot use Argo to install Argo itself, and kind MetalLB configuration depends on the host Docker network.

Consequences: Bootstrap credentials and MetalLB remain explicit prerequisites outside steady-state reconciliation. Fresh-cluster automation waits for all roots before generating validation traffic.

## MetalLB Is Kind-Only Bootstrap Infrastructure

Status: Accepted.

Decision: Use MetalLB layer 2 mode for kind and derive its address pool from the live Docker network.

Why: Layer 2 supplies the required `LoadBalancer` behavior without unrelated BGP router configuration. Runtime discovery avoids machine-specific committed subnets.

Consequences: MetalLB remains outside Argo. Cloud providers must use provider load balancing instead of reusing the kind configuration.

## Istio Uses Sidecar Mode And Istio CNI

Status: Accepted.

Decision: Use Istio sidecar mode, revision-based injection, strict mTLS, and Istio CNI. Do not add Cilium or ambient mode in the current phase.

Why: Sidecar mode is the mature Istio path and makes proxy identity, L7 authorization, and telemetry explicit. Deferring Cilium keeps the current networking model understandable.

Consequences: kind's default CNI does not provide the desired general NetworkPolicy enforcement. That limitation must be stated rather than hidden behind policy manifests.

## User And Management Ingress Are Separate

Status: Accepted.

Decision: Run independent Istio user and management gateway data planes. Use Gateway API routes and keep backend Services internal.

Why: Application traffic and administrative interfaces have different exposure and policy needs.

Consequences: Kind allocates two MetalLB addresses. Argo CD and Grafana share the management gateway as a playground convenience, while the application uses the user gateway.

## Three Argo Roots Model Ownership Zones

Status: Accepted.

Decision: Reconcile shared platform, application, and observability children through three independent roots.

Why: The single kind cluster stands in for boundaries that may become separate cloud clusters later.

Consequences: Child waves are local creation order, not a global readiness graph. Application availability cannot depend on observability-root health. Hard readiness dependencies stay inside one child `Application`.

## Wrapper Charts Own Helm Dependencies

Status: Accepted.

Decision: Use lightweight single-source wrapper charts with committed `Chart.lock` files.

Why: Each Argo child can point at one source path that contains the upstream dependency declaration and local values. The lock preserves the dependency digest.

Consequences: Refresh locks only for intentional dependency changes. Commit locks and ignore downloaded archives. Do not reintroduce Argo multi-source chart-plus-values applications.

## MinIO Is Generic Kind-Local Object Storage

Status: Accepted.

Decision: Keep MinIO in the shared platform zone and provision consumer-specific buckets, users, and policies separately.

Why: Object storage is a platform capability used by observability, not an observability component itself.

Consequences: Mimir, Tempo, and Loki receive scoped credentials and verified HTTPS. Cloud profiles replace MinIO with provider object storage and workload identity.

## Observability Uses Grafana Backends And Alloy

Status: Accepted.

Decision: Use Alloy for collection, Mimir for metrics, Tempo for traces, Loki for logs, and Grafana for query and visualization.

Why: This models separate collection, ingest, storage, tenancy, and query responsibilities rather than treating one Prometheus server as the entire observability system.

Consequences: Application telemetry uses OTLP, node metrics retain scrape compatibility, and container logs come from Kubernetes stdout and stderr files rather than application OTLP logs.

## Cross-Zone Telemetry Uses Native Authentication

Status: Accepted.

Decision: Protect Alloy-to-backend ingest with native mutual TLS, exact client identity, route allowlists, and gateway-controlled tenant selection.

Why: Istio identity belongs to the logical application cluster. A remote observability environment may be meshless or use another identity system. `X-Scope-OrgID` selects a tenant but is not authentication.

Consequences: Mimir, Tempo, and Loki have dedicated Envoy ingest gateways. Raw internal Services remain a documented kind limitation until network separation or an enforcing CNI removes bypass paths.

## Grafana Uses Dedicated PostgreSQL

Status: Accepted.

Decision: Keep PostgreSQL in the Grafana umbrella chart and start with a fresh database instead of migrating disposable SQLite metadata.

Why: One Argo `Application` allows certificate and database resource waves to establish PostgreSQL before Grafana starts. Git provisioning recreates dashboards and datasources without carrying local SQLite history.

Consequences: Grafana uses password-authenticated mutual TLS with hostname verification. PostgreSQL has its own retained volume. The old Grafana PVC remains mounted until a separate cleanup decision removes it.

## HTTP/2 Follows Observability

Status: Accepted.

Decision: Keep HTTP/1.1 as the baseline, then run a reversible mesh-only HTTP/2 experiment before adding application h2c support.

Why: Protocol changes are easier to understand when telemetry can show the actual proxy and application behavior.

Consequences: HTTP/2 remains roadmap work. Any h2c adoption must declare the Service protocol and preserve HTTP/1.1 compatibility unless removal is deliberate.
