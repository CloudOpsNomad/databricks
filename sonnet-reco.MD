# Sonnet Production Readiness Review — Workday Databricks CI/CD

**Review type:** read-only assessment. No workflow files were modified.
**Scope:** `.github/workflows/workday-cicd-{dev,stage,prod}.yml` and the operational context surrounding them.
**Date reviewed:** 2026-04-27

---

## Confirmed Context

The following are team-confirmed facts that this review treats as givens, not gaps.

| Fact | Status |
|---|---|
| Runners are AWS CodeBuild, not GitHub-hosted | Confirmed |
| CodeBuild service role can `sns:Publish` to notification topics | Confirmed |
| GitHub App installed at org level; bot merge permitted under branch protection | Confirmed |
| Direct pushes to `main` and `stage` blocked by branch protection | Confirmed |
| Lead engineer approval click on GitHub `production-approval` is the intended human gate | Confirmed intentional |
| Unit and smoke tests running on the production workspace are an accepted risk | Risk accepted |
| Squash merges used on `develop`; hotfix must land on `stage` before `main` by team policy | Confirmed |

---

## Acceptance Criteria Status

### AC 1 — Git-Flow Implementation — Met

**What is in place:**
- Three-branch promotion flow: `develop` → `stage` → `main`.
- Each workflow validates the PR source branch and title pattern via a `validate_pr` job.
  - Dev accepts `feature/`, `fix/`, `refactor/`, `docs/`, `chore/`, `NDP-*`, and sync branches.
  - Stage accepts `develop`, `hotfix/NDP-*`, and main-sync branches.
  - Prod accepts `stage` and `hotfix/NDP-*` with title pattern enforcement.
- Branch protection enforces "no direct push," which is the correct mechanism for this guarantee.

**Considerations:**
- The hotfix path to `main` is permitted by the `validate_pr` shell check. Stage-first is team policy, not a code gate. If that discipline erodes, add a `git merge-base --is-ancestor` check that confirms the hotfix SHA already exists on `stage` before allowing the PR to proceed.
- Re-confirm allowed branch patterns annually as naming conventions change.

---

### AC 2 — Automated Linting and Unit Testing — Met

**What is in place:**
- All three workflows run identical `lint` and `secret_scan` jobs on every PR.
  - `pylint` via `nbqa` with `--fail-under=9.0`.
  - `sqlfluff lint` for SQL files.
  - Bandit for Python SAST.
  - `grep` regex check for hardcoded credentials.
  - `pip-audit` against the exported `prod` dependency group.
- `unit_test` submits bronze and silver notebook jobs via `databricks jobs submit --json` before any DAB deploy step.
- Build fails if any job in this chain fails.

**Considerations:**

1. **CLI exit-code propagation.** The `databricks jobs submit` command must block and return a non-zero exit code when the Databricks job itself fails — not just when submission fails. Confirm which CLI version is installed in the CodeBuild image and whether `--wait` is needed. If the CLI exits 0 on submission regardless of job outcome, the unit test gate is not a real gate.

2. **Lint score threshold.** `fail-under=9.0` is a reasonable starting point, but notebooks naturally accumulate minor style issues over time. If the score becomes a noisy blocker for clearly safe changes, consider per-file ratchet configuration in `.pylintrc` or separate thresholds per notebook layer (bronze, silver).

3. **Test coverage gap.** Unit tests cover the bronze and silver notebook layers. Confirm the gold/final transformation layer is covered or explicitly excluded from scope by design.

4. **Smoke test distinctness.** The smoke test falls back to the silver unit-test JSON when no dedicated `workday/tests/smoke/smoke_test_run.json` exists. This couples smoke signal to unit-test quality. A dedicated smoke notebook should verify only production-critical signals: cluster reachability, secrets access, bundle job availability, and a lightweight read-only data check.

---

### AC 3 — Environment Isolation — Met

**What is in place:**
- Each environment uses dedicated `DATABRICKS_HOST_*`, `DATABRICKS_CLIENT_ID_*`, `DATABRICKS_CLIENT_SECRET_*`, and `DATABRICKS_CLUSTER_ID_*`.
- Bundle target is controlled via `DATABRICKS_TARGET` (`unit_dev`, `stage`, `prod`).
- Workflow triggers are scoped to their respective target branches.

**Considerations:**

1. **Bundle definition.** No `databricks.yml` was found at the workspace root during this review. Bundle targets referenced in the workflow (`unit_dev`, `stage`, `prod`) must exist in the bundle configuration and match exactly. Confirm before go-live.

2. **Workspace path ownership.** If production notebook import paths reference `/Workspace/Users/<person>@company.com/...`, a user offboarding or email change will silently break the pipeline. Move to a service-principal-owned path such as `/Workspace/Shared/workday/` or a dedicated app folder.

