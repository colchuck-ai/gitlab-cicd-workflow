# gitlab-cicd-workflow

This [GitLab CI/CD component](https://docs.gitlab.com/ci/components/) project provides reusable components for deployment and release workflows. These components allow you to control:

- When jobs are run
- Which GitLab environments jobs deploy to

## Contents

- [gitlab-cicd-workflow](#gitlab-cicd-workflow)
  - [Contents](#contents)
  - [Usage](#usage)
  - [Release Workflow](#release-workflow)
    - [Deployment Workflow](#deployment-workflow)
  - [Components](#components)
    - [`workflow` Component](#workflow-component)
    - [`run-if-protected` Component](#run-if-protected-component)
      - [`run-if-protected` Component Usage](#run-if-protected-component-usage)
    - [`run-if-unprotected` Component](#run-if-unprotected-component)
      - [`run-if-unprotected` Component Usage](#run-if-unprotected-component-usage)
      - [`workflow` Component Usage](#workflow-component-usage)
    - [`deploy-if-protected` Component](#deploy-if-protected-component)
      - [`deploy-if-protected` Component Inputs](#deploy-if-protected-component-inputs)
      - [`deploy-if-protected` Component Usage](#deploy-if-protected-component-usage)
    - [`deploy-if-unprotected` Component](#deploy-if-unprotected-component)
      - [`deploy-if-unprotected` Component Inputs](#deploy-if-unprotected-component-inputs)
      - [`deploy-if-unprotected` Component Usage](#deploy-if-unprotected-component-usage)
    - [`stack-trigger-rules` Component](#stack-trigger-rules-component)
      - [`stack-trigger-rules` Component Inputs](#stack-trigger-rules-component-inputs)
      - [`stack-trigger-rules` Component Usage](#stack-trigger-rules-component-usage)
  - [Disclaimer](#disclaimer)

## Usage

## Release Workflow

```yaml
include:
  - component: gitlab.example.com/engineering/dev-tools/internal/gitlab-cicd-workflow/workflow@1
  - component: gitlab.example.com/engineering/dev-tools/internal/gitlab-cicd-workflow/run-if-protected@1

stages:
  - unit-test
  - release

test:
  stage: unit-test
  script:
    - echo "running unit tests ..."

release:
  stage: release
  extends:
    - .run-if-protected
  script:
    - echo "publishing release ..."
```

### Deployment Workflow

```yaml
include:
  - component: gitlab.example.com/engineering/dev-tools/internal/gitlab-cicd-workflow/workflow@1
  - component: gitlab.example.com/engineering/dev-tools/internal/gitlab-cicd-workflow/deploy-if-unprotected@1
    inputs:
      environment-deployment-tier: testing
      environment-name: test/manual
      when: manual
  - component: gitlab.example.com/engineering/dev-tools/internal/gitlab-cicd-workflow/deploy-if-unprotected@1
    inputs:
      environment-deployment-tier: testing
      environment-name: test/auto
  - component: gitlab.example.com/engineering/dev-tools/internal/gitlab-cicd-workflow/deploy-if-protected@1
    inputs:
      environment-deployment-tier: staging
      environment-name: stg
  - component: gitlab.example.com/engineering/dev-tools/internal/gitlab-cicd-workflow/deploy-if-protected@1
    inputs:
      environment-deployment-tier: production
      environment-name: prod
      when: manual

stages:
  - unit-test
  - deploy-test
  - deploy-staging
  - test-staging
  - deploy-prod

unit-test:
  stage: unit-test
  script:
    - echo "running unit tests ..."

deploy-test-manual:
  stage: deploy-test
  extends:
    - .deploy-if-unprotected:testing:test/manual
  script:
    - echo "running manually triggered test deployment ..."

deploy-test-auto:
  stage: deploy-test
  extends:
    - .deploy-if-unprotected:testing:test/auto
  script:
    - echo "deploying to test environment ..."

deploy-staging:
  stage: deploy-staging
  extends:
    - .deploy-if-protected:staging:stg
  script:
    - echo "deploying to staging environment ..."

test-staging:
  stage: test-staging
  extends:
    - .run-if-protected
  script:
    - echo "running tests on the staging environment ..."

deploy-prod:
  stage: deploy-prod
  extends:
    - .deploy-if-protected:production:prod
  script:
    - echo "deploying to prod environment ..."
```

## Components

### `workflow` Component

Defines pipeline rules to support:

- Merge request pipelines
- Branch pipelines (if the branch is not the source of an open merge request)
- Skips tag pipelines and branch pipelines for branches with open merge requests

---

### `run-if-protected` Component

Defines a hidden job `.run-if-protected` that runs jobs in branch pipelines if the branch is protected.

#### `run-if-protected` Component Usage

```yaml
include:
  - component: gitlab.example.com/engineering/dev-tools/internal/gitlab-cicd-workflow/run-if-protected@1

stages:
  - test

test-protected:
  stage: test
  extends:
    - .run-if-protected
  script:
    - echo "Running tests on protected branches..."
```

### `run-if-unprotected` Component

Defines a hidden job `.run-if-unprotected` that runs jobs in:

- Merge request pipelines if the source branch is unprotected
- Branch pipelines if the branch is unprotected and not the source of an open merge request

#### `run-if-unprotected` Component Usage

```yaml
include:
  - component: gitlab.example.com/engineering/dev-tools/internal/gitlab-cicd-workflow/run-if-unprotected@1

stages:
  - test

test-unprotected:
  stage: test
  extends:
    - .run-if-unprotected
  script:
    - echo "Running tests on unprotected branches..."
```

#### `workflow` Component Usage

```yaml
include:
  - component: gitlab.example.com/engineering/dev-tools/internal/gitlab-cicd-workflow/workflow@1
  - component: gitlab.example.com/engineering/dev-tools/internal/gitlab-cicd-workflow/run-if-protected@1
  - component: gitlab.example.com/engineering/dev-tools/internal/gitlab-cicd-workflow/run-if-unprotected@1

stages:
  - unit-test

unit-test:
  stage: unit-test
  script:
    - echo "running unit tests ..."

unit-tests-test-env:
  stage: unit-test
  extends:
    - .run-if-unprotected
  script:
    - echo "running unit tests for unprotected branches ..."

unit-tests-prod-env:
  stage: unit-test
  extends:
    - .run-if-protected
  script:
    - echo "running unit tests for protected branches ..."
```

---

### `deploy-if-protected` Component

Defines a hidden job `.deploy-if-protected:<environment-deployment-tier>:<environment-name>` that:

- Uses `rules` to run in branch pipelines if the branch is protected
- Defines an `environment` using `environment-deployment-tier` and `environment-name` inputs

#### `deploy-if-protected` Component Inputs

| Name                          | Type     | Description                                                 | Required | Default      |
| ----------------------------- | -------- | ----------------------------------------------------------- | -------- | ------------ |
| `environment-deployment-tier` | `string` | The deployment tier (`testing`, `staging`, or `production`) | yes      |              |
| `environment-name`            | `string` | Name of the GitLab environment                              | yes      |              |
| `when`                        | `string` | When to trigger the job (`on_success` or `manual`)          | no       | `on_success` |

#### `deploy-if-protected` Component Usage

```yaml
include:
  - component: gitlab.example.com/engineering/dev-tools/internal/prod-deployment@1
    inputs:
      environment-deployment-tier: production
      environment-name: prod

stages:
  - deploy

deploy-prod:
  stage: deploy
  extends:
    - .prod-deployment:production:prod
  script:
    - echo "deploying to prod ..."
```

---

### `deploy-if-unprotected` Component

Defines a hidden job `.deploy-if-unprotected:<environment-deployment-tier>:<environment-name>` that:

- Uses `rules` to run in:
  - Merge request pipelines if the source branch is unprotected and does **not** match `hotfix-branch-re`
  - Branch pipelines if the branch is unprotected, is not the source of an open merge request, and does **not** match `hotfix-branch-re`
- Defines an `environment` using `environment-deployment-tier` and `environment-name` inputs

#### `deploy-if-unprotected` Component Inputs

| Name                          | Type     | Description                                                                             | Required | Default         |
| ----------------------------- | -------- | --------------------------------------------------------------------------------------- | -------- | --------------- |
| `hotfix-branch-re`            | `string` | `re2` regex for identifying hotfix branches, which are exempt from non-prod deployments | no       | `/^hotfix-.+$/` |
| `environment-deployment-tier` | `string` | Non-production deployment tier (`other`, `development`, or `testing`)                   | yes      |                 |
| `environment-name`            | `string` | Name of the GitLab environment                                                          | yes      |                 |
| `when`                        | `string` | When to trigger the job (`on_success` or `manual`)                                      | no       | `on_success`    |

#### `deploy-if-unprotected` Component Usage

```yaml
include:
  - component: gitlab.example.com/engineering/dev-tools/internal/deploy-if-unprotected@1
    inputs:
      environment-deployment-tier: testing
      environment-name: test

stages:
  - deploy

deploy-test:
  stage: deploy
  extends:
    - .deploy-if-unprotected:testing:test
  script:
    - echo "deploying to the test environment ..."
```

### `stack-trigger-rules` Component

Defines a hidden job `.stack-trigger-rules:<stack>` that conditionally triggers stack-specific pipelines based on various conditions including:

- Manual triggers via the `RUN_STACK` variable (using RE2 regex pattern matching)
- File changes in specified paths
- Pipeline source (web UI, API, merge request, push, schedule)
- Branch protection status

#### `stack-trigger-rules` Component Inputs

| Name            | Type     | Description                                                               | Required | Default              |
| --------------- | -------- | ------------------------------------------------------------------------- | -------- | -------------------- |
| `stack`         | `string` | The stack name to use for path filtering and conditional execution        | yes      |                      |
| `changes-paths` | `array`  | Array of file paths to check for changes that would trigger this pipeline | yes      |                      |
| `compare-to`    | `string` | The branch or tag to compare changes against                              | no       | `$CI_DEFAULT_BRANCH` |

#### `stack-trigger-rules` Component Usage

```yaml
include:
  - component: gitlab.example.com/engineering/dev-tools/internal/gitlab-cicd-workflow/stack-trigger-rules@1
    inputs:
      stack: core
      changes-paths:
        - "environments/*/multi/global/core/**/*.tf"
        - "modules/multi-global-core/**/*.tf"

stages:
  - stack-triggers

trigger-core:
  stage: stack-triggers
  extends: .stack-trigger-rules:core
  trigger:
    include: .gitlab/core.gitlab-ci.yml
```

This component provides sophisticated pipeline triggering logic:

1. Runs pipelines for stacks matching the `RUN_STACK` regex pattern
2. Skips tag pipelines and branches with open merge requests
3. For scheduled pipelines, only runs if `RUN_STACK` matches
4. For web/API triggers, uses change detection against the default branch
5. For merge request events, runs for stacks with detected changes
6. For push events, runs for stacks with detected changes

---

## Disclaimer

We built this because we needed it. We welcome contributions. If you want to consume the artifacts released by this project, do so with the same consideration you would take when consuming an open source component.
