## Fast.BI Data Platform CI/CD Workflows

This repository provides Argo Workflows and CI/CD templates to build, test, validate, and deploy Fast.BI data platform assets across branches, merge requests, and main releases.

- **Repository name**: `data-platform-cicd-workflows`
- **Scope**: Orchestrate CI/CD for data applications, dbt artifacts, DAGs, and maintenance jobs
- **Stack**: Argo Workflows (YAML), Bash scripts, GitHub/GitLab CI templates; integrates with Terraform/Terragrunt infra

## Overview

Standardized workflows for the Fast.BI data platform:
- **Branch CI**: fast feedback (linting, tests, selective compilation, PVC cleanup)
- **Merge Request E2E**: full end-to-end validations prior to merge
- **Main**: promotion flows (compilations, governance imports, quality checks)
- **Maintenance**: reusable steps and cleanup jobs

### Repository Structure

```text
bi-platform-cicd-workflows/
  0_branch/                    # Branch (feature) CI scripts
  1_merge_request/             # MR E2E CI scripts
  2_main/                      # Main (release) CI scripts
  fast_bi_argo_workflows/      # Argo Workflows and CI templates
    branch_workflow.yaml
    mr_e2e_workflow.yaml
    main_workflow.yaml
    template.github.argo-workflows.yml
    template.gitlab.gitlab-ci.yml
  README.md
  README_TEMPLATE.md
```

## Architecture

Argo Workflows on Kubernetes orchestrate containerized steps executing scripts from `0_branch/`, `1_merge_request/`, and `2_main/`.

- **Branch Workflow (`branch_workflow.yaml`)**: fast, incremental checks for feature branches
- **MR E2E Workflow (`mr_e2e_workflow.yaml`)**: comprehensive validations and end-to-end tests
- **Main Workflow (`main_workflow.yaml`)**: promotion and scheduled pipeline runs on `main`
- **CI Templates**: GitHub/GitLab snippets to trigger Argo workflows consistently

## Using the CI Templates

### GitLab

```yaml
include:
  - local: "fast_bi_argo_workflows/template.gitlab.gitlab-ci.yml"

variables:
  ARGO_NAMESPACE: "argo"

mr_e2e:
  extends: .fast_bi_argo_workflows
  variables:
    WORKFLOW_FILE: "fast_bi_argo_workflows/mr_e2e_workflow.yaml"
```

### GitHub Actions

```yaml
name: MR E2E
on:
  pull_request:

jobs:
  mr-e2e:
    uses: ./.github/workflows/template.argo-workflows.yml
    with:
      workflow_file: fast_bi_argo_workflows/mr_e2e_workflow.yaml
      argo_namespace: argo
```

## Common Workflow Inputs

- `ARGO_NAMESPACE`: Argo namespace (e.g., `argo`)
- `WORKFLOW_FILE`: Path to the workflow YAML
- `GIT_REF` / `COMMIT_SHA`: ref for checkout steps
- `REGISTRY`, `IMAGE_TAG`: image registry and tag
- `DBT_PROJECT_DIR`, `DAG_DIR`: paths for model and orchestration artifacts

## Main Functionality

- **Branch CI**: lint YAML/SQL, detect changes, validate dependencies, compile DAG/dbt manifest, PVC cleanup
- **MR E2E**: end-to-end data model validations, tagging, orchestration variable imports, cleanup
- **Main**: production compilations, data catalog/governance imports, data quality deployments, incremental refresh helpers

## Testing

Run workflows via Argo with parameters:

```bash
argo submit fast_bi_argo_workflows/mr_e2e_workflow.yaml \
  -n ${ARGO_NAMESPACE:-argo} \
  -p git_ref=${GIT_REF:-HEAD} \
  --watch
```

## Troubleshooting

- **Stuck/Pending**: check controller and pod events (`kubectl describe wf/<name>`)
- **PVC issues**: use `0_branch/k8s-pvc-cleanup.sh`
- **Missing variables**: verify CI variables and Argo parameters

### Getting Help

- **Documentation**: `https://wiki.fast.bi`
- **Issues**: `https://github.com/fast-bi/bi-platform-docker-images/issues`
- **Email**: support@fast.bi

## 🤝 Contributing

1. Open an issue or discussion
2. Create a feature branch
3. Update workflows/scripts and docs
4. Validate via MR E2E workflow
5. Open a merge/pull request

## License

This project is licensed under the MIT License - see the `LICENSE` file for details.

```
MIT License

Copyright (c) 2025 Fast.BI

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

This repository is part of the Fast.BI platform infrastructure.
