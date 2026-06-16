# Build Spec Patterns

OneDev CI/CD is defined in `.onedev-buildspec.yml` at repository root. Get full schema with `tod build get-spec-schema`.

## Table of Contents

1. [Minimal CI Example](#minimal-ci-example)
2. [Multi-Job Pipeline](#multi-job-pipeline)
3. [Docker Build & Push](#docker-build--push)
4. [Matrix Build](#matrix-build)
5. [Using Services](#using-services)
6. [Job Secrets & Parameters](#job-secrets--parameters)
7. [Build Promotion](#build-promotion)
8. [Step Templates](#step-templates)
9. [Conditional Execution](#conditional-execution)
10. [Reusing Build Specs](#reusing-build-specs)

## Minimal CI Example

```yaml
version: 1.0.0
jobs:
  - name: ci
    steps:
      - !CheckoutStep
        name: checkout
        cloneDepth: 50
      - !CommandStep
        name: build and test
        image: node:18
        commands: |
          npm ci
          npm test
    triggers:
      - !BranchUpdateTrigger {}
    jobExecutors:
      - !ServerDockerExecutor
        osSelector: ubuntu
```

## Multi-Job Pipeline

```yaml
version: 1.0.0
jobs:
  - name: build
    steps:
      - !CheckoutStep
        name: checkout
      - !CommandStep
        name: compile
        image: maven:3.9
        commands: mvn package -DskipTests
      - !PublishArtifactStep
        name: publish jar
        artifacts: target/*.jar
    triggers:
      - !BranchUpdateTrigger {}

  - name: test
    steps:
      - !CheckoutStep
        name: checkout
      - !CommandStep
        name: run tests
        image: maven:3.9
        commands: mvn test
      - !PublishJUnitReportStep
        name: publish test results
        reportFiles: target/surefire-reports/*.xml
    jobDependencies:
      - jobName: build
        requireSuccessful: true
    triggers:
      - !BranchUpdateTrigger {}

  - name: deploy
    steps:
      - !CheckoutStep
        name: checkout
      - !CommandStep
        name: deploy to staging
        image: alpine/k8s
        commands: |
          kubectl apply -f k8s/
    jobDependencies:
      - jobName: test
        requireSuccessful: true
    triggers:
      - !BranchUpdateTrigger
        branches: main
```

## Docker Build & Push

```yaml
version: 1.0.0
jobs:
  - name: docker-build
    steps:
      - !CheckoutStep
        name: checkout
      - !BuildImageStep
        name: build image
        platforms: linux/amd64,linux/arm64
        buildArgs:
          - name: VERSION
            value: '@build_version@'
      - !PublishImageStep
        name: push to registry
        registry: !ProjectContainerRegistry {}
        tags:
          - '@build_version@'
          - latest
    triggers:
      - !TagCreateTrigger {}
    jobExecutors:
      - !ServerDockerExecutor
        osSelector: ubuntu
```

## Matrix Build

```yaml
version: 1.0.0
jobs:
  - name: test-matrix
    steps:
      - !CheckoutStep
        name: checkout
      - !CommandStep
        name: test
        image: '@matrix:runtime@'
        commands: npm test
    matrix:
      - name: runtime
        values:
          - node:18
          - node:20
          - node:22
    triggers:
      - !BranchUpdateTrigger {}
```

## Using Services

```yaml
version: 1.0.0
jobs:
  - name: integration-test
    services:
      - name: postgres
        image: postgres:15
        envVars:
          - name: POSTGRES_PASSWORD
            value: test
          - name: POSTGRES_DB
            value: testdb
        readinessCheck: !HttpCheck
          port: 5432
          path: /
          timeout: 60000
      - name: redis
        image: redis:7
    steps:
      - !CheckoutStep
        name: checkout
      - !CommandStep
        name: run integration tests
        image: node:18
        commands: |
          export DATABASE_URL=postgres://postgres:test@postgres:5432/testdb
          export REDIS_URL=redis://redis:6379
          npm run test:integration
    triggers:
      - !BranchUpdateTrigger {}
```

## Job Secrets & Parameters

```yaml
version: 1.0.0
jobs:
  - name: deploy
    params:
      - name: environment
        type: Choice
        choices:
          - staging
          - production
        allowEmpty: false
      - name: dryRun
        type: Boolean
        defaultValue: false
    steps:
      - !CheckoutStep
        name: checkout
      - !CommandStep
        name: deploy
        image: alpine/k8s
        commands: |
          echo "Deploying to @param:environment@"
          kubectl set image deployment/app app=myimage:@build_version@
          '@param:dryRun@' && echo "Dry run, skipping rollout" || kubectl rollout status deployment/app
    jobSecrets:
      - name: kubeconfig
      - name: deploy-token
    triggers:
      - !ManualTrigger {}
```

## Build Promotion

```yaml
version: 1.0.0
jobs:
  - name: ci
    steps:
      - !CheckoutStep
        name: checkout
      - !CommandStep
        name: build
        image: node:18
        commands: npm ci && npm run build
      - !PublishArtifactStep
        name: publish dist
        artifacts: dist/**
    triggers:
      - !BranchUpdateTrigger {}

  - name: deploy-staging
    steps:
      - !CommandStep
        name: deploy
        image: alpine/k8s
        commands: kubectl apply -f k8s/staging/
    jobDependencies:
      - jobName: ci
        requireSuccessful: true
        artifacts: '**'
    triggers:
      - !BuildPromotionTrigger
        promoters:
          - name: Promote to Staging

  - name: deploy-production
    steps:
      - !CommandStep
        name: deploy
        image: alpine/k8s
        commands: kubectl apply -f k8s/production/
    jobDependencies:
      - jobName: deploy-staging
        requireSuccessful: true
    triggers:
      - !BuildPromotionTrigger
        promoters:
          - name: Promote to Production
```

## Step Templates

Define reusable step sequences in project settings, then reference in build spec:

```yaml
version: 1.0.0
jobs:
  - name: ci
    steps:
      - !CheckoutStep
        name: checkout
      - !UseTemplateStep
        name: setup node
        templateName: setup-node
        paramMap:
          nodeVersion: '18'
      - !CommandStep
        name: test
        image: node:18
        commands: npm test
    triggers:
      - !BranchUpdateTrigger {}
```

## Conditional Execution

```yaml
version: 1.0.0
jobs:
  - name: deploy
    condition: '"@branch@" == "main" || "@branch@".startsWith("release/")'
    steps:
      - !CheckoutStep
        name: checkout
      - !CommandStep
        name: deploy
        image: alpine/k8s
        commands: kubectl apply -f k8s/
    triggers:
      - !BranchUpdateTrigger {}
```

## Reusing Build Specs

Include common build spec fragments:

```yaml
version: 1.0.0
imports:
  - !Include 
    projectPath: shared/ci-templates
    revision: main
    path: node-ci.yml
jobs:
  - name: ci
    steps:
      - !CheckoutStep
        name: checkout
      - !UseTemplateStep
        name: standard ci
        templateName: node-ci
    triggers:
      - !BranchUpdateTrigger {}
```

## Common Step Types Reference

| Step Type | Purpose |
|-----------|---------|
| `!CheckoutStep` | Clone repository with optional depth/lfs/submodules |
| `!CommandStep` | Run shell commands in specified image |
| `!BuildImageStep` | Build Docker image from Dockerfile |
| `!RunContainerStep` | Run a container as part of pipeline |
| `!PublishArtifactStep` | Publish files as build artifacts |
| `!PublishReportStep` | Publish generic HTML reports |
| `!PublishJUnitReportStep` | Publish JUnit XML test results |
| `!PublishCoberturaReportStep` | Publish Cobertura coverage reports |
| `!PublishCheckstyleReportStep` | Publish Checkstyle results |
| `!PublishDependencyScanResultStep` | Publish dependency vulnerability scan |
| `!PublishImageScanResultStep` | Publish container image scan results |
| `!SetBuildVersionStep` | Set build version from file or command |
| `!UseTemplateStep` | Reference a step template |

## Common Trigger Types Reference

| Trigger Type | Fires When |
|--------------|-----------|
| `!BranchUpdateTrigger` | Code pushed to branch |
| `!TagCreateTrigger` | Tag created |
| `!PullRequestUpdateTrigger` | PR opened/updated |
| `!CronTrigger` | Scheduled time reached |
| `!ManualTrigger` | User manually triggers |
| `!DependencyIngestTrigger` | Dependency updated |
| `!BuildPromotionTrigger` | Build promoted |
| `!ScheduleTrigger` | One-time scheduled execution |

## Common Executor Types Reference

| Executor | Runs On |
|----------|---------|
| `!ServerDockerExecutor` | Docker on OneDev server |
| `!RemoteDockerExecutor` | Docker on remote agent machine |
| `!ServerShellExecutor` | Server shell/bash |
| `!RemoteShellExecutor` | Remote agent shell |
| `!KubernetesExecutor` | Kubernetes cluster |
