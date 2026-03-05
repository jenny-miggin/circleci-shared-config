# CircleCI Shared Config - Approaches

Two methods for sharing CircleCI pipeline configuration across multiple repos. Both use this repository as the source of truth for shared pipeline logic.

## Method 1 - Child repo imports a shared template via URL orb

Each child repo maintains its own `config.yml` which pulls in a shared job template from this repo as a [URL orb](https://circleci.com/docs/orb-concepts/#url-orbs). The child repo controls its own workflow and passes repo-specific values as job parameters.

### Config files

- **Shared template:** [`build-n-push-ecr-image-template.yml`](https://github.com/jenny-miggin/circleci-shared-config/blob/main/build-n-push-ecr-image-template.yaml)
- **Child repo example:** [`my-child-repo/.circleci/config.yml`](https://github.com/jenny-miggin/my-child-repo/blob/use-url-orb-template/.circleci/config.yml)

### How it works

1. The child repo imports the template as a URL orb in its `config.yml`
2. Repo-specific values (`image_name`, `ecr_repo`, `region` etc.) are passed as job parameters in the workflow
3. Optionally, a child repo can use [`override-with`](https://circleci.com/docs/guides/orchestrate/how-to-override-config/) to swap the template job for its own local implementation

```yaml
# my-child-repo/.circleci/config.yml
orbs:
  template: "https://raw.githubusercontent.com/jenny-miggin/circleci-shared-config/refs/heads/main/build-n-push-ecr-image-template.yaml"

workflows:
  build-and-deploy:
    jobs:
      - template/build_and_deploy_image:
          image_name: "my-service"
          ecr_repo: "my-service-ecr"
          region: "eu-west-1"
```

### Pros

- Simple to understand: the child repo config is self-contained and explicit
- No CircleCI GitHub App pipeline definitions required
- Repo-specific values are visible and reviewable in a PR

### Cons

- Every repo needs its own `config.yml` to maintain
- Workflow structure can drift and each team can change their own config independently
- Rolling out a pipeline change (e.g. adding a security scan) requires a PR in every repo

## Method 2 - Central config driven by project environment variables

A single central config file drives the pipeline for all child repos. Repo-specific values are not stored in any config file - they are set once as [project-level environment variables](https://circleci.com/docs/set-environment-variable/#set-an-environment-variable-in-a-project) in CircleCI. Child repos require zero config files by default.

### Config files

- **Central config:** [`build-n-push-ecr-image-template.yml`](https://github.com/jenny-miggin/circleci-shared-config/blob/central-config-using-envvars/build-n-push-ecr-image-template.yml)
- **Child repo example** *(only needed for repos with a custom test job):* [`my-child-repo/.circleci/config.yml`](https://github.com/jenny-miggin/my-child-repo/blob/repo-specific-jobs/.circleci/config.yml)

### How it works

1. A CircleCI [pipeline definition](https://circleci.com/docs/pipeline-overview/#pipeline-definitions) (requires the [GitHub App integration](https://circleci.com/docs/github-apps-integration/)) points each child repo at `build-n-push-ecr-image-template.yml` as its config source instead of the repo's own `.circleci/config.yml`
2. A [trigger](https://circleci.com/docs/triggers-overview/) is set up for each child repo to run the pipeline on push
3. Jobs read repo-specific values from [project-level environment variables](https://circleci.com/docs/set-environment-variable/#set-an-environment-variable-in-a-project) set in CircleCI
4. The workflow structure is identical across all repos and can only be changed in one place

```yaml
# build-n-push-ecr-image-template.yml - no hardcoded repo values, reads from env vars at runtime
jobs:
  build_and_deploy_image:
    steps:
      - run:
          name: Build Docker image
          command: echo "Building $IMAGE_NAME"   # $IMAGE_NAME set as project env var
      - run:
          name: Push image to ECR
          command: |
            docker push ${CONTAINER_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}
```

### Repo-specific config files

Repos that need custom behaviour beyond what project env vars can provide can still maintain their own `config.yml`. A separate [pipeline definition](https://circleci.com/docs/pipeline-overview/#pipeline-definitions) and [trigger](https://circleci.com/docs/triggers-overview/) can be set up for those repos, pointing to their own config file instead of the central one. This keeps the standard pipeline as the default while giving teams an escape hatch when genuinely needed.

### Environment variables to set per project

See the CircleCI docs for [how to set project-level environment variables](https://circleci.com/docs/set-environment-variable/#set-an-environment-variable-in-a-project).

| Variable | Description | Example |
|---|---|---|
| `AWS_ROLE_ARN` | IAM role ARN for OIDC authentication | `arn:aws:iam::123456789:role/my-role` |
| `AWS_REGION` | AWS region | `eu-west-1` |
| `IMAGE_NAME` | Docker image name | `payments-service` |
| `ECR_REPO` | ECR repository name | `payments-service-ecr` |
| `SSM_PREFIX` | SSM parameter store prefix | `/payments-service` |
| `SOURCE` | Source identifier | `payments-service` |
| `CONTAINER_ACCOUNT_ID` | AWS account ID for ECR | `123456789` |

### Pros

- No per-repo config files by default - onboarding a new repo only requires setting env vars, creating a [pipeline definition](https://circleci.com/docs/pipeline-overview/#pipeline-definitions), and adding a [trigger](https://circleci.com/docs/triggers-overview/)
- Workflow structure is enforced centrally - teams cannot deviate from the standard pipeline
- Pipeline changes roll out to all repos instantly from a single commit to this repo
- Repos that need custom behaviour can opt in to their own config via a separate pipeline definition and trigger

### Cons

- Environment variables must be set correctly per project which is less visible than values in a config file PR. These can be managed via API.
- Slightly more setup per repo (pipeline definition + trigger) compared to just adding a `config.yml`

## Comparison

| | Method 1 | Method 2 |
|---|---|---|
| Number of repos | Few | Many (10s-100s) |
| GitHub App required | No | Yes |
| Per-repo config files | Yes (`config.yml`) | No (optional for custom behaviour) |
| Repo-specific values | Job parameters in `config.yml` | [Project env vars](https://circleci.com/docs/set-environment-variable/#set-an-environment-variable-in-a-project) in CircleCI |
| Workflow enforcement | No - each repo can change it | Yes - locked in central config |
| Onboarding a new repo | PR to add `config.yml` | Set env vars + [pipeline definition](https://circleci.com/docs/pipeline-overview/#pipeline-definitions) + [trigger](https://circleci.com/docs/triggers-overview/) |
| Rolling out pipeline changes | PR per repo | Single PR in this repo |
| Custom behaviour per repo | Via `override-with` in `config.yml` | Via separate pipeline definition + trigger |