# k8s-playground-platform-config

Kubernetes desired state, local bootstrap support, and future cloud infrastructure code for the k8s playground platform.

Kind is the implemented reference environment. The current platform includes Argo CD, cert-manager, Istio, Gateway API, Alloy, Mimir, Tempo, Loki, Grafana, PostgreSQL, MinIO, and the playground application. GCP and AWS are future provider tracks.

## Repository Ownership

This repository owns application and platform resources, wrapper charts, kind configuration, local lifecycle tasks, platform validation, and future Terraform or OpenTofu infrastructure code.

The adjacent [`k8s-playground-argocd-apps`](https://github.com/mblayman/k8s-playground-argocd-apps) repository owns Argo CD `Application` composition, including component selection, source paths, destinations, projects, sync policy, and child-Application waves.

## Quick Start

The workflows require a GNU/Linux environment, a running Docker daemon, `kind`, and `kubectl`. Mise installs the additional tools pinned by this repository.

Install pinned tools and validate the desired state:

```sh
mise install
mise run validate:all
```

Create the complete kind platform and configure local hostnames:

```sh
mise run cluster:create
mise run cluster:hosts:sync
```

Delete it when finished:

```sh
mise run cluster:hosts:remove
mise run cluster:delete
```

See [operations](docs/operations.md) for prerequisites, access, recovery, and component-level workflows.

## Documentation

- [Current architecture](docs/architecture.md) describes the implemented platform and portability boundaries.
- [Operations](docs/operations.md) covers the supported kind lifecycle and recovery procedures.
- [Validation](docs/validation.md) defines static, Argo wiring, and live runtime checks.
- [Roadmap](docs/roadmap.md) contains unfinished local, GCP, and AWS work.
- [Architecture decisions](docs/decisions.md) records the rationale that still shapes the repository.
