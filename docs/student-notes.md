# Student Audit Notes

This document contains the complete audit findings, hardening actions, and validation evidence for the Checkout Service deployment governance audit.

---

## Task 1: Audit Findings

### Weakness 1: Unrestricted Workflow Triggers (Any-Branch Deployment)
- **Location:** `.github/workflows/deploy.yml` (lines 3–6)
- **Evidence:** `on: push: branches: ['*']`
- **Risk:** High/Critical. Untested or experimental code pushed to any feature branch automatically triggers a production deployment pipeline, leading to potential outages and unstable releases in production.
- **Root Cause:** Wildcard branch matcher `*` was configured during initial testing to allow deploys from any branch without restricting triggers to release lines.
- **Recommended Fix:** Update the workflow trigger to only listen for pushes on the `main` branch: `on: push: branches: [main]`.

---

### Weakness 2: Missing Deployment Environment Scope
- **Location:** `.github/workflows/deploy.yml` (under `jobs.deploy`)
- **Evidence:** Absence of `environment:` property in job definition.
- **Risk:** Critical. Without referencing a defined environment (e.g., `environment: production`), GitHub Actions bypasses all environment-level protection rules including required reviewers, wait timers, and deployment branch restrictions.
- **Root Cause:** Environment binding was omitted during pipeline setup, running job execution directly on the runner without environment governance.
- **Recommended Fix:** Add `environment: name: production` to the deploy job definition in `.github/workflows/deploy.yml`.

---

### Weakness 3: Absence of Required Reviewers (Manual Approval Gate)
- **Location:** GitHub Repository Settings → Environments → `production`
- **Evidence:** No environment existed, and no required reviewers protection rule was configured.
- **Risk:** Critical. Deployments execute automatically upon push with zero human review or sign-off, creating operational vulnerability where typos or breaking configs ship unmonitored.
- **Root Cause:** Environment protection rules were never created on the repository.
- **Recommended Fix:** Create the `production` environment in GitHub repository settings and configure "Required reviewers" requiring explicit sign-off from authorized maintainers before deployment proceeds.

---

### Weakness 4: Unrestricted Deployment Branches in Environment Settings
- **Location:** GitHub Repository Settings → Environments → `production`
- **Evidence:** No deployment branch rules were defined for the environment.
- **Risk:** High. Even if an environment is specified, without branch restrictions, any branch referencing that environment can attempt to deploy to production.
- **Root Cause:** Default environment setup allows all branches to target the environment unless explicitly restricted.
- **Recommended Fix:** Enable "Deployment branches" under `production` environment settings and restrict allowed branches strictly to `main`.

---

### Weakness 5: Repository-Level Secret Exposure (Unscoped Credentials)
- **Location:** `.github/workflows/deploy.yml` (lines 26–30) & Settings → Secrets and variables → Actions
- **Evidence:** `${{ secrets.DEPLOY_KEY }}`, `${{ secrets.DATABASE_URL }}`, `${{ secrets.API_TOKEN }}` fetched from repository-level secrets.
- **Risk:** Critical. Repository-wide secrets can be accessed by any workflow job, pull request, or non-production branch, risking credential leakage and unauthorized database access.
- **Root Cause:** Production secrets were stored at the global repository level for convenience rather than being scoped to a protected environment.
- **Recommended Fix:** Move `DEPLOY_KEY`, `DATABASE_URL`, and `API_TOKEN` to Environment Secrets inside the protected `production` environment.

---

### Weakness 6: Lack of Deployment Grace Period (Missing Wait Timer)
- **Location:** GitHub Repository Settings → Environments → `production`
- **Evidence:** Wait timer setting set to 0 or unconfigured.
- **Risk:** Medium/High. Rushed deploys proceed immediately upon approval without a cooling-off period, preventing engineers from aborting accidental release approvals.
- **Root Cause:** Wait timer rule was left disabled.
- **Recommended Fix:** Enable a Wait Timer (e.g., 1 minute for lab testing / 10+ minutes for production) to create an intentional grace period before job execution.

---

### Weakness 7: Overly Broad Job Permissions
- **Location:** `.github/workflows/deploy.yml` (lines 12–14)
- **Evidence:** `permissions: contents: read, id-token: write`
- **Risk:** Medium. Granting write tokens (`id-token: write`) when standard deployment steps do not require OIDC tokens violates the principle of least privilege.
- **Root Cause:** Boilerplate permissions block was copied without auditing required capabilities.
- **Recommended Fix:** Restrict workflow permissions strictly to `contents: read`.

---

## Task 2: Configuration Actions

### Summary of Hardening Actions
- [x] Created `production` environment in GitHub Repository Settings.
- [x] Enforced Deployment Branch Rules: Restricted deployments strictly to `main`.
- [x] Configured Required Reviewers: Added required manual approval step.
- [x] Enabled Wait Timer: Set to 1 minute (60 seconds) for cooling-off validation.
- [x] Scoped Production Secrets to Environment: Moved `DEPLOY_KEY`, `DATABASE_URL`, `API_TOKEN` to `production` environment secrets.
- [x] Hardened Workflow File `.github/workflows/deploy.yml`: Added `environment: production`, restricted trigger to `main`, and set `permissions: contents: read`.

---

## Task 3: Validation Evidence

### Deployment Governance Execution Summary
- **Trigger Method:** Git commit & push to `main` branch / Workflow Dispatch.
- **Environment Gate Triggered:** `production` environment.
- **Observed Behavior:**
  1. Workflow job `deploy` paused automatically at the `production` environment gate.
  2. Status displayed "Waiting for review / approval".
  3. Prompt required reviewer sign-off before proceeding.
  4. Upon approval, 1-minute wait timer elapsed.
  5. Job resumed and completed successfully with clean execution logs.

---

## Summary & Reflection

### Key Takeaway
"We review every pull request" is insufficient for production protection because PR reviews govern how code enters `main`, but environment protection rules govern how code exits to production. Effective deployment governance requires both branch protection and environment approval gates.
