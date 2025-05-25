---
layout: post
title: "Real-life Postmortem: Lessons from a Bad Deploy"
date: 2025-05-25 23:23:02 +0000
comments: true
---

![Real Life Postmortem Header](/assets/images/postmortem_500.png)

It’s easy to get lulled into complacency when a deployment pipeline runs smoothly for months. The changes feel routine. The muscle memory kicks in. But it only takes one seemingly minor oversight to remind you that software systems are fragile—and production is always one bad deploy away from a wake-up call.

> *“It was just a small config change—how bad could it be?”*

That was our thinking the morning we pushed what seemed like an innocuous update to production. What followed was a tense two-hour firefight, locked user accounts, support tickets piling up, and a scramble to identify the root cause before real damage was done.  
This is the story of a production incident that began with a one-line config change and escalated into a full-blown outage affecting thousands of users. We’re sharing this not just to document what happened, but to reflect on the deeper lessons and process improvements that came out of it.

---

## The Incident

At 10:47 AM, a configuration change was deployed to our production environment. The change was meant to enable a new feature flag for a subset of users. The change itself passed all unit tests and had been deployed to staging without issue.

However, within 3 minutes of going live, we noticed a sharp spike in 500 errors. By 10:52 AM, our on-call engineer was alerted via PagerDuty: **"Auth API failure rate > 5%"**.
Users began reporting they couldn’t log in. Our support team was overwhelmed. It was clear: something was seriously broken.

---

## The Incident Timeline

Let’s walk through how this unfolded.

**10:47 AM**
The new config is deployed to production via CI/CD.

**10:50 AM**
Datadog reports a sudden drop in successful login attempts. We also see a spike in 500 responses from the auth service.

**10:52 AM**
PagerDuty alerts the on-call SRE. Internal Slack lights up with support agents reporting login failures from customers.

**10:55 AM**
We attempt a rollback of the config file. No immediate improvement.

**11:10 AM**
Deeper investigation begins. The app logs show errors around an `undefined variable reference` in the login flow.

**11:25 AM**
The engineering lead discovers that the logic introduced to enable the feature depends on an env variable that doesn’t exist in production.

**11:32 AM**
We patch the config to remove the feature flag override and redeploy.

**11:39 AM**
Error rates begin to normalize. Users are able to log in again.

**12:12 PM**
All systems operational. Post-incident review begins.

**Total time to resolution:** 1 hour, 25 minutes
**Impact:** 80% of login attempts failed during the incident window.

---

## The Setup

We were rolling out a new internal feature toggle for a revamped user onboarding flow. The backend implementation was already merged, tested, and deployed behind a feature flag. All we needed was to enable the flag for a small subset of users in production.

The flag was configured through a centralized YAML file that our application consumed at runtime. Here’s the commit that kicked it all off:

```yaml
features:
  onboarding_revamp:
    enabled: true
    allowlist_env: USER_ONBOARDING_BETA
```

Simple, right? Except we missed one crucial thing: `USER_ONBOARDING_BETA` didn’t exist in the production environment.

---

## Root Cause

After a quick rollback didn’t solve the issue, we investigated deeper.
Turns out, the new config referenced a **non-existent environment variable**. In staging, this variable was set manually for a separate test, so the issue never showed up there.
In production, however, the missing variable caused our authentication service to fail a critical conditional check. That failure triggered a fallback path that wasn’t well-tested—and it crashed, taking the login flow down with it.
The login service was extended to check whether the new onboarding flow should be triggered. The check looked like this (simplified):

```python
if os.getenv('USER_ONBOARDING_BETA') == 'true':
    use_new_flow()
else:
    use_old_flow()
```

In staging, this variable was set manually for testing purposes. But in production, it didn’t exist. The `getenv()` call returned `None`, which caused the condition to bypass the new flow—but worse, the downstream fallback assumed a complete context object which wasn’t populated when the flag was disabled. That’s where the crash happened.
Because we didn’t have validation or runtime guards for missing variables, the failure happened silently until the service tried to use an uninitialized value.

---

## Contributing Factors

This incident wasn’t caused by a single bug—it was the result of several small oversights lining up in the wrong way.

### 1. **Lack of config validation**

We had no runtime or CI/CD validation for referencing undefined environment variables in configuration files.

### 2. **Environment drift**

Staging had a manually set variable. Production did not. Our deployment process didn’t surface this discrepancy.

### 3. **No feature flag gating at runtime**

We used a raw env var to gate a feature instead of using our internal feature flag service, which provides safety checks and rollout strategies.

### 4. **Insufficient fallback testing**

The fallback code path had little test coverage and made assumptions about objects being initialized.

### 5. **Overconfidence in "safe" changes**

This change bypassed peer review due to being categorized as a non-code change. That false sense of safety cost us dearly.

---

## Recovery and Response

Our on-call engineer did an excellent job coordinating a rapid rollback, engaging relevant stakeholders, and escalating the issue. Still, this incident exposed cracks in our incident response procedures, such as:

* Delays in root cause identification due to noisy logs
* Lack of automated rollback of environment variables
* Incomplete runbook for this particular service

We’ve since taken several corrective actions.

---

## What We’ve Fixed

### 1. Schema validation for all config files

We introduced JSON Schema definitions for all critical config files and integrated schema validation into the CI pipeline. Broken configs fail the build early.

### 2. Mandatory environment variable checks

The app now refuses to start if required environment variables are missing. We also maintain a `.env.required` file per environment.

### 3. Feature flag gating via service, not env vars

All feature toggles must now go through our internal feature management platform. It provides percentage-based rollouts, validation, and audit trails.

### 4. Test coverage for fallback paths

We wrote new tests to explicitly trigger fallback scenarios and validate that they behave gracefully under missing or corrupted state.

### 5. Deployment guardrails

Every deployment, even for config-only changes, must go through the same code review and validation process. No more "safe" shortcuts.

---

## Key Lessons Learned

* **No change is too small to break production.**
  Treat every deploy with the same discipline.

* **Staging must match production.**
  Environment drift is a silent killer.

* **Validate everything.**
  Failing fast in CI is better than failing live.

* **Always test fallback paths.**
  Just because something is rarely used doesn’t mean it’s not critical.

---

## Cultural Shifts and Team Reflection

More than the technical lessons, this incident prompted us to reevaluate how we think about safety, trust, and process discipline.

* We now treat configuration changes as **first-class citizens** in the change management process.
* We recognized the dangers of **tribal knowledge**—engineers assuming certain variables exist because “they always have.”
* We acknowledged that **peer review is not optional**, regardless of perceived risk.
* And perhaps most importantly: **we normalized talking about failure** openly, without blame.

---

## Final Thoughts

Incidents like this are painful—but they’re also valuable. They force us to confront weak spots, fix blind spots, and grow stronger as a team.

Yes, a login outage caused short-term pain for users and a few frantic hours for the team. But in exchange, we now have tighter guardrails, better observability, and a stronger engineering culture that values reliability as much as velocity.

No change is too small to deserve scrutiny. We were lucky this didn’t last longer or cause data loss. But we won’t rely on luck next time.

## Appendix: What You Can Do Today

Here are some takeaways you can apply in your own team right now:

* Add config schema validation to your CI/CD process.
* Document all required environment variables per environment.
* Use feature flag platforms instead of raw env vars.
* Review fallback paths with the same rigor as primary logic.
* Never skip peer review—even for “simple” changes.

If this resonated with your team, we’d love to hear your stories. What’s the “small change” that taught you a big lesson?

**#SRE #IncidentResponse #LessonsLearned #30DaysOfDevOps**