3. **Bundle coverage.** DAB deploy only manages what is defined in the bundle. Production jobs, schedules, and parameters that were created manually in the Databricks UI are invisible to this pipeline and will drift over time. Audit and migrate them into the bundle before treating the system as fully automated.

---

### AC 4 — Secure Secret Management — Met (with ongoing controls required)

**What is in place:**
- No credentials in workflow YAML. All Databricks calls receive host, client ID, and client secret from GitHub secrets.
- `secret_scan` runs Bandit, regex grep, and `pip-audit` on every PR.
- Artifact (`bandit-report-workday`) is uploaded for review.

**Considerations:**

1. **Scan completeness.** Regex and Bandit scans are heuristic. They catch common patterns but will not catch all forms of credential leakage. Enable GitHub Advanced Security secret scanning at the org level if not already active.

2. **Notebook-level enforcement.** The CI scan runs on the file before it reaches Databricks. Add a test or review convention that confirms all production-facing notebooks retrieve sensitive values through `dbutils.secrets.get()` or the AWS Secrets Manager SDK, not through hardcoded constants or environment variable reads from non-CI sources.

3. **Secret rotation cadence.** Databricks service principal secrets and the GitHub App private key should rotate on a defined schedule. Rotation without updating GitHub secrets will fail the next workflow run silently at authentication. Keep a rotation calendar and test after rotation with a dry-run workflow dispatch.

---

### AC 5 — Automated Deployment on Merge to `main` — Met

**What is in place:**
- PR pre-merge path: validate → lint → secret scan → unit test → bundle validate → DAB deploy (prod) → smoke test → `ci_gate` → `merge_release_pr`. The bot merges only after all gates pass.
- Post-merge push path: `validate_bundle` → `pre_deploy` → `production-approval` → `deploy_bundle` → `smoke_test` → `tag_release` → notify.
- No one manually deploys anything in the Databricks UI. The Databricks service principal performs all deployments.

**Human approval model:**
The post-merge path intentionally pauses at the `production-approval` GitHub Environment. A lead engineer clicks **Approve and deploy** in GitHub Actions. This is the team's deliberate control point. It satisfies the Jira AC because the AC specifies "no manual intervention in the Databricks UI" — not "no human in the loop at all." The GitHub approval gate is a governance mechanism, not a workaround.

**Considerations:**

1. **`production-approval` environment configuration.** For the approval to be a real control:
   - Set at least one required reviewer (a lead engineer or team).
   - Disallow self-review so the person who merges the PR cannot also approve the deploy.
   - Restrict deployment branches to `main` only.
   - Keep production Databricks credentials (`DATABRICKS_CLIENT_SECRET_PROD`, etc.) in the protected `production` environment, not in org-level secrets.

2. **Double deploy.** Each release deploys twice to production: once pre-merge (during PR smoke validation) and once post-merge (approved, tagged run). This is intentional and accepted. Document it so the team does not raise it as a bug. If cost becomes a concern later, explore a SHA-matching optimization that skips the post-merge deploy when the incoming SHA was already smoked on the PR path.

3. **Job schedule coverage.** Automation is only as complete as the bundle definition. Any Databricks job or schedule not declared in the bundle will not be updated by CI and must be managed manually. Confirm bundle coverage before closing this AC.

---

### AC 6 — Auditability — Partially Met

**What is in place:**
- Successful post-merge smoke creates an annotated Git tag: `deploy/prod-<UTC-timestamp>-<short-sha>`.
- Tag annotation records the full SHA and workflow run URL.
- SNS notification and PR comment include run link and outcome.

**Open decisions:**

| Question | Current state | What to decide |
|---|---|---|
| Pre-merge prod deploy tagged? | No | Is this deploy event in scope for auditors? |
| Versioning scheme | Timestamp + short SHA | Is this acceptable, or is semver (`v1.4.2`) required? |
| Tag push permissions | Uses `github.token` | Does the org ruleset allow the bot to push tags to `main`? |

**Considerations:**

1. If auditors interpret "every deployment to production" to include pre-merge deploys, add a tagging step inside the PR path after a successful smoke. Use a naming convention that distinguishes it from the post-merge tag, such as `deploy/prod-pr-<pr-number>-<sha>`.

2. If the project adopts semver, a `VERSION` file or a release PR convention that bumps the version before merge gives a clean source of truth. The `tag_release` step can then read that value instead of generating a timestamp string.

3. The post-merge tag is the stronger audit artifact because it is tied to the approved, human-reviewed deployment. Ensure the tag creation step in `tag_release` is confirmed working on a dry-run before go-live, including the `git push origin` step through CodeBuild with the workflow token.

