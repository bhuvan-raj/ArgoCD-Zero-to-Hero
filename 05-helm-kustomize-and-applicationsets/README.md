# Helm, Kustomize and ApplicationSets

![ArgoCD][badge-argocd] ![Helm][badge-helm] ![Kustomize][badge-kustomize] ![Kubernetes][badge-k8s] ![Level][badge-level]

> Managing real-world configuration with ArgoCD: how it renders Helm charts and Kustomize overlays, how to structure environments in Git, and how to generate Applications at scale with ApplicationSets.

This module builds on [Architecture and Core Concepts][mod-03] and [Application Management and Sync][mod-04]. Those modules introduced the source types, value overrides and the ApplicationSet idea. Here we go deeper.

---

## Table of Contents

- [Learning Objectives](#learning-objectives)
- [The Big Picture](#the-big-picture)
- [Choosing a Config Tool](#choosing-a-config-tool)
- [Helm with ArgoCD](#helm-with-argocd)
- [Kustomize with ArgoCD](#kustomize-with-argocd)
- [Helm and Kustomize Together](#helm-and-kustomize-together)
- [Structuring Environments in Git](#structuring-environments-in-git)
- [Why ApplicationSets?](#why-applicationsets)
- [ApplicationSet Anatomy](#applicationset-anatomy)
- [Generators](#generators)
- [Templating](#templating)
- [Combining Generators](#combining-generators)
- [ApplicationSet Sync and Deletion Policies](#applicationset-sync-and-deletion-policies)
- [Progressive Syncs](#progressive-syncs)
- [Hands-On Lab](#hands-on-lab)
- [Common Pitfalls](#common-pitfalls)
- [Best Practices](#best-practices)
- [Knowledge Check](#knowledge-check)
- [Summary](#summary)
- [Series Navigation](#series-navigation)

---

## Learning Objectives

By the end of this module you will be able to:

- Explain how ArgoCD renders Helm charts and how that differs from `helm install`.
- Use Helm values files, inline values and parameters, and know which one wins.
- Build Kustomize bases and overlays and deploy them with ArgoCD.
- Choose a sensible Git layout for multiple environments.
- Write ApplicationSets using the list, cluster, Git, matrix, merge and pull request generators.
- Control what happens to generated Applications on update and deletion.

---

## The Big Picture

ArgoCD does not care how manifests are produced. The Repository Server turns *whatever is in Git* into plain Kubernetes YAML, and the Application Controller deploys that YAML.

```text
   Git repository
   +---------------------------------------------+
   |  Helm chart   Kustomize overlay   Plain YAML |
   +----------------------+----------------------+
                          |
                          v
               +---------------------+
               |   Repository Server |
               |  helm template      |
               |  kustomize build    |
               |  (or read as-is)    |
               +----------+----------+
                          |
                          v
                 Plain Kubernetes YAML
                          |
                          v
               +---------------------+
               | Application         |
               | Controller          |
               | diff + sync         |
               +---------------------+
```

> **Key idea:** the *rendered output* is what gets compared and synced. Changing a Helm value or a Kustomize patch changes the output, which ArgoCD sees as a difference from live state.

---

## Choosing a Config Tool

| | Plain YAML | Helm | Kustomize |
|---|-----------|------|-----------|
| **Approach** | Copy and edit | Go templates + values | Patch and overlay, no templating |
| **Best for** | Tiny apps, learning | Third-party charts, parameterised apps | Your own apps with small per-environment differences |
| **Learning curve** | Lowest | Medium | Medium |
| **Reuse** | Copy-paste | Chart + values | Base + overlays |
| **Readability** | Highest | Can get dense | Good |
| **Packaging and versioning** | None | Chart versions, repositories | Git revision only |

A common, practical rule:

- **Third-party software** (Prometheus, ingress controllers, cert-manager): use their **Helm chart**.
- **Your own services**: use **Kustomize** or a Helm chart you maintain.
- Mixing is normal. ArgoCD treats each Application independently.

---

## Helm with ArgoCD

### ArgoCD uses `helm template`, not `helm install`

This surprises many people. ArgoCD runs **`helm template`** to render YAML, then applies it itself.

| | `helm install` | ArgoCD |
|---|----------------|--------|
| Renders templates | Yes | Yes (`helm template`) |
| Creates a Helm release object (Secret) | Yes | **No** |
| `helm list` shows it | Yes | **No** |
| `helm rollback` works | Yes | **No** (use ArgoCD history or Git) |
| Tracks resources | Helm release | ArgoCD tracking labels/annotations |

Consequences:

- Do not manage the same release with both Helm CLI and ArgoCD.
- Anything that depends on Helm release state (for example `lookup` against the release itself) behaves differently.
- ArgoCD is the single owner of the lifecycle.

### Helm chart from Git

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: webapp-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/gitops.git
    targetRevision: main
    path: charts/webapp
    helm:
      releaseName: webapp
      valueFiles:
        - values.yaml
        - ../../envs/prod/webapp-values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: webapp
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

### Helm chart from a chart repository

```yaml
source:
  repoURL: https://prometheus-community.github.io/helm-charts
  chart: kube-prometheus-stack
  targetRevision: 65.1.0          # chart version, not Git revision
  helm:
    releaseName: monitoring
    valuesObject:
      grafana:
        enabled: true
      prometheus:
        prometheusSpec:
          retention: 15d
```

When `chart:` is set instead of `path:`, `targetRevision` is the **chart version**. Pin it. A floating version means deployments change without a commit.

### Helm chart from an OCI registry

```yaml
source:
  repoURL: registry.example.com/charts      # no oci:// prefix in the Application
  chart: webapp
  targetRevision: 1.4.0
```

The OCI registry must be registered in ArgoCD as a Helm repository with OCI enabled, with credentials if it is private.

### Value precedence

Several ways exist to provide values. When the same key is set in more than one place, **later wins**:

```text
   chart's own values.yaml        (lowest priority)
          |
          v
   helm.valueFiles   (in listed order, later files override earlier ones)
          |
          v
   helm.values / helm.valuesObject   (inline)
          |
          v
   helm.parameters   (like --set)    (highest priority)
```

| Field | Format | Typical use |
|-------|--------|-------------|
| `valueFiles` | Paths to files in the repo | Per-environment values kept in Git. **Preferred.** |
| `values` | Inline YAML as a string | Quick overrides |
| `valuesObject` | Inline YAML as a structured object | Same, but cleaner and mergeable |
| `parameters` | Name/value pairs | One-off overrides such as an image tag set by CI |
| `fileParameters` | Name + file path | Load a parameter's value from a file |

### Useful Helm options

| Option | Purpose |
|--------|---------|
| `releaseName` | Release name used in templates (`.Release.Name`). Defaults to the Application name. |
| `valueFiles` | List of values files. |
| `ignoreMissingValueFiles: true` | Do not fail if an optional values file is absent. |
| `skipCrds: true` | Do not install the chart's CRDs. |
| `passCredentials: true` | Send repository credentials to chart download URLs on other domains. |
| `version` | Helm major version to use. |

```yaml
helm:
  releaseName: webapp
  skipCrds: false
  ignoreMissingValueFiles: true
  valueFiles:
    - values.yaml
    - values-prod.yaml
    - values-local-overrides.yaml   # optional
```

### Helm hooks become ArgoCD hooks

Helm hook annotations are translated into ArgoCD sync phases:

| Helm hook | ArgoCD phase |
|-----------|--------------|
| `pre-install`, `pre-upgrade` | `PreSync` |
| `post-install`, `post-upgrade` | `PostSync` |
| `test` hooks | Ignored |
| Other hooks (`pre-delete`, `post-delete`, `post-rollback`, etc.) | Not mapped to sync phases in all versions. Check the documentation for your ArgoCD version. |

Hook delete policies map across too: `before-hook-creation` becomes `BeforeHookCreation`, `hook-succeeded` becomes `HookSucceeded`, `hook-failed` becomes `HookFailed`.

> Unsupported hooks are ignored. If a chart relies heavily on Helm hooks, test it with ArgoCD in a non-production cluster first.

### Helm dependencies

If a chart declares dependencies (`Chart.yaml`), the Repository Server resolves them when rendering. Commit `Chart.lock` so the resolved versions are reproducible, and make sure the dependency repositories are reachable from the repo server.

### Inspect what ArgoCD renders

```bash
argocd app manifests webapp-prod
```

To reproduce locally for debugging:

```bash
helm template webapp charts/webapp \
  -f charts/webapp/values.yaml \
  -f envs/prod/webapp-values.yaml \
  --namespace webapp
```

---

## Kustomize with ArgoCD

Kustomize customises plain YAML without templates. You keep a **base** and layer **overlays** on top.

```text
   app/
   +-- base/
   |   +-- deployment.yaml
   |   +-- service.yaml
   |   +-- kustomization.yaml
   +-- overlays/
       +-- dev/
       |   +-- kustomization.yaml
       +-- staging/
       |   +-- kustomization.yaml
       +-- prod/
           +-- kustomization.yaml
           +-- replica-patch.yaml
```

### Base

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

### Overlay

```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namePrefix: prod-
namespace: webapp-prod

images:
  - name: my-org/webapp
    newTag: 1.4.0

replicas:
  - name: webapp
    count: 4

patches:
  - path: replica-patch.yaml
```

### Pointing ArgoCD at an overlay

```yaml
source:
  repoURL: https://github.com/my-org/gitops.git
  targetRevision: main
  path: app/overlays/prod        # ArgoCD detects kustomization.yaml
```

ArgoCD runs `kustomize build` on that path and deploys the result.

### Overrides on the Application

```yaml
source:
  path: app/overlays/prod
  kustomize:
    namePrefix: prod-
    images:
      - my-org/webapp:1.4.0
```

| Field | Purpose |
|-------|---------|
| `namePrefix` / `nameSuffix` | Add a prefix or suffix to resource names. |
| `images` | Override image names or tags. |
| `namespace` | Set the namespace on all resources. |
| `replicas` | Override replica counts. |
| `commonLabels` / `commonAnnotations` | Add labels or annotations everywhere. |
| `patches` | Inline patches. |
| `components` | Apply reusable Kustomize components. |

> **A common CI pattern:** the pipeline builds an image, then updates the `images:` entry (or `newTag`) in the overlay and commits it. ArgoCD sees the commit and deploys. Git stays the record of what is running.

### Kustomize build options

Global flags are set in the `argocd-cm` ConfigMap:

```yaml
data:
  kustomize.buildOptions: --load-restrictor LoadRestrictionsNone
```

> Loosening restrictions or enabling extra features reduces the safety of manifest rendering. Do it deliberately and only where needed.

---

## Helm and Kustomize Together

There are two common ways to combine them.

### Option 1: Kustomize inflates a Helm chart

Kustomize can render a chart itself with `helmCharts`:

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

helmCharts:
  - name: ingress-nginx
    repo: https://kubernetes.github.io/ingress-nginx
    version: 4.11.3
    releaseName: ingress
    namespace: ingress-nginx
    valuesFile: values.yaml

patches:
  - path: add-annotations.yaml
```

ArgoCD must be told to allow it:

```yaml
# argocd-cm
data:
  kustomize.buildOptions: --enable-helm
```

This lets you patch a third-party chart's output without forking the chart.

### Option 2: Multiple sources

Keep the chart where it lives and your values in your own repo. See [Multiple Sources][mod-04-multi] in module 04.

| | Kustomize `helmCharts` | Multiple sources |
|---|------------------------|------------------|
| Post-render patching | Yes | No |
| Requires extra ArgoCD config | `--enable-helm` | No |
| Values in your own repo | Yes | Yes |
| Simplicity | Medium | High |

---

## Structuring Environments in Git

How you lay out Git is one of the most important design decisions in GitOps.

### Pattern 1: Folder per environment (recommended to start)

One branch, one folder per environment.

```text
   gitops-repo/
   +-- apps/
   |   +-- webapp/
   |       +-- base/
   |       +-- overlays/
   |           +-- dev/
   |           +-- staging/
   |           +-- prod/
   +-- applicationsets/
       +-- webapp.yaml
```

**Pros:** easy to see all environments side by side, simple promotion by copying or editing a folder, works well with ApplicationSet Git directory generators.

### Pattern 2: Branch per environment

`dev`, `staging` and `main` branches, each tracked by the matching Application.

**Cons:** drift between branches, painful merges, harder to review a promotion. Generally discouraged.

### Pattern 3: Separate repositories

An **app repo** (source code, built by CI) and a **config repo** (manifests, watched by ArgoCD).

```text
   App repo                       Config repo
   +----------------+  CI builds   +-----------------------+
   | source code    |------------->| image tag updated in  |
   | Dockerfile     |  and commits | overlays/<env>        |
   +----------------+              +-----------+-----------+
                                               |
                                               v
                                           ArgoCD
```

**Pros:** clean separation of concerns, config commits do not trigger application builds, access control per repo.

### Promotion across environments

```text
   dev  ------>  staging  ------>  prod
    |               |                |
  auto-sync    auto-sync        manual sync
                                (or PR-gated)
```

Promote by changing the image tag or values in the next environment's folder, via a pull request. The PR is the approval gate and the audit trail.

---

## Why ApplicationSets?

Imagine deploying one service to three environments and three clusters. Hand-writing nine `Application` manifests is repetitive and error-prone. Adding a tenth means copy, paste, edit.

An **ApplicationSet** replaces that with a template plus a generator.

```text
   +---------------------+      +---------------------+
   |  Generator          |      |  Template           |
   |  (where do values   |  +   |  (shape of one      |
   |   come from?)       |      |   Application)      |
   +----------+----------+      +----------+----------+
              |                            |
              +-------------+--------------+
                            |
                            v
              +-----------------------------+
              |  ApplicationSet controller  |
              +--------------+--------------+
                             |
          +------------------+------------------+
          v                  v                  v
   +-------------+    +-------------+    +-------------+
   | Application |    | Application |    | Application |
   |   dev       |    |   staging   |    |   prod      |
   +-------------+    +-------------+    +-------------+
```

The ApplicationSet controller continuously reconciles: add a cluster, folder or list item and an Application appears; remove it and the Application is removed.

---

## ApplicationSet Anatomy

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: webapp
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]

  generators:                      # 1. where do the parameters come from?
    - list:
        elements:
          - env: dev
          - env: prod

  template:                        # 2. what does each Application look like?
    metadata:
      name: "webapp-{{ .env }}"
    spec:
      project: default
      source:
        repoURL: https://github.com/my-org/gitops.git
        targetRevision: main
        path: "apps/webapp/overlays/{{ .env }}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "webapp-{{ .env }}"

  syncPolicy:                      # 3. how does the ApplicationSet itself behave?
    preserveResourcesOnDeletion: false
```

The `template` section is a normal Application spec with placeholders. Each generated parameter set fills in the placeholders.

---

## Generators

### List generator

A fixed list you write yourself.

```yaml
generators:
  - list:
      elements:
        - cluster: dev
          url: https://kubernetes.default.svc
        - cluster: prod
          url: https://prod.example.com
```

**Use when:** a small, rarely changing set of targets.

### Cluster generator

One Application per cluster registered in ArgoCD.

```yaml
generators:
  - clusters:
      selector:
        matchLabels:
          env: prod            # only clusters labelled env=prod
```

Available parameters include `name`, `server`, and cluster labels under `metadata.labels`.

```yaml
template:
  metadata:
    name: "monitoring-{{ .name }}"
  spec:
    destination:
      server: "{{ .server }}"
      namespace: monitoring
```

Register and label a cluster:

```bash
argocd cluster add my-prod-context --label env=prod
```

**Use when:** deploying the same add-on (monitoring agent, ingress, policy engine) to every cluster. Adding a cluster automatically deploys it.

### Git directory generator

One Application per **folder** in a Git repository.

```yaml
generators:
  - git:
      repoURL: https://github.com/my-org/gitops.git
      revision: main
      directories:
        - path: apps/*
        - path: apps/legacy-*
          exclude: true
```

Parameters: `path.path` (full path), `path.basename` (folder name), and `path.segments`.

```yaml
template:
  metadata:
    name: "{{ .path.basename }}"
  spec:
    source:
      repoURL: https://github.com/my-org/gitops.git
      targetRevision: main
      path: "{{ .path.path }}"
    destination:
      server: https://kubernetes.default.svc
      namespace: "{{ .path.basename }}"
```

**Use when:** one folder per app or per environment. Adding a folder to Git creates the Application. This is the most GitOps-friendly generator.

### Git file generator

One Application per **config file** (JSON or YAML) found in Git. The file's contents become parameters.

```yaml
generators:
  - git:
      repoURL: https://github.com/my-org/gitops.git
      revision: main
      files:
        - path: "envs/**/config.json"
```

```json
{
  "env": "prod",
  "cluster": "https://prod.example.com",
  "replicas": 5
}
```

**Use when:** each environment needs several parameters (cluster, replica count, hostname) and you want them in Git as data, not hard-coded in the ApplicationSet.

### Pull request generator

One Application per open **pull request**, for preview environments.

```yaml
generators:
  - pullRequest:
      github:
        owner: my-org
        repo: webapp
        tokenRef:
          secretName: github-token
          key: token
        labels:
          - preview
      requeueAfterSeconds: 120
```

Parameters include `number`, `branch`, `branch_slug` and `head_sha`.

```yaml
template:
  metadata:
    name: "webapp-pr-{{ .number }}"
  spec:
    source:
      repoURL: https://github.com/my-org/webapp.git
      targetRevision: "{{ .head_sha }}"
      path: deploy
    destination:
      server: https://kubernetes.default.svc
      namespace: "preview-{{ .number }}"
    syncPolicy:
      automated:
        prune: true
      syncOptions:
        - CreateNamespace=true
```

When the PR is closed or merged, the Application and its namespace contents are removed.

> Configure a webhook for pull requests, otherwise the generator only notices changes on its polling interval.

### SCM provider generator

One Application per **repository** discovered in a GitHub organisation, GitLab group or Bitbucket project, optionally filtered by path or label.

**Use when:** every repo in an organisation follows the same convention and should be deployed automatically.

### Generator summary

| Generator | One Application per... | Typical use |
|-----------|------------------------|-------------|
| List | Entry in a list | Few fixed targets |
| Cluster | Registered cluster | Cluster add-ons |
| Git directory | Folder in Git | One folder per app or environment |
| Git file | Config file in Git | Per-environment parameters as data |
| Pull request | Open pull request | Preview environments |
| SCM provider | Repository in an organisation | Org-wide conventions |
| Matrix | Combination of two generators | Apps x clusters |
| Merge | Merged result of generators | Defaults with overrides |

---

## Templating

ApplicationSets support two template engines.

| Engine | Syntax | Enabled by |
|--------|--------|------------|
| Fasttemplate (default, legacy) | `{{ env }}` or `{{path.basename}}` | Nothing |
| **Go templates** (recommended) | `{{ .env }}`, `{{ .path.basename }}` | `goTemplate: true` |

Go templates give you conditionals, functions and nested access. Set `missingkey=error` so a typo in a parameter name fails loudly instead of producing a blank value.

```yaml
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  template:
    metadata:
      name: "{{ .path.basename }}-{{ .env }}"
      labels:
        team: "{{ .team | lower }}"
```

### Per-element differences with `templatePatch`

When most Applications are alike but a few need extra fields (for example automated sync only in dev), use `templatePatch`:

```yaml
spec:
  goTemplate: true
  generators:
    - list:
        elements:
          - env: dev
            autoSync: "true"
          - env: prod
            autoSync: "false"
  template:
    metadata:
      name: "webapp-{{ .env }}"
    spec:
      project: default
      # ... source and destination ...
  templatePatch: |
    {{- if eq .autoSync "true" }}
    spec:
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
    {{- end }}
```

---

## Combining Generators

### Matrix generator

Produces the **cross product** of two generators.

```text
   Git directories:  [ frontend, backend ]
                          x
   Clusters:         [ dev, prod ]
                          =
   4 Applications:   frontend-dev, frontend-prod,
                     backend-dev,  backend-prod
```

```yaml
generators:
  - matrix:
      generators:
        - git:
            repoURL: https://github.com/my-org/gitops.git
            revision: main
            directories:
              - path: apps/*
        - clusters:
            selector:
              matchLabels:
                tier: standard
template:
  metadata:
    name: "{{ .path.basename }}-{{ .name }}"
  spec:
    source:
      repoURL: https://github.com/my-org/gitops.git
      targetRevision: main
      path: "{{ .path.path }}"
    destination:
      server: "{{ .server }}"
      namespace: "{{ .path.basename }}"
```

**Use when:** every app should go to every cluster in a group.

### Merge generator

Combines the output of several generators by a shared key, letting later generators **override** earlier ones. Use it for defaults plus exceptions:

```yaml
generators:
  - merge:
      mergeKeys:
        - server
      generators:
        - clusters: {}                # defaults for every cluster
        - list:
            elements:
              - server: https://prod.example.com
                replicas: "5"         # override for one cluster
```

---

## ApplicationSet Sync and Deletion Policies

### What happens when the ApplicationSet changes

The controller keeps generated Applications in line with the template. By default it can create, update and delete them. You can restrict this:

```yaml
spec:
  syncPolicy:
    applicationsSync: create-only
```

| `applicationsSync` | Create | Update | Delete |
|--------------------|:------:|:------:|:------:|
| `sync` (default) | Yes | Yes | Yes |
| `create-update` | Yes | Yes | No |
| `create-delete` | Yes | No | Yes |
| `create-only` | Yes | No | No |

`create-only` is useful when teams are expected to customise generated Applications by hand and you do not want the ApplicationSet to overwrite them.

### What happens when the ApplicationSet is deleted

Deleting an ApplicationSet deletes the Applications it generated. If those Applications carry the cascade finalizer (`resources-finalizer.argocd.argoproj.io`), their **workloads are deleted too**.

To keep the deployed resources:

```yaml
spec:
  syncPolicy:
    preserveResourcesOnDeletion: true
```

> **Treat ApplicationSet deletion as dangerous.** A single `kubectl delete applicationset` can remove every environment. Protect with RBAC, and consider `preserveResourcesOnDeletion: true` for production sets.

### Letting other controllers change generated Applications

If something else legitimately edits generated Applications (for example an image updater changing a target revision), tell the controller to ignore those fields:

```yaml
spec:
  ignoreApplicationDifferences:
    - jsonPointers:
        - /spec/source/targetRevision
```

---

## Progressive Syncs

By default, all generated Applications sync as soon as they differ. **Progressive syncs** let an ApplicationSet roll out changes in stages, for example dev first, then staging, then prod.

```yaml
spec:
  strategy:
    type: RollingSync
    rollingSync:
      steps:
        - matchExpressions:
            - key: env
              operator: In
              values: [dev]
        - matchExpressions:
            - key: env
              operator: In
              values: [staging]
          maxUpdate: 50%
        - matchExpressions:
            - key: env
              operator: In
              values: [prod]
          maxUpdate: 1
```

```text
   Step 1: dev   ---> wait until Healthy
   Step 2: staging ---> wait until Healthy
   Step 3: prod  ---> one at a time
```

> Progressive syncs have been an **alpha** feature and must be explicitly enabled on the ApplicationSet controller. Check the documentation for your ArgoCD version before relying on it in production.

---

## Hands-On Lab

**Goal:** deploy one Kustomize app to three environments from a single ApplicationSet, then add an environment by adding a folder.

### Prerequisites

- ArgoCD installed (module 02) and the `argocd` CLI logged in.
- A Git repository you control.

### Step 1: Create the repository layout

```text
   gitops-lab/
   +-- apps/
       +-- webapp/
           +-- base/
           |   +-- deployment.yaml
           |   +-- service.yaml
           |   +-- kustomization.yaml
           +-- overlays/
               +-- dev/
               |   +-- kustomization.yaml
               +-- staging/
               |   +-- kustomization.yaml
               +-- prod/
                   +-- kustomization.yaml
```

`base/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: webapp
          image: nginx:1.27
          ports:
            - containerPort: 80
```

`base/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  selector:
    app: webapp
  ports:
    - port: 80
      targetPort: 80
```

`base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

`overlays/dev/kustomization.yaml` (repeat for `staging` and `prod`, changing the namespace and replica count):

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namespace: webapp-dev
replicas:
  - name: webapp
    count: 1
```

Use `count: 2` for staging and `count: 3` for prod.

Commit and push everything.

### Step 2: Render locally to verify

```bash
kubectl kustomize apps/webapp/overlays/prod
```

You should see the Deployment with 3 replicas in namespace `webapp-prod`.

### Step 3: Create the ApplicationSet

```yaml
# webapp-appset.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: webapp
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - git:
        repoURL: https://github.com/<you>/gitops-lab.git
        revision: main
        directories:
          - path: apps/webapp/overlays/*
  template:
    metadata:
      name: "webapp-{{ .path.basename }}"
    spec:
      project: default
      source:
        repoURL: https://github.com/<you>/gitops-lab.git
        targetRevision: main
        path: "{{ .path.path }}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "webapp-{{ .path.basename }}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

```bash
kubectl apply -f webapp-appset.yaml
```

### Step 4: Verify

```bash
kubectl get applicationset -n argocd
argocd app list
kubectl get deploy -A | grep webapp
```

You should see three Applications: `webapp-dev`, `webapp-staging` and `webapp-prod`, each deployed to its own namespace with the right replica count.

### Step 5: Add an environment by adding a folder

Copy `overlays/prod` to `overlays/qa`, change the namespace to `webapp-qa`, commit and push.

```bash
argocd app list
```

A new `webapp-qa` Application appears **without touching the ApplicationSet**.

### Step 6: Promote an image

Change the image tag in `overlays/dev/kustomization.yaml`:

```yaml
images:
  - name: nginx
    newTag: "1.27.2"
```

Commit and push. Only `webapp-dev` changes. Repeat for staging and prod to simulate promotion.

### Step 7: Switch to a Helm-based variation (optional)

Replace the Kustomize overlays with a Helm chart and per-environment values files, and change the Application template to use `helm.valueFiles`. Compare how each approach feels for your own apps.

### Step 8: Clean up

```bash
kubectl delete applicationset webapp -n argocd
```

Because the generated Applications use the default behaviour, they are deleted along with the ApplicationSet. Confirm and remove the leftover namespaces if any remain:

```bash
kubectl get applications -n argocd
kubectl get ns | grep webapp
```

---

## Common Pitfalls

| Pitfall | Why it happens | What to do |
|---------|----------------|------------|
| `helm list` shows nothing for ArgoCD apps | ArgoCD uses `helm template`, so no Helm release exists | Use `argocd app get` and ArgoCD history |
| Floating chart version deploys unexpectedly | `targetRevision: "*"` or a range | Pin an exact chart version |
| Value override does not apply | A higher-priority source (parameters) wins | Check precedence; prefer `valueFiles` and avoid mixing |
| Kustomize overlay cannot find `../../base` | Wrong `path` in the Application | Point at the overlay, not the base |
| Third-party chart cannot be patched | Charts are rendered, not editable | Use Kustomize `helmCharts` or chart values |
| ApplicationSet generates nothing | Generator matches no folders, clusters or files | Check paths and selectors; read the ApplicationSet controller logs |
| Template placeholder is blank | Typo in a parameter name with the default template engine | Use `goTemplate: true` with `missingkey=error` |
| Application changes keep being reverted | The ApplicationSet overwrites manual edits | Use `create-only` or `ignoreApplicationDifferences` |
| Deleting an ApplicationSet wiped an environment | Cascade finalizer on generated Applications | Use `preserveResourcesOnDeletion: true` and protect with RBAC |
| Pull request Applications appear slowly | Only polling, no webhook | Configure a webhook and lower `requeueAfterSeconds` |
| Generated Application name too long or invalid | Combined parameters exceed Kubernetes name limits | Shorten names; use `nameNormalized` or Go template functions |

### Debugging ApplicationSets

```bash
kubectl get applicationset webapp -n argocd -o yaml          # conditions and status
kubectl describe applicationset webapp -n argocd
kubectl logs -n argocd deploy/argocd-applicationset-controller
```

---

## Best Practices

- **Pin chart versions** and image tags. Never track `latest` or a floating range.
- Prefer **values files in Git** over inline parameters, so changes are reviewable.
- Use **Kustomize for your own apps** and **Helm for third-party software**, and do not be afraid to mix them.
- Do **not** manage one release with both Helm CLI and ArgoCD.
- Start with a **folder-per-environment** layout; avoid long-lived branches per environment.
- Separate **app repo** and **config repo** when CI and deployment ownership differ.
- Use **Go templates** with `missingkey=error` in ApplicationSets.
- Prefer the **Git directory generator** for app and environment discovery, and the **cluster generator** for add-ons.
- Gate production promotion with **pull requests** or manual sync.
- Protect ApplicationSets: restrict who can edit or delete them, and use `preserveResourcesOnDeletion` where appropriate.
- Always **render locally** (`helm template`, `kubectl kustomize`) before committing a big change.

---

## Knowledge Check

1. Does ArgoCD run `helm install`? What does it run instead?
2. Why does `helm list` not show applications deployed by ArgoCD?
3. Which Helm value source has the highest priority?
4. When `chart:` is used in a source, what does `targetRevision` mean?
5. How do you apply a Kustomize overlay in ArgoCD?
6. Name two ways to combine Helm charts with your own configuration.
7. Which generator would you use to deploy an ingress controller to every registered cluster?
8. Which generator creates preview environments for pull requests?
9. What does the matrix generator produce?
10. What happens to workloads if you delete an ApplicationSet, and how can you prevent it?

<details>
<summary>Answers</summary>

1. No. It runs `helm template` to render manifests and applies them itself.
2. No Helm release object is created. ArgoCD tracks resources with its own labels or annotations.
3. `helm.parameters`, which behave like `--set`.
4. The chart version, not a Git revision.
5. Set `path` to the overlay directory containing `kustomization.yaml`. ArgoCD runs `kustomize build` on it.
6. Kustomize `helmCharts` (needs `--enable-helm`), or an Application with multiple sources.
7. The cluster generator.
8. The pull request generator.
9. The cross product of two generators, for example every app on every cluster.
10. The generated Applications are deleted, and with the cascade finalizer their workloads go too. Use `preserveResourcesOnDeletion: true` to keep them.

</details>

---

## Summary

- ArgoCD renders **Helm** with `helm template` and **Kustomize** with `kustomize build`, then syncs the resulting plain YAML.
- There is **no Helm release** object, so ArgoCD is the sole owner of the lifecycle.
- Helm value precedence runs from `valueFiles` up to inline values and finally `parameters`.
- **Kustomize** suits your own apps with small per-environment differences; **Helm** suits third-party charts and heavily parameterised apps.
- A **folder-per-environment** layout, optionally split into an app repo and a config repo, is the most practical Git structure.
- **ApplicationSets** generate Applications from a template and a generator: list, cluster, Git directory, Git file, pull request, SCM provider, matrix and merge.
- Use **Go templates**, control behaviour with `applicationsSync` and `preserveResourcesOnDeletion`, and treat ApplicationSet deletion as a high-risk action.

---

## Series Navigation

| Module | Topic |
|--------|-------|
| [Fundamentals of GitOps][mod-01] | Principles, pull vs push, Git as the source of truth |
| [ArgoCD Installation][mod-02] | Installing ArgoCD and accessing the UI and CLI |
| [ArgoCD Architecture and Core Concepts][mod-03] | Components, reconciliation, Applications, Projects |
| [Application Management and Sync][mod-04] | Creating, syncing, drift, rollback, deletion, troubleshooting |
| **Helm, Kustomize and ApplicationSets** | You are here |

<!-- Reference-style link definitions -->
[mod-01]: ../01-fundamentals-of-gitops/README.md
[mod-02]: ../02-argocd-installation/README.md
[mod-03]: ../03-argocd-architecture-and-core-concepts/README.md
[mod-04]: ../04-application-management-and-sync/README.md
[mod-04-multi]: ../04-application-management-and-sync/README.md#multiple-sources
[badge-argocd]: https://img.shields.io/badge/Argo%20CD-EF7B4D?logo=argo&logoColor=white
[badge-helm]: https://img.shields.io/badge/Helm-0F1689?logo=helm&logoColor=white
[badge-kustomize]: https://img.shields.io/badge/Kustomize-326CE5?logo=kubernetes&logoColor=white
[badge-k8s]: https://img.shields.io/badge/Kubernetes-326CE5?logo=kubernetes&logoColor=white
[badge-level]: https://img.shields.io/badge/Level-Intermediate-orange
