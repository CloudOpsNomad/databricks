# Production Readiness Review — Workday CI/CD

Scope: `.github/workflows/workday-cicd-{dev,stage,prod}.yml` and supporting setup. This review **does not modify code**; it records the **current state**, **considerations**, and **what to decide before calling the work production-ready** against the Jira acceptance criteria.

> Operational facts confirmed by the team:
>
> - Runners are **AWS CodeBuild** (not GitHub-hosted Linux). The CodeBuild service role is allowed to **`sns:Publish`** to the topic ARNs.
> - The CI **GitHub App** is enabled at the **org level**; the bot is permitted to merge per branch protection rules.
> - **Direct pushes to `main` / `stage`** are blocked by branch protection.
> - **Unit tests against prod** are an accepted, signed-off risk for this repo.
> - **`develop`** uses **squash** merges from feature branches.
> - **Hotfix** policy: even hotfixes land in **`stage`** before `main` (process, not enforced in shell).

---

## Acceptance criteria — current state vs ticket

### 1. Git-Flow implementation — **Met**

- Workflows enforce branch+title patterns:
  - **dev**: `pull_request` → `develop`; sources include `feature/feat/major/fix/refactor/docs/chore`/`NDP-*`, plus `chore/sync-stage-to-develop-*`.
  - **stage**: `pull_request` → `stage`; sources `develop`, `hotfix/NDP-*`, `chore/sync-main-to-stage-*`.
  - **prod**: `pull_request` → `main`; sources `stage` or `hotfix/NDP-*` (with title rules).
- **Direct push prohibition** is enforced **outside YAML** by branch protection (already in place).

**Considerations:** Keep the prod **PR base check** strict (`main`) and re-affirm hotfix policy with the team yearly so wording in `validate_pr` and the actual rule remain aligned.

---

### 2. Automated linting & unit testing — **Met (with notes)**

- Each environment runs **`lint`** (`pylint` via `nbqa`, `sqlfluff`) and **`secret_scan`** (Bandit, grep, `pip-audit`).
- **`unit_test`** runs **bronze + silver** notebooks via `databricks jobs submit` from JSON. Every workflow chains `unit_test` **before** any DAB validate/deploy. Failures fail the build.

**Considerations:**

- **`databricks jobs submit`** returns immediately on submission; the **CLI version in your CodeBuild image** matters. Verify the version used **waits and surfaces non-zero on Databricks job failure**; if not, add an explicit poll/`--no-wait`/exit-code check.
- Treat **`fail-under=9.0`** for pylint as a policy lever. If notebooks land near that score, expect periodic flakiness; consider a **per-notebook ratchet**.
- Add **a real smoke notebook** under `workday/tests/smoke/smoke_test_run.json`. Today the smoke step **falls back to silver unit-test JSON**, which couples smoke quality to silver tests.

---

### 3. Environment isolation — **Met**

- Dedicated **`DATABRICKS_HOST_*`** vars, **`DATABRICKS_CLIENT_ID_*` / `_SECRET_*`** secrets, and **`DATABRICKS_CLUSTER_ID_*`** per environment.
- Bundle target switches via **`DATABRICKS_TARGET`** (`unit_dev`, `stage`, `prod`).

**Considerations:**

- The repo currently has **no `databricks.yml`** committed (no DAB definition discovered at the workspace root in this review). **Bundle targets must exist** in `databricks.yml` (or wherever your bundle root is) and **match** the workflow’s target names — confirm before go-live.
- Notebook **import paths** are hard-coded to **`/Workspace/Users/user@databricks.com/...`**. For prod isolation prefer a **service principal-owned folder** (e.g. `/Workspace/Shared/...` or a dedicated `/Workspace/Apps/workday/...`) so user lifecycle changes do not break imports.

---

### 4. Secure secret management — **Met (process); validate end-to-end**

- Workflows scan for hardcoded credentials and run `pip-audit` on the **prod** dependency group.
- All Databricks calls receive credentials via `secrets.*` (no embedded tokens).

