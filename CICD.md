# Workday GitHub Actions CI/CD

This document describes the **Databricks Asset Bundle (DAB)** workflows under `.github/workflows/` for **Dev**, **Stage**, and **Production**, and how they fit your **runners and org setup** (e.g. CodeBuild, org-level GitHub App, branch protection).

| Workflow | File | Primary trigger |
|----------|------|-------------------|
| Dev | `workday-cicd-dev.yml` | PR → `develop` (`workday/**`, workflow file) |
| Stage | `workday-cicd-stage.yml` | PR → `stage` (same path rules) |
| Prod | `workday-cicd-prod.yml` | PR → `main`, `push` → `main`, `workflow_dispatch` (same path rules) |

---

## Dev (`workday-cicd-dev.yml`)

**Purpose:** Gate changes entering `develop` without deploying the bundle to a long-lived prod-like target from this pipeline alone.

**Flow (high level):**  
`validate_pr` → `lint` → `secret_scan` → `unit_test` (Databricks **unit_dev**) → `bundle_validate` → `smoke_test` (after validate only; no `bundle deploy` in dev) → `ci_gate` → optional auto-merge / SNS / PR comment.

**Notable behavior:**

- **No DAB deploy** in dev: `databricks bundle validate` only; smoke validates behavior after validate.
- On smoke failure, the workflow resets the checkout toward the PR base and **re-validates** the bundle (no deploy rollback, by design).
- **Permissions:** `contents: read`, `pull-requests: write` (merge uses a GitHub App token where configured).

---

## Stage (`workday-cicd-stage.yml`)

**Purpose:** Integrate work on **`stage`** with full **validate + deploy + smoke** before merge.

**Flow:**  
`validate_pr` (allowed sources: `develop`, `hotfix/NDP-*`, `chore/sync-main-to-stage-*`; title rules) → same **lint** / **secret_scan** pattern as prod → `unit_test` (Databricks **stage**) → `deploy_bundle` (`validate` + `deploy`) → `smoke_test` → `ci_gate` → SNS → optional **auto-merge** (`gh pr merge` via CI App).

**Rollback (smoke failure):** Checkout is aligned to **`origin/<base>`** at the PR base workflow context, then **`bundle deploy`** again so the stage workspace matches the **target branch head** used for rollback, then the job **fails** so the PR does not show a green smoke gate.

**Permissions:** `contents: read`, `pull-requests: write`.

---

## Production (`workday-cicd-prod.yml`)

**Purpose:** Controlled promotion **`stage` → `main`** with **pre-merge** prod deploy + smoke + automated merge, and a **post-merge** pass (approval, deploy, tag, notify).

### Triggers

- **`pull_request`** → `main` (paths: `workday/**`, this workflow).
- **`push`** → `main` (same paths).
- **`workflow_dispatch`**.

### Concurrency

- **`workday-prod-main`**, `cancel-in-progress: false` — at most one concurrent run for this pipeline group (reduces overlapping prod deploys).

### PR path (`pull_request`, release to `main`)

1. **`validate_pr`** — Source branch must be **`stage`** or match **`hotfix/NDP-*`**. Titles: **`release/NDP-*-YYYY-MM-DD`** from `stage`, **`hotfix/NDP-*-YYYY-MM-DD`** from hotfix branches. **Process:** even for hotfixes, the team requires changes to hit **`stage`** before `main` where applicable; that ordering is **not** asserted by this shell (would need GitHub API / extra checks to prove “merged to stage first”).
2. **`pre_deploy`** — Records **`previous_sha`** (PR: base SHA; used heavily on **push** rollback).
3. **Quality:** `lint` → `secret_scan` → `unit_test` (uses **prod** host/secrets/cluster — **accepted** by the team for this repo).
4. **`validate_bundle`** — `databricks bundle validate --target prod`.
5. **`approval`** — **Skipped** on `pull_request` (deploy is allowed when approval job is `skipped`).
6. **`deploy_bundle`** — `databricks bundle deploy --target prod` (GitHub Environment **`production`**).
7. **`smoke_test`** — `databricks jobs submit` from `workday/tests/smoke/smoke_test_run.json` if present, else **silver** unit test JSON as fallback.
8. **`ci_gate`** — Fails the PR path unless **deploy + smoke** succeeded (and upstream jobs).
9. **`merge_release_pr`** — If `ci_gate` succeeded, merges the PR with **`gh pr merge … --merge`** using the **CI GitHub App** token.

**Rollback on failed smoke (PR):** In the smoke job, on failure: `git fetch` + `git reset --hard origin/<base>` (for `main` PRs, **`origin/main`**), then **`bundle deploy`** to prod again, **SNS**, **`exit 1`** — **no merge**.

### Push path (`push` / `workflow_dispatch`)

1. **`approval`** — **Required** (`environment: production-approval`).
2. **`deploy_bundle`** → **`smoke_test`** → **`tag_release`** (annotated tag `deploy/prod-<UTC>-<sha>`) → **`notify`** (PR comment + SNS where applicable).

**Rollback on failed smoke (push):** Checkout **`pre_deploy.previous_sha`** (typically **`HEAD^1`** of the merge commit), redeploy bundle, fail job.

### Why prod deploy runs twice (PR run + `push` run)

Two separate workflow runs are **by design**, not an accident:

1. **`pull_request` run** — Validates the **candidate** merge, **deploys** that ref to prod, **smoke-tests**, then **`merge_release_pr`** merges into `main`. This is what **blocks a bad merge**: smoke must pass before Git gets the merge commit.

