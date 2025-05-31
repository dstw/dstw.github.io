---
layout: post
title: "Designing a Progressive Delivery Pipeline with GitHub Actions and Argo Rollouts"
date: 2025-05-31 20:18:22 +0700
comments: true
---

**Progressive delivery** isn’t just a buzzword—it's a strategic evolution of continuous delivery practices aimed at safer, more controlled releases. While tools like **GitHub Actions** (CI/CD automation) and **Argo Rollouts** (Kubernetes-native canary and blue-green deployment controller) are excellent enablers, the underlying value comes from the **deployment strategy** and **observability-first mindset**.

This article walks through the design of a modern progressive delivery pipeline—starting with a grounding in **deployment strategies**, then progressing through a tool-agnostic pipeline architecture, with GitHub Actions and Argo Rollouts as practical examples. We'll also touch on popular **alternatives** at each stage.

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

---

## Designing the Pipeline

A typical progressive delivery pipeline includes these core stages:

1. **CI: Build & Test**
   Use GitHub Actions to run tests, build Docker images, and push them to a container registry.

2. **CD: Push Artifact & Update Manifests**
   Automatically update Kubernetes manifests (e.g., image tag) and push to a Git repo.
    📌 Git becomes the source of truth.

3. **GitOps Sync**
   Argo CD detects changes and syncs them to the cluster. It ensures that your live state matches Git.

4. **Progressive Rollout & Metrics Evaluation**
   Argo Rollouts gradually shifts traffic to the new version (e.g., 10% → 30% → 100%), with pauses for metric checks.

5. **Promotion / Rollback Decisioning**
   Use metrics (e.g., error rate from Prometheus) to auto-promote or rollback. Optional manual approval gates.

Let’s break this down using **GitHub Actions** + **Argo Rollouts**.

---

## Stage 1: Continuous Integration (CI)

### **Goal:**

Build, test, and package your application automatically on every change to ensure it's **always in a deployable state**.

### **What Happens in This Stage:**

1. **Trigger:**

   * A developer pushes code to the `main` or `feature` branch on GitHub.
   * GitHub Actions detects the change and starts the CI workflow.

2. **Checkout Code:**

   * Uses the `actions/checkout@v3` action to pull the latest source code.

3. **Run Unit Tests and Static Checks:**

   * Executes language-specific tests (e.g., `npm test`, `go test`, `pytest`).
   * Runs linters (e.g., ESLint, Flake8) to enforce coding standards.

4. **Build Docker Image:**

   * Compiles the application and builds a Docker image.
   * Tags the image using the Git SHA to uniquely identify the build (`myapp:<SHA>`).

5. **Push to Registry:**

   * Pushes the built image to a container registry (e.g., GitHub Container Registry, ECR, GCR, Docker Hub).

---

### **Example GitHub Actions CI Workflow**

```yaml
name: CI

on:
  push:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Docker
        uses: docker/setup-buildx-action@v2

      - name: Run tests
        run: |
          make test  # Or any test runner

      - name: Build and push Docker image
        env:
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t ghcr.io/myorg/myapp:$IMAGE_TAG .
          echo ${{ secrets.GITHUB_TOKEN }} | docker login ghcr.io -u USERNAME --password-stdin
          docker push ghcr.io/myorg/myapp:$IMAGE_TAG
```
---

### **Why This Stage Matters**

* **Shift-left testing:** Catches errors early.
* **Repeatability:** Ensures builds are consistent and reproducible.
* **Speed:** Automated builds reduce manual effort.
* **Auditability:** Each image is traceable to a Git commit.

---

## Stage 2: Continuous Delivery (Manifest Update)

### **Goal:**

Update the container image reference in a Kubernetes manifest (using Argo Rollouts for progressive deployment), then commit and push this change to Git. This triggers Argo CD to apply the update automatically.

---

### Prerequisites:

* You're using **GitHub Actions** for CI/CD.
* Your Kubernetes manifest is a plain YAML file — typically an `Argo Rollout` resource, not a standard `Deployment`.
* Argo CD is set up to monitor the Git repository for changes.
* Argo Rollouts is installed in the cluster and replaces your standard Deployment object.

---

### Argo Rollout Manifest Example (`rollout.yaml`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause: {duration: 30s}
        - setWeight: 60
        - pause: {duration: 60s}
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: ghcr.io/myorg/myapp:oldsha
          ports:
            - containerPort: 8080
