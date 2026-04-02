# ⚡ Fast.BI — Data Platform CI/CD Workflows

CI/CD pipelines for the Fast.BI data platform, built on **Argo Workflows** running on Kubernetes. Three workflow phases cover every stage of the data engineering lifecycle — from the first commit on a feature branch to production deployment on `main`.

---

## 📑 Table of Contents

- [Repository Structure](#-repository-structure)
- [How It Works](#-how-it-works)
- [Phase 1 — Branch Workflow](#-phase-1--branch-workflow-branch_workflowyaml)
- [Phase 2 — Merge Request E2E Workflow](#-phase-2--merge-request-e2e-workflow-mr_e2e_workflowyaml)
- [Phase 3 — Main Workflow](#-phase-3--main-workflow-main_workflowyaml)
- [CI Templates](#-ci-templates)
- [Infrastructure & Secrets](#-infrastructure--secrets)
- [Templates Directory](#-templates-directory)
- [Troubleshooting](#-troubleshooting)
- [Contributing](#-contributing)

---

## 📁 Repository Structure

```text
data-platform-cicd-workflows/
├── branch_workflow.yaml               # Phase 1: Feature branch quality checks
├── mr_e2e_workflow.yaml               # Phase 2: Merge request end-to-end validation
├── main_workflow.yaml                 # Phase 3: Production deployment on main
├── template.github.argo-workflows.yml # GitHub Actions trigger template
├── template.gitlab.gitlab-ci.yml      # GitLab CI trigger template
└── templates/
    └── dbt-quality-check/             # Setup templates for dbt quality tools
        ├── .pre-commit-config.yaml    # dbt-checkpoint hooks (copy to dbt project root)
        ├── packages_addition.yml      # dbt-project-evaluator package snippet
        ├── dbt_project_addition.yml   # dbt_project.yml config for evaluator
        ├── Dockerfile.patch           # dbt-workflow-core image changes
        └── macros/
            └── cleanup_evaluator_schema.sql  # dbt macro: drop evaluator CI schema
```

---

## 🔄 How It Works

All workflows share the same core infrastructure:

| Component | Details |
|-----------|---------|
| **Orchestrator** | Argo Workflows on Kubernetes |
| **PVC** | Shared `/workdir` PersistentVolumeClaim per workflow run |
| **Secrets** | `fastbi-argo-workflows-secrets` Kubernetes secret |
| **Docker images** | `dbt_workflow_core_image`, `data_analysis_import_core_image` |
| **Artifact transport** | PVC (branch/MR) · S3/MinIO artifacts (main) |
| **Change detection** | `git diff-tree` — folder-level and full file path |
| **Warehouses** | BigQuery · Snowflake · Redshift · Microsoft Fabric |

### Git Provider Integration

```
Git push / PR open
      │
      ▼
GitHub Actions / GitLab CI  ──trigger──▶  Argo Workflows
      │                                         │
      └── template.github.argo-workflows.yml    └── runs on Kubernetes
          template.gitlab.gitlab-ci.yml
```

---

## 🌿 Phase 1 — Branch Workflow (`branch_workflow.yaml`)

**Trigger**: every push to a feature branch  
**Goal**: fast feedback to the data engineer before requesting a review  
**Blocking**: none — all checks use `continueOn: failed: true`; failures are visible but do not block the branch

### Pipeline Steps

```
Git Push on Feature Branch
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                     BRANCH WORKFLOW                              │
│                                                                  │
│  1. checkout          Clone repo at commit SHA                   │
│         │                                                        │
│         ▼                                                        │
│  2. detect-changes    git diff-tree → changed_folders.txt        │
│                                     → changed_files.txt          │
│         │                                                        │
│         ▼                                                        │
│  3. lint-yaml  ⚠️     yamllint on changed folders                │
│         │                                                        │
│         ▼                                                        │
│  4. lint-sql   ⚠️     sqlfluff on changed folders                │
│         │                                                        │
│         ▼                                                        │
│  5. check-dbt-quality ⚠️  (see detail below)                    │
│         │                                                        │
│         ▼                                                        │
│  6. validating-da-dependencies ⚠️  DA dependency graph check     │
│         │                         (runs when check_on_branch)    │
│         ▼                                                        │
│  7. k8s-pvc-cleanup   Always runs — releases PVC                 │
└─────────────────────────────────────────────────────────────────┘
```

> ⚠️ = `continueOn: failed: true` — check reports findings but never blocks the branch

### Step 5 — `check-dbt-quality` (detail)

Replaces the legacy `dbt-coverage` tool with two modern checks that give engineers **clear, actionable quality feedback** on every commit:

```
check-dbt-quality
       │
       ├─── 🔍 dbt-checkpoint  (presence checks on changed files only)
       │         │
       │         ├── dbt parse            (fast compile — no warehouse needed)
       │         ├── check-model-has-properties-file
       │         ├── check-model-has-description
       │         ├── check-source-has-description
       │         ├── check-macro-has-description
       │         ├── check-model-has-tests-by-name  (unique + not_null)
       │         └── check-source-has-tests-by-name
       │
       └─── 🏗️ dbt-project-evaluator  (structural health — real warehouse run)
                 │
                 ├── dbt build --select package:dbt_project_evaluator
                 ├── checks 27 rules across modeling / testing /
                 │   documentation / structure / performance / governance
                 │   → https://dbt-labs.github.io/dbt-project-evaluator/main/rules/
                 │
                 └── 🧹 cleanup
                       dbt run-operation cleanup_evaluator_schema
                       (drops dbt_evaluator_ci schema from warehouse)
```

**Prerequisites in the dbt project:**

| Item | Action |
|------|--------|
| `.pre-commit-config.yaml` | Copy from `templates/dbt-quality-check/` to dbt project root |
| `dbt-project-evaluator` | Add from `templates/dbt-quality-check/packages_addition.yml` to `packages.yml` |
| `dbt_project.yml` config | Merge from `templates/dbt-quality-check/dbt_project_addition.yml` |
| Docker image | Apply `templates/dbt-quality-check/Dockerfile.patch` to `dbt-workflow-core` |

---

## 🔀 Phase 2 — Merge Request E2E Workflow (`mr_e2e_workflow.yaml`)

**Trigger**: merge request / pull request open or update  
**Goal**: full end-to-end validation — DAG runs, data tagging, and cleanup — before code merges  
**Blocking**: configured per step; critical steps block merge via Git provider branch protection rules

### Pipeline Steps

```
Merge Request Opened / Updated
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   MR E2E WORKFLOW                                │
│                                                                  │
│  1. checkout                Clone repo at MR head SHA            │
│         │                                                        │
│         ▼                                                        │
│  2. detect-changes          git diff-tree → changed folders      │
│                             + tag-aware changed files            │
│         │                                                        │
│         ▼                                                        │
│  3. lint-yaml-e2e  ⚠️       yamllint on dbt/DAG files            │
│         │                                                        │
│         ▼                                                        │
│  4. detect-changed-dbt-tags   Reads manifest.json, matches       │
│         │                     changed models/macros/tests to     │
│         │                     DBT_TAGS in dynamic_dag_config.yml │
│         │                     → matched_dags.txt                 │
│         ▼                                                        │
│  5. data-orchestration-variables-import-e2e                      │
│         │                   Imports Airflow variables for        │
│         │                   tag-matched DAGs only                │
│         ▼                                                        │
│  6. compile-dbt-manifest-e2e   dbt compile per changed folder    │
│         │                                                        │
│         ▼                                                        │
│  7. compile-dag-e2e            Helm/DAG compilation for          │
│         │                      tag-matched DAGs only             │
│         ▼                                                        │
│  8. data-model-e2e-dag-run-trigger                               │
│         │                   Triggers DAG run in Airflow /        │
│         │                   Cloud Composer                        │
│         ▼                                                        │
│  9. data-model-e2e-dag-status-check                              │
│         │                   Polls DAG run until complete         │
│         ▼                                                        │
│ 10. validating-da-dependencies-e2e                               │
│         │                   Validates Data Analysis dependencies  │
│         ▼                                                        │
│ 11. data-model-e2e-data-tagging                                  │
│         │                   Tags data assets (lineage/governance) │
│         ▼                                                        │
│ 12. data-model-e2e-data-cleanning                                │
│         │                   dbt run-operation cleanup_dbt_dataset │
│         │                   Removes E2E test data from warehouse  │
│         ▼                                                        │
│ 13. data-model-e2e-data-orchestration-cleanup                    │
│         │                   Resets Airflow DAG state              │
│         ▼                                                        │
│ 14. k8s-pvc-cleanup         Always runs — releases PVC           │
└─────────────────────────────────────────────────────────────────┘
```

### Tag-Based DAG Matching

The `detect-changed-dbt-tags` step ensures only relevant DAGs are compiled and run:

```
changed_files.txt
      │
      ▼
  manifest.json (per folder)
      │  reads tags for each changed model / macro / test
      ▼
  dynamic_dag_config.yml
      │  matches model tags ↔ DBT_TAGS per DAG
      ▼
  matched_dags.txt  →  "folder:dag_name" per line  |  "SKIP" if no match
      │
      ▼
  Steps 5–7 skip folders/DAGs not in matched_dags.txt
```

---

## 🚀 Phase 3 — Main Workflow (`main_workflow.yaml`)

**Trigger**: push to `main` / `master` branch  
**Goal**: production promotion — compile artifacts, import variables, deploy governance, quality, and catalog  
**Transport**: S3/MinIO artifacts (no shared PVC between steps)

### Pipeline Steps

```
Push to main / master
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                     MAIN WORKFLOW                                │
│                                                                  │
│  1. checkout                Clone repo at commit SHA             │
│         │                                                        │
│         ▼                                                        │
│  2. detect-changes          git diff-tree (two-commit range)     │
│                             → changed_folders.txt                │
│                             → changed_files.txt  (full paths)    │
│         │                                                        │
│         ▼                                                        │
│  3. data-orchestration-variables-import                          │
│         │                   Tag detection: manifest.json         │
│         │                   → matched_dags.txt                   │
│         │                   Imports Airflow variables for        │
│         │                   tag-matched DAGs only                │
│         ▼                                                        │
│  4a. compile-dbt-manifest   dbt compile per changed folder  ─┐  │
│  4b. compile-dag            Helm/DAG compilation            ◄─┘  │
│         │                   (parallel — both run together)       │
│         ▼                                                        │
│  5. refresh-dbt-incremental-modules                              │
│         │                   Triggers incremental model refresh   │
│         │                   for changed dbt models               │
│         ▼                                                        │
│  6a. data-model-da-dependencies-import    ─┐                     │
│  6b. data-model-data-governance-metadata-import                  │
│  6c. data-model-data-catalog-deployment    ├── parallel          │
│  6d. data-model-data-quality-deployment   ─┘                     │
│         │                                                        │
│         ▼                                                        │
│  7. k8s-pvc-cleanup         Always runs — releases PVC           │
└─────────────────────────────────────────────────────────────────┘
```

### Artifact Flow (Main)

Unlike Branch/MR which use a shared PVC, Main uses artifact-based transport so steps can run on different pods:

```
checkout  ──repo-artifact──────────────────────────────┐
              │                                         │
detect-changes──changed-files-artifact──────────────┐  │
              │  changed-full-files-artifact───────┐ │  │
              ▼                                    │ │  │
data-orchestration-variables-import                │ │  │
   ◄── changed-full-files-artifact ────────────────┘ │  │
   ──► matched-dags-artifact ──────────────────┐     │  │
       compiled-repo-artifact ─────────────┐   │     │  │
                                           ▼   ▼     ▼  ▼
                                        compile-dag  compile-dbt-manifest
```

---

## 🔧 CI Templates

### GitHub Actions

```yaml
# .github/workflows/branch.yml
name: Branch CI
on:
  push:
    branches-ignore: [main, master]

jobs:
  branch-ci:
    uses: ./.github/workflows/template.argo-workflows.yml
    with:
      workflow_file: branch_workflow.yaml
      argo_namespace: argo
```

```yaml
# .github/workflows/mr-e2e.yml
name: MR E2E
on:
  pull_request:

jobs:
  mr-e2e:
    uses: ./.github/workflows/template.argo-workflows.yml
    with:
      workflow_file: mr_e2e_workflow.yaml
      argo_namespace: argo
```

```yaml
# .github/workflows/main.yml
name: Main Deploy
on:
  push:
    branches: [main, master]

jobs:
  main-deploy:
    uses: ./.github/workflows/template.argo-workflows.yml
    with:
      workflow_file: main_workflow.yaml
      argo_namespace: argo
```

### GitLab CI

```yaml
# .gitlab-ci.yml
include:
  - local: "template.gitlab.gitlab-ci.yml"

branch_ci:
  extends: .fast_bi_argo_workflows
  variables:
    WORKFLOW_FILE: "branch_workflow.yaml"
  rules:
    - if: '$CI_PIPELINE_SOURCE == "push" && $CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH'

mr_e2e:
  extends: .fast_bi_argo_workflows
  variables:
    WORKFLOW_FILE: "mr_e2e_workflow.yaml"
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

main_deploy:
  extends: .fast_bi_argo_workflows
  variables:
    WORKFLOW_FILE: "main_workflow.yaml"
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
```

---

## 🔐 Infrastructure & Secrets

All workflows read credentials from the `fastbi-argo-workflows-secrets` Kubernetes secret:

| Secret Key | Used By | Purpose |
|------------|---------|---------|
| `DATA_WAREHOUSE_PLATFORM` | all | `bigquery` / `snowflake` / `redshift` / `fabric` |
| `GCP_SA_SECRET` | BigQuery | Base64-encoded service account JSON |
| `GCP_PROJECT_NAME` | BigQuery | GCP project ID |
| `SNOWFLAKE_PRIVATE_KEY` | Snowflake | RSA private key (PEM) |
| `SNOWFLAKE_PASSPHRASE` | Snowflake | Key passphrase |
| `REDSHIFT_HOST/PORT/USER/PASSWORD/DATABASE` | Redshift | Connection params |
| `FABRIC_SERVER/DATABASE/PORT/USER/PASSWORD/AUTHENTICATION` | Fabric | Connection params |
| `AIRFLOW_SECRET_FILE_NAME` | MR/Branch | Path to Airflow variables YAML |
| `ARGO_MAIN_ADDRESS` | Exit handler | Argo server URL for status reporting |

---

## 📦 Templates Directory

The `templates/dbt-quality-check/` directory contains files to set up the two dbt quality tools in your data repository. Copy these files to the appropriate locations in your dbt project and `dbt-workflow-core` image:

| Template file | Copy to |
|---------------|---------|
| `.pre-commit-config.yaml` | dbt project root |
| `packages_addition.yml` | merge into dbt project `packages.yml` |
| `dbt_project_addition.yml` | merge into dbt project `dbt_project.yml` |
| `Dockerfile.patch` | apply to `dbt-workflow-core` Dockerfile |
| `macros/cleanup_evaluator_schema.sql` | `dbt-workflow-core/macros/` |

---

## 🛠️ Troubleshooting

| Symptom | Check |
|---------|-------|
| Workflow stuck / pending | `kubectl describe wf/<name> -n argo` |
| PVC not released | `kubectl delete pvc <name> -n argo` — or wait for `k8s-pvc-cleanup` step |
| Step fails immediately | Check `fastbi-argo-workflows-secrets` has all required keys |
| dbt-checkpoint warnings | Model missing description or tests — see step log for file names |
| dbt-project-evaluator warnings | See rule explanation at [dbt-project-evaluator rules](https://dbt-labs.github.io/dbt-project-evaluator/main/rules/) |
| Evaluator schema not cleaned up | Run `dbt run-operation cleanup_evaluator_schema --args '{"schema_name": "dbt_evaluator_ci", "dry_run": False}'` manually |
| No DAGs matched (SKIP) | Changed files have no dbt tags matching `DBT_TAGS` in `dynamic_dag_config.yml` |
| DAG compile fails | Verify `dbt_airflow_variables.yml` and `dynamic_dag_config.yml` are present in the folder |

---

## 🤝 Contributing

1. Open an issue or discussion describing the change
2. Create a feature branch from `master`
3. Update the workflow YAML and any relevant templates
4. Validate by running the branch workflow on your feature branch
5. Open a pull / merge request — the MR E2E workflow runs automatically

---

## 📄 License

MIT License — Copyright © 2025 Fast.BI

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

---

*Part of the Fast.BI data platform infrastructure.*
