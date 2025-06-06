---
layout: post
title: "Debugging a Broken CI Pipeline in GitHub Actions: A Real-World Walkthrough"
date: 2025-06-06 18:18:00 +0700
comments: true
---

## 🧩 *“It Works on My Machine” — 5 CI/CD Debugging Stories That Didn’t Care*

In modern development, CI/CD pipelines are the backbone of shipping fast, safe, and consistent software. We trust them to be reliable, reproducible, and immune to the quirks of individual developer environments.

And yet…

> Sometimes your perfectly functional local code hits the CI wall and crumbles.
> Other times, it sails through the CI — only to break in production.
> Occasionally, it works in both places — and still does the wrong thing.

This article shares **five real-world debugging cases** — drawn from production systems — where GitHub Actions CI/CD pipelines broke in surprising ways:

* A **Node.js monorepo** where environment paths caused broken builds in CI.
* A **Python app** where a test failed *only* in the cloud.
* A **PHP microservice** where CI skipped a critical script step.
* A **Go project** where a race condition was invisible locally.
* A sneaky **Docker cache issue** that served outdated code *with a smile*.

Each case was painful to debug. Each taught us something valuable. And each reinforces the same truth:

> **CI/CD is software too — and it needs testing, observability, and care like everything else.**

Let’s dive in.

---

> *“It worked yesterday!” – Every developer ever.*

CI pipelines are fantastic—when they work. But what happens when they break right before a crucial release? In this post, we’ll walk through a real-world example of a broken GitHub Actions CI pipeline, how it was diagnosed, and how we restored it step-by-step.

Whether you're a DevOps engineer, software developer, or tech lead, this post aims to equip you with practical strategies and tips for efficiently debugging GitHub Actions pipelines.

---

## 🚨 The Situation: Unexpected CI Failure on a Hotfix Branch

We maintain a moderately complex Node.js microservice that runs automated builds and tests on every push using GitHub Actions. One Friday afternoon, just before a minor production hotfix, we noticed something odd:

```text
❌ CI failed on the hotfix branch.
```

This was unexpected, especially since the same commit on the `main` branch had passed CI earlier that week. We hadn’t made any changes to the `.github/workflows/ci.yml` file either.

### The Workflow File

Here’s a simplified version of our `ci.yml` workflow:

```yaml
name: CI Pipeline

on:
  push:
    branches:
      - main
      - hotfix/*
  pull_request:

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
      - run: npm install
      - run: npm run test
```

---

## 🔍 Step 1: Reading the Logs

The first rule of debugging CI: **always read the logs carefully.** In our case, the failure was during the `npm install` step:

```bash
npm ERR! code ERESOLVE
npm ERR! ERESOLVE could not resolve dependency: 
npm ERR! peer some-package@">=2.0.0 <3.0.0" from another-package@1.1.0
```

Looks like a dependency conflict. But why now?

---

## 🔍 Step 2: Reproducing Locally

To isolate the problem, we tried the following on a fresh clone:

```bash
git checkout hotfix/fix-cors
rm -rf node_modules package-lock.json
npm install
```

💥 Boom — same error locally.

This confirmed that the issue was not GitHub-specific but a real dependency resolution problem. After checking the `package.json` file, we realized that the version range of `some-package` was loose, and the new release (published just hours ago) had introduced breaking changes.

---

## 🛠️ Step 3: Locking Down Dependencies

This was an **example of a non-deterministic CI pipeline** due to non-locked dependencies. We resolved it by:

1. Updating `package.json` to use strict versions:

   ```json
   "some-package": "2.4.1",
   "another-package": "1.1.0"
   ```

2. Regenerating the lockfile:

   ```bash
   rm package-lock.json
   npm install
   git add package-lock.json
   ```

3. Pushing the changes:

   ```bash
   git commit -m "Fix: Lock dependencies to resolve CI failure"
   git push origin hotfix/fix-cors
   ```

✅ CI passed.

---

## 🔄 Step 4: Improving Pipeline Robustness

After resolving the immediate issue, we took time to make the pipeline more robust against similar issues in the future.

### ✅ Added a Dependency Audit Step

```yaml
- name: Audit Dependencies
  run: npm audit --audit-level=moderate
```

### ✅ Enforced Dependency Locking with `package-lock.json`

We added a lint step to verify the lockfile is committed and not out-of-sync:

```yaml
- name: Check for Modified Lockfile
  run: |
    git diff --exit-code package-lock.json || (echo "package-lock.json modified. Run npm install and commit changes." && exit 1)
```

