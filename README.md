# gitops-homelab

GitOps repository for my homelab Kubernetes cluster.

This repo is the source of truth for:
- Argo CD control-plane objects
- platform infrastructure running in the cluster
- application workloads

Argo CD continuously reconciles the cluster to the state defined here.

---

## Repository layout

```text
apps/
  <env>/
    <app>/
      ...

argocd/
  projects/
    platform-project.yaml
  apps.yaml
  cert-manager.yaml
  cert-manager-config.yaml
  argo-rollouts.yaml
  namespaces.yaml

bootstrap/
  root-application.yaml

infra/
  <env>/
    argo-rollouts/
      ...
    cert-manager/
      ...
    cert-manager-config/
      ...
    namespaces/
      ...
```

## Structure and responsibilities

### `bootstrap/`
Contains the initial manual bootstrap manifest.

- `root-application.yaml` is the only manifest applied manually with `kubectl`
- after that, Argo CD manages the rest

### `argocd/`
Contains Argo CD control-plane resources.

This includes:
- `AppProject`
- `ApplicationSet`
- standalone `Application` resources for infra components

`root` points to this directory.

### `apps/`
Contains workload applications.

Convention:

apps/<environment>/<application>

Examples:
- apps/prod/whoami
- apps/dev/whoami

These are generated into Argo CD `Application`s by the `ApplicationSet` in `argocd/apps.yaml`.

### `infra/`
Contains platform and cluster-level components.

Convention:

infra/<environment>/<component>

Examples:
- infra/prod/cert-manager
- infra/prod/argo-rollouts
- infra/prod/namespaces

Infra components are managed as explicit Argo CD `Application`s, not through the `apps` `ApplicationSet`.

---

## GitOps model

The reconciliation flow is:

Git → Argo CD root app → Argo CD Applications / ApplicationSets #→ cluster

### Control plane
Managed in `argocd/`:
- projects
- appsets
- infra app definitions

### Workloads
Managed in `apps/`

### Infrastructure
Managed in `infra/`

---

## Naming conventions

### Application folders
apps/<env>/<app>

### Infrastructure folders
infra/<env>/<component>

### Generated Argo CD application names

The `apps` `ApplicationSet` generates application names using:

<app>-<env>

Examples:
- whoami-prod
- whoami-dev

---

## Namespace strategy

Namespaces are managed centrally under:

infra/<env>/namespaces

Applications should not normally create or own their own namespaces.

This keeps namespace ownership explicit and avoids conflicts between apps.

---

## Environment model

The repository is structured to support multiple environments.

Current convention:

- prod
- dev (future)

Example:

apps/prod/whoami
apps/dev/whoami

---

## Argo CD design decisions

### Root app

`bootstrap/root-application.yaml` creates the `root` Argo CD application.

`root` points to:

argocd/

This ensures Argo CD manages its own configuration declaratively.

### AppProject

`argocd/projects/platform-project.yaml` defines:
- allowed repositories
- allowed destinations
- resource permissions

### ApplicationSet

`argocd/apps.yaml` generates applications from the `apps/` structure.

### Standalone infra apps

Infrastructure like cert-manager and Argo Rollouts are managed as separate Argo CD `Application`s.

This avoids ownership conflicts and keeps infra separate from workloads.

---

## Certificates and TLS

cert-manager is split into:

### Controller
argocd/cert-manager.yaml
infra/<env>/cert-manager/

### Configuration
argocd/cert-manager-config.yaml
infra/<env>/cert-manager-config/

This separates lifecycle from configuration.

---

## Vendored upstream manifests

Some components vendor upstream manifests (e.g. Argo Rollouts).

Guidelines:
- pin versions
- document upgrades
- avoid using `latest`

---

## Operational rules

- Make changes in Git first
- Avoid manual cluster edits unless necessary
- Ensure each resource has a single owner
- Use `prune: false` when adopting existing resources

---

## Workflow

1. Change manifests
2. Commit and push
3. Argo CD syncs automatically
4. Verify via UI or `kubectl`

---

## Useful commands

### Dashboards via a local Kubernetes tunnel

#### Argo CD

Forward the Argo CD API server from the cluster to your local machine:

```bash
kubectl port-forward -n argocd service/argocd-server 8080:443
```

Open `https://localhost:8080`. The browser will show a certificate warning when the
self-signed or cluster-managed certificate is not trusted locally.

The initial admin password can be read with:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 --decode && echo
```

If the forward runs on a remote host or k3s host, create an SSH tunnel before
opening the URL:

```bash
ssh -L 8080:localhost:8080 <user>@<k3s-host>
```

Keep the `kubectl port-forward` command running on the remote host or in an
SSH session on that host.

#### Argo Rollouts

Install the Argo Rollouts `kubectl` plugin once, then start the dashboard from
the local machine or a host with access to the cluster:

```bash
kubectl argo rollouts dashboard
```

Open `http://localhost:3100/rollouts`. This dashboard is provided by the
`kubectl argo rollouts` plugin; it is not exposed by the controller installation
in this repository.

### Applications
```bash
kubectl get applications -n argocd
kubectl get applicationsets -n argocd
```

### Inspect
`kubectl describe application <name> -n argocd`

### Refresh
`kubectl annotate application <name> -n argocd argocd.argoproj.io/refresh=hard --overwrite`

---

## Bootstrap

 `kubectl apply -f bootstrap/root-application.yaml`

---

## Goals

- fully GitOps-managed cluster
- clean separation of apps and infra
- reproducible state
- scalable multi-environment structure

---

## Future improvements

- monitoring/logging stack
- secret management
- backup strategy
- multi-environment setup
- advanced rollout strategies