```

Your CI job's output will be a new image tag like `ghcr.io/myorg/myapp:<sha>` that you need to update in this manifest.

---

### GitHub Actions Job to Update the Image Tag

```yaml
jobs:
  update-rollout-manifest:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout manifest repository
        uses: actions/checkout@v3
        with:
          repository: myorg/k8s-manifests
          token: ${{ secrets.GITHUB_TOKEN }}
          ref: main

      - name: Update Rollout manifest with new image
        run: |
          NEW_IMAGE_TAG="ghcr.io/myorg/myapp:${{ github.sha }}"
          sed -i "s|image: .*|image: ${NEW_IMAGE_TAG}|" path/to/rollout.yaml

      - name: Commit and push changes
        run: |
          git config user.name "ci-bot"
          git config user.email "ci@bot.com"
          git add path/to/rollout.yaml
          git commit -m "Update rollout image to ${{ github.sha }}"
          git push origin main
```

> **Note:** Make sure your GitHub Actions runner has permission to push changes to the manifest repo (`GITHUB_TOKEN` must have `contents: write` scope).

---

### What Happens After

* Argo CD, which continuously watches the `main` branch of the manifest repository, detects the updated `rollout.yaml`.
* It applies the change to the cluster.
* Argo Rollouts takes over the actual deployment:

  * Starts rolling out the new image.
  * Follows the defined canary strategy (e.g., 20% → 60% → 100%).
  * Pauses as defined between steps.

---

### Benefits of This Approach

* **Simple & native:** No extra templating tools needed.
* **Traceable:** Every rollout is versioned in Git.
* **Safe rollout:** Canary strategy is enforced by Argo Rollouts.
* **Rollback-ready:** Reverting the Git commit reverts the deployment.

---

## Stage 3: GitOps Sync (CD Trigger)

### **Goal:**

Deploy the updated manifest from GitHub directly to your Kubernetes cluster by triggering a rollout via GitHub Actions.

---

### Context:

Since we're **not using a GitOps controller** like Argo CD, the sync between Git and the cluster will be **manually triggered by GitHub Actions** — but we still follow GitOps principles:

* **Declare state in Git**
* **Update manifests via commits**
* **Drive deployments via version control**

---

### What Happens in This Stage:

1. **GitHub Actions updates the manifest** (done in Stage 2).
2. In this stage, the same or subsequent GitHub Actions job will:

   * Connect to the Kubernetes cluster.
   * Apply the updated manifest using `kubectl`.
   * This triggers Argo Rollouts to begin a progressive rollout (canary or blue-green).

---

### GitHub Actions Example for Manual Sync

```yaml
jobs:
  sync-to-cluster:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout updated manifests
        uses: actions/checkout@v3

      - name: Set up kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'latest'

      - name: Authenticate with Kubernetes
        run: |
          echo "${KUBE_CONFIG}" > kubeconfig.yaml
          export KUBECONFIG=$PWD/kubeconfig.yaml

      - name: Apply the rollout manifest
        run: |
          kubectl apply -f path/to/rollout.yaml
```

> Replace `KUBE_CONFIG` with your base64-encoded kubeconfig, stored as a GitHub Actions secret.

### What This Triggers:

* **Argo Rollouts sees a new image tag** in the `Rollout` manifest.
* It **initiates a progressive delivery** strategy (canary, blue-green, etc.) as defined in the spec.
* The rollout proceeds through its steps — controlled by `setWeight`, `pause`, and optionally, manual promotion.

This stage completes the delivery loop — syncing Git changes into the cluster without relying on any external CD tool other than Argo Rollouts, while still preserving the spirit of GitOps: Git is the source of truth.

---

## Stage 4: Progressive Rollout (Canary Deployment)

### **Goal:**

Gradually roll out the new version of your application to a subset of users, monitor it, and promote it to full production **step by step**, using the native capabilities of **Argo Rollouts**.

---

### What Is a Canary Deployment (again)?

A **canary deployment** gradually shifts traffic from the stable (old) version of an application to a new version in controlled steps. It allows you to catch issues early, reduce blast radius, and roll back if needed — all **without downtime**.

---

### Example Argo Rollout Spec (Canary Strategy):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 4
  strategy:
    canary:
      canaryService: myapp-canary
      stableService: myapp-stable
      steps:
        - setWeight: 20
        - pause: {duration: 1m}
        - setWeight: 60
        - pause: {}
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: ghcr.io/myorg/myapp:<new-sha>
```
---

