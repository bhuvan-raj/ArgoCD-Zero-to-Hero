# Application Management and Sync

![ArgoCD][badge-argocd] ![GitOps][badge-gitops] ![Kubernetes][badge-k8s] ![Level][badge-level]

> Day-to-day operation of ArgoCD: creating and updating Applications, previewing and running syncs, handling drift and ignored differences, rolling back, controlling deletion, and troubleshooting failed syncs.

This module builds on [ArgoCD Architecture and Core Concepts][mod-03]. If terms like *desired state*, *sync status* or *selfHeal* are unfamiliar, read that module first.

---

## Table of Contents

- [Learning Objectives](#learning-objectives)
- [Application Lifecycle](#application-lifecycle)
- [Creating Applications](#creating-applications)
- [Inspecting Applications](#inspecting-applications)
- [Refresh vs Hard Refresh](#refresh-vs-hard-refresh)
- [Previewing Changes with Diff](#previewing-changes-with-diff)
- [Running a Sync](#running-a-sync)
- [Selective Sync](#selective-sync)
- [Automated Sync in Practice](#automated-sync-in-practice)
- [Sync Retries and Failed Syncs](#sync-retries-and-failed-syncs)
- [Handling Drift and Ignored Differences](#handling-drift-and-ignored-differences)
- [Resource Tracking](#resource-tracking)
- [Customizing Helm and Kustomize Sources](#customizing-helm-and-kustomize-sources)
- [Multiple Sources](#multiple-sources)
- [Custom Health Checks](#custom-health-checks)
- [Sync Windows](#sync-windows)
- [History and Rollback](#history-and-rollback)
- [Deleting Applications](#deleting-applications)
- [Protecting Resources from Pruning and Deletion](#protecting-resources-from-pruning-and-deletion)
- [Troubleshooting Sync Problems](#troubleshooting-sync-problems)
- [Hands-On Lab](#hands-on-lab)
- [CLI Cheat Sheet](#cli-cheat-sheet)
- [Best Practices](#best-practices)
- [Knowledge Check](#knowledge-check)
- [Summary](#summary)
- [Series Navigation](#series-navigation)

---

## Learning Objectives

By the end of this module you will be able to:

- Create Applications using the UI, the CLI and declarative manifests.
- Preview changes with `diff` and `--dry-run` before syncing.
- Sync a whole application or only selected resources.
- Explain what automated sync will and will not do.
- Stop noisy, expected differences from showing as `OutOfSync`.
- Roll back a bad release the right way.
- Delete Applications safely and protect critical resources.
- Diagnose the most common sync failures.

---

## Application Lifecycle

```text
   Create Application
          |
          v
   +--------------+   Git changes / drift   +--------------+
   |   Synced     |------------------------>|  OutOfSync   |
   |   Healthy    |<------------------------|              |
   +------+-------+       Sync succeeds     +------+-------+
          |                                        |
          | upgrade, tune, override                | Sync fails
          v                                        v
   Update Application                       +--------------+
          |                                 |  Sync Failed |
          v                                 |  / Degraded  |
   Rollback if needed                       +------+-------+
          |                                        |
          v                                        v
   Delete Application                      Troubleshoot and retry
```

Operating an application means moving it around this loop safely. The rest of this module covers each step.

---

## Creating Applications

There are three ways to create an Application. They all produce the same `Application` custom resource.

| Method | Best for |
|--------|----------|
| **Declarative YAML** | The GitOps way. Reviewable, repeatable, stored in Git. Recommended. |
| **CLI** | Quick experiments and scripting. |
| **Web UI** | Learning and exploring. |

### Declarative (recommended)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: webapp
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/webapp-config.git
    targetRevision: main
    path: overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: webapp
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f webapp.yaml
```

### CLI

```bash
argocd app create webapp \
  --repo https://github.com/my-org/webapp-config.git \
  --revision main \
  --path overlays/dev \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace webapp \
  --project default \
  --sync-option CreateNamespace=true
```

### Web UI

1. Click **+ New App**.
2. Fill in **Application Name** and **Project**.
3. Choose a **Sync Policy** (Manual or Automatic).
4. Set the **Repository URL**, **Revision** and **Path**.
5. Set the **Cluster URL** and **Namespace**.
6. Click **Create**.

> The UI and CLI create the same custom resource as the YAML above. If you create apps imperatively, export them to Git afterwards with `argocd app get webapp -o yaml` so Git stays the source of truth.

---

## Inspecting Applications

```bash
# List all applications
argocd app list

# Detailed status of one application
argocd app get webapp

# The same as YAML
argocd app get webapp -o yaml

# The rendered manifests ArgoCD will apply
argocd app manifests webapp

# Resource tree, sync history and conditions
kubectl describe application webapp -n argocd
```

Example `argocd app get` output:

```text
Name:               argocd/webapp
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          webapp
URL:                https://argocd.example.com/applications/webapp
Repo:               https://github.com/my-org/webapp-config.git
Target:             main
Path:               overlays/dev
SyncWindow:         Sync Allowed
Sync Policy:        <none>
Sync Status:        OutOfSync from main (a1b2c3d)
Health Status:      Healthy

GROUP  KIND        NAMESPACE  NAME    STATUS     HEALTH
       Service     webapp     webapp  Synced     Healthy
apps   Deployment  webapp     webapp  OutOfSync  Healthy
```

Read it top-down: the overall **Sync Status**, then the per-resource table showing exactly which resources differ.

---

## Refresh vs Hard Refresh

| Action | What it does | When to use |
|--------|--------------|-------------|
| **Refresh** | Re-fetches Git (using cached manifests where valid) and re-compares with live state. | You pushed a commit and do not want to wait for the poll. |
| **Hard Refresh** | Same as refresh, but **invalidates the manifest cache** and regenerates manifests from scratch. | Helm dependencies or remote bases changed, or you suspect stale cached manifests. |
| **Sync** | Actually applies changes to the cluster. | You want the cluster to change. |

```bash
argocd app get webapp --refresh
argocd app get webapp --hard-refresh
```

> A refresh never changes the cluster. It only updates ArgoCD's *view* of whether the app is in sync.

---

## Previewing Changes with Diff

Always look before you sync.

```bash
argocd app diff webapp
```

The output is a standard diff between **live state** (cluster) and **desired state** (Git):

```diff
===== apps/Deployment webapp/webapp ======
 spec:
-  replicas: 2
+  replicas: 4
   template:
     spec:
       containers:
       - name: webapp
-        image: my-org/webapp:1.3.0
+        image: my-org/webapp:1.4.0
```

You can also compare against a different revision before changing the application:

```bash
argocd app diff webapp --revision feature/new-limits
```

In the UI, open the application and click **App Diff** to see the same view.

---

## Running a Sync

### Manual sync

```bash
argocd app sync webapp
```

### Dry run

Validate what a sync would do without applying anything:

```bash
argocd app sync webapp --dry-run
```

### Common sync flags

| Flag | Effect |
|------|--------|
| `--prune` | Delete resources that are in the cluster but no longer in Git. |
| `--dry-run` | Preview only. |
| `--force` | Use `kubectl replace --force` semantics (delete and recreate). Disruptive. |
| `--revision <rev>` | Sync to a specific branch, tag or commit instead of the tracked one. |
| `--resource <group:kind:name>` | Sync only the named resource(s). |
| `--label key=value` | Sync only resources matching a label. |
| `--async` | Return immediately without waiting for completion. |
| `--timeout <seconds>` | Maximum time to wait for the sync. |
| `--retry-limit <n>` | Retry failed syncs up to n times. |

### Wait for the result

```bash
argocd app wait webapp --sync --health --timeout 300
```

This is useful in CI/CD pipelines: trigger the sync, then block until the application is `Synced` and `Healthy` or the timeout is reached.

### What a sync actually does

```text
  1. Render manifests for the target revision
  2. Run PreSync hooks
  3. Apply resources in sync-wave order
  4. Wait for each wave to become healthy
  5. Run PostSync hooks
  6. Record the result in the application's history
```

### Stopping a running sync

```bash
argocd app terminate-op webapp
```

---

## Selective Sync

You do not have to sync everything at once. Syncing a single resource is handy for testing or for staged rollouts.

```bash
# Only the Deployment
argocd app sync webapp --resource apps:Deployment:webapp

# Only the Service (core group is an empty string)
argocd app sync webapp --resource :Service:webapp

# Everything with a given label
argocd app sync webapp --label tier=frontend
```

In the UI, click **Sync**, then untick the resources you do not want to apply under **Synchronize resources**.

> Selective sync leaves the application `OutOfSync` for anything you skipped. That is expected.

---

## Automated Sync in Practice

Automated sync is configured in the sync policy (see [Core Concepts][mod-03]). The details below are the behaviours that surprise people.

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
    allowEmpty: false
```

| Behaviour | Detail |
|-----------|--------|
| **One attempt per revision** | Automated sync tries once per unique combination of commit and application parameters. If it fails, it does not hammer the cluster. It tries again when something changes, or when `retry` is configured. |
| **`allowEmpty: false` (default)** | Protects you from an accidental sync that would delete every resource because the source path rendered nothing. Leave it off unless you really intend to empty an app. |
| **Rollback is blocked** | You cannot use `argocd app rollback` while automated sync is enabled. Disable it first, or better, revert in Git. |
| **Applies on detecting differences** | Triggered by new commits **and** by cluster drift (when `selfHeal` is true). |

Enable or disable it from the CLI:

```bash
argocd app set webapp --sync-policy automated --auto-prune --self-heal
argocd app set webapp --sync-policy none
```

---

## Sync Retries and Failed Syncs

Transient failures (an API server briefly unavailable, a CRD not yet registered) are common. Configure retries so they resolve themselves:

```yaml
syncPolicy:
  automated:
    selfHeal: true
  retry:
    limit: 5
    backoff:
      duration: 10s
      factor: 2
      maxDuration: 5m
```

```text
  attempt 1 fails  -->  wait 10s
  attempt 2 fails  -->  wait 20s
  attempt 3 fails  -->  wait 40s
  attempt 4 fails  -->  wait 80s
  attempt 5 fails  -->  give up, status: Failed
```

Typical sync operation states:

| State | Meaning |
|-------|---------|
| `Running` | A sync is in progress. |
| `Succeeded` | All resources applied and hooks completed. |
| `Failed` | A resource or hook failed. See the operation message. |
| `Terminating` | The sync is being cancelled. |

---

## Handling Drift and Ignored Differences

Some fields are legitimately changed by something other than Git, and ArgoCD will report them as `OutOfSync` forever. Common examples:

- An HPA changes `spec.replicas` on a Deployment.
- A mutating webhook injects a sidecar or default values.
- A controller writes to `status` or `metadata` fields.
- A CA bundle or certificate is rotated by cert-manager.

Tell ArgoCD to ignore those fields.

### Per application

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas

    - group: admissionregistration.k8s.io
      kind: MutatingWebhookConfiguration
      jqPathExpressions:
        - .webhooks[]?.clientConfig.caBundle
```

| Method | Use when |
|--------|----------|
| `jsonPointers` | You know the exact field path. Simple and readable. |
| `jqPathExpressions` | You need to match inside lists or use patterns. |
| `managedFieldsManagers` | Ignore all fields owned by a specific field manager (for example `kube-controller-manager`). |

### Make ignored fields survive a sync

By default, ignored differences are skipped when *comparing*, but a sync can still overwrite them. To make a sync respect them too:

```yaml
syncPolicy:
  syncOptions:
    - RespectIgnoreDifferences=true
```

### Globally

For cluster-wide rules, configure `resource.customizations.ignoreDifferences.<group_kind>` in the `argocd-cm` ConfigMap.

### Ignore a resource that ArgoCD does not manage

Resources that exist in the namespace but are not defined in Git can clutter the view. Mark resources that should never affect sync status:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/compare-options: IgnoreExtraneous
```

> Only ignore fields you understand. Ignoring too much defeats the purpose of GitOps.

---

## Resource Tracking

ArgoCD must know which cluster resources belong to which Application, so it can detect drift and know what to prune.

| Tracking method | How it works |
|-----------------|--------------|
| `label` | Adds the label `app.kubernetes.io/instance: <app-name>`. Simple, but conflicts with other tools that use the same label, and label values are length-limited. |
| `annotation` | Adds an annotation `argocd.argoproj.io/tracking-id` containing the app, group, kind, namespace and name. Avoids label clashes. |
| `annotation+label` | Uses both. Helps tools that look up resources by label. |

The method is set in the `argocd-cm` ConfigMap:

```yaml
data:
  application.resourceTrackingMethod: annotation
```

> The default tracking method has changed between ArgoCD major versions. Check the default for the version you installed before relying on label-based assumptions.

---

## Customizing Helm and Kustomize Sources

You can override values on the Application itself without touching the chart or base in Git. Use this for environment-specific tweaks, and keep anything long-lived in Git.

### Helm

```yaml
source:
  repoURL: https://github.com/my-org/charts.git
  targetRevision: main
  path: charts/webapp
  helm:
    releaseName: webapp
    valueFiles:
      - values.yaml
      - values-prod.yaml
    parameters:
      - name: image.tag
        value: "1.4.0"
    values: |
      replicaCount: 3
      ingress:
        enabled: true
```

```bash
argocd app set webapp --helm-set image.tag=1.4.0
argocd app set webapp --values values-prod.yaml
```

### Kustomize

```yaml
source:
  repoURL: https://github.com/my-org/app-config.git
  targetRevision: main
  path: overlays/prod
  kustomize:
    namePrefix: prod-
    images:
      - my-org/webapp:1.4.0
```

```bash
argocd app set webapp --kustomize-image my-org/webapp:1.4.0
```

> **Remember:** parameters set on the Application live in the Application object, not in your Git repo's manifests. Be deliberate about where each setting lives.

---

## Multiple Sources

A single Application can combine several sources, for example a public Helm chart from one repo with your own values file from another.

```yaml
spec:
  sources:
    - repoURL: https://charts.example.com
      chart: webapp
      targetRevision: 2.5.0
      helm:
        valueFiles:
          - $values/envs/prod/values.yaml

    - repoURL: https://github.com/my-org/gitops-values.git
      targetRevision: main
      ref: values
```

```text
   +------------------------+      +---------------------------+
   | Helm chart repository  |      | Git repo with values      |
   | chart: webapp 2.5.0    |      | ref: values               |
   +-----------+------------+      +-------------+-------------+
               |                                 |
               +---------------+-----------------+
                               |
                               v
                     One Application, one sync
```

The `ref` name (`values` here) is referenced from the chart using `$values/...`. This keeps third-party charts untouched while your configuration stays in your own repo.

---

## Custom Health Checks

ArgoCD ships health checks for built-in Kubernetes kinds and many popular CRDs. For custom resources it does not know, you can define a health check in Lua inside `argocd-cm`:

```yaml
data:
  resource.customizations.health.example.com_Widget: |
    hs = {}
    if obj.status ~= nil and obj.status.phase == "Ready" then
      hs.status = "Healthy"
      hs.message = "Widget is ready"
    else
      hs.status = "Progressing"
      hs.message = "Waiting for Widget"
    end
    return hs
```

The script receives the resource as `obj` and must return a table with a `status` (`Healthy`, `Progressing`, `Degraded`, `Suspended`, `Missing`) and an optional `message`. Without a health check, a custom resource is typically reported as healthy as soon as it exists, which can hide real problems and break sync-wave ordering.

---

## Sync Windows

Sync windows restrict **when** syncs may happen. They are defined on an AppProject.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  syncWindows:
    # Allow syncs only on weekdays 09:00-17:00 UTC
    - kind: allow
      schedule: "0 9 * * 1-5"
      duration: 8h
      applications:
        - "*"
      timeZone: UTC

    # Block all syncs during the Friday evening freeze
    - kind: deny
      schedule: "0 17 * * 5"
      duration: 63h
      applications:
        - "*"
      manualSync: false
```

| Field | Meaning |
|-------|---------|
| `kind` | `allow` or `deny`. A `deny` window always wins over an `allow` window. |
| `schedule` | Cron expression for when the window starts. |
| `duration` | How long the window lasts. |
| `applications`, `namespaces`, `clusters` | What the window applies to. |
| `manualSync` | If `true`, manual syncs are still allowed during a `deny` window. |

```bash
argocd proj windows list production
```

---

## History and Rollback

Every successful sync is recorded in the application's history.

```bash
argocd app history webapp
```

```text
ID  DATE                           REVISION
5   2026-11-18 10:12:01 +0000 UTC  main (a1b2c3d)
6   2026-11-19 14:30:44 +0000 UTC  main (e4f5a6b)
7   2026-11-20 09:05:12 +0000 UTC  main (c7d8e9f)
```

### Rollback with the CLI

```bash
# Disable automated sync first, otherwise rollback is refused
argocd app set webapp --sync-policy none

argocd app rollback webapp 6
```

Rollback re-applies the manifests from a previous history entry. The cluster returns to the old state, but **Git still contains the bad commit**, so the application will show as `OutOfSync`.

### Rollback the GitOps way (recommended)

```bash
git revert <bad-commit>
git push
```

ArgoCD then syncs the reverted state. Git history remains the single, auditable record of what happened and why.

```text
   CLI rollback                          Git revert
   ------------                          ----------
   Fast, immediate                       One extra step
   Git and cluster disagree              Git and cluster agree
   Must disable auto-sync                Works with auto-sync
   Good for emergencies                  Preferred default
```

> In an emergency, a CLI rollback is a valid first step to stop the bleeding. Follow it with a Git revert so the repository reflects reality.

---

## Deleting Applications

Deleting an Application has two possible effects: remove only the Application object, or also remove the resources it deployed.

### Cascading delete

If the Application has the finalizer `resources-finalizer.argocd.argoproj.io`, or you delete with `--cascade`, ArgoCD deletes the deployed resources first, then the Application.

```bash
argocd app delete webapp --cascade
kubectl delete application webapp -n argocd     # with the finalizer in place
```

### Non-cascading delete

Only the Application object is removed. The workloads keep running, unmanaged.

```bash
argocd app delete webapp --cascade=false
```

| Delete type | Application object | Deployed resources |
|-------------|--------------------|--------------------|
| Cascading (finalizer or `--cascade`) | Deleted | **Deleted** |
| Non-cascading | Deleted | **Kept** |

### Propagation policy

When cascading, you can control how Kubernetes removes dependents:

```bash
argocd app delete webapp --propagation-policy foreground
```

Options are `foreground`, `background` and `orphan`.

> **Be careful with App of Apps.** Deleting a parent Application with the cascade finalizer can delete every child Application, and therefore every workload. Treat it as the dangerous operation it is.

---

## Protecting Resources from Pruning and Deletion

Some resources must never be removed by ArgoCD: a PersistentVolumeClaim holding data, a namespace shared with other teams, a database.

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-options: Prune=false
```

| Annotation value | Effect |
|------------------|--------|
| `Prune=false` | ArgoCD will not delete this resource when it disappears from Git. |
| `Delete=false` | ArgoCD will not delete this resource when the Application is deleted. |
| `Prune=false,Delete=false` | Both protections (comma-separated). |

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-options: Prune=false,Delete=false
```

---

## Troubleshooting Sync Problems

Start with the facts: what does ArgoCD say, and what do the components say?

```bash
argocd app get webapp                       # status, conditions, per-resource state
argocd app diff webapp                      # what differs
argocd app manifests webapp                 # what is being rendered
kubectl describe application webapp -n argocd
kubectl logs -n argocd deploy/argocd-repo-server            # manifest generation problems
kubectl logs -n argocd statefulset/argocd-application-controller   # sync/compare problems
```

| Symptom | Likely cause | What to check or do |
|---------|--------------|---------------------|
| `ComparisonError` / repo unreachable | Wrong URL, missing credentials, network or DNS issue | `argocd repo list`, repo-server logs |
| `rpc error: ... authentication required` | Missing or expired repo credentials | Re-add the repository credentials |
| `Unable to generate manifests` | Bad Helm values, broken Kustomize overlay, invalid YAML | `argocd app manifests`, repo-server logs |
| `the server could not find the requested resource` | CRD not installed yet | Install the CRD first (sync wave) or enable retry |
| `namespace not found` | Destination namespace does not exist | Add `CreateNamespace=true` |
| `is not permitted in project` | AppProject blocks the repo, destination or kind | Update the AppProject whitelists |
| Stuck `OutOfSync` after every sync | A controller or webhook mutates fields | Add `ignoreDifferences` |
| `Progressing` forever | Resource never becomes healthy (image pull, failing probe) | `kubectl describe` the pod, `kubectl get events` |
| Sync hangs on a hook | Hook Job is failing or never completes | `kubectl logs job/<hook>`, check delete policy |
| `one or more objects failed to apply` | Invalid manifest, immutable field change | Read the per-resource message, consider `Replace=true` as a last resort |
| Sync blocked by a window | An active `deny` sync window | `argocd proj windows list <project>` |

### A debugging order that works

```text
  1. argocd app get          -> What is the status and message?
  2. argocd app diff         -> What is different, exactly?
  3. argocd app manifests    -> Is rendering correct?
  4. kubectl describe/events -> Why is the resource unhealthy?
  5. component logs          -> Repo-server (render) or controller (apply)
```

---

## Hands-On Lab

**Goal:** practise creating, previewing, syncing, drifting, rolling back and deleting an application.

### Prerequisites

- A running cluster with ArgoCD installed (module 02).
- The `argocd` CLI logged in.
- A Git repository you control (fork the example repo or create your own) containing a simple Deployment and Service.

### Step 1: Create the application with manual sync

```bash
argocd app create lab-app \
  --repo https://github.com/<you>/<your-repo>.git \
  --revision main \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace lab \
  --sync-option CreateNamespace=true
```

Check the status. It should be `OutOfSync` and `Missing`:

```bash
argocd app get lab-app
```

### Step 2: Preview, then sync

```bash
argocd app diff lab-app
argocd app sync lab-app --dry-run
argocd app sync lab-app
argocd app wait lab-app --sync --health
```

### Step 3: Make a change in Git

Edit the Deployment (for example change `replicas` from `1` to `3`), commit and push.

```bash
argocd app get lab-app --refresh
argocd app diff lab-app
```

The application becomes `OutOfSync`. Sync it and confirm the change:

```bash
argocd app sync lab-app
kubectl get deployment -n lab
```

### Step 4: Cause drift

```bash
kubectl scale deployment guestbook-ui -n lab --replicas=6
argocd app get lab-app
```

The app is `OutOfSync` because the cluster no longer matches Git. Fix it with a sync, then enable self-heal and repeat:

```bash
argocd app set lab-app --sync-policy automated --self-heal
kubectl scale deployment guestbook-ui -n lab --replicas=6
kubectl get deployment guestbook-ui -n lab -w
```

ArgoCD reverts the change within moments.

### Step 5: Ignore an expected difference

Add an `ignoreDifferences` block for `/spec/replicas` on the Deployment, apply it, then scale manually again. The app should now stay `Synced`.

### Step 6: Break something and roll back

Push a commit with an invalid image tag so pods fail to start. Observe `Progressing` or `Degraded`, then recover:

```bash
argocd app set lab-app --sync-policy none
argocd app history lab-app
argocd app rollback lab-app <previous-id>
```

Finish the proper way: `git revert` the bad commit and push.

### Step 7: Protect and delete

Add `argocd.argoproj.io/sync-options: Delete=false` to one resource, then:

```bash
argocd app delete lab-app --cascade
kubectl get all -n lab
```

Confirm which resource survived, then clean up manually.

---

## CLI Cheat Sheet

| Task | Command |
|------|---------|
| List apps | `argocd app list` |
| Show app | `argocd app get <app>` |
| Show rendered manifests | `argocd app manifests <app>` |
| Refresh | `argocd app get <app> --refresh` |
| Hard refresh | `argocd app get <app> --hard-refresh` |
| Diff | `argocd app diff <app>` |
| Dry-run sync | `argocd app sync <app> --dry-run` |
| Sync | `argocd app sync <app>` |
| Sync with prune | `argocd app sync <app> --prune` |
| Sync one resource | `argocd app sync <app> --resource <group:kind:name>` |
| Wait for healthy | `argocd app wait <app> --sync --health` |
| Cancel a running sync | `argocd app terminate-op <app>` |
| Enable auto-sync | `argocd app set <app> --sync-policy automated --auto-prune --self-heal` |
| Disable auto-sync | `argocd app set <app> --sync-policy none` |
| History | `argocd app history <app>` |
| Rollback | `argocd app rollback <app> <id>` |
| Delete (cascade) | `argocd app delete <app> --cascade` |
| Delete (keep resources) | `argocd app delete <app> --cascade=false` |
| App logs | `argocd app logs <app>` |

---

## Best Practices

- Define Applications **declaratively** and keep them in Git.
- Run `argocd app diff` and `--dry-run` before syncing anything important.
- Prefer **`git revert`** over CLI rollback, and use CLI rollback only to stop an incident.
- Turn on **automated sync with `selfHeal`** for environments where Git should be the only way to change things. Keep production gated if you need approvals.
- Be cautious with **`prune`** and always protect stateful resources with `Prune=false` and `Delete=false`.
- Use **retry with backoff** to absorb transient failures such as CRD ordering.
- Use **`ignoreDifferences`** sparingly and only for fields you understand.
- Put long-lived configuration in Git, not in Application-level parameter overrides.
- Use **sync windows** to protect production during freezes.
- Treat deletion of App of Apps parents as a high-risk operation.
- Always check the **repo-server logs** for rendering problems and the **controller logs** for apply problems.

---

## Knowledge Check

1. What is the difference between a refresh and a hard refresh?
2. Which command shows what a sync would change without applying it?
3. Why might an automated sync not retry after a failure?
4. Why does `argocd app rollback` fail when automated sync is enabled?
5. How do you stop an HPA-managed `replicas` field from showing as `OutOfSync`?
6. What is the difference between `Prune=false` and `Delete=false`?
7. What happens to deployed resources if you delete an Application with `--cascade=false`?
8. Which logs do you check first for a Helm rendering error?
9. When do you use `Replace=true`, and why is it risky?
10. A `deny` sync window and an `allow` window overlap. Which wins?

<details>
<summary>Answers</summary>

1. A refresh re-compares Git with live state, using cached manifests where valid. A hard refresh also invalidates the manifest cache and regenerates manifests.
2. `argocd app diff <app>` for a diff, or `argocd app sync <app> --dry-run` for a dry run.
3. Automated sync attempts once per unique commit and parameter combination. It tries again on a new change, or repeatedly if `retry` is configured.
4. Rollback would immediately be overwritten by automated sync, so ArgoCD refuses it. Disable automated sync first, or revert in Git.
5. Add an `ignoreDifferences` entry for `/spec/replicas` on the Deployment. Add `RespectIgnoreDifferences=true` if a sync should also leave it alone.
6. `Prune=false` protects a resource when it is removed from Git. `Delete=false` protects it when the Application is deleted.
7. They are kept in the cluster, but are no longer managed by ArgoCD.
8. The `argocd-repo-server` logs.
9. When an apply fails because of an immutable field change. It is risky because it replaces the resource, which can cause downtime or data loss.
10. The `deny` window wins.

</details>

---

## Summary

- Create Applications **declaratively** so the definition itself lives in Git.
- **Diff** and **dry-run** before you sync; **refresh** to update ArgoCD's view and **hard refresh** to bypass the cache.
- Sync everything, or **selectively** sync individual resources.
- **Automated sync** tries once per revision, protects against empty syncs, and blocks CLI rollback.
- Use **retry** for transient failures, **`ignoreDifferences`** for expected drift, and **custom health checks** for CRDs.
- Constrain when changes land with **sync windows**.
- Roll back with **`git revert`**; use CLI rollback only for emergencies.
- Control deletion with **finalizers**, `--cascade`, and the `Prune=false` / `Delete=false` annotations.
- Troubleshoot in order: **status, diff, manifests, events, logs**.

---

## Series Navigation

| Module | Topic |
|--------|-------|
| [Fundamentals of GitOps][mod-01] | Principles, pull vs push, Git as the source of truth |
| [ArgoCD Installation][mod-02] | Installing ArgoCD and accessing the UI and CLI |
| [ArgoCD Architecture and Core Concepts][mod-03] | Components, reconciliation, Applications, Projects, ApplicationSets |
| **Application Management and Sync** | You are here |

<!-- Reference-style link definitions -->
[mod-01]: ../01-fundamentals-of-gitops/README.md
[mod-02]: ../02-argocd-installation/README.md
[mod-03]: ../03-argocd-architecture-and-core-concepts/README.md
[badge-argocd]: https://img.shields.io/badge/Argo%20CD-EF7B4D?logo=argo&logoColor=white
[badge-gitops]: https://img.shields.io/badge/GitOps-Declarative%20Delivery-blue
[badge-k8s]: https://img.shields.io/badge/Kubernetes-326CE5?logo=kubernetes&logoColor=white
[badge-level]: https://img.shields.io/badge/Level-Intermediate-orange
