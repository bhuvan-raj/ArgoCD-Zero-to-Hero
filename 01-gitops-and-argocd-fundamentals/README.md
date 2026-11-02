# 01 - GitOps and Argo CD Fundamentals

![GitOps](https://img.shields.io/badge/GitOps-Fundamentals-orange)
![Argo CD](https://img.shields.io/badge/Argo%20CD-Continuous%20Delivery-EF7B4D)
![Kubernetes](https://img.shields.io/badge/Kubernetes-Declarative-326CE5)

## Overview

GitOps is a modern approach to managing application deployments and infrastructure using Git as the single source of truth. It follows a declarative model where the desired state of an application is stored in a Git repository, and automated tools ensure that the actual environment matches the defined configuration.

Argo CD is a declarative, GitOps-based Continuous Delivery tool for Kubernetes. It continuously monitors Git repositories and helps maintain the desired state of applications running inside Kubernetes clusters.

In this module, we will understand the fundamentals of GitOps, its principles, deployment models, and how Argo CD simplifies Kubernetes application delivery.

## Learning Objectives

By the end of this module, you will understand:

- What GitOps is and why it is needed.
- Traditional deployment vs GitOps-based deployment.
- Core principles of GitOps.
- Declarative vs imperative configuration.
- Push-based vs Pull-based deployment models.
- Desired state vs Actual state.
- What Argo CD is and why we use it.
- How Argo CD fits into the Kubernetes CI/CD workflow.
- Benefits and limitations of GitOps.

## Table of Contents

1. [Introduction to GitOps](#1-introduction-to-gitops)
2. [Core Principles of GitOps](#2-core-principles-of-gitops)
3. [Traditional CI/CD vs GitOps](#3-traditional-cicd-vs-gitops)
4. [Desired State vs Actual State](#4-desired-state-vs-actual-state)
5. [Introduction to Argo CD](#5-introduction-to-argo-cd)
6. [GitOps Workflow with Argo CD](#6-gitops-workflow-with-argo-cd)
7. [Important GitOps Concepts](#7-important-gitops-concepts)
8. [Advantages of GitOps](#8-advantages-of-gitops)
9. [Key Takeaways](#key-takeaways)
10. [References](#references)

---

## 1. Introduction to GitOps

- What is GitOps?
- Why do we need GitOps?
- Evolution of Kubernetes deployment practices.
- Problems with traditional deployment approaches.
- Git as the single source of truth.
- GitOps workflow and operating model.

## 2. Core Principles of GitOps

Understand the four fundamental principles of GitOps:

### Declarative Configuration

- What is declarative configuration?
- Declarative vs imperative approach.
- Managing Kubernetes resources using YAML manifests.

### Version Controlled

- Storing application configurations in Git.
- Git commits, branches, and pull requests.
- Tracking and auditing deployment changes.

### Automated Synchronization

- Automatically applying desired configurations.
- Eliminating manual deployment operations.

### Continuous Reconciliation

- Comparing desired and actual states.
- Detecting configuration differences.
- Maintaining consistency between Git and Kubernetes.

## 3. Traditional CI/CD vs GitOps

- Traditional CI/CD deployment workflow.
- GitOps-based deployment workflow.
- Push-based deployment model.
- Pull-based deployment model.
- Differences between Push and Pull approaches.
- Advantages of GitOps over traditional deployment practices.

## 4. Desired State vs Actual State

- What is the desired state?
- What is the actual state?
- How Kubernetes maintains the desired state.
- Understanding reconciliation.
- Introduction to configuration drift.
- Why maintaining the desired state is important.

## 5. Introduction to Argo CD

- What is Argo CD?
- Why do we need Argo CD?
- Argo CD as a GitOps Continuous Delivery tool.
- How Argo CD works at a high level.
- Argo CD integration with Kubernetes.
- Argo CD vs traditional CI/CD tools.
- Benefits of using Argo CD.

## 6. GitOps Workflow with Argo CD

Understand the end-to-end GitOps deployment workflow:

1. Developers write application code.
2. Code is pushed to a Git repository.
3. CI pipelines build and test the application.
4. Container images are built and pushed to a container registry.
5. Kubernetes manifests are updated in a Git repository.
6. Argo CD detects the configuration changes.
7. Argo CD compares the desired state with the actual cluster state.
8. The Kubernetes cluster is synchronized with the desired configuration.

### Architecture Overview

```text
       Developer
           |
           v
      Git Repository
           |
           v
      CI Pipeline
   (Build and Test)
           |
           v
    Container Registry
           |
           v
   Update Kubernetes
       Manifests
           |
           v
    GitOps Repository
           |
           v
        Argo CD
           |
           v
   Kubernetes Cluster
           |
           v
   Application Running
```

## 7. Important GitOps Concepts

- Single Source of Truth.
- Declarative Infrastructure.
- Continuous Reconciliation.
- Configuration Drift.
- Desired State Management.
- Immutable Infrastructure (introduction).
- Git-based Rollback.
- Auditability and Traceability.

## 8. Advantages of GitOps

- Consistent application deployments.
- Version-controlled configurations.
- Improved deployment automation.
- Easy rollback using Git history.
- Better visibility and auditability.
- Reduced manual intervention.
- Improved operational consistency.
- Supports multi-environment and multi-cluster management.

## Key Takeaways

- GitOps uses Git as the single source of truth.
- Kubernetes configurations are maintained declaratively.
- GitOps controllers continuously reconcile the desired and actual states.
- Push-based and pull-based deployments are different deployment approaches.
- Argo CD implements GitOps-based Continuous Delivery for Kubernetes.
- GitOps improves consistency, automation, and traceability.

## References

- [Argo CD Official Documentation][ref-argocd]
- [Argo CD Core Concepts][ref-core]
- [OpenGitOps - GitOps Principles][ref-opengitops]
- [Kubernetes Official Documentation][ref-k8s]

[ref-argocd]: https://argo-cd.readthedocs.io/en/stable/
[ref-core]: https://argo-cd.readthedocs.io/en/stable/core_concepts/
[ref-opengitops]: https://opengitops.dev/
[ref-k8s]: https://kubernetes.io/docs/home/