### Step-by-Step Rollout Process

1. **New Rollout Applied:**
   The updated manifest with a new image tag is applied (from Stage 3). This triggers Argo Rollouts to begin a new rollout.

2. **First Step - 20% Traffic to New Version:**
   Argo Rollouts updates **20% of the pods** with the new image and routes **20% of traffic** to `myapp-canary` (if configured with ingress or service mesh).

3. **Pause for 1 Minute:**
   During this pause, Argo Rollouts waits. You can:

   * Manually inspect the rollout via `kubectl argo rollouts get rollout myapp`.
   * Run tests or checks to validate the new version.

4. **Second Step - 60% Traffic:**
   Argo Rollouts proceeds to update more pods and route more traffic:

   * Now **60%** of traffic goes to the new version.

5. **Pause Indefinitely:**
   The rollout pauses, waiting for a **manual promotion**. This is useful if you want a final approval step before full rollout.

6. **Manual Promotion (optional):**
   If you're satisfied with the canary, promote it using the CLI:

   ```bash
   kubectl argo rollouts promote myapp
   ```

7. **Rollout Completes:**
   All pods are updated to the new version. The `stableService` now points to the new version.

---

### Monitoring Rollouts

You can monitor rollout progress in real-time:

```bash
kubectl argo rollouts get rollout myapp --watch
```

This shows:

* Status (Paused / Progressing / Healthy / Aborted)
* Current step and weight
* Number of updated pods
* Errors (if any)

---

### If Something Goes Wrong

You can **abort** the rollout at any time:

```bash
kubectl argo rollouts abort myapp
```

This will:

* Revert traffic to the stable version
* Scale down the canary pods

---

## Stage 5: Metrics-Driven Decisions

### Goal:

* Use **automated checks** where safe (e.g., staging).
* Allow **manual judgment** where needed (e.g., production).
* Blend both paths with GitHub Actions + Argo Rollouts.

---

## Option A: Automated Promotion (Safe Environments)

### How It Works:

1. After rollout pauses at a step (`pause: {duration: 2m}` or `pause: {}`), GitHub Actions:

   * Polls the rollout status.
   * Runs a **scripted health check** (e.g., `curl`, `kubectl logs`, exit codes).
   * If the app is healthy, it automatically promotes:

```yaml
- name: Check rollout status
  run: |
    STATUS=$(kubectl argo rollouts get rollout myapp -o jsonpath='{.status.pauseConditions}')
    echo "Pause Status: $STATUS"

- name: Run health check (basic)
  run: |
    curl -f http://myapp-canary.my-namespace.svc.cluster.local/health || exit 1

- name: Promote rollout if healthy
  if: success()
  run: kubectl argo rollouts promote myapp
```

This gives you **auto-promotion logic**, driven by GitHub Actions.

---

## Option B: Manual Promotion (Critical Environments)

### How It Works:

1. Rollout pauses with `pause: {}` (indefinite).
2. GitHub Actions sends a Slack/email/PR comment (if configured).
3. Engineers:

   * View dashboards, logs, alerts.
   * Use `kubectl argo rollouts promote` or `abort` based on judgment.

```yaml
- name: Notify about manual gate
  run: echo "Canary rollout paused. Please inspect and run 'kubectl argo rollouts promote myapp' manually."
```

You can also add an **approval step** in GitHub Actions:

```yaml
- name: Await manual approval
  uses: hmarr/auto-approve-action@v2
  if: github.event_name == 'pull_request'
```

Or simply wait for human input via a CLI tool, Slack ping, or dashboard review.

---

## Hybrid Strategy (Best of Both)

Combine both strategies:

* Use **automated checks** for basic validations.
* Pause rollout indefinitely for **final manual confirmation**.

Example `Rollout` steps:

```yaml
steps:
  - setWeight: 20
  - pause: {duration: 2m}    # auto-checks + auto-promote
  - setWeight: 60
  - pause: {}                # manual gate before full rollout
```
---

## Summary

| Strategy         | When to Use               | How                                     |
| ---------------- | ------------------------- | --------------------------------------- |
| Auto Promotion   | Staging/dev               | GitHub Action health checks + `promote` |
| Manual Promotion | Production                | Human verifies + uses `kubectl promote` |
| Hybrid           | Most real-world pipelines | Auto partial → manual final approval    |

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