**Considerations:**

- The **scan is heuristic** (regex). Pair it with a **periodic external scan** (e.g. GitHub secret scanning, AWS access analyzer) and a policy that **all production-bound code paths** read **only** from **Databricks Secrets** (`dbutils.secrets.get`) or **AWS Secrets Manager**. Add **a unit test** that fails when a notebook references local env or constants where a secret is expected.

---

### 5. Automated CD on merge to `main` — **Partially met, by design**

What you have:

- **PR pre-merge gate** on prod: `validate → quality → unit → bundle validate → deploy → smoke → ci_gate → merge_release_pr` (auto-merge after smoke passes).
- **Post-merge `push` run** on `main`: `approval → deploy → smoke → tag → notify`.

So upon a **successful merge to `main`**, the pipeline **does** update prod automatically (via the **`push`** workflow). The **second deploy** is intentional: it carries **lead-engineer approval** (`production-approval` environment) and **audit tagging** that the PR path skips.

**Considerations:**

- The Jira AC says “**without manual intervention** in the Databricks UI.” Today the `push` path requires **a human to approve the `production-approval` GitHub Environment** before deploy. That is **not Databricks UI** intervention, but it is **a manual GitHub click**. If the AC means **fully automated** post-merge, decide one of:
  - Keep the approval (interpret AC as “no Databricks UI clicks”), **or**
  - Move approval to the **PR side** and skip approval on `push`, **or**
  - Make `push` deploy idempotent and bypass approval when the SHA matches a PR run already approved (more code).
- **Job schedules** updated by `bundle deploy`: this is what DABs do via `bundle.yml` job definitions. Confirm your bundle defines all **Workflows/Jobs** you want managed; anything created **only in the UI** is **out of scope** for this CI/CD and will drift.

---

### 6. Auditability (Git tag per prod deploy) — **Partially met**

What you have (`tag_release` on `push`):

- After successful `push` smoke, an **annotated tag** `deploy/prod-<UTC-timestamp>-<short-sha>` is created and pushed.
- Tag message records the SHA. **PR comment** + **SNS** include a run link.

**Considerations:**

- The **PR-path deploy** is **not** Git-tagged. If auditors require **every prod deploy** (including the pre-merge one) to have a tag, add a separate tag (e.g. `deploy/prod-pr-…`) on the PR success path.
- **Version number** is encoded as **timestamp + short SHA**, not **semver**. If the AC means **semver**, add a release scheme (e.g. `v1.4.2`) sourced from a `VERSION` file or a Git tag bumped by a release PR.
- **Tag push** uses `github.token`. With CodeBuild + org rules, confirm the token is **allowed to push tags** to `main` (some org rulesets block bot tag pushes by default).

---

## Cross-cutting considerations

### A. Runner / build environment

- CodeBuild image must include: **bash**, **`jq`**, **`curl`**, **Python 3.10**, **`uv`**, **AWS CLI v2**, and **`gh`** (GitHub CLI). The Databricks CLI is installed in-step.
- If `gh` is **missing** in CodeBuild, **`merge_release_pr`** and **PR comments** fail. Validate once with a no-op workflow that runs `gh --version` on the actual runner.

### B. Concurrency & parallel PRs

- Prod uses **`concurrency: workday-prod-main`** with `cancel-in-progress: false` — **good**: no overlapping prod deploys.
- Stage has **no concurrency**: two stage PRs can deploy to the **same stage workspace** in parallel. This is usually acceptable, but if smoke from PR A fails and rolls back while PR B is mid-deploy, results can interleave. Consider adding **`concurrency: workday-stage-${{ github.workflow }}`**.

### C. Rollback semantics

