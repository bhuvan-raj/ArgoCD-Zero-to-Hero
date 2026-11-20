# ArgoCD Architecture and Core Concepts

![ArgoCD][badge-argocd] ![GitOps][badge-gitops] ![Kubernetes][badge-k8s] ![Level][badge-level]

> How ArgoCD is built and how it thinks: the components that make up an installation, the reconciliation loop, and the core objects (Applications, Projects, sync policies, hooks and ApplicationSets) you will use in every later module.

---

## Table of Contents

- [Learning Objectives](#learning-objectives)
- [ArgoCD in One Picture](#argocd-in-one-picture)
- [ArgoCD Components](#argocd-components)
- [How the Components Work Together](#how-the-components-work-together)
- [The Reconciliation Loop](#the-reconciliation-loop)
- [Core Concepts](#core-concepts)
- [Application](#application)
- [Sync Status and Health Status](#sync-status-and-health-status)
- [Sync Policies](#sync-policies)
- [Sync Options](#sync-options)
- [Sync Phases, Hooks and Waves](#sync-phases-hooks-and-waves)
- [Projects](#projects)
- [Repositories and Clusters](#repositories-and-clusters)
- [Supported Manifest Formats](#supported-manifest-formats)
- [App of Apps and ApplicationSets](#app-of-apps-and-applicationsets)
- [ArgoCD Configuration Objects](#argocd-configuration-objects)
- [Hands-On: Explore Your Installation](#hands-on-explore-your-installation)
- [Common Pitfalls](#common-pitfalls)
- [Knowledge Check](#knowledge-check)
- [Summary](#summary)
- [Series Navigation](#series-navigation)

---

## Learning Objectives

By the end of this module you will be able to:

- Name each ArgoCD component and explain what it does.
- Describe how a change in Git ends up as a change in the cluster.
- Explain the difference between **desired state**, **live state**, **sync status** and **health status**.
- Write a basic `Application` and `AppProject` manifest.
- Choose between manual and automated sync, and understand `prune` and `selfHeal`.
- Use sync hooks and waves to control deployment ordering.
- Understand when to reach for App of Apps or ApplicationSets.

---

## ArgoCD in One Picture

ArgoCD is a declarative, GitOps continuous delivery controller for Kubernetes. Git is the source of truth; ArgoCD continuously makes the cluster match it.

```text
   +-------------+        +-----------------------------------------+
   |  Developer  |        |             Kubernetes Cluster          |
   +------+------+        |                                         |
          |               |   +-----------------------------------+ |
          | git push      |   |              ArgoCD               | |
          v               |   |                                   | |
   +-------------+  pull  |   |  compares   DESIRED  vs   LIVE    | |
   |  Git Repo   |<-------+---|  state      (Git)       (cluster) | |
   | (manifests) |        |   |                                   | |
   +-------------+        |   +-----------------+-----------------+ |
                          |                     | apply / sync      |
                          |                     v                   |
                          |   +-----------------------------------+ |
                          |   |  Deployments, Services, ConfigMaps |
                          |   +-----------------------------------+ |
                          +-----------------------------------------+
```

---

## ArgoCD Components

A standard installation runs several workloads in the `argocd` namespace.

| Component | Kubernetes workload | Responsibility |
|-----------|--------------------|----------------|
| **API Server** | `argocd-server` (Deployment) | Exposes the gRPC/REST API used by the Web UI, CLI and CI/CD systems. Handles authentication, RBAC enforcement, and webhook events from Git providers. |
| **Repository Server** | `argocd-repo-server` (Deployment) | Clones Git repositories and generates Kubernetes manifests from Helm, Kustomize, plain YAML or plugins. Caches results. |
| **Application Controller** | `argocd-application-controller` (StatefulSet) | The core control loop. Compares live cluster state with desired state, reports status, and performs syncs. |
| **Redis** | `argocd-redis` (Deployment) | Cache for repository and application state. Not a database; it can be rebuilt. |
| **Dex** (optional) | `argocd-dex-server` (Deployment) | Identity service that connects ArgoCD to external SSO providers (OIDC, LDAP, SAML, GitHub, etc.). |
| **ApplicationSet Controller** | `argocd-applicationset-controller` (Deployment) | Generates many `Application` objects from templates and generators. |
| **Notifications Controller** | `argocd-notifications-controller` (Deployment) | Sends alerts (Slack, email, webhooks) on application events. |

> **Where does ArgoCD keep its state?** In Kubernetes itself. Applications, Projects, repository credentials and cluster credentials are all stored as Kubernetes custom resources and Secrets. There is no separate ArgoCD database.

### API Server

- Serves the Web UI, the `argocd` CLI and the API.
- Authenticates users (local accounts or SSO through Dex/OIDC).
- Enforces RBAC before any action is performed.
- Receives webhooks from GitHub, GitLab, Bitbucket and others so changes are picked up immediately instead of waiting for the next poll.

### Repository Server

- Holds a local cache of the Git repositories it has been asked to read.
- Renders manifests with the right tool (Helm, Kustomize, directory of YAML, or a config management plugin).
- Is deliberately **stateless** and scalable. It never talks to the destination cluster.

### Application Controller

- Runs as a Kubernetes controller, continuously watching running applications.
- Compares the **live state** with the **target state** produced by the repo server.
- Detects `OutOfSync` applications and, depending on policy, syncs them.
- Executes lifecycle hooks (`PreSync`, `Sync`, `PostSync`, `SyncFail`).

### Redis

- Caches manifests, application state and cluster state to reduce load on Git and the Kubernetes API.
- Safe to lose; ArgoCD repopulates it.

### Dex

- Optional. Only needed if you use SSO with an external identity provider.
- Without it you rely on the built-in `admin` and local accounts.

### ApplicationSet Controller

- Watches `ApplicationSet` resources.
- Uses **generators** (list, cluster, Git, matrix and more) to stamp out `Application` objects at scale, for example one per cluster or one per environment folder.

### Notifications Controller

- Watches applications and evaluates triggers (sync failed, health degraded, sync succeeded).
- Sends messages using configured templates and services.

---

## How the Components Work Together

```text
 +-----------+   +-----------+   +--------------------+
 |  Web UI   |   |  argocd   |   |  CI/CD / Webhooks  |
 |           |   |    CLI    |   |  (Git providers)   |
 +-----+-----+   +-----+-----+   +----------+---------+
       |               |                    |
       +---------------+--------------------+
                       |
                       v
              +------------------+        +-------------+
              |   API Server     |<------>|     Dex     |
              | (auth, RBAC, API)|  SSO   | (optional)  |
              +--------+---------+        +-------------+
                       |
                       v
        +---------------------------------+
        |   Application Controller        |
        |   (compare, sync, health)       |
        +------+----------------+---------+
               |                |
   request     |                | read/write live state
   manifests   v                v
        +--------------+   +------------------------+
        | Repo Server  |   |  Target Kubernetes     |
        | (Git, Helm,  |   |  cluster(s)            |
        |  Kustomize)  |   +------------------------+
        +------+-------+
               |
               v
        +--------------+        +---------+
        |  Git Repos   |        |  Redis  |  (cache for all of the above)
        +--------------+        +---------+
```

A typical flow:

1. A developer pushes a change to the Git repository.
2. ArgoCD notices the change, either by polling (every 3 minutes by default) or through a webhook.
3. The **Application Controller** asks the **Repository Server** to render the manifests for the tracked revision.
4. The controller compares the rendered manifests (desired state) with the resources in the cluster (live state).
5. If they differ, the application is marked `OutOfSync`.
6. A sync is performed, either manually or automatically, and the controller applies the changes using the Kubernetes API.
7. The controller evaluates resource health and reports the final status in the UI, CLI and API.

---

## The Reconciliation Loop

At the heart of ArgoCD is a loop that never stops:

```text
        +-----------------------------+
        |  1. Fetch desired state     |
        |     (render Git manifests)  |
        +--------------+--------------+
                       |
                       v
        +-----------------------------+
        |  2. Read live state         |
        |     (from the cluster)      |
        +--------------+--------------+
                       |
                       v
        +-----------------------------+
        |  3. Diff desired vs live    |
        +--------------+--------------+
                       |
              +--------+--------+
              |                 |
              v                 v
        +-----------+     +-----------+
        |  Synced   |     | OutOfSync |
        +-----------+     +-----+-----+
                                |
                     manual or automated sync
                                |
                                v
                      +-------------------+
                      |  4. Apply changes |
                      +---------+---------+
                                |
                                +--------> back to step 1
```

Two things can cause drift:

- **Git changed:** the desired state moved. This is the normal deployment path.
- **The cluster changed:** someone edited a resource directly (for example with `kubectl edit`). With `selfHeal` enabled, ArgoCD reverts it.

---

## Core Concepts

| Concept | Meaning |
|---------|---------|
| **Application** | A custom resource that connects a Git source (repo, path, revision) to a destination (cluster, namespace). |
| **Application source type** | The tool used to build manifests: Helm, Kustomize, Jsonnet, plain directory or a plugin. |
| **Target state** (desired state) | What the manifests in Git say the cluster should look like. |
| **Live state** | What is actually running in the cluster right now. |
| **Sync status** | Whether live state matches target state. |
| **Health status** | Whether the resources are actually working. |
| **Sync** | The act of making live state match target state. |
| **Refresh** | Comparing the latest Git state with the cached state and live state, without changing the cluster. |
| **Project (AppProject)** | A logical grouping of applications with guardrails: allowed repos, destinations, and resource kinds. |
| **Repository** | A Git (or Helm/OCI) source registered with ArgoCD. |
| **Cluster** | A destination Kubernetes cluster registered with ArgoCD. |
| **ApplicationSet** | A template plus generators that produces many Applications. |

---

## Application

The `Application` custom resource is the most important object in ArgoCD. It answers three questions: **what** to deploy, **from where**, and **to where**.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:                                # FROM WHERE
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook

  destination:                           # TO WHERE
    server: https://kubernetes.default.svc
    namespace: guestbook

  syncPolicy:                            # HOW
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

| Field | Purpose |
|-------|---------|
| `metadata.namespace` | Applications live in the `argocd` namespace (where ArgoCD is installed). |
| `finalizers` | `resources-finalizer.argocd.argoproj.io` makes deleting the Application also delete the resources it deployed (cascading delete). |
| `spec.project` | The AppProject that governs this application. `default` allows everything. |
| `spec.source.repoURL` | Git (or Helm/OCI) repository URL. |
| `spec.source.targetRevision` | Branch, tag, commit SHA or `HEAD`. |
| `spec.source.path` | Directory in the repo containing the manifests. |
| `spec.destination.server` | The cluster API URL. `https://kubernetes.default.svc` means the cluster ArgoCD runs in. |
| `spec.destination.namespace` | Default target namespace. |
| `spec.syncPolicy` | Whether and how syncs happen. |

> `spec.destination` can use either `server` (the cluster URL) or `name` (the cluster's registered name), not both.

---

## Sync Status and Health Status

ArgoCD reports two independent things. Do not confuse them.

### Sync status

Does the cluster match Git?

| Status | Meaning |
|--------|---------|
| `Synced` | Live state matches target state. |
| `OutOfSync` | Live state differs from target state. |
| `Unknown` | ArgoCD cannot determine the state (for example the repo cannot be read). |

### Health status

Are the resources working?

| Status | Meaning |
|--------|---------|
| `Healthy` | Resource is operating correctly. |
| `Progressing` | Not healthy yet, but still moving toward healthy (for example a rollout in progress). |
| `Degraded` | Resource has failed or cannot reach a healthy state. |
| `Suspended` | Resource is paused (for example a suspended CronJob). |
| `Missing` | Resource exists in Git but not in the cluster. |
| `Unknown` | Health could not be determined. |

### Why both matter

| Sync | Health | What it means |
|------|--------|---------------|
| Synced | Healthy | Everything matches Git and is working. |
| Synced | Degraded | The cluster matches Git, but the app is broken (bad image, failing probe). Fix Git. |
| OutOfSync | Healthy | Running fine, but differs from Git (manual change or new commit not yet applied). |
| OutOfSync | Degraded | Differs from Git and is also broken. Investigate. |

---

## Sync Policies

The sync policy decides whether ArgoCD applies changes on its own.

### Manual sync (default)

ArgoCD detects the difference and marks the app `OutOfSync`, but waits for a human (or pipeline) to trigger the sync.

```bash
argocd app sync guestbook
```

### Automated sync

ArgoCD syncs automatically when it detects a difference between Git and the cluster.

```yaml
syncPolicy:
  automated: {}
```

Automated sync can be tuned with two important flags:

| Flag | Behaviour when `false` (default) | Behaviour when `true` |
|------|----------------------------------|-----------------------|
| `prune` | Resources removed from Git are **left** in the cluster. | Resources removed from Git are **deleted** from the cluster. |
| `selfHeal` | Manual changes made in the cluster are **not** reverted until the next sync. | Manual changes in the cluster are **automatically reverted** to match Git. |

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

```text
  Someone runs:  kubectl scale deployment guestbook --replicas=10
                                   |
                                   v
        Git says replicas: 2  -->  live says replicas: 10  --> OutOfSync
                                   |
                       selfHeal: true  |  selfHeal: false
                                   |             |
                                   v             v
                   ArgoCD resets to 2     Stays at 10 until a sync
```

> **Recommendation:** Use manual sync while learning and for production-critical apps with approval gates. Use automated sync with `prune` and `selfHeal` for environments where Git should be the only way to change things.

---

## Sync Options

Sync options fine-tune how a sync is performed. They can be set per application (`syncPolicy.syncOptions`) or per resource using an annotation.

| Option | Purpose |
|--------|---------|
| `CreateNamespace=true` | Create the destination namespace if it does not exist. |
| `PruneLast=true` | Delete pruned resources after all other resources are healthy. |
| `ApplyOutOfSyncOnly=true` | Apply only resources that are out of sync, instead of everything. Useful for large apps. |
| `ServerSideApply=true` | Use Kubernetes server-side apply. Helpful for very large resources such as CRDs. |
| `Replace=true` | Use `kubectl replace` instead of `apply`. Use with care; it is destructive. |
| `Validate=false` | Skip kubectl validation. |
| `PrunePropagationPolicy=foreground` | Control how dependents are deleted when pruning. |

```yaml
syncPolicy:
  syncOptions:
    - CreateNamespace=true
    - PruneLast=true
    - ServerSideApply=true
```

### Retry on failure

```yaml
syncPolicy:
  automated: {}
  retry:
    limit: 5
    backoff:
      duration: 5s
      factor: 2
      maxDuration: 3m
```

---

## Sync Phases, Hooks and Waves

Real deployments often need ordering: run a database migration **before** the app starts, or run a smoke test **after** it is up. ArgoCD provides **hooks** and **waves** for that.

### Sync phases

```text
   PreSync  -->  Sync  -->  PostSync
                   |
                   +--(on failure)--> SyncFail
```

| Phase | When it runs | Typical use |
|-------|--------------|-------------|
| `PreSync` | Before the main sync | Database migrations, backups |
| `Sync` | During the main sync (default for normal resources) | The application itself |
| `PostSync` | After all resources are synced and healthy | Smoke tests, notifications |
| `SyncFail` | Only if the sync fails | Cleanup, alerts |

### Hooks

A hook is a normal Kubernetes resource (often a `Job`) with annotations:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: myorg/migrations:1.4.0
          command: ["./migrate", "up"]
      restartPolicy: Never
```

| Hook delete policy | Meaning |
|--------------------|---------|
| `HookSucceeded` | Delete the hook resource after it succeeds. |
| `HookFailed` | Delete the hook resource after it fails. |
| `BeforeHookCreation` | Delete the previous hook resource before creating a new one (default). |

### Sync waves

Within a phase, **waves** order resources. Lower numbers go first; the default wave is `0`. ArgoCD waits for each wave to be healthy before starting the next.

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"   # namespace / config first
```

```text
  Wave -1 :  Namespace, ConfigMap, Secret
     |
     v   (wait until healthy)
  Wave  0 :  Deployment, Service        <- default
     |
     v   (wait until healthy)
  Wave  1 :  Ingress
```

---

## Projects

An **AppProject** is a boundary that restricts what applications can do. It is how ArgoCD supports multi-team and multi-tenant use.

A project can restrict:

- **Source repositories:** which Git repos applications may deploy from.
- **Destinations:** which clusters and namespaces applications may deploy to.
- **Resource kinds:** which cluster-scoped and namespaced resources are allowed or denied.
- **Roles:** project-scoped RBAC for teams, with optional JWT tokens for automation.
- **Sync windows:** time windows in which syncs are allowed or blocked.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-payments
  namespace: argocd
spec:
  description: Payments team applications

  sourceRepos:
    - https://github.com/my-org/payments-*

  destinations:
    - server: https://kubernetes.default.svc
      namespace: payments-*

  clusterResourceWhitelist: []          # no cluster-scoped resources allowed

  namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota

  roles:
    - name: developer
      description: Read and sync payments apps
      policies:
        - p, proj:team-payments:developer, applications, get, team-payments/*, allow
        - p, proj:team-payments:developer, applications, sync, team-payments/*, allow
      groups:
        - my-org:payments-devs
```

> The built-in `default` project allows any source, destination and resource. Create dedicated projects instead of putting everything in `default`.

---

## Repositories and Clusters

### Repositories

ArgoCD needs to know how to reach your Git repositories. Public repos work without credentials. Private repos need credentials, registered via UI, CLI or declaratively as Secrets.

```bash
# HTTPS with a token
argocd repo add https://github.com/my-org/private-repo.git \
  --username my-user \
  --password <token>

# SSH
argocd repo add git@github.com:my-org/private-repo.git \
  --ssh-private-key-path ~/.ssh/id_ed25519

argocd repo list
```

Repository credentials are stored as Kubernetes Secrets in the `argocd` namespace, labelled `argocd.argoproj.io/secret-type: repository`.

### Clusters

The cluster ArgoCD runs in is registered automatically as `https://kubernetes.default.svc` (named `in-cluster`). To deploy to other clusters, register them:

```bash
argocd cluster add <kubectl-context-name>
argocd cluster list
```

This creates a ServiceAccount in the target cluster and stores its credentials as a Secret in the `argocd` namespace.

```text
                   +----------------------+
                   |  ArgoCD (hub)        |
                   +----+-----------+-----+
                        |           |
          +-------------+           +-------------+
          v                                       v
 +------------------+                   +------------------+
 | dev cluster      |                   | prod cluster     |
 | (in-cluster)     |                   | (registered)     |
 +------------------+                   +------------------+
```

---

## Supported Manifest Formats

The Repository Server can render several formats. The source type is detected automatically from the contents of `path`, or can be set explicitly.

| Format | Detected by | Example source block |
|--------|-------------|----------------------|
| **Plain YAML / JSON** | A directory of manifests | `path: manifests/` |
| **Helm** | `Chart.yaml` | `helm: { valueFiles: [values-prod.yaml] }` |
| **Kustomize** | `kustomization.yaml` | `kustomize: { namePrefix: prod- }` |
| **Jsonnet** | `.jsonnet` files | `directory: { jsonnet: {...} }` |
| **Config management plugin** | Plugin configuration | Custom tooling |

### Helm example

```yaml
source:
  repoURL: https://github.com/my-org/charts.git
  targetRevision: main
  path: charts/webapp
  helm:
    valueFiles:
      - values-prod.yaml
    parameters:
      - name: image.tag
        value: "1.4.0"
```

### Kustomize example

```yaml
source:
  repoURL: https://github.com/my-org/app-config.git
  targetRevision: main
  path: overlays/production
```

---

## App of Apps and ApplicationSets

Managing dozens of Applications by hand does not scale. ArgoCD offers two patterns.

### App of Apps

One parent Application points at a directory that contains child `Application` manifests. Syncing the parent creates all the children.

```text
          +----------------------+
          |   root Application   |
          +----------+-----------+
                     | points to Git folder of Application manifests
        +------------+------------+
        v            v            v
  +-----------+ +-----------+ +-----------+
  |  app: web | | app: api  | | app: db   |
  +-----------+ +-----------+ +-----------+
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/gitops.git
    targetRevision: main
    path: apps               # folder full of Application manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### ApplicationSet

An `ApplicationSet` generates Applications from a template and a **generator**. Add a new cluster or folder, and a new Application appears automatically.

| Generator | Creates one Application per... |
|-----------|--------------------------------|
| **List** | Item in a fixed list |
| **Cluster** | Cluster registered in ArgoCD |
| **Git (directories)** | Directory in a Git repo |
| **Git (files)** | Config file in a Git repo |
| **Matrix / Merge** | Combination of other generators |
| **Pull Request** | Open pull request (preview environments) |

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: webapp-environments
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - env: dev
            namespace: webapp-dev
          - env: staging
            namespace: webapp-staging
          - env: prod
            namespace: webapp-prod
  template:
    metadata:
      name: "webapp-{{env}}"
    spec:
      project: default
      source:
        repoURL: https://github.com/my-org/webapp.git
        targetRevision: main
        path: "overlays/{{env}}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{namespace}}"
      syncPolicy:
        automated:
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

| | App of Apps | ApplicationSet |
|---|-------------|----------------|
| Approach | Hand-written child manifests in Git | Template + generator |
| Best for | A fixed, curated set of apps | Many similar apps (per cluster, per env, per team) |
| Adding a new app | Add a new manifest | Add an element, folder or cluster |

---

## ArgoCD Configuration Objects

ArgoCD is configured through ConfigMaps and Secrets in the `argocd` namespace.

| Object | Kind | Purpose |
|--------|------|---------|
| `argocd-cm` | ConfigMap | Main configuration: reconciliation timeout, resource customizations, SSO/Dex settings, status badge, UI banner. |
| `argocd-rbac-cm` | ConfigMap | Global RBAC policies and default role. |
| `argocd-cmd-params-cm` | ConfigMap | Command-line parameters for components (for example running the API server insecure behind an ingress). |
| `argocd-secret` | Secret | Admin password hash, server signing key, webhook secrets. |
| `argocd-initial-admin-secret` | Secret | One-time initial admin password. Delete after first login. |
| `argocd-notifications-cm` | ConfigMap | Notification triggers, templates and services. |
| Repository / cluster Secrets | Secret | Credentials, labelled with `argocd.argoproj.io/secret-type`. |

### Default RBAC at a glance

```text
  p, role:org-admin, applications, *,    */*, allow
  p, role:readonly,  applications, get,  */*, allow
  g, my-org:platform-team, role:org-admin
```

Policies follow the form `p, <subject>, <resource>, <action>, <object>, <allow|deny>`.

---

## Hands-On: Explore Your Installation

Use the cluster from the installation module to see these components for yourself.

### List the components

```bash
kubectl get pods -n argocd
```

Expected pods (names will have suffixes):

```text
argocd-application-controller-0
argocd-applicationset-controller-...
argocd-dex-server-...
argocd-notifications-controller-...
argocd-redis-...
argocd-repo-server-...
argocd-server-...
```

### See the workload types

```bash
kubectl get deployments,statefulsets -n argocd
```

> The Application Controller is a **StatefulSet**. Everything else is a Deployment.

### Inspect ArgoCD's custom resources

```bash
kubectl get crds | grep argoproj.io
```

You should see `applications.argoproj.io`, `appprojects.argoproj.io` and `applicationsets.argoproj.io`.

### Look at the configuration

```bash
kubectl get configmap -n argocd
kubectl get configmap argocd-cm -n argocd -o yaml
```

### Create your first Application declaratively

Save the following as `guestbook.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

Apply it and watch the status:

```bash
kubectl apply -f guestbook.yaml
kubectl get applications -n argocd
```

The app appears as `OutOfSync` because the sync policy is manual. Sync it:

```bash
argocd app sync guestbook
argocd app get guestbook
```

### Watch self-heal in action

Enable automated sync with self-heal:

```bash
argocd app set guestbook --sync-policy automated --self-heal
```

Break the app on purpose:

```bash
kubectl scale deployment guestbook-ui -n guestbook --replicas=5
kubectl get deployment guestbook-ui -n guestbook -w
```

ArgoCD detects the drift and reverts the replica count to what Git defines.

### Clean up

```bash
argocd app delete guestbook --cascade
```

---

## Common Pitfalls

| Pitfall | Why it happens | What to do |
|---------|----------------|------------|
| App stuck `OutOfSync` right after sync | Controllers or mutating webhooks change fields after apply (defaults, injected sidecars) | Use `ignoreDifferences` for those fields |
| Deleted a file in Git but the resource is still running | `prune` is off by default | Enable `prune: true` or sync with `--prune` |
| Manual `kubectl` changes keep disappearing | `selfHeal` is on | Make the change in Git instead |
| Applications deployed to the wrong place | Using the `default` project with no restrictions | Create AppProjects and limit destinations |
| Changes take minutes to appear | Default polling interval is about 3 minutes | Configure a Git webhook |
| Deleting an Application leaves resources behind | Missing cascade finalizer | Add `resources-finalizer.argocd.argoproj.io` or delete with `--cascade` |
| Secrets committed to Git in plain text | Git is the source of truth for everything | Use Sealed Secrets, External Secrets Operator or SOPS (covered in a later module) |

---

## Knowledge Check

1. Which component renders Helm charts and Kustomize overlays into plain manifests?
2. Where does ArgoCD store Applications and repository credentials?
3. What is the difference between *sync status* and *health status*?
4. An app shows `Synced` and `Degraded`. Is the problem in Git or in the cluster?
5. What do `prune` and `selfHeal` each do?
6. How do you run a database migration before the application deploys?
7. Which resource restricts the repos and namespaces a team may deploy to?
8. When would you pick an ApplicationSet over App of Apps?
9. How does ArgoCD usually detect a new commit, and how can you make it faster?
10. Which component is optional and only needed for SSO?

<details>
<summary>Answers</summary>

1. The **Repository Server** (`argocd-repo-server`).
2. In Kubernetes: Applications and Projects are custom resources; credentials are Secrets in the `argocd` namespace. There is no separate database.
3. Sync status asks whether live state matches Git; health status asks whether the resources are actually working.
4. The cluster matches Git, so the issue is with the workload itself (image, config, probes). The fix usually goes in Git.
5. `prune` deletes cluster resources that were removed from Git. `selfHeal` reverts manual changes made directly in the cluster.
6. A `PreSync` hook, usually a `Job` annotated with `argocd.argoproj.io/hook: PreSync`.
7. An **AppProject**.
8. When you need many similar Applications (per cluster, environment or team) generated from a template.
9. By polling Git (about every 3 minutes by default). A Git provider webhook makes it near-instant.
10. **Dex**.

</details>

---

## Summary

- ArgoCD is a set of cooperating components: **API Server**, **Repository Server**, **Application Controller**, **Redis**, and optional **Dex**, **ApplicationSet** and **Notifications** controllers.
- All ArgoCD state lives in Kubernetes objects. There is no external database.
- The controller runs a **reconciliation loop** comparing **desired state** (Git) with **live state** (cluster).
- **Sync status** tells you whether the cluster matches Git; **health status** tells you whether it works.
- **Sync policies** control automation; `prune` and `selfHeal` decide how strictly Git is enforced.
- **Hooks** and **waves** control ordering within a deployment.
- **AppProjects** provide guardrails; **App of Apps** and **ApplicationSets** scale management across many applications.

---

## Series Navigation

| Module | Topic |
|--------|-------|
| [Fundamentals of GitOps][mod-01] | Principles, pull vs push, Git as the source of truth |
| [ArgoCD Installation][mod-02] | Installing ArgoCD and accessing the UI and CLI |
| **ArgoCD Architecture and Core Concepts** | You are here |

<!-- Reference-style link definitions -->
[mod-01]: ../01-fundamentals-of-gitops/README.md
[mod-02]: ../02-argocd-installation/README.md
[badge-argocd]: https://img.shields.io/badge/Argo%20CD-EF7B4D?logo=argo&logoColor=white
[badge-gitops]: https://img.shields.io/badge/GitOps-Declarative%20Delivery-blue
[badge-k8s]: https://img.shields.io/badge/Kubernetes-326CE5?logo=kubernetes&logoColor=white
[badge-level]: https://img.shields.io/badge/Level-Beginner%20to%20Intermediate-brightgreen
