# GPT Production Readiness Recommendation - Workday Databricks CI/CD

Scope: review of the Dev, Stage, and Prod GitHub Actions CI/CD design for Databricks notebooks. This document is a recommendation and readiness assessment only. It does not change workflow behavior.

## Executive Verdict

The CI/CD design is ready for a controlled production rollout, assuming the already-confirmed organization setup remains in place:

- AWS CodeBuild runners have the required runtime permissions, including SNS publish access.
- The GitHub App is installed at the organization level and the bot is allowed to merge under branch protection.
- Direct pushes to `main` and `stage` are blocked by repository branch protection.
- Production intentionally requires a lead engineer to approve the GitHub `production-approval` environment before the post-merge deploy continues.
- Unit and smoke tests running against the production workspace are an accepted risk decision for this project.

The only acceptance criterion that still needs a reviewer/auditor decision is auditability scope: whether the existing post-merge deployment tag is enough, or whether the pre-merge production deploy also needs its own tag and whether timestamp-plus-SHA is acceptable as the version format.

## Acceptance Criteria Readiness

### 1. Git-Flow Implementation - Met

The workflows support a multi-branch model across `develop`, `stage`, and `main`.

- Dev PRs target `develop`.
- Stage PRs target `stage`.
- Production PRs target `main`.
- Direct pushes to protected branches are handled through branch protection, which is the correct enforcement layer for this requirement.
- Production PRs are expected to come from `stage`, with a hotfix path available when required by process.

Consideration: keep the hotfix rule explicit in team process. If hotfixes must always pass through `stage` before `main`, consider adding an automated check later that verifies the hotfix SHA exists on `stage`.

### 2. Automated Linting and Unit Testing - Met

Each workflow includes quality gates before deployment:

- Python linting through `pylint` / `nbqa`.
- SQL linting through `sqlfluff`.
- Security checks through Bandit, regex secret checks, and `pip-audit`.
- Notebook unit tests submitted to Databricks from JSON using `databricks jobs submit`.
- Unit tests are positioned before any DAB deployment step.

Considerations:

- Confirm the Databricks CLI version used by CodeBuild waits for submitted job completion and returns a non-zero exit code when the Databricks run fails. If it only submits and exits successfully, add explicit polling.
- Keep the lint score threshold under review. A strict threshold is useful, but it should not become a recurring operational blocker for low-risk notebook changes.
- Unit test JSON submissions should remain environment-specific by cluster and parameters, not by duplicating test logic.

### 3. Environment Isolation - Met

The workflow design separates Dev, Stage, and Prod through environment-specific variables and secrets:

- Separate Databricks hosts.
- Separate service principal credentials.
- Separate cluster IDs.
- Separate bundle targets such as `unit_dev`, `stage`, and `prod`.

Considerations:

- Confirm the bundle definition exists in the actual bundle root and that target names match the workflow values exactly.
- Prefer service-principal-owned workspace paths for production notebooks instead of user-owned `/Workspace/Users/...` paths, so user lifecycle changes do not break deployments.
- Confirm all production job schedules and notebooks are declared in the bundle. Anything still created only through the Databricks UI can drift from Git.

### 4. Secure Secret Management - Met, With Runtime Validation Needed

The workflows avoid embedding credentials in YAML and use GitHub secrets/variables for Databricks authentication. The secret scan also checks for common hardcoded secret patterns.

Considerations:

- Regex secret scanning is useful but not complete. Keep GitHub secret scanning or equivalent external scanning enabled.
- Add or maintain tests/code review checks that verify notebooks read secrets from Databricks Secrets or AWS Secrets Manager, not from hardcoded constants.
- Schedule credential rotation for Databricks service principal secrets and the GitHub App private key.

### 5. Automated Deployment on Merge to `main` - Met

This is met under the intended interpretation of the Jira requirement: no manual Databricks UI intervention.

The production process still includes a deliberate human control:

- A PR to `main` must pass validation, linting, security scan, unit tests, DAB validate, prod DAB deploy, and smoke test before the bot merges it.
- After merge to `main`, the `push` workflow runs the production deployment path.
- The post-merge deploy pauses at the GitHub `production-approval` environment.
- A lead engineer clicks **Approve and deploy** in GitHub Actions.
- CI then performs the Databricks deployment and smoke test using the configured service principal.

This approval click is a production control point, not manual Databricks deployment. It supports the ticket statement that production changes are approved by a lead engineer before affecting Gold or production telemetry.

Considerations:

- Configure the `production-approval` GitHub Environment with required reviewers.
- Disallow self-review if segregation of duties is required.
- Restrict the environment to `main`.
- Keep production Databricks credentials scoped to the protected `production` environment.

