---
layout: post
title: "Debugging a Broken CI Pipeline in GitHub Actions"
date: 2025-06-06 18:18:00 +0700
comments: true
---

![Debugging Pipeline Illustration](/assets/images/debug_pipeline.jpg)

# “It Works on My Machine”

"What 5 CI/CD Failures Taught Us About Shipping Software"

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

# 🧵 A Node.js Monorepo Where Environment Paths Caused Broken Builds in CI

### …and Why “It Works on My Machine” Wasn’t Enough

---

## The Modern Monorepo Setup

We had what looked like a clean, scalable architecture: a **Node.js monorepo** using [Yarn Workspaces](https://classic.yarnpkg.com/en/docs/workspaces/) to manage multiple internal packages under one repository.

The structure:

```
/repo
  /packages
    /core       # shared logic & utilities
    /web        # web frontend using React
    /cli        # internal CLI tool
  package.json  # declares "workspaces": ["packages/*"]
```

Each subdirectory (`/web`, `/cli`, etc.) was its own package, with dependencies and scripts. The root managed dependencies across the entire workspace. Shared binaries like `eslint`, `tsc`, and our custom CLI tool lived in `devDependencies` at the root.

Locally, things were smooth:

* Developers used tools like `tsc`, `eslint`, `cli` directly from any workspace.
* `yarn install` hoisted dependencies and created symlinks under `node_modules/.bin/`.
* Running tests and builds from any subproject Just Worked™.

But then we shipped the first CI setup using GitHub Actions.
And things broke — hard.

---

## The CI Workflow: A Basic Start

Here’s the initial GitHub Actions workflow for CI:

```yaml
name: Build & Test

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 18

      - name: Install dependencies
        run: yarn install --frozen-lockfile

      - name: Run tests
        run: yarn workspace web test
```

We assumed this would mimic local development.

Instead, the CI blew up:

```
error Command failed with exit code 127.
sh: tsc: command not found
```

This error appeared for various binaries (`tsc`, `eslint`, `cli`, etc.) — all of which worked locally.

---

## The Investigation Begins

At first, we suspected something wrong with `yarn install`. But digging deeper, we realized:

* The error wasn’t about **missing packages**.
* It was about **missing binaries** — those normally exposed in `node_modules/.bin`.

To confirm, we added a debug step to inspect the `$PATH`:

```yaml
- run: echo $PATH
```

The result?

```text
/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
```

No `node_modules/.bin` in sight.

Meanwhile, `tsc`, `eslint`, etc., were symlinked correctly under:

```text
./node_modules/.bin/
```

So why were they invisible?

---

## Root Cause: PATH Assumptions and Subshells

There were **two main problems**, both related to assumptions developers often make when working locally.

### 1. **GitHub Actions Runs Commands in Fresh Subshells**

Each `run:` block in GitHub Actions executes in its own non-interactive shell. That means:

* Your `PATH` isn’t persistent across steps unless you export it explicitly.
* Changes to the environment in one step don’t survive into the next.

Unlike your terminal, which remembers when you `export PATH`, CI is stateless unless told otherwise.

### 2. **Monorepo Layout Obscures Context**

Within a monorepo, binaries often exist **only at the root**. If you `cd` into a workspace folder like `packages/web`, `node_modules/.bin` is not there — unless the workspace has its own install (which ours didn't, by design).

So when CI moved into `packages/web` to run tests, it no longer had access to shared binaries unless they were explicitly referenced.

---

## Testing Hypotheses

We tried changing the working directory in the `run` block:

```yaml
- name: Debug tsc
  working-directory: packages/web
  run: |
    ls ../../node_modules/.bin
    which tsc
```

Still:

```text
which: no tsc in (...)
```

`tsc` was present at `../../node_modules/.bin/tsc`, but it wasn't in the `$PATH`.

We even tried calling the binary directly:

```yaml
- run: ../../node_modules/.bin/tsc
```

This worked — but hardcoding paths wasn't sustainable across multiple workspaces and environments.

---

## The Fix: Explicitly Extend PATH

The most portable fix was to **add the workspace’s `.bin` directory to `PATH`** using GitHub Actions’ environment manipulation feature:

```yaml
- name: Add node_modules/.bin to PATH
  run: echo "${{ github.workspace }}/node_modules/.bin" >> $GITHUB_PATH
```

Now, every shell in the job has access to binaries like `tsc`, `eslint`, and `cli` — no matter which directory we run from.

We also simplified test calls by **running them from the root**, not per workspace:

```yaml
- run: yarn workspace web test
```

This ensured the scripts inherited the correct environment and binary access.

---

## What We Learned

1. **Your local terminal is lying to you.**
   Tools like VS Code, `direnv`, and your shell profile (.zshrc, .bashrc) help you without you realizing it. CI is a clean room — it only runs what you tell it.

2. **Monorepos introduce binary scope issues.**
   Binaries installed at the root may not be visible in subfolders unless the path is adjusted.

3. **GitHub Actions doesn’t persist shell environments across steps.**
   To make your pipeline reproducible, always treat each `run:` step as a new shell.

4. **Explicit is better than implicit.**
   Add `node_modules/.bin` to the `PATH`. Prefer fully-qualified script calls. Avoid relying on magical behavior from `yarn`.

---

## Final Working CI Snippet

Here’s the corrected and robust version of the CI workflow:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - uses: actions/setup-node@v4
        with:
          node-version: 18

      - run: yarn install --frozen-lockfile

      - name: Add node_modules/.bin to PATH
        run: echo "${{ github.workspace }}/node_modules/.bin" >> $GITHUB_PATH

      - name: Run tests
        run: yarn workspace web test
```

---

## TL;DR

> If your monorepo CI pipeline fails with “command not found” for a binary you know is installed —
> check your `PATH`.
> CI isn’t ignoring your code. It’s just following exactly what you told it to do.

---

# 🐍 A Python App Where a Test Failed *Only* in the Cloud

### …and How We Discovered the OS Knew More Than We Did

---

## The Setup

We had a small but growing Python web app that served as an internal API for reporting and metrics. The stack was lean:

* **Flask** for HTTP routing
* **SQLAlchemy** for the ORM
* **pytest** for tests
* **GitHub Actions** for CI

Developers ran the full test suite locally before pushing to main.
CI validated every pull request and ensured quality before merging.

Everything worked.

Until one day, a contributor pushed a simple patch — a few lines touching a timestamp formatting function — and CI failed:

```bash
=================================== FAILURES ===================================
___________ test_format_timestamp_with_tz[test_input_1-expected_output_1] ______
>       assert format_timestamp("2023-10-15T13:45:30Z") == "2023-10-15 13:45 UTC"
E       AssertionError: assert '2023-10-15 15:45 UTC' == '2023-10-15 13:45 UTC'
```

Locally? All green.

---

## “It Works on My Machine”

The developer confirmed:

```bash
pytest tests/test_time_utils.py
```

Passed on:

* macOS
* Windows WSL
* Ubuntu 22.04 (Docker)

But in CI — `ubuntu-latest` on GitHub Actions — the test failed. Every time.

---

## The Test in Question

The failing test checked a simple utility:

```python
def format_timestamp(iso_str):
    dt = datetime.fromisoformat(iso_str.replace("Z", "+00:00"))
    return dt.strftime("%Y-%m-%d %H:%M UTC")
```

And the test:

```python
@pytest.mark.parametrize("test_input,expected", [
    ("2023-10-15T13:45:30Z", "2023-10-15 13:45 UTC"),
])
def test_format_timestamp_with_tz(test_input, expected):
    assert format_timestamp(test_input) == expected
```

So… why was CI formatting the time as **15:45**, not **13:45**?

---

## The Deep Dive

We added a debug print:

```python
print(repr(dt))  # datetime object
```

In CI:

```text
datetime.datetime(2023, 10, 15, 15, 45, 30, tzinfo=datetime.timezone.utc)
```

Wait — **15:45**?

Locally, the exact same line produced:

```text
datetime.datetime(2023, 10, 15, 13, 45, 30, tzinfo=datetime.timezone.utc)
```

Same code. Same input string. Different result.

We added more debug output in CI:

```python
import os, time
print("TZ ENV:", os.environ.get("TZ"))
print("System TZ:", time.tzname)
```

Output:

```text
TZ ENV: None
System TZ: ('UTC', 'UTC')
```

Which... made it *weirder*. CI **claimed** it was in UTC. But the timestamp result was **2 hours ahead**.

---

## The Root Cause: A Library's Dirty Secret

We eventually isolated the issue:
**`datetime.fromisoformat` was behaving inconsistently across platforms.**

We confirmed:

```python
dt = datetime.fromisoformat("2023-10-15T13:45:30+00:00")
```

Was **interpreted correctly** in all environments, **but**:

```python
dt = datetime.fromisoformat("2023-10-15T13:45:30Z")
```

Was silently **misparsed on some OS/Python combinations**, including:

* Ubuntu 22.04
* Python 3.10.12 (used by `ubuntu-latest`)
* CI environments without explicit locale/timezone setups

Why?

Because **`fromisoformat()` does *not* support the "Z" suffix in Python <3.11**. It's a known quirk:

* `Z` is short for "Zulu time" (UTC)
* Python’s `fromisoformat()` never supported it until Python 3.11+

In our function, we manually replaced `"Z"` with `"+00:00"`, but in CI the string must have been **already altered by a dependency**, or inconsistently parsed. Possibly even influenced by a lower-level locale or `dateutil` conflict.

Bottom line: **you can’t trust `fromisoformat("...Z")` before 3.11**.

---

## The Fix

We swapped out `fromisoformat()` entirely and used `dateutil.parser`, which handles `Z` and other ISO formats properly:

```python
from dateutil import parser

def format_timestamp(iso_str):
    dt = parser.isoparse(iso_str)
    return dt.strftime("%Y-%m-%d %H:%M UTC")
```

And pinned it in `requirements.txt`:

```text
python-dateutil>=2.8.2
```

Now the result was consistent across:

* Local environments
* Docker
* GitHub Actions CI
* macOS vs Linux

Tests passed. CI green.

---

## What We Learned

1. **Python standard library support for date formats is fragmented.**
   `datetime.fromisoformat()` seems robust — until it quietly fails on "Z" time. You won’t notice until your app hits CI or production.

2. **Dependencies can alter behavior in invisible ways.**
   An innocent helper may mutate or sanitize strings before you parse them, changing how you interpret timezone offsets.

3. **CI catches edge cases local machines hide.**
   Your laptop has locale settings, system libraries, and Python patches that may not exist in the cloud. Let CI test the *real* world.

4. **Reproducibility needs isolation.**
   Once this failed, we moved all test runs to Docker with fixed versions and timezone configs — even locally — to prevent silent mismatches.

---

## Final CI Fix (Plus Docker)

We added a reproducible test stage with timezone lock:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    container: 
      image: python:3.10-slim
      env:
        TZ: UTC

    steps:
      - uses: actions/checkout@v3
      - run: pip install -r requirements.txt
      - run: pytest
```

This **locked the timezone** to UTC at the container level — and ensured the Python runtime acted predictably.

---

## TL;DR

> Your local environment is lying to you — again.
> Timezones, locales, and Python quirks make CI an essential gate for catching subtle platform bugs.

If your Python test passes on your machine but fails in CI, look at:

* System timezones
* Python versions
* Locale / encoding configs
* And especially: how you parse dates

---

# 🐘 A PHP Microservice Where CI Skipped a Critical Script Step

### …and the Deployment That Quietly Broke Everything

---

## The Background

We had a lightweight PHP microservice responsible for validating user-submitted data before forwarding it to our backend system. It didn’t do anything too fancy — just some `symfony/validator` rules, logging, and response shaping.

CI/CD was managed through **GitHub Actions**, with a workflow file that handled:

* Linting via `phpcs`
* Running PHPUnit tests
* Packaging the service into a Docker container
* Pushing the image to AWS ECR

We’d deployed this microservice dozens of times without issue. But then, one Friday afternoon…

---

## A Silent Breakage

An engineer added a new validation rule and updated the unit tests. Locally:

```bash
composer test
```

✅ All tests passed.  
✅ CI. All green.  
✅ Deployment to staging. No errors in the pipeline.

But QA flagged an issue:

> “The validation step isn’t triggering anymore. We can submit obviously invalid data, and it goes through just fine.”

Our validation microservice was accepting nonsense — empty strings, negative values, invalid email formats — all passed through like water through a sieve.

---

## Reproducing Locally

On local machines:

```bash
composer install
composer test
php -S localhost:8000 -t public
```

Everything behaved as expected.

The new validation rule rejected bad input. It wasn’t a logic bug.

But the deployed version was doing… nothing.

---

## The Mystery

We SSH’d into the ECS container.

Checked the PHP version: ✅  
Checked the container’s environment: ✅  
Pulled up logs: ✅  
No errors.

So we added logging inside the validator service:

```php
file_put_contents('/tmp/validate.log', json_encode($input));
```

Nothing was being logged.

We confirmed: the validator class was never being invoked. It was being **autoloaded**, but its constructor wasn’t running.

That led us back to one of the most innocuous files in PHP apps: `composer.json`.

---

## The Culprit

We noticed this in `composer.json`:

```json
"scripts": {
  "post-install-cmd": [
    "@php bin/setup-validation-metadata"
  ]
}
```

That script populated metadata used by the validator — without it, the system had *no rules* to run.

Locally, this worked because developers always ran:

```bash
composer install
```

Which triggered `post-install-cmd`.

But in CI, our Dockerfile looked like this:

```dockerfile
COPY . /var/www/
RUN composer install --no-dev --no-scripts
```

💥

> `--no-scripts` was **skipping** the critical setup step — silently.

There were no warnings. No errors. Just… skipped behavior. And since the validator didn’t throw errors when misconfigured, we didn’t notice until QA found it by hand.

---

## The Fix

We adjusted the `Dockerfile`:

```dockerfile
RUN composer install --no-dev && \
    composer run post-install-cmd
```

Alternatively, if using Composer 2.2+, you can whitelist individual scripts even with `--no-scripts`:

```bash
composer install --no-dev --no-scripts
composer run-script setup-validation-metadata
```

This ensured our validation config was rebuilt every time — just like in local dev.

---

## The Follow-up

To prevent this type of bug from going unnoticed again, we added:

1. A **smoke test** to assert that validation rules were loaded:

   ```php
   $this->assertNotEmpty($validator->getMetadataFor(MyRequestDto::class)->getConstraints());
   ```

2. An **end-to-end test** that actually hit the service over HTTP in CI with bad data.

3. A **linting rule** in CI to warn if `composer.json` has post-install scripts and CI uses `--no-scripts`.

---

## Lessons Learned

* **CI scripts may silently skip critical project logic.**
  Flags like `--no-scripts` are great for speed — but dangerous if you rely on Composer hooks.

* **What CI skips, it also skips silently.**
  Unlike failed tests or missing files, Composer won’t yell if your scripts are skipped.

* **Validate your assumptions, not just your code.**
  If your code relies on setup steps, test that they ran — not just that “no errors” occurred.

* **Production != Dev**
  Especially in Docker-based CI, every flag or layer cache decision can affect runtime behavior.

---

## Final GitHub Actions Fix

In our `ci.yml`, we updated the build stage:

```yaml
- name: Install dependencies
  run: composer install --no-dev

- name: Run validation metadata script
  run: composer run post-install-cmd
```

And added a CI test:

```yaml
- name: Smoke test validator
  run: php tests/ValidateMetadataSmokeTest.php
```

---

## TL;DR

> Your CI pipeline might run green while skipping half the setup —
> and PHP won’t raise its voice about it.

Always check:

* Do you rely on `composer` scripts?
* Are they skipped in `--no-scripts` mode?
* Do your tests actually hit runtime behavior, or just isolated logic?

CI can pass while your app does nothing at all.

---

# 🦫 A Go Project Where a Race Condition Was Invisible Locally

### …Until CI Flushed It Out Under Pressure

---

## The Setup

We had a backend Go microservice responsible for aggregating metrics from various upstream APIs. The service polled data every few seconds, cached responses, and exposed a JSON endpoint for downstream consumers.

Architecture-wise:

* Scheduled fetch via `time.Ticker`
* Caching in a shared in-memory map
* Served via `net/http`
* Tested with Go's built-in `testing` package
* Built and deployed via GitHub Actions

It was fast. It was reliable. It was… *not as thread-safe as we thought*.

---

## The CI Bug That Shouldn’t Exist

A team member submitted a pull request that reorganized how we fetched and cached data:

```go
var dataCache = make(map[string]MetricData)

func updateMetrics() {
	resp := fetchRemoteMetrics()
	for key, value := range resp {
		dataCache[key] = value
	}
}
```

Tests passed locally. Coverage was high.
CI ran all tests — and *intermittently failed* with errors like:

```
fatal error: concurrent map writes
```

Sometimes CI was green. Sometimes it crashed.
Even rerunning the same commit would randomly fail.

Locally, it was solid as a rock.

---

## The Confusion

Why was CI catching errors that never happened locally?

We tried:

```bash
go test -race ./...
```

Locally, all tests passed — even with `-race`.

CI? Still threw the `concurrent map writes` panic occasionally, but not every run.

That’s when we noticed something subtle:

* CI machines used **4+ vCPUs**
* Devs typically ran on **1-2 cores**

The concurrency window was simply wider in CI. Race conditions that were *possible* locally were just too rare to manifest.

---

## The Offending Code

Here’s the simplified version of what we had:

```go
var dataCache = map[string]MetricData{}

func fetchAndUpdate() {
	newData := fetchRemoteMetrics()
	for k, v := range newData {
		dataCache[k] = v
	}
}

func metricsHandler(w http.ResponseWriter, r *http.Request) {
	json.NewEncoder(w).Encode(dataCache)
}
```

At the same time:

* A background goroutine called `fetchAndUpdate()` every 5 seconds.
* HTTP handlers read from `dataCache`.

The result? **Concurrent reads and writes** to a plain `map`.
Go maps are **not thread-safe**. Ever.

But since most machines don’t hit both read and write in the same nanosecond, we got away with it — until CI ran things fast enough to expose it.

---

## The Fix

We switched to `sync.Map` — Go's concurrency-safe map type.

```go
var dataCache sync.Map

func fetchAndUpdate() {
	newData := fetchRemoteMetrics()
	for k, v := range newData {
		dataCache.Store(k, v)
	}
}

func metricsHandler(w http.ResponseWriter, r *http.Request) {
	cache := make(map[string]MetricData)
	dataCache.Range(func(key, value any) bool {
		cache[key.(string)] = value.(MetricData)
		return true
	})
	json.NewEncoder(w).Encode(cache)
}
```

Now all concurrent access was safe, and we no longer relied on race-prone native maps.

---

## CI Defense

We added a dedicated CI step:

```yaml
- name: Run tests with race detector
  run: go test -race ./...
```

We also updated local tooling:

```bash
alias gotest='go test -race ./...'
```

And added this to `Makefile` to enforce dev use:

```make
test:
	go test -race ./...
```

---

## What We Learned

* **Race conditions may *exist* locally but only *trigger* under CI conditions.**
  More cores = more parallelism = more chaos = more bugs.

* **Go’s map is a time bomb under concurrent access.**
  Even if you're only writing once every few seconds, you *must* guard map access when reads and writes happen in parallel.

* **The `-race` flag is essential — but not sufficient alone.**
  Use it with parallel tests, stress tests, and in environments that actually mimic real deployment concurrency.

* **Don’t just rely on unit tests — test usage under load.**
  In Go, even a tiny web service needs concurrency safety from day one.

---

## TL;DR

> If your Go code uses a `map`, and goroutines can *possibly* read/write it at the same time, you're living dangerously.

CI exposed a race condition that our local machines didn’t. Why?

* Local: 1-2 cores, low parallelism
* CI: multiple cores, full CPU usage, triggers bugs faster

Fix:

* Replace `map` with `sync.Map` (or use `sync.RWMutex`)
* Always test with `go test -race`
* Add concurrent access tests in CI to simulate real usage

---

# 🐳 A Sneaky Docker Cache Issue That Served Outdated Code *with a Smile*

### …and How CI Deployed an Old Binary While Saying "Everything’s Fine"

---

## The Background

We had a simple Go service containerized with Docker. The app built a binary, packaged it into a small Alpine image, and exposed a `/health` and `/version` endpoint.

Here's our `Dockerfile`:

```dockerfile
FROM golang:1.20 AS builder

WORKDIR /app
COPY . .
RUN go build -o server .

FROM alpine:3.18

WORKDIR /app
COPY --from=builder /app/server .
CMD ["./server"]
```

Our GitHub Actions workflow looked like:

```yaml
- name: Build Docker image
  run: docker build -t my-app:${{ github.sha }} .

- name: Push Docker image
  run: docker push my-app:${{ github.sha }}

- name: Deploy to staging
  run: some-deployment-script.sh
```

Each push triggered a new CI run, built a Docker image, and deployed to staging.

And for months — it worked flawlessly.

Until one week, it didn't.

---

## The Incident

A small refactor added a new `/metrics` endpoint.
The PR was reviewed. Merged. CI was green.
Staging was updated. QA pinged the new endpoint:

```bash
curl https://staging.example.com/metrics
```

And got:

```
404 page not found
```

They checked the `/version` endpoint. It responded with the **previous commit hash**.

The deployment had gone through — but the new binary hadn’t made it in.

CI logs claimed success. Docker image pushed. Deployment complete.

But the container was serving **yesterday’s binary**.

---

## The Investigation

We pulled the image directly from the registry:

```bash
docker run --rm my-app:<latest-sha> /app/server --version
```

Still showed the old SHA.

So we re-ran the build job with `--no-cache`:

```bash
docker build --no-cache -t my-app:debug .
```

Now it showed the correct version.

Lightbulb moment.

We’d been using Docker layer cache — and it had silently re-used an outdated build step.

---

## The Root Cause

In this line:

```dockerfile
COPY . .
RUN go build -o server .
```

The `COPY . .` step **didn’t trigger a cache bust** unless file timestamps or content changed — and in our case, Docker *thought nothing had changed*.

Here’s what happened:

* Developer made changes
* Git added the changes
* But the Docker layer cache, running on CI runners, still had an old `COPY` layer cached due to persistent volume caching
* `go build` ran on unchanged files
* Result: Old binary copied into the final image, and no one noticed

CI passed. Tests passed. Even image push succeeded.

Just… with the wrong code inside.

---

## The Fix

We modified the `Dockerfile` to **ensure cache busting** on the actual source code:

### Solution A: Use checksums as cache key

```dockerfile
COPY go.mod ./
COPY go.sum ./
RUN go mod download

COPY . .
RUN go build -o server .
```

This at least makes changes in `go.mod` invalidate cache before `COPY . .`.

But more robust:

### Solution B: Force cache busting with `--no-cache`

In CI:

```yaml
- name: Build Docker image (no cache)
  run: docker build --no-cache -t my-app:${{ github.sha }} .
```

Or:

### Solution C: Use content hashing as build arg

```dockerfile
ARG CACHEBUST=1
COPY . .
RUN go build -o server .
```

Then in CI:

```yaml
- run: docker build --build-arg CACHEBUST=$(date +%s) -t my-app:${{ github.sha }} .
```

This ensures a fresh build every time, even if Docker's cache thinks otherwise.

---

## CI/CD Improvements

To prevent this type of issue again:

1. We added a `/buildinfo` endpoint that exposed:

   * Git SHA
   * Build time
   * Docker image tag

2. We diff the deployed hash vs the latest commit hash in post-deploy validation

3. We **disabled persistent layer caching in CI** runners unless explicitly enabled

4. We lint for `COPY . .` as an anti-pattern in large multi-stage builds

---

## Lessons Learned

* **Docker is fast because it assumes your files didn’t change.**
  If its cache logic says "same inputs", it skips steps — even if you're relying on side effects like fresh binaries.

* **CI/CD isn’t just about green lights.**
  It's about *verifying* that what you built is what you deployed.

* **Reproducibility ≠ correctness.**
  You might be consistently building the wrong image — and not even know it.

* **Don’t trust Docker layer cache in critical production builds.**
  Use `--no-cache` or proper hashing to bust stale layers.

---

## TL;DR

> Docker silently served an old binary because we reused the build layer.
> CI passed. QA failed. Debugging took hours.

Fixes:

* Use `--no-cache` in CI builds
* Add `/version` or `/buildinfo` to your app
* Validate deploy hash != last commit hash
* Avoid `COPY . .` without explicit cache busting strategies

----

# 🧭 CI/CD as a First-Class Citizen in Your Debugging Mindset

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

Good CI pipelines are invisible when they work. Great engineers investigate them when they don’t.

**#CICD #Troubleshooting #30DaysOfDevOps**
