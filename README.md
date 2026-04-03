
---

# CI/CD Pipeline with GitHub Actions and Argo CD on Kubernetes (EKS)

## Overview

This repository demonstrates a complete CI/CD pipeline for building, caching, testing, scanning, and deploying a Java application to a Kubernetes cluster using Argo CD.

The pipeline uses GitHub Actions for Continuous Integration and Continuous Deployment. The workflow performs the following:

* Builds a Java application using Maven
* Caches dependencies for faster builds
* Runs automated tests
* Performs Static Application Security Testing (SAST) using CodeQL
* Logs in to Docker Hub or AWS ECR using stored credentials
* Builds a new Docker image on every workflow trigger
* Tags the image uniquely using `github.sha`
* Pushes the image to a Docker repository
* Automatically updates the Kubernetes deployment manifest
* Allows Argo CD to detect and sync changes to the Kubernetes cluster

Argo CD compares the desired state (GitHub repository) with the actual state (Kubernetes cluster) and automatically synchronizes changes. It also provides safe rollback capabilities without downtime.

---

# Prerequisites

Before starting, ensure you have the following:

1. An EKS cluster or any running Kubernetes cluster
2. Argo CD installed on the Kubernetes cluster
3. kubectl installed and configured
4. Github

---

# Setup

## Step 1: Configure GitHub Actions Workflow

Create a workflow file inside:

```
.github/workflows/
```

### Branch Strategy

Create a new feature branch when working with GitHub Actions.

* This isolates development work from the `main` branch.
* The `main` branch remains clean and production-ready.
* Open a Pull Request (PR) from the feature branch to `main`.
* Review the code.
* Merge into `main` after approval.

---

### Path Ignore Configuration

The workflow uses `paths-ignore` to prevent unnecessary workflow triggers.

The following paths are excluded:

1. `kubernetes/` folder

   * Contains Kubernetes manifest files.
   * Argo CD handles Kubernetes deployment.
   * Changes in this folder should not trigger the CI workflow.

2. `README.md`

   * Documentation updates should not trigger builds.
   * README changes can be pushed independently.

---

### Jobs in the Workflow

Jobs are sets of tasks executed within the workflow.

#### 1. Checkout and Build

* Checks out source code.
* Builds the Java application using Maven.

#### 2. Caching

* Caches Maven dependencies.
* Improves build speed by reusing stored dependencies.

#### 3. Build Artifact

* Stores the compiled application as a GitHub Actions artifact.
* Can be downloaded in dependent jobs.

#### 4. Security Scan (CodeQL)

* Performs static code analysis.
* Detects hardcoded secrets.
* Identifies vulnerabilities.
* Detects SQL injection risks.

#### 5. Docker Login, Build, Tag, and Push

This represents the CD process.

* Logs in to Docker Hub or AWS ECR using repository secrets.
* Builds a new Docker image.
* Tags the image using `github.sha` for uniqueness.
* Uses `sed` to update the image tag inside the Kubernetes deployment manifest.
* Pushes the new image to the Docker repository.

---

## Step 2: Argo CD Setup and Role

### Role of Argo CD

Argo CD is a GitOps tool that:

* Monitors a Git repository (desired state).
* Compares it with the actual state of the Kubernetes cluster.
* Automatically syncs changes when differences are detected.
* Allows safe rollbacks.
* Provides a UI to monitor application state.

Argo CD runs inside the Kubernetes cluster and continuously watches the configured Git repository.

Git becomes the single source of truth.

---

### Install Argo CD on EKS Using Helm

Add the Helm repository:

```
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

Install Argo CD:

```
helm install argocd argo/argo-cd --namespace argocd --create-namespace
```

---

### Expose Argo CD Server (LoadBalancer)

Patch the service:

```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Check the service:

```
kubectl get svc -n argocd
```

---

### Retrieve Argo CD Admin Password

```
kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 --decode
```

Login with:

* Username: `admin`
* Retrieved password

---

### Configure Argo CD Application

Create an `application.yaml` file (Application CRD) that includes:

* GitHub repository URL
* Target branch
* Kubernetes cluster endpoint
* Path to Kubernetes manifests

Apply the configuration:

```
kubectl apply -f application.yaml
```

Push the file to the GitHub repository.

Argo CD will now monitor the repository and automatically sync changes to the Kubernetes cluster.

---

## Step 3: Workflow Flow

1. Developer builds and pushes code to GitHub.
2. GitHub Actions:

   * Builds the application.
   * Runs tests.
   * Caches dependencies.
   * Runs CodeQL security scan.
   * Builds and pushes Docker image.
3. `sed` updates the image tag in the Kubernetes deployment manifest.
4. A Pull Request is opened to merge into `main`.
5. After approval, changes are merged into `main`.
6. Argo CD detects the change.
7. Argo CD syncs the new state to the Kubernetes cluster automatically.
8. No manual `kubectl apply` commands are required.
9. Git becomes the single source of truth.
10. Argo CD acts as the watcher and enforcer of desired state.

---

# Architecture Summary

Developer → GitHub → GitHub Actions (CI/CD) → Docker Registry → GitHub (Updated Manifest) → Argo CD → Kubernetes (EKS)

---

# Key Benefits

* Automated CI/CD pipeline
* Secure code scanning with CodeQL
* Immutable Docker image tagging
* GitOps-based deployment
* Automatic synchronization
* Zero-downtime rollback capability
* Git as the single source of truth

---

