---
layout: post
title: "Bash Like an Engineer: Smarter Scripting for DevOps"
date: 2025-05-29 12:20:35 +0700
comments: true
---

![Smart Bash Scripting Illustration](/assets/images/bash_smart_scripting.png)

When you think of Bash, you probably think of quick fixes, hacked-together cron jobs, or obscure one-liners you copied from Stack Overflow at 2 a.m. But Bash scripting doesn’t have to be messy or fragile. In fact, when treated with the same discipline we apply to other languages, Bash can be a powerful, safe, and maintainable tool—especially for DevOps workflows.

In this post, I’ll walk you through:

* Bash tricks every DevOps engineer should know
* How to write structured, testable, and debug-friendly scripts
* Safety practices to avoid production disasters
* A real-world example from my toolkit

Let’s dive in.

---

## Thinking Like a Software Engineer in Bash

Many DevOps scripts start as a quick fix—and stay that way. But you can bring **software engineering principles** into Bash scripting with just a few simple habits:

### Use Functions to Structure Your Code

Instead of writing everything in a top-down blob, split your logic into functions:

```bash
#!/usr/bin/env bash

set -euo pipefail

log_info() {
  echo "[INFO] $*"
}

backup_database() {
  # some logic here
  log_info "Starting database backup"
}

main() {
  backup_database
}

main "$@"
```

Why? Easier testing, readability, and future reuse. Think of `main()` as the script’s entrypoint, just like in C or Python.

---

## Self-Debugging Bash: Know What Your Script is Doing

Ever had a script silently fail? Or worse, behave unpredictably? Here’s how to make it **transparent and debuggable**.

### Enable Fail-Fast Mode

Start every serious Bash script with this:

```bash
set -euo pipefail
IFS=$'\n\t'
```

* `-e`: exit on error
* `-u`: error on unset variables
* `-o pipefail`: fail if any command in a pipeline fails
* `IFS`: safer word splitting

### Add Traps for Debugging

Catch script exits, signals, or errors:

```bash
trap 'echo "ERROR: Script failed at line $LINENO"; exit 1' ERR
trap 'echo "Script exited cleanly."' EXIT
```

Now, when something goes wrong, you know *where*.

---

### Trace with `set -x` — See What Your Script is Doing

Sometimes your Bash script doesn’t crash, but it still misbehaves. Maybe it’s skipping a step, passing the wrong variable, or silently doing something unexpected.

This is where `set -x` shines. It enables **execution tracing**, printing every command and its arguments before they run.

```bash
set -x  # Turn on command tracing
```

Or, if you're running a script externally:

```bash
bash -x script.sh
```

**Example:**

```bash
#!/usr/bin/env bash
set -x

name="world"
echo "Hello, $name!"
```

**Output:**

```bash
+ name=world
+ echo 'Hello, world!'
Hello, world!
```

You’ll see exactly what the script is doing, which helps identify subtle bugs—like incorrect substitutions, misordered logic, or silently failing branches.

### Heads-up: `set -x` is Extremely Verbose

> **⚠️ Warning:** `set -x` can generate a *flood of output*, especially in larger scripts or when looping over files, parsing input, or running background processes. It can also inadvertently log **sensitive data**—such as passwords in command-line arguments or environment variables.

**So when should you use it?**

* ✔️  During development or debugging
* ❌ Not in production unless output is redirected and scrubbed
* ✔️  Only when explicitly enabled by a flag or environment variable

### Best Practice: Conditional Tracing with `DEBUG` Mode

To avoid hardcoding `set -x` into your script, use a simple flag pattern like this:

```bash
[[ "${DEBUG:-0}" -eq 1 ]] && set -x
```

This checks whether the environment variable `DEBUG` is set to `1`. If so, it enables tracing.

**Now you can run:**

```bash
./script.sh          # Normal mode
DEBUG=1 ./script.sh  # Debug mode with tracing
```

You get debug output only when you *want* it — and your logs stay clean by default.

### Optional: Support a `--debug` Flag

If you prefer using CLI flags, add this:

```bash
DEBUG=0

while [[ $# -gt 0 ]]; do
  case "$1" in
    --debug) DEBUG=1 ;;
  esac
  shift
done

[[ "$DEBUG" -eq 1 ]] && set -x
```

Now you can run:

```bash
./script.sh --debug
```

### Protect Sensitive Data

Always be mindful of what you log when `set -x` is active. For example:

```bash
PASSWORD="secret"
curl -u "admin:$PASSWORD" https://example.com  # Will leak password in trace!
```

If your script deals with secrets, avoid enabling full tracing, or mask critical values before logging them.

By using conditional `DEBUG` tracing, you maintain control over verbosity and security while keeping the powerful insights of `set -x` just a flag away.

---

## Bash Safety 101: Guard Rails for Production Scripts

Writing production-ready Bash scripts requires a **defensive mindset**. Bash is powerful but unforgiving—small oversights can lead to major failures. Here’s how to reduce risk with safer scripting practices.

### Quote Everything

Unquoted variables are one of the most common causes of unexpected behavior in Bash scripts. They can break commands when variables contain spaces, are empty, or have special characters.

```bash
rm -rf "$TARGET_DIR"    # Safe
rm -rf $TARGET_DIR      # Dangerous if empty or contains spaces
```

If `$TARGET_DIR` is unset or blank, the second line becomes `rm -rf`, potentially deleting everything in the current directory.

### Validate Input

Assume nothing. Scripts often fail due to missing or malformed input. Validate arguments before using them:

```bash
[[ -z "${1:-}" ]] && { echo "Usage: $0 <target_dir>"; exit 1; }
```

This ensures the required argument is present before the script proceeds.

### Use `mktemp` for Temporary Files

Hardcoding temporary filenames invites collisions and race conditions. Instead, use `mktemp` to safely generate unique filenames:

```bash
TMP_FILE=$(mktemp)
```

Always clean up temporary files after use to avoid clutter or leaks.

### Be Explicit with Destructive Commands

When using commands like `rm`, `mv`, or `cp`, add verbosity (`-v`) and consider a dry-run mode for testing. Log actions clearly before execution. It’s better to be overly cautious than to guess in production.

---

## Testing Your Bash Scripts (Yes, It’s Possible!)

Testing isn't just for compiled languages or high-level frameworks. Bash scripts can and should be tested—especially if they're part of your infrastructure automation. The [Bats framework](https://github.com/bats-core/bats-core) makes this possible.

### What is Bats?

Bats (Bash Automated Testing System) provides a simple, readable way to write test cases for your scripts. It helps you verify script behavior in isolation or as part of CI pipelines.

### Example Test Case

```bash
@test "Backup completes successfully" {
  run ./my_backup.sh
  [ "$status" -eq 0 ]
}
```

This test runs the script and checks that it exits with a status code of `0` (success). You can also assert on the script’s output or any file system changes it performs.

### Setup Instructions

1. Install Bats:

   ```bash
   git clone https://github.com/bats-core/bats-core.git
   sudo ./bats-core/install.sh /usr/local
   ```

2. Create a test file:

   ```bash
   mkdir tests
   touch tests/test_my_script.bats
   ```

3. Run tests:

   ```bash
   bats tests/
   ```

Incorporating Bats into your CI/CD pipeline allows you to catch regressions early and ensure your Bash scripts behave as expected across environments.


## Example CI Testing with GitHub Actions

You can automate Bash script testing using [Bats](https://github.com/bats-core/bats-core) and GitHub Actions:

**Example Structure**:

```
.
├── my_script.sh
├── tests/
│   └── test_my_script.bats
├── .github/workflows/ci.yml
```

**Sample Bats Test (`tests/test_my_script.bats`)**:

```bash
@test "Script runs without error" {
  run ./my_script.sh
  [ "$status" -eq 0 ]
}
```

**GitHub Actions Workflow**:

```yaml
name: Test Bash Scripts
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: git clone --depth=1 https://github.com/bats-core/bats-core.git
      - run: sudo ./bats-core/install.sh /usr/local
      - run: chmod +x *.sh
      - run: bats tests/
```

This lets you test Bash scripts like real software—automatically and reliably.

---

## Real-World Toolkit Example: Safe Cleanup Script

Let’s walk through a practical script that safely deletes old log files, commonly used in log rotation or maintenance routines.

```bash
#!/usr/bin/env bash

set -euo pipefail
IFS=$'\n\t'
```

### Fail-Fast Settings

* `-e`: Exit immediately on any error.
* `-u`: Treat unset variables as errors.
* `-o pipefail`: Exit if any command in a pipeline fails.
* `IFS`: Sets a safer internal field separator to handle file names with spaces or tabs.

---

```bash
LOG_DIR="/var/log/myapp"
DAYS_TO_KEEP=7
```

### Configuration Block

Defines constants up front. Makes the script easy to adapt without editing logic later on.

---

```bash
log() { echo "[$(date +'%F %T')] $*"; }
```

### Logging Function

Adds a timestamp to every message, improving traceability, especially when logs are redirected to files or viewed in CI output.

---

```bash
clean_old_logs() {
  log "Looking for files older than $DAYS_TO_KEEP days in $LOG_DIR"
  find "$LOG_DIR" -type f -mtime +$DAYS_TO_KEEP -print -delete
  log "Cleanup complete."
}
```

### Cleanup Logic

* `find` locates files older than `DAYS_TO_KEEP`.
* `-print` shows which files will be removed.
* `-delete` performs the deletion.

This combination gives visibility before destructive action is taken, and the modular design improves testability.

---

```bash
main() {
  [[ -d "$LOG_DIR" ]] || { echo "Log directory not found: $LOG_DIR"; exit 1; }
  clean_old_logs
}
```

### Entrypoint

Encapsulates the logic and validates prerequisites. Ensures the script doesn’t run if the target directory doesn’t exist.

---

```bash
trap 'log "Script exited unexpectedly at line $LINENO"; exit 1' ERR
trap 'log "Script completed."' EXIT
```

### Error and Exit Traps

* Logs the line number on failure for easier debugging.
* Confirms successful completion when the script ends.

---

### Summary: Why This Script is Safe and Reliable

* Uses fail-fast settings to avoid silent failures
* Validates all inputs and environment dependencies
* Provides clear, timestamped logging for observability
* Is structured with functions, making it easier to maintain and test
* Uses traps to improve debugging and error reporting

This pattern can be adapted for many use cases—from cleanup to backups to automation. It’s a strong foundation for writing safer Bash in real-world systems.

---

## Final Thoughts

Bash is everywhere in DevOps—from provisioning to automation, deployment, and recovery. But too often, it’s treated like a second-class citizen. With a bit of structure, discipline, and tooling, you can make your Bash scripts:

* **Maintainable**
* **Safe**
* **Testable**
* **Debuggable**

And most importantly—**trusted by your future self and your team.**

Next time you write a script, don’t just make it work. Make it **engineered**.

**#Bash #Scripting #30DaysOfDevOps**