2. **`push` run** — Fires on the **new tip of `main`** after the merge. It runs the **post-merge contract**: **`production-approval`**, deploy, smoke, **`tag_release`**, **`notify`**. That is where the **annotated deploy tag** and **merged-PR comment** logic live.

So you get **two `bundle deploy` calls** for one promotion when both succeed. They should apply the **same (or equivalent) bundle content**; the second run also satisfies **lead approval** and **audit tagging** that are intentionally skipped on the PR path. **Cost/time tradeoff:** you keep a strict pre-merge gate and a full post-merge audit trail. **Optional later change:** skip redundant deploy on `push` only when you can reliably detect “already deployed from the PR run for this SHA” (easy to get wrong; document any shortcut).

---

## Shared patterns (lint & secret scan)

Across **dev / stage / prod**, the **lint** and **`secret_scan`** job **steps** are aligned: same **nbqa pylint** / **sqlfluff**, same **Bandit + grep + `uv export --group prod` + pip-audit**, artifact **`bandit-report-workday`**.

---

## Secrets and variables (checklist)

Configure in **GitHub Actions** (repo or org). Names below match the workflows.

**Databricks (per environment suffix `_DEV` / `_STAGE` / `_PROD` where applicable):**

- `vars.DATABRICKS_HOST_*`
- `secrets.DATABRICKS_CLIENT_ID_*`, `secrets.DATABRICKS_CLIENT_SECRET_*`
- `secrets.DATABRICKS_CLUSTER_ID_*` (unit/smoke job submit)

**Notifications:**

- `secrets.DEV_SNS_TOPIC_ARN`, `secrets.STAGE_SNS_TOPIC_ARN`, `secrets.PROD_SNS_TOPIC_ARN`

**CI merge / comments (stage + prod):**

- `secrets.CI_APP_ID`, `secrets.CI_APP_PRIVATE_KEY`
- `create-github-app-token` uses **`owner: databricks`** — must match the org where the app is installed (org-level app is already enabled in your setup).

**Runners and AWS for SNS:** Workflows use `runs-on: ubuntu-latest` in YAML; your **CodeBuild** runners replace that in practice. **SNS** works when the **CodeBuild service role** (or equivalent) is allowed **`sns:Publish`** on the topic ARNs — no `configure-aws-credentials` step is required in that model.

**Tag push (prod):** `tag_release` uses **`github.token`**. If org policy **blocks** `GITHUB_TOKEN` from creating tags, use a **PAT** or a **scoped token** with `contents: write` on tags.

---

## Production readiness summary

### Strengths

- **Clear separation** of dev (validate-only), stage (full PR deploy + smoke + auto-merge), prod (pre-merge prod gate + post-merge CD).
- **Unit tests run before any DAB deploy** in all three workflows (`needs` chains).
- **Lint and secret scan** are consistent across environments.
- **Prod PR path:** **deploy → smoke → gate → merge**; failure rolls back Databricks bundle toward **`main`** and does not merge.
- **Concurrency** on prod reduces overlapping deploys.
- **SNS** on CodeBuild runners with an IAM role allowed to publish to the topic.
- **Org-level GitHub App** and **branch protection / bot merge** are configured so automation can merge when checks pass.

### Remaining technical notes (not “missing org setup”)

1. **`validate_pr` “not main” exit 0:** If `TARGET_BRANCH != main`, the branch step **exits 0** (success). Harmless with `on.pull_request.branches: [main]`, but it is a **fail-open** edge case if triggers ever change.
2. **Hotfix “via stage first”:** Policy is documented in the workflow echo; **enforcing** “merged to `stage` before PR to `main`” would need extra automation (e.g. GitHub API checks), not shell on `head_ref` alone.
3. **Audit tag vs pre-merge deploy:** First prod deploy on a PR is **not** Git-tagged until the **`push`** run’s **`tag_release`** — **accepted** unless auditors require a tag on the pre-merge SHA.
4. **`merge_release_pr` uses `--merge`:** **`develop`** uses **squash** for feature PRs; **`main`** may still use **merge commits** for releases. If **`main`** is **squash-only**, switch to **`gh pr merge --squash`** and revisit **`HEAD^1`** rollback on **`push`** (squash parents differ from merge commits).
5. **Tag push:** If org policy blocks **`GITHUB_TOKEN`** from pushing tags, use a PAT or ruleset exception for the bot.

### Suggested go-live checklist

- [x] Branch protection on **`main`** / **`stage`** (including required checks such as **`ci_gate`** where applicable).
- [ ] GitHub Environments **`production`** and **`production-approval`** — approvers and secrets scoped correctly.
- [x] CI App at org level; merge/bot behavior verified for release PRs.
- [x] **SNS** from CodeBuild role verified.
- [ ] **Databricks** vars/secrets per environment; **`DATABRICKS_TARGET`** matches bundle targets (`unit_dev`, `stage`, `prod`).
- [ ] Optional: **`workday/tests/smoke/smoke_test_run.json`** for a dedicated smoke job spec (instead of silver fallback only).

---

## File map

```
.github/workflows/
├── workday-cicd-dev.yml      # develop PRs
├── workday-cicd-stage.yml    # stage PRs
└── workday-cicd-prod.yml     # main PR + push + dispatch
```

For job-level acceptance-style comments, see the numbered **`# ---`** blocks at the top of each job in the YAML files.

---

*This file reflects the workflows as of the last repository update. Re-run a short review whenever you change triggers, environments, or merge policy.*
