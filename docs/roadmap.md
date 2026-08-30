# Roadmap

Kind remains the working reference environment. Every portability change must preserve `mise run validate:all` and the complete kind create, smoke-test, and delete lifecycle.

This roadmap contains unfinished work only. Current behavior belongs in [architecture.md](architecture.md), and rationale belongs in [decisions.md](decisions.md).

## Local Platform Work

### Identity

Objective: add Keycloak as a local OAuth 2.0 and OIDC provider, then use it for Grafana human login.

Scope: deploy Keycloak, define explicit claim-to-role mapping, disable inappropriate self-service paths, and retain a controlled local administrator account.

Exit condition: Grafana login and role mapping work through OIDC, and a tested break-glass login remains available.

### Profiling And eBPF Telemetry

Objective: add profiling without weakening the collector namespace or treating a reserved bucket as an installed backend.

Scope: deploy Pyroscope, configure storage and Grafana integration, then evaluate Beyla for useful service telemetry and profiling-adjacent signals.

Exit condition: profiles can be collected, retained, and queried. Beyla is either installed with bounded privileges and demonstrated value or rejected with a recorded reason.

### Mesh-Generated Tracing

Objective: add Istio-generated traces after the application OTLP path is stable.

Scope: route selected Envoy telemetry through Alloy without duplicating application spans or introducing sensitive request data.

Exit condition: mesh spans correlate with application traces, sampling and cardinality are bounded, and probes and sensitive values remain excluded.

### Alerting

Objective: use Mimir ruler and Alertmanager as the primary platform alerting path.

Scope: define Git ownership for rules, provision rules, configure grouping, inhibition, silences, repeat intervals, notification targets, and receiver credential handling.

Exit condition: selected cluster, workload, ingress, security, backend, and missing-telemetry failures produce actionable alerts without PHI or sensitive request data.

### HTTP/2

Objective: compare HTTP/2 behavior with the observed HTTP/1.1 baseline.

Scope: start with a reversible ingress-gateway-to-application-sidecar upgrade over Istio mTLS. Add application h2c only if the mesh-only experiment does not answer the learning goal.

Exit condition: telemetry proves the negotiated protocol and preserves existing HTTP/1.1 clients unless compatibility is removed deliberately.

### Secrets Management

Objective: replace ad hoc bootstrap credentials when new secret classes or cloud providers justify a dedicated system.

Scope: evaluate SOPS with `age`, External Secrets Operator with provider Secret Manager services, or a split model for different secret classes.

Exit condition: the selected design covers Git storage, local bootstrap, cloud workload identity, rotation, recovery, and least-privilege access without committing plaintext Secrets.

### Network Policy

Objective: add enforceable network isolation when the cluster CNI or cloud provider supports the intended policy model.

Scope: revisit the CNI decision, isolate raw observability Services, and prove that authenticated gateways are the only cross-zone ingest path.

Exit condition: policy tests demonstrate both allowed traffic and denied bypass paths without weakening Istio identity controls.

## GCP Provider Track

The GCP work should progress in independently verifiable slices:

| Phase | Outcome |
| --- | --- |
| Portability audit | Identify every kind-specific address, host path, credential, storage, DNS, certificate, and networking assumption |
| Infrastructure foundation | Establish a dedicated Terraform or OpenTofu area in this repository and retain one low-cost project, API, billing, state, and optional DNS foundation |
| Cost safety | Establish low quotas, budget alerts, mandatory session labels, expiry metadata, and cleanup that runs outside the cluster |
| Empty-cluster tracer | Create a zonal GKE Standard cluster, install Argo CD, validate the control path, and destroy all session resources |
| Argo composition | Add `clusters/gcp` roots without changing the working `clusters/kind` roots or copying portable desired state wholesale |
| Application path | Add GCP load balancing, temporary access, certificates, storage classes, Istio, and the application smoke test |
| Cloud services | Replace MinIO with GCS and static object-storage credentials with Workload Identity |
| Collector profile | Replace kind kubelet certificates and host-path assumptions with a verified GKE collection design |
| Observability | Bring central backends and Grafana over one component at a time |
| Teardown proof | Confirm that no labeled GKE, Compute Engine, load-balancing, disk, address, bucket, DNS, certificate, secret, or IAM resource remains |
| Cost measurement | Record creation time, teardown time, and billed cost, then update estimates from observed data |

The first profile should use zonal GKE Standard. The current Istio CNI and Alloy node integration need node-level access, so Autopilot is not the initial target.

Start with a small on-demand node pool for stable bring-up. Add a Spot profile after the regular lifecycle is reproducible. Do not buy committed-use discounts for an intermittent session cluster.

Keep regional high availability, backups, Cloud NAT, Cloud SQL, and other recurring managed services out of the first profile unless a milestone specifically studies them.

## GCP Cost Safety

Credits reduce a bill but do not prevent leaked resources. Before the first session:

- Label every resource with a session ID and expiry.
- Run expiry cleanup outside the cluster.
- Constrain vCPU, disk, address, and load-balancer quotas.
- Configure project and billing-account alerts.
- Treat alerts as delayed notifications, not hard spending caps.
- Keep an emergency programmatic billing response as a final guard, not normal teardown.
- Delete Kubernetes resources that own cloud infrastructure before destroying the cluster.
- Audit for residual resources after every early teardown.

### Dated Cost Snapshot

The following planning snapshot was checked on August 29, 2026. Recalculate it with the Google Cloud Pricing Calculator before provisioning.

Assumptions: `us-central1`, zonal GKE Standard, three `e2-standard-2` nodes, low traffic, and complete deletion after each session.

| Session | Planning Estimate |
| --- | ---: |
| 2 hours | `$0.75-$2` |
| 4 hours | `$1-$3` |
| 8 hours | `$2-$5` |
| 40 hours in one month | `$10-$25` |

Program notes checked on the same date:

- Eligible new customers could receive a `$300` welcome credit for 90 days. Delay activation until a deployable GCP slice is ready.
- The GKE free tier provided `$74.40` in monthly credits for one zonal Standard or Autopilot cluster per billing account. It covered the cluster management fee, not nodes, disks, networking, or load balancers.
- Formal education grants were issued through eligible faculty and institutions.
- Google Skills credits paid for hosted labs and could not fund this project's resources.

Verify current terms at:

- [Google Cloud Free Program](https://cloud.google.com/free/docs/free-cloud-features)
- [GKE pricing](https://cloud.google.com/kubernetes-engine/pricing)
- [Google Cloud education credits](https://edu.google.com/programs/credits/)
- [Google Cloud for students](https://cloud.google.com/edu/students)

## AWS Provider Track

Begin AWS work only after GCP establishes the provider contract and exposes which differences belong in infrastructure, Argo composition, or component values.

The first AWS milestone will define an EKS session profile and `clusters/aws` roots without adding AWS concepts to portable component configuration. It must carry forward the same expiry, cleanup, quota, cost measurement, and kind-regression requirements.

## Open Decisions

- Which secret classes belong in encrypted Git and which belong in provider Secret Manager services?
- Which enforceable CNI and NetworkPolicy model should follow the current kindnet phase?
- Should the mature cloud shape keep observability in the application cluster or use a separate cluster?
- Which session resources, if any, should persist between cloud sessions?