### 6. Auditability - Partially Met

The production `push` path creates an annotated deployment tag after successful smoke testing. This gives a clear post-merge link between production deployment, Git SHA, timestamp, and workflow run.

Current auditability is strong for post-merge deployment, but two points need a conscious decision:

- The pre-merge production deploy used for smoke validation is not tagged.
- The current version format is timestamp plus short SHA, not semantic versioning.

Considerations:

- If auditors define "every deployment" literally, add a tag for the pre-merge production deploy as well.
- If auditors require a formal version number, add a release version source such as a `VERSION` file or semver tag.
- Confirm the GitHub token or bot identity is allowed by org rulesets to push deployment tags.

## Production Rollout Considerations

### Approval Model

The lead engineer approval click in GitHub is a positive control, not a weakness. It should remain in place if the business wants human review before production telemetry or Gold-layer assets are updated.

### Double Deploy Behavior

Production deploys can happen twice for a release:

- First during the PR flow, to prove the candidate can deploy and pass smoke before merge.
- Again after merge to `main`, so the final main-branch SHA is deployed, approved, smoked, tagged, and notified.

This is acceptable if documented. If runtime cost or deployment time becomes a concern, consider a future optimization that skips the second deploy only when the post-merge SHA matches the already-smoked PR candidate.

### Rollback

Rollback behavior is defined:

- Prod PR smoke failure: reset local checkout to `origin/main`, redeploy bundle, report failure, and block merge.
- Prod post-merge smoke failure: redeploy from the previous baseline SHA and report failure.
- Stage smoke failure: redeploy from the target branch baseline.

Consideration: document a manual escalation path if rollback itself fails, such as redeploying from the last known-good `deploy/prod-*` tag.

### Smoke Test Quality

The smoke test currently has a fallback to a silver unit-test JSON if a dedicated smoke JSON is unavailable. That is functional, but it makes smoke less distinct from unit testing.

Recommendation: create a dedicated smoke test definition that checks only production-critical signals, such as notebook import, bundle-deployed job availability, cluster access, secrets access, and a small read-only validation query.

### Databricks Bundle Coverage

The deployment can only manage what is defined in the Databricks Asset Bundle. For production readiness, confirm that the bundle owns:

- Production job definitions.
- Notebook paths.
- Schedules.
- Parameters.
- Cluster or job compute references.
- Environment-specific paths such as S3 buckets and workspace folders.

Anything still managed only in the Databricks UI remains outside Git audit and can drift.

### Runner and Tooling

Because the runner is CodeBuild, validate the actual image includes or can install:

- Bash.
- Python 3.10.
- `jq`.
- `curl`.
- AWS CLI v2.
- `uv`.
- GitHub CLI `gh`.
- Databricks CLI.

The `gh` binary is especially important because the bot merge and PR comments depend on it.

### Cost and Data Safety

Production unit and smoke tests are accepted, but they should stay safe:

- Use isolated test schemas or read-only checks.
- Avoid writes to Gold production tables.
- Avoid contention with scheduled production workflows.
- Monitor DBU impact from repeated PR validation.

### Stage Concurrency

Production already has a single-flight concurrency model. Stage should be reviewed for parallel PR behavior because multiple stage deploys can interfere with each other if they target the same workspace and one fails during another deployment.

Recommendation: consider a stage-level concurrency group in a later hardening change.

## Go-Live Checklist

- Confirm `production-approval` has required reviewers and self-review is disabled.
- Confirm branch protection blocks direct pushes to `main` and `stage`.
- Confirm the GitHub App can merge and comment from CodeBuild.
- Confirm CodeBuild can publish to SNS.
- Confirm CodeBuild includes `gh` or installs it before use.
- Confirm Databricks CLI job submission waits for actual run success/failure.
- Confirm bundle targets match workflow target names.
- Confirm all production jobs/schedules are represented in the bundle.
- Confirm the tag push succeeds under org rulesets.
- Decide whether pre-merge production deploys require tags.
- Decide whether timestamp-plus-SHA is enough for versioning or semver is required.
- Add a dedicated smoke test JSON when practical.
- Create a rollback runbook for failed rollback attempts.

## Final Recommendation

Proceed with a controlled production rollout. The pipeline has the important safety properties: branch flow, quality gates, environment separation, secret scanning, prod approval, smoke testing, rollback, notifications, and post-merge tagging.

The main item to settle with the ticket reviewer is auditability wording. If post-merge timestamp/SHA tags are accepted, the current design satisfies the intent. If auditors require a tag for every pre-merge production deploy or semantic version numbers, add that before closing the auditability acceptance criterion.