- **Stage** rollback uses **`origin/<base>` (= `origin/stage`)** at the moment smoke fails — this is **`stage` HEAD as it currently stands**, which can include **other already-merged stage PRs** (not necessarily a known-good).
- **Prod PR** rollback uses **`origin/main`** (target branch head). Same caveat — `main` could have advanced.
- **Prod push** rollback uses **`pre_deploy.previous_sha`** (the merge commit’s `HEAD^1`). This works for **merge commits**. If `main` ever switches to **squash**, `HEAD^1` of a squashed commit is the **previous `main` tip**, which is also fine — but the **diff** isn’t the PR’s change set, so re-deploying `HEAD^1` is what you actually want for rollback.

**Considerations:** Document an **escalation runbook** when rollback also fails (out-of-band redeploy of a known-good tag, e.g. `deploy/prod-…`).

### D. Data and cost blast radius (prod unit + smoke)

- Unit + smoke against **prod** workspace incurs cost on **every PR** to `main`. Watch for:
  - **DBU cost** on the prod cluster.
  - **Concurrency** with scheduled jobs (use a **dedicated PR cluster** or cluster pool).
  - **Side effects on tables** — make sure tests **read** or write only to **isolated test schemas/paths** (verify `notebook_task.base_parameters.env="prod"` does **not** flip notebooks into writing to live tables).

### E. Drift between Databricks UI and the bundle

- DAB deploy will **manage** what is in the bundle. UI-created jobs/notebooks **will not be touched**. Consider a periodic job that reports **bundle drift** (e.g. unmanaged jobs in prod) so AC #5 doesn’t silently regress.

### F. Hotfix policy enforcement

- Branch + title now allow `hotfix/NDP-*` again. **Stage-first** is policy, not check. If you want an actual gate, options:
  - GitHub API check in `validate_pr` that the **head SHA exists on `stage`** (e.g. `git merge-base --is-ancestor HEAD origin/stage`).
  - Alternatively, **block direct hotfix → main** PRs and require hotfixes to come **only from `stage`** after they land there.

### G. Observability

- SNS subjects/messages include workflow name, PR, run URL — good for paging.
- Consider a **single dashboard** (e.g. CloudWatch / Datadog) that watches: `bundle deploy` duration, smoke pass rate, rollback frequency, and **time-to-recover** from rollback to next green deploy.

### H. Secret rotation

- Keep a routine for rotating **`DATABRICKS_CLIENT_SECRET_*`** and the **CI App private key**. Rotation **breaks the workflow silently** if it lags secrets — schedule a quarterly rotation drill.

---

## Decision log to confirm before flipping the switch

| # | Decision | Suggested default |
|---|---|---|
| 1 | Approval on `push` stays as gate | **Yes** — preserve audit trail; doc that AC means “no Databricks UI clicks” |
| 2 | Tag pre-merge prod deploy too | **Optional** — only if auditors require tag per deploy event |
| 3 | Move from timestamp+SHA tag to **semver** | **Optional** — add a `VERSION` bump step on release PRs if needed |
| 4 | Add `concurrency` to stage | **Yes** — single-flight per stage workflow |
| 5 | Add real smoke notebook + JSON | **Yes** — replace silver fallback in smoke |
| 6 | Strict hotfix “stage-first” gate | **Optional** — only if process is repeatedly violated |
| 7 | Service-principal owned `/Workspace` folder for imports | **Yes** for prod robustness |
| 8 | Drift detection job (bundle vs UI) | **Yes** for AC #5 longevity |

---

## Verdict

**Production-ready for a controlled rollout** of the Workday flow into prod, given your CodeBuild + GitHub App + branch protection setup. Two **AC items remain conditional**:

- **AC #5 (CD on merge):** **Met** if “without manual intervention” is interpreted as **“no Databricks UI clicks.”** **Not met** if it requires a fully unattended GitHub Action with **no environment approval click**.
- **AC #6 (auditability):** **Met** for **post-merge** deploys via `tag_release`; **only partially met** for pre-merge prod deploys (no Git tag), and **versioning is not semver**.

Make a call on **#5 and #6** with the ticket reviewer; everything else above is a **set of choices and follow-ups**, not blockers, for the current pipeline.
