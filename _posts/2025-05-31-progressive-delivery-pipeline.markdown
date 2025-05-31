---
layout: post
title: "Designing a Progressive Delivery Pipeline with GitHub Actions and Argo Rollouts"
date: 2025-05-31 20:18:22 +0700
comments: true
---

In the age of cloud-native applications and DevOps practices, software delivery is evolving beyond traditional CI/CD pipelines. Progressive delivery—a modern approach that includes strategies like canary releases, blue-green deployments, and feature flags—enables safer, controlled rollouts to production.

In this post, we’ll walk through how to design a **progressive delivery pipeline using GitHub Actions** for CI and **Argo Rollouts** for advanced Kubernetes deployment strategies.

---

## The Foundation: Deployment Strategies

Before implementing progressive delivery, it's crucial to understand the **deployment strategies** that enable or inspire it:

### 1. **Recreate (Big Bang)**

* Stop the old version, then deploy the new one.
* **Risky**, leads to **downtime**.
* **Used in:** legacy systems or simple environments.

### 2. **Blue-Green Deployment**

* Run two production environments (blue & green).
* Switch traffic from old (blue) to new (green) after verification.
* Fast rollback by switching back.

### 3. **Canary Deployment**

* Gradually shift a small portion of traffic to the new version.
* Monitor key metrics (latency, errors, CPU).
* If healthy, continue the rollout; if not, roll back.

### 4. **Rolling Update**

* Incrementally replace instances of the old version with the new one.
* Common in Kubernetes (`RollingUpdate` strategy).
* Doesn't support traffic shifting or rollback natively.

### 5. **A/B Testing**

* Deploy multiple versions simultaneously to different user segments.
* Requires application-level logic or feature flags.

### 6. **Feature Flags (Toggle-Based Delivery)**

* Deploy code with flags to control which features are exposed.
* Supports experimentation, A/B testing, kill switches.

> **Progressive delivery** combines **canary**, **feature flagging**, and **observability** to deliver changes incrementally, reduce blast radius, and allow early feedback.

---

## What is Progressive Delivery?

**Progressive delivery** is a release pattern that minimizes risk by gradually rolling out changes to production. Instead of deploying a new version to all users at once, it’s released incrementally to subsets of traffic or user segments, using automation and observability to detect issues early.

### Key Characteristics:

* **Incremental exposure**: Traffic moves in stages (e.g., 10% → 30% → 100%)
* **Gate checks**: Health metrics dictate whether to proceed or halt
* **Rollbacks**: Fast, automated rollback on failure
* **Control points**: Manual approval or full automation

> Progressive delivery is the evolution of continuous delivery—it prioritizes **safety**, **observability**, and **granular control** over **speed alone**.

### **Key benefits:**

* **Reduced risk** in production.
* **Faster feedback** loops.
* **Better observability** for changes.

Let’s break this down using **GitHub Actions** + **Argo Rollouts**.

---

## Tooling Overview

### GitHub Actions

GitHub Actions is a powerful CI/CD automation platform tightly integrated with GitHub repositories. It can build, test, and package your application on every pull request or push.

### Argo Rollouts

Argo Rollouts is a Kubernetes controller and set of CRDs (Custom Resource Definitions) that provides advanced deployment strategies like:

* Canary deployments
* Blue-green deployments
* Progressive analysis using Prometheus, Wavefront, and others

Together, they form a complete pipeline for building, testing, and deploying apps with confidence.

---

## Pipeline Architecture

Here’s the high-level flow of our pipeline:

1. **Code Push**: A developer pushes code to GitHub.
2. **CI Workflow**: GitHub Actions builds the app, runs tests, and pushes the container image to a registry.
3. **CD Trigger**: GitHub Actions updates the Kubernetes manifest or Rollout resource in a Git repo (GitOps).
4. **Argo Rollouts**: Kubernetes picks up the change and rolls it out using a canary or blue-green strategy.
5. **Metrics Check**: Rollouts are monitored based on success metrics (e.g., via Prometheus).
6. **Auto Promotion or Rollback**: Depending on metrics and manual judgment, the rollout proceeds or aborts.

---

## Step-by-Step Implementation

### 1. Define the Argo Rollout Resource

Here’s an example `Rollout` manifest for a canary deployment:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
spec:
  replicas: 4
  strategy:
    canary:
      steps:
        - setWeight: 25
        - pause: { duration: 5m }
        - setWeight: 50
        - pause: { duration: 5m }
        - setWeight: 100
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: ghcr.io/your-org/my-app:latest
        ports:
        - containerPort: 8080
```

### 2. Create a GitHub Actions Workflow

Place this in `.github/workflows/deploy.yml`:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Log in to GitHub Container Registry
      uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Build and push Docker image
      run: |
        docker build -t ghcr.io/your-org/my-app:${{ github.sha }} .
        docker push ghcr.io/your-org/my-app:${{ github.sha }}

    - name: Update rollout manifest
      run: |
        sed -i "s|ghcr.io/your-org/my-app:.*|ghcr.io/your-org/my-app:${{ github.sha }}|" k8s/rollout.yaml
        git config user.name "github-actions"
        git config user.email "actions@github.com"
        git commit -am "Update image to ${{ github.sha }}"
        git push
```

> Optionally, if you're using **Argo CD** with GitOps, it will automatically detect the manifest change and sync it to the cluster.

---

## Integrating Observability

To make progressive delivery meaningful, it must be tied to observable metrics. Argo Rollouts supports real-time analysis with:

* **Prometheus**
* **New Relic**
* **Datadog**
* **Wavefront**

Example `analysisTemplate`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate-check
spec:
  metrics:
    - name: success-rate
      interval: 1m
      count: 3
      successCondition: result > 95
      provider:
        prometheus:
          address: http://prometheus-server
          query: |
            sum(rate(http_requests_total{status=~"2.."}[1m])) / 
            sum(rate(http_requests_total[1m])) * 100
```

Attach this template to your rollout under `spec.strategy.canary.analysis`.

---

## 🧪 Manual Promotion & Rollback

Use `kubectl argo rollouts` or the [Argo Rollouts Dashboard](https://argoproj.github.io/argo-rollouts/dashboard/) to promote or abort deployments:

```bash
kubectl argo rollouts get rollout my-app
kubectl argo rollouts promote my-app
kubectl argo rollouts abort my-app
```

---

## Best Practices for Progressive Delivery

* **Start with staging**: Use progressive delivery in lower environments first.
* **Decouple deploy from release**: Use feature flags to separate code deployment from feature exposure.
* **Automate observability**: Rollouts should halt or rollback based on real metrics.
* **Immutable images**: Always deploy using image digests or commit-based tags.
* **Fail fast**: Set tight error budgets and rollback on first signs of failure.
* **Document your rollout plans**: Define steps, thresholds, and rollback conditions explicitly.

---

## Final Thoughts

**Progressive delivery** is about strategy, not tooling. Tools like **GitHub Actions** and **Argo Rollouts** make it easier, but the value lies in how well you manage **risk, observability, and user exposure**.

By blending GitOps, canary rollouts, and real-time telemetry, your team can move faster—without sacrificing safety or sleep.