### ✅ Introduced Dependency Pinning Policy

We enforced a policy to use exact versions for all production dependencies.

---

## 🧠 Lessons Learned

### 1. **Always Read the Logs.**

CI logs are verbose for a reason. The answer is often hidden in plain sight.

### 2. **Reproduce Locally.**

If the error happens in CI, try running the same commands in a clean local environment.

### 3. **Lock Your Dependencies.**

Non-deterministic builds will break at the worst possible time. Lock everything. Commit your lockfiles.

### 4. **Pin Tooling Versions.**

We later realized that `actions/setup-node@v4` always installs the latest minor version unless pinned. We switched to:

```yaml
- uses: actions/setup-node@v4.0.2
```

To avoid surprises.

### 5. **Don't Ignore Broken CI.**

If CI is failing but your app still “works,” don’t ignore it. It’s a sign of tech debt that will come back to haunt you.

---

## 🧰 Bonus: Tools to Help Debug GitHub Actions Faster

* **act**: Run GitHub Actions locally. Great for quick iteration.
  👉 [https://github.com/nektos/act](https://github.com/nektos/act)
* **tmate**: Add an SSH debug session during CI runs.
  👉 [https://github.com/marketplace/actions/debug-with-tmate](https://github.com/marketplace/actions/debug-with-tmate)
* **workflow\_run**: Trigger downstream jobs conditionally if upstream CI passes.

---

## 📦 Final Thoughts

CI pipelines are the safety net for modern software teams. But they’re not immune to the entropy of ever-changing dependencies and environments.

By adopting strict versioning, reading logs carefully, and investing in proper diagnostics, you can turn a mysterious CI failure into a clear-cut engineering win.

---

### ✅ TL;DR Checklist for Debugging Broken GitHub Actions:

* [x] Review logs in detail
* [x] Try to reproduce locally
* [x] Check for recently updated dependencies
* [x] Pin versions (Node, Actions, packages)
* [x] Commit and validate lockfiles
* [x] Add safety checks in your workflow

---

# Debugging a Broken CI Pipeline in GitHub Actions: When It *Only* Fails in CI

> “It works on my machine” — a phrase that sends shivers down every DevOps engineer's spine.

GitHub Actions is a powerful CI/CD platform—but sometimes, debugging a broken pipeline can feel like solving a mystery. What’s worse? When the pipeline **fails in CI but works perfectly on your local machine**.

In this blog, we walk through a real-world example of this exact scenario, how we approached the problem, and what strategies helped us find and fix the root cause.

---

## ⚠️ The Incident: CI Failing on a Merge Request, but Local Works Fine

Our engineering team was preparing a feature branch merge for a Python Flask API. The GitHub Actions workflow ran tests on all PRs to `main`. But suddenly:

```text
❌ CI job failed: "Test Backend / Run Unit Tests"
```

This PR passed locally on every developer machine. No recent changes had been made to the CI workflow file. Strange.

Here’s the CI job snippet:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: |
          python -m pip install -r requirements.txt
      - name: Run unit tests
        run: |
          pytest tests/
```

And here was the error:

```bash
ImportError: cannot import name 'url_quote' from 'werkzeug.urls'
```

But this didn’t make sense. Running `pytest` locally worked fine across multiple developer environments. We verified the Python version, dependency versions, and OS (all using Ubuntu).

---

## 🔍 Step 1: Double-Checking Python and Package Versions

We printed dependency versions both locally and in CI:

```bash
pip freeze | grep werkzeug
```

* ✅ Locally: `Werkzeug==2.3.0`
* ❌ In CI: `Werkzeug==3.0.0`

That was the first clue. CI was installing newer versions of dependencies than expected.

Our `requirements.txt` file did **not pin** `werkzeug`:

```
Flask>=2.2
```

Turns out `Flask 2.2` pulls in `Werkzeug 3.0`, which had removed `url_quote`. Locally, everyone had older versions cached or used existing virtual environments.

---

## 🧠 Step 2: Why CI Behaves Differently Than Local

CI builds from scratch. No caching. No environment pollution.

Local environments are often not clean—even when you `pip install`, you might rely on previously installed transitive dependencies. In contrast, CI starts from a fresh virtual machine each time.

Lesson: *If it’s not pinned, it’s not predictable.*

---

## 🛠️ Step 3: Pin Everything

To avoid future surprises, we:

1. Created a frozen lock file:

```bash
pip freeze > requirements.txt
```

2. Replaced open-ended dependencies in `requirements.in` (we use `pip-tools`) like:

```txt
Flask>=2.2  →  Flask==2.2.5
Werkzeug==2.3.0
```

3. Added a safety net in our CI:

```yaml
- name: Validate dependency versions
  run: |
    pip freeze | grep Werkzeug | grep "==2.3.0" || exit 1
```

4. Rebuilt the CI pipeline.

Now both local and CI environments use the same packages.

---

## 🔁 Step 4: Making CI Fail More Transparently

The CI failure message originally gave no clue *why* the import broke. We updated our `pytest.ini` to include:

```ini
[pytest]
log_cli = true
log_cli_level = DEBUG
```

Now, failed tests provided better tracebacks and environmental context.

We also used GitHub's `upload-artifact` action to persist test logs:

```yaml
- name: Upload Test Logs
  if: failure()
  uses: actions/upload-artifact@v4
  with:
    name: pytest-logs
    path: logs/
```

---

## 🧪 Bonus: Use Matrix Testing to Detect Version-Specific Failures

To prevent future version mismatches, we introduced matrix testing:

```yaml
strategy:
  matrix:
    python-version: ['3.9', '3.10', '3.11']
```

This caught other hidden issues and ensured forward compatibility.

---

## 💡 Key Takeaways

### 🔧 1. CI is your clean room

CI environments start fresh. That’s a blessing for reproducibility, but a curse for hidden local assumptions.

### 📌 2. Pin dependencies aggressively

Lock versions of both direct and transitive dependencies in production software, especially in Python.

### 🧹 3. Validate your assumptions

Use tools like `pip freeze`, logging, and artifact upload to add transparency to failures.

### 🧰 4. Use local CI emulators

Try [`act`](https://github.com/nektos/act) to simulate GitHub Actions locally, and spot divergence early.

---

## 🧵 TL;DR: Checklist When CI Fails but Local Passes

* [x] Compare package versions with `pip freeze`, `npm ls`, or `composer show`.
* [x] Print runtime logs in CI to see context (`log_cli` or verbose flags).
* [x] Re-run CI with debugging (use `debug-tmate` to SSH into a CI machine).
* [x] Pin all dependencies—don’t rely on semver ranges.
* [x] Add matrix testing for environment parity.

---

## 📦 Final Thoughts

The worst CI bugs are the ones you *can't* reproduce. But they're also the most valuable to fix—because they expose hidden assumptions in your build and deployment process.

The next time CI fails mysteriously, treat it as a signal. It’s not just a failure—it's your system asking for more rigor.

---

# Debugging a Broken GitHub Actions Pipeline in a PHP Project (When CI Breaks but Your Laptop Doesn't)

When maintaining a PHP web application, you may have your code, tests, and tools working perfectly on your development machine — only to have your CI pipeline break with a cryptic error. This is the story of such a case in a Laravel project.

---

## 🧩 The Scenario: PHPStan Fails Only in CI

We recently introduced [PHPStan](https://phpstan.org/) — a popular static analysis tool — into our Laravel-based project. Locally, `vendor/bin/phpstan analyse` ran cleanly.

But the CI pipeline started failing on every pull request:

```text
Fatal error: Uncaught Error: Class 'Illuminate\Support\Facades\Route' not found
```

At first glance, this made no sense:

* The Laravel project used Composer correctly.
* All packages were installed.
* No error locally.

The job looked like this:

```yaml
name: CI

on: [push, pull_request]

jobs:
  analyse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          tools: composer, phpstan

      - name: Install dependencies
        run: composer install --no-interaction --prefer-dist

      - name: Run PHPStan
        run: vendor/bin/phpstan analyse
```

We were stuck. Why would PHPStan crash in CI?

---

## 🕵️ Step 1: Investigate the Stack Trace

PHPStan’s error pointed to something like:

```php
Illuminate\Support\Facades\Route::get(...);
```

This is standard Laravel usage. But the trace revealed that **autoloading failed**. That’s our next clue.

---

## 🛠️ Step 2: Composer Autoload and Caching

We enabled verbose logging in CI:

```yaml
- name: Dump Autoloader
  run: composer dump-autoload -o -vvv
```

And added a debug step:

```yaml
- name: Check Autoload File
  run: ls -l vendor/composer/autoload_*.php
```

Everything looked fine, but PHPStan still failed. So we suspected caching.

We were using this caching block:

```yaml
- name: Cache Composer dependencies
  uses: actions/cache@v3
  with:
    path: vendor
    key: ${{ runner.os }}-php-${{ hashFiles('composer.lock') }}
    restore-keys: |
      ${{ runner.os }}-php-
```

We commented out the cache — and CI passed.

🎯 **Root Cause Identified**: The Composer cache stored a `vendor` directory built under an older PHP version and outdated dependencies. The autoloader became invalid in the new environment, breaking class resolution.

---

## ✅ Step 3: Fixes Implemented

### ✅ Use Composer Cache *Correctly*

Never cache `vendor/` directly unless you're absolutely certain of environment parity.

We switched to caching only the Composer cache directory:

```yaml
- name: Cache Composer files
  uses: actions/cache@v3
  with:
    path: ~/.composer/cache
    key: ${{ runner.os }}-composer-${{ hashFiles('composer.lock') }}
```

This allowed Composer to re-download packages if needed, but still benefit from caching.

### ✅ Enforce Clean Builds

We added a `composer validate` and `composer install --no-scripts --no-autoloader` step to detect inconsistencies.

---

## 🛡️ Step 4: Harden the CI Pipeline

To ensure we never cache `vendor/` again by mistake, we added a CI lint check:

```yaml
- name: Validate no vendor cache
  run: |
    if grep -q "vendor" .github/workflows/*.yml; then
      echo "Don't cache vendor/ — it's error-prone!"
      exit 1
    fi
```

We also added a separate GitHub Actions workflow just for static analysis and type checking:

```yaml
name: Static Analysis

on: [push, pull_request]

jobs:
  phpstan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          tools: phpstan, composer
      - run: composer install --no-dev --no-scripts
      - run: phpstan analyse
```

---

## 🔁 Recap: CI Breaks but Local Works — in PHP

| 🔍 Symptom                           | ✅ Root Cause                                 |
| ------------------------------------ | -------------------------------------------- |
| `Class not found` in PHPStan         | Composer cache from stale `vendor/` dir      |
| CI only fails, local works fine      | Local uses valid autoload from fresh install |
| Static analysis fails inconsistently | Cache is environment-specific                |

---

## 💡 Key Takeaways

### 🚫 1. Don't cache `vendor/` in CI

It introduces subtle, environment-specific bugs. Use Composer’s internal cache instead.

### 🛠️ 2. Validate Composer and Autoload

Add checks like `composer validate`, `composer dump-autoload`, and version consistency in CI.

### 🔍 3. Isolate Static Analysis

Run tools like PHPStan or Psalm in a clean, separate job — not mixed with build/test.

### 🧪 4. Use verbose flags in CI

More logs = faster debugging. Use `-v` or `--debug` for Composer, PHPUnit, PHPStan, etc.

---

## 📦 Final Words

CI/CD pipelines are the gatekeepers of code quality. When they break for reasons that don’t happen locally, they expose gaps in reproducibility and environment isolation.

These aren't just bugs — they’re opportunities to build more resilient systems.

Next time your Laravel or PHP project breaks in CI, ask yourself:

* Is this failure deterministic?
* Is caching introducing state I didn’t expect?
* Can I replicate this in a clean Docker container?

Sometimes, the best debugging tool is to rebuild from scratch — the same way your CI does.

---

# 🐳 When Docker Betrays You: Debugging a CI/CD Cache Issue That Doesn't Happen Locally

**"It builds fine on my machine."**
**"CI says otherwise."**

This is the story of a subtle Docker caching bug that broke our production deploy — despite everything working perfectly during local testing.

---

## 🎯 The Context

We were working on a PHP microservice deployed as a Docker container. Our `Dockerfile` used multi-stage builds for better performance and caching:

```Dockerfile
# Build stage
FROM php:8.2-cli AS build
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --prefer-dist

# Final image
FROM php:8.2-cli
WORKDIR /app
COPY --from=build /app /app
COPY . .
CMD ["php", "index.php"]
```

Everything was fine locally. CI built it. We deployed it. But then… 🎭

---

## 🧨 The Problem: Outdated Source Code in Production

After a successful build and push from GitHub Actions, we deployed the image to ECS.

But our new features were **not showing up**. Somehow, it was running **old code**.

🤯 Logs showed the application running with a previous release.
🤖 CI logs said everything built successfully.
✅ Tag was `myapp:latest` — exactly as expected.

Locally, everything worked. Even pushing manually from a local machine resulted in the correct behavior.

So what was going on?

---

## 🔍 Step 1: Check the Build Logs

We added verbose output to our GitHub Actions workflow:

```yaml
jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build & Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: myorg/myapp:latest
```

🪵 We noticed this:

```
#3 [internal] load metadata for docker.io/library/php:8.2-cli
#4 [internal] load build context
#5 [1/4] FROM docker.io/library/php:8.2-cli@sha256:...
...
#9 exporting to image
```

The output said the image was built and pushed — but it didn't actually **rebuild the image**. Most steps used **cache hits**.

---

## 🤯 Step 2: Understand What Docker Is Caching

The critical line in the Dockerfile:

```Dockerfile
COPY . .
```

This step comes **after** the `COPY composer.json composer.lock` and `composer install` in the build stage. But:

* **We were copying the source code too late.**
* Docker saw no changes to `composer.json` → cache hit.
* The `COPY . .` didn't re-trigger because Docker cached the final image layer **from a previous job**.

The problem was that our GitHub Actions runner didn’t always detect a code change — because Docker layer cache **was being reused** from a previous build.

---

## 🧪 Step 3: Reproduce Locally (You Can’t)

On a dev laptop, changes to the working directory **always** invalidated the Docker cache:

```bash
docker build -t myapp:latest .
```

Boom — rebuilt from scratch.

But in GitHub Actions, Docker used **BuildKit remote caching** unless told otherwise. And since cache hits were pulled from previous builds, the `COPY . .` layer sometimes reused stale context.

---

## 🛠️ Step 4: Force Cache Invalidation in CI

We added an explicit cache key and Git SHA:

```yaml
- name: Build and Push with Cache Busting
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: myorg/myapp:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
    build-args: |
      CACHEBUST=${{ github.sha }}
```

And updated the Dockerfile to respect it:

```Dockerfile
ARG CACHEBUST=1
COPY . .
```

This tiny `ARG` acts as a **cache buster** — forcing Docker to re-evaluate the layer when the Git SHA changes.

---

## 🧩 Bonus Fix: Don't Copy Everything Blindly

To avoid copying test files, `.git`, or random unneeded files, we added a `.dockerignore`:

```text
.git
tests/
node_modules/
*.log
```

That sped up CI and avoided unexpected cache pollution.

---

## 🔒 Step 5: Harden the Pipeline

We added:

* A post-build validation script to test whether expected files existed in the final container:

```yaml
- name: Run smoke test inside container
  run: |
    docker run --rm myorg/myapp:latest php -r "file_exists('index.php') || exit(1);"
```

* Unique version tags (`myapp:sha-${{ github.sha }}`) to avoid pushing "latest" from every PR.

---

## 🧠 Lessons Learned

| Issue                       | Root Cause                          | Fix                                         |
| --------------------------- | ----------------------------------- | ------------------------------------------- |
| Image built from old source | Docker cache reused stale layers    | Use `CACHEBUST` arg with Git SHA            |
| CI "builds" were no-ops     | BuildKit optimized too aggressively | Use cache-from/cache-to explicitly          |
| Deployment had wrong code   | `COPY . .` was not triggered        | Add `.dockerignore`, change build structure |

---

## ✅ Key Takeaways

1. **Docker cache is powerful — and dangerous** in CI if not controlled.
2. **GitHub Actions uses BuildKit and layer caching across jobs** — this can lead to stale images.
3. **Use a `CACHEBUST` argument** tied to your commit hash.
4. **Don't rely solely on `:latest` tags** — tag with Git SHAs or release versions.
5. **Add smoke tests inside your CI/CD pipeline** to validate your container post-build.

---

## 🧭 Final Thoughts

CI/CD is often about finding edge cases your local environment hides. Docker's smart caching is a blessing for speed — but it will happily give you yesterday's leftovers if you're not careful.

Always assume **CI has a memory**, and sometimes it remembers the wrong things.

----

## 🧯 CI/CD as a First-Class Citizen in Your Debugging Mindset

These stories highlight a simple but often neglected truth: **your pipeline is part of your product**.

It builds your code.
It decides what gets tested.
It decides what gets shipped.
And it’s subject to all the same bugs, assumptions, and regressions as the code it delivers.

So the next time you debug a failure, ask not only:

* “What changed in the code?”

But also:

* “What changed in the pipeline?”
* “What assumptions am I making about CI behavior?”
* “What’s different between CI and local?”

**Good CI pipelines are invisible when they work. Great engineers investigate them when they don’t.**

Make your CI/CD system something you understand — not just something you inherit.
And always remember:

> If it works on your machine, but not in CI — **CI is right.**
