# Operate The Kind Platform

Run commands from the repository root unless a section says otherwise.

## Prerequisites

Install the tools pinned by the repository:

```sh
mise install
```

The workflows require a GNU/Linux-style userland, Bash 4 or newer, `mise`, a running Docker daemon, `kind`, `kubectl`, `curl`, `openssl`, `base64`, and standard Unix utilities. Mesh repair also requires `jq`. Hostname synchronization may use `sudo` and `resolvectl`.

Internet access is required for upstream manifests, Helm dependencies, container images, and the public Argo roots.

Run static validation before a substantial desired-state change:

```sh
mise run validate:all
```

See the [validation contract](validation.md) for targeted checks and limitations.

## Create The Platform

Create the complete reference environment:

```sh
mise run cluster:create
```

The command assumes that the target cluster does not exist. It creates the three-node kind cluster, bootstraps MetalLB and Argo CD, creates local-only Secrets, applies all three Argo roots, waits for convergence, and tests the application through the user gateway.

The smoke test discovers the live gateway address and uses `curl --resolve`, so cluster creation does not require an existing `/etc/hosts` entry.

After creation, configure the local hostnames:

```sh
mise run cluster:hosts:sync
```

## Manage Local Hostnames

The platform uses:

```text
app.k8s-playground.test
argocd.k8s-playground.test
grafana.k8s-playground.test
```

Inspect the managed `/etc/hosts` block without changing it:

```sh
mise run cluster:hosts:check
```

Synchronize the block with the current gateway addresses:

```sh
mise run cluster:hosts:sync
```

The synchronization command changes only the marked playground block, displays a diff, creates `/etc/hosts.k8s-playground.backup`, and preserves unrelated entries. It refuses malformed markers, conflicting unmanaged hostnames, symbolic-link targets, and missing or invalid gateway addresses.

Remove only the managed block:

```sh
mise run cluster:hosts:remove
```

You can test against another hosts file with `--hosts-file`:

```sh
mise run cluster:hosts:check --hosts-file /path/to/hosts
```

## Access Services

List the current external addresses and URLs:

```sh
mise run cluster:external-ips
```

After hostname synchronization, use:

| Service | URL |
| --- | --- |
| Application | `https://app.k8s-playground.test` |
| Argo CD | `https://argocd.k8s-playground.test` |
| Grafana | `https://grafana.k8s-playground.test` |

The local CA is intentionally not installed in the host trust store. Browser certificate warnings are expected.

Verify the application without changing host resolution or trusting the local CA:

```sh
mise run gateway:smoke-test
```

Expected response:

```text
Howdy from k8s-playground-service
```

Retrieve local administrator credentials:

```sh
mise run argocd:admin-password
mise run grafana:admin-password
```

Use a local Argo CD port-forward when the management gateway is unavailable:

```sh
mise run argocd:port-forward
```

The default local URL is `http://localhost:8080`. Select another port with `--port`.

## Inspect Status

Use the focused status commands when a stage fails:

```sh
mise run kind:status
mise run metallb:status
mise run argocd:status
mise run istio:status
mise run gateway:status
mise run app:status
mise run grafana-postgresql:status
mise run grafana:status
```

These commands display state. They do not replace readiness checks or data-path tests.

Run active readiness checks with:

```sh
mise run istio:wait
mise run gateway:wait
mise run management-gateway:wait
mise run gateway:smoke-test
```

## Reconcile Argo Applications

Reapply all three roots and wait for the expected children:

```sh
mise run argocd:bootstrap-roots
```

Force one child to synchronize:

```sh
mise run argocd:sync-app --app grafana
```

Use `--timeout 600` when a component needs more time. A forced sync is useful after a source path changes or after diagnosing a failed operation. It is not a substitute for publishing desired state that Argo can read.

## Restore Bootstrap Secrets

Fresh cluster creation runs all credential bootstrap tasks automatically. Before syncing dependent desired state into an existing cluster, run:

```sh
mise run minio:bootstrap-secrets
mise run mimir:bootstrap-secrets
mise run tempo:bootstrap-secrets
mise run loki:bootstrap-secrets
mise run grafana-postgresql:bootstrap-secrets
mise run grafana:bootstrap-secrets
```

The tasks preserve compatible existing Secrets. Mimir, Tempo, and Loki credentials are mirrored into the `minio` and `observability` namespaces because provisioning and consumers need the same scoped credential.

If Argo CD is also absent, run `mise run argocd:install` before the Secret tasks and `mise run argocd:bootstrap-roots` afterward.

## Repair After Laptop Suspension

Long workstation suspensions can leave paused kind containers with expired Istio workload certificates or projected service-account tokens. Pods may still report `Ready` while requests fail with HTTP 503.

Run:

```sh
mise run kind:repair-mesh
```

The task inspects each running `istio-proxy`, identifies certificates that are expired or near expiry, deletes only affected controller-managed pods, waits for their workloads, and reruns the external smoke test.

This repair is specific to kind on a suspendable workstation. Continuously running Kubernetes nodes should rotate workload certificates normally.

## Test Mimir Certificate Renewal

cert-manager normally renews certificates without operator action. To force renewal of the Mimir gateway and Alloy Mimir-client certificates:

```sh
mise run mimir:renew-ingest-certificates
```

The task requires both certificate fingerprints to change and both `Certificate` resources to return `Ready`. It does not send test telemetry or prove uninterrupted exporter traffic.

## Delete The Platform

Remove host mappings and delete the kind cluster:

```sh
mise run cluster:hosts:remove
mise run cluster:delete
```

Cluster deletion does not remove `/etc/hosts` entries, so the two commands are separate. It also leaves `/etc/hosts.k8s-playground.backup`, local image caches, and preserved failed-validation workspaces alone.

## Recover A Failed Bring-Up

`cluster:create` has no resume marker and fails if the named kind cluster already exists. The clean recovery is:

```sh
mise run cluster:hosts:remove
mise run cluster:delete
mise run cluster:create
mise run cluster:hosts:sync
```

To diagnose before rebuilding, use the status commands above. If the cluster is sound and only one later stage failed, rerun that public component task, such as `metallb:bootstrap`, `argocd:install`, the bootstrap Secret tasks, or `argocd:bootstrap-roots`.

Do not invoke hidden mise tasks or scripts directly during normal operation. Run `mise tasks` to see the complete public task inventory.