---

## Cross-Cutting Considerations

### Pipeline concurrency

- Prod: `concurrency: workday-prod-main` with `cancel-in-progress: false` prevents overlapping prod runs. This is correct.
- Stage: no concurrency group. Two concurrent stage PRs can both deploy to the same stage workspace. If one fails and triggers rollback while the other is mid-deploy, the stage workspace may end up in an ambiguous state. Evaluate adding a `concurrency` group for stage as a follow-up hardening item.

### Rollback integrity

- Stage smoke failure rolls back to `origin/stage` head at the time of failure. If another PR has already merged to stage, "rollback" deploys that merged state, not a known-good prior state. This is acceptable given how stage is used but should be understood.
- Prod PR smoke failure rolls back to `origin/main` head. Same caveat: main may have advanced if another PR merged concurrently. The concurrency group on prod reduces this risk significantly.
- Prod post-merge smoke failure rolls back to `HEAD^1` of the merge commit, which is the previous tip of `main`. This is the cleanest rollback target.

**Write a rollback runbook.** Document what to do when rollback itself fails (network issue, Databricks API timeout, or the rollback target is also broken). The manual recovery path is: redeploy from the last annotated `deploy/prod-*` tag by running `databricks bundle deploy` locally from the tagged SHA.

### CodeBuild runner toolchain

All of the following must be available in the CodeBuild image or installed at runtime:

- Python 3.10
- `uv` (for dependency export and virtual environment setup)
- AWS CLI v2 (for SNS publish)
- GitHub CLI `gh` (for `merge_release_pr` and PR comments — this is the most commonly missing tool in CodeBuild images)
- `jq` (for JSON manipulation in smoke/unit test steps)
- `curl` and `bash`
- Databricks CLI (installed in-step via `install.sh`)

Validate `gh --version`, `uv --version`, `jq --version`, and `aws --version` in a no-op workflow before enabling the full pipeline.

### Observability and alerting

- SNS notifications cover success and failure for smoke tests and rollback events. That is the minimum required for on-call awareness.
- Consider adding a dashboard that tracks: deploy frequency, smoke pass rate, rollback frequency, and approval-to-deploy latency. CloudWatch metrics or a Datadog integration against GitHub Actions run data are practical options.

### Cost and production data safety

- Unit and smoke tests run against the production workspace on every PR to `main`. Costs are accepted but should be monitored.
- Confirm test notebooks do not write to live Gold tables or affect production telemetry. Use isolated test schemas or strictly read-only operations. Passing `env="prod"` as a notebook parameter is not sufficient safety if the notebook logic does not branch on that parameter for write paths.

---

## Go-Live Checklist

- [ ] `production-approval` environment has required reviewers, self-review disabled, and `main`-only branch restriction.
- [ ] Branch protection blocks direct push to `main` and `stage`.
- [ ] GitHub App token/permissions validated from CodeBuild (merge, comment, tag push).
- [ ] SNS publish tested from CodeBuild in a real workflow run.
- [ ] `gh`, `uv`, `jq`, AWS CLI v2 confirmed available in CodeBuild image.
- [ ] Databricks CLI job-submit exit-code behavior confirmed (blocks on run completion, returns non-zero on failure).
- [ ] Bundle targets (`unit_dev`, `stage`, `prod`) confirmed in `databricks.yml`.
- [ ] All production jobs and schedules confirmed as bundle-managed or explicitly documented as out-of-scope.
- [ ] Tag push to `main` confirmed working through org rulesets and workflow token.
- [ ] Decision made on pre-merge prod deploy tagging (required or not).
- [ ] Decision made on versioning format (timestamp+SHA or semver).
- [ ] Dedicated smoke test JSON at `workday/tests/smoke/smoke_test_run.json` created.
- [ ] Rollback runbook documented and accessible to on-call team.
- [ ] Credential rotation schedule established for Databricks secrets and GitHub App key.

---

## Verdict

**Ready for controlled production rollout.** The pipeline implements the complete promotion flow (develop → stage → main), enforces quality gates, isolates environments, uses secure credential handling, and has defined rollback paths with notifications at every failure point.

**AC 5 (Automated CD):** Met. The GitHub approval click is the intended human gate, not Databricks UI intervention.

**AC 6 (Auditability):** Conditionally met. Post-merge deployments are tagged with timestamp and SHA. Two decisions are needed before marking this closed: whether pre-merge prod deployments must also be tagged, and whether timestamp+SHA satisfies the "version number" requirement or whether semver is expected.

The go-live checklist above is the concrete work remaining before those decisions can be confirmed and the ticket fully signed off.
