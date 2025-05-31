---
layout: post
title: "Designing a Progressive Delivery Pipeline with GitHub Actions and Argo Rollouts"
date: 2025-06-01 06:15:55 +0700
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

* **Incremental exposure**: Traffic moves in stages (e.g., 25% → 50% → 100%)
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
* Progressive analysis using Prometheus, Datadog, New Relic and others

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

## Step 1: Define the Argo Rollout Resource

The foundation of any progressive delivery pipeline using Argo Rollouts starts with replacing the standard Kubernetes `Deployment` object with a more intelligent and flexible `Rollout` resource. This custom resource gives us fine-grained control over how new versions of our application are introduced into a live environment, including support for strategies like **canary deployments**, **blue-green deployments**, and **automated metric-based analysis**.

Let’s break down a basic Rollout manifest configured for a **canary deployment** strategy:

{% raw %}
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
{% endraw %}

### Key Sections Explained

#### `apiVersion`, `kind`, and `metadata`

This identifies the resource as an Argo Rollout. By setting `kind: Rollout` and using Argo’s API group (`argoproj.io/v1alpha1`), we’re telling Kubernetes to delegate control of this workload to the **Argo Rollouts controller**, which manages more advanced deployment logic than a vanilla `Deployment`.

#### `spec.replicas`

We define the desired state of the application to run with four replicas. During a canary rollout, Argo will manage the distribution of these replicas between the **stable** and **canary** versions of your application, based on the rollout steps defined next.

#### `strategy.canary.steps`

This section is where the progressive magic happens. The canary strategy defines how new versions are incrementally exposed to users:

* **`setWeight: 25`**
  Shift 25% of traffic to the new version. This means 1 out of 4 replicas will run the updated container.

* **`pause: { duration: 5m }`**
  Wait for 5 minutes before proceeding. This window allows time to monitor key metrics like latency, errors, and CPU usage.

* **`setWeight: 50`**
  If everything looks healthy, move to 50% traffic — 2 out of 4 pods now run the new version.

* **`pause: { duration: 5m }`**
  Another pause for validation, helping catch any regressions before the full rollout.

* **`setWeight: 100`**
  At this point, the rollout is complete — all pods now run the new version.

This step-based approach mitigates risk by enabling early detection of issues. If a problem is discovered during any pause, the rollout can be **automatically or manually aborted**, limiting the blast radius.

#### `selector` and `template`

The `selector` matches pods based on the `app: my-app` label, and the `template` defines the pod specification just like a standard Kubernetes Deployment.

The `template.spec.containers.image` field is the most important here — this is where you specify the container image to deploy. In a GitOps workflow, this field will be updated dynamically (via GitHub Actions or Argo CD), and Argo Rollouts will detect the change to begin a new deployment cycle.

> 💡 Tip: Always use **tagged images or digests** (e.g., `my-app:1.2.3` or `my-app@sha256:...`) instead of mutable tags like `latest` for predictable and traceable deployments.

---

### Why Use `Rollout` Instead of `Deployment`?

While a Kubernetes `Deployment` supports basic strategies like **rolling updates**, it lacks built-in support for:

* Gradual traffic shifting
* Health-based gating
* Real-time metric analysis
* Automated rollback

By using `Rollout`, we unlock the full spectrum of **progressive delivery capabilities**, allowing safer, observable, and more controlled software releases in production.

---

In the next step, we’ll wire up this rollout with a GitHub Actions workflow that builds and publishes a container image, and automatically updates this manifest to trigger the deployment process.

---

## Step 2: Create a GitHub Actions Workflow

Once the Argo Rollout manifest is in place, the next step is to **automate the build and deployment process** using GitHub Actions. This pipeline ensures that every code change pushed to the `main` branch results in a new container image, an updated Kubernetes manifest, and a progressive rollout via Argo.

Here’s a straightforward example of a GitHub Actions workflow file to achieve this:

> **`.github/workflows/deploy.yml`**

{% raw %}
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
{% endraw %}
---

### What This Workflow Does

Let’s break it down step by step:

#### 1. Trigger on Push to `main`

{% raw %}
```yaml
on:
  push:
    branches:
      - main
```
{% endraw %}

The workflow triggers automatically whenever new code is pushed to the `main` branch — typically after merging a pull request. This ensures production deployments only occur after explicit approvals.

---

#### 2. Build and Push the Docker Image

{% raw %}
```yaml
docker build -t ghcr.io/your-org/my-app:${{ github.sha }} .
docker push ghcr.io/your-org/my-app:${{ github.sha }}
```
{% endraw %}

This command:

* Builds a Docker image from the current repository.
* Tags the image with the short SHA of the commit (e.g., `a1b2c3d`).
* Pushes it to the [GitHub Container Registry](https://ghcr.io).

> Using commit SHAs as tags ensures that each deployment is uniquely versioned and easy to trace back to a specific code change.

---

#### 3. Update the Argo Rollout Manifest

{% raw %}
```bash
sed -i "s|ghcr.io/your-org/my-app:.*|ghcr.io/your-org/my-app:${{ github.sha }}|" k8s/rollout.yaml
```
{% endraw %}

This `sed` command modifies the `image:` field in the `rollout.yaml` manifest to reference the freshly built image.

This is the change Argo CD (or another GitOps controller) will detect — triggering a new progressive rollout using the updated container.

---

#### 4. Commit and Push the Manifest

{% raw %}
```bash
git commit -am "Update image to ${{ github.sha }}"
git push
```
{% endraw %}

Once the manifest is updated, we commit and push it back to the repository. This creates a GitOps-compatible workflow — where **Git is the source of truth** for deployment state.

> 💡 Tip: Make sure Argo CD is configured to watch this repository and path so it can pick up the change automatically.

---

This GitHub Actions workflow creates a clean, end-to-end CI/CD process:

1. **Detects changes** on the `main` branch.
2. **Builds and pushes** a new Docker image to GHCR.
3. **Updates** the rollout manifest with the new image tag.
4. **Commits and pushes** the change to Git.
5. **Triggers Argo Rollouts** to begin a progressive canary deployment.

All of this happens without needing to manually tweak YAML or run deployment scripts — just commit your code and push.

---

## Step 3: Integrate Observability for Safe Progressive Delivery

Progressive delivery is only as effective as the insights you gather during rollouts. Without proper observability, canary strategies become guesswork. Argo Rollouts addresses this by integrating with popular monitoring tools to **automatically evaluate metrics** during each rollout step.

### Real-Time Analysis with Metrics

Argo Rollouts supports built-in integrations with several observability platforms:

* **Prometheus**
* **Datadog**
* **New Relic**

These tools provide the telemetry required to make intelligent rollout decisions — such as **pausing**, **promoting**, or **aborting** a deployment based on live traffic behavior.

### Example: Prometheus-Based Analysis

Here’s an `AnalysisTemplate` that evaluates HTTP success rates using Prometheus:

{% raw %}
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
{% endraw %}

**What this does:**

* Every **1 minute**, it runs the query.
* Repeats this **3 times**.
* Expects a **success rate > 95%** to consider the step healthy.

This template can be attached to your rollout under the canary strategy:

{% raw %}
```yaml
strategy:
  canary:
    steps:
      - setWeight: 25
      - pause: { duration: 5m }
      - analysis:
          templates:
            - templateName: success-rate-check
```
{% endraw %}

This ensures each phase of the rollout is **guarded by real-time health checks**.

---

### Manual Promotion and Rollback

Even with automation in place, you may occasionally need to intervene manually — especially during production rollouts.

Argo Rollouts provides a CLI (`kubectl argo rollouts`) and a web-based dashboard to:

* **Check rollout status**
* **Manually promote** to the next step
* **Abort** an unhealthy rollout immediately

### Useful Commands:

{% raw %}
```bash
# Check rollout status
kubectl argo rollouts get rollout my-app

# Promote to the next canary step
kubectl argo rollouts promote my-app

# Abort the rollout (revert to stable)
kubectl argo rollouts abort my-app
```
{% endraw %}

> Pro tip: You can also use Slack or Prometheus alerts to notify engineers before manual intervention is needed.

---

## Best Practices for Safe Rollouts

To keep your progressive delivery strategy both **safe** and **repeatable**, follow these best practices:

### Pin Image Tags

Avoid using `:latest`. Always tag Docker images with Git SHAs:

{% raw %}
```bash
ghcr.io/your-org/my-app:<commit-sha>
```
{% endraw %}

This guarantees immutability and traceability across builds.

---

### Use GitOps Principles

Let Git be the single source of truth. Commit all manifest changes (including rollout strategies and analysis templates) to version control. This:

* Enables rollbacks via `git revert`
* Provides a full audit trail
* Encourages peer-reviewed configuration changes

---

### Monitor During Pauses

During each pause step (`pause: { duration: 5m }`), closely observe dashboards, metrics, and logs. For extra safety:

* Set up alerts based on metric degradation
* Notify your team via Slack or PagerDuty

---

### Segment Traffic with Ingress or Service Mesh

Progressive rollouts are most effective when paired with **traffic segmentation**. Use tools like:

* **NGINX Ingress Controller**
* **Istio**
* **Linkerd**

This enables targeting only a subset of users or regions with the new version before promoting it globally.

---

### Define Tight Error Budgets

Fail fast. Don’t wait until customers complain. Instead:

* Define strict **success conditions** (e.g., success rate > 95%, latency < 500ms)
* Abort the rollout automatically if those conditions aren't met
* Rely on Argo Rollouts' integration with metrics to enforce this

---

## Final Thoughts

**Progressive delivery** is about strategy, not tooling. Tools like **GitHub Actions** and **Argo Rollouts** make it easier, but the value lies in how well you manage **risk, observability, and user exposure**.

By blending GitOps, canary rollouts, and real-time telemetry, your team can move faster—without sacrificing safety or sleep.

**#CICD #Tools #Deployment #30DaysOfDevOps**
