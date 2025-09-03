# GitHub Actions Documentation

## Table of Contents
1. [Introduction](#introduction)
2. [Core Concepts](#core-concepts)
3. [Workflow Structure](#workflow-structure)
4. [Current Pipeline Analysis](#current-pipeline-analysis)
5. [Best Practices](#best-practices)
6. [Common Patterns](#common-patterns)
7. [Security Considerations](#security-considerations)
8. [Troubleshooting](#troubleshooting)

## Introduction

GitHub Actions is a CI/CD platform that allows you to automate your build, test, and deployment pipeline. You can create workflows that build and test every pull request to your repository, or deploy merged pull requests to production.

## Core Concepts

### Workflows
- YAML files stored in `.github/workflows/`
- Define automated processes triggered by events
- Can contain one or more jobs

### Events
- Triggers that start workflow runs
- Examples: `push`, `pull_request`, `schedule`, `workflow_dispatch`

### Jobs
- Set of steps executed on the same runner
- Run in parallel by default
- Can have dependencies using `needs`

### Steps
- Individual tasks within a job
- Can run commands or use actions

### Actions
- Reusable units of code
- Can be from marketplace or custom

### Runners
- Servers that run workflows
- GitHub-hosted or self-hosted

## Workflow Structure

```yaml
name: Workflow Name
on: [trigger events]
jobs:
  job-name:
    runs-on: runner-type
    steps:
      - name: Step Name
        uses: action@version
        with:
          parameter: value
```

## Current Pipeline Analysis

Your current CI/CD pipeline (`cicd.yml`) implements a comprehensive DevSecOps workflow:

### Pipeline Stages

#### 1. Compile Stage
```yaml
compile:
  runs-on: self-hosted
  steps:
    - uses: actions/checkout@v4
    - name: Set up JDK 17
      uses: actions/setup-java@v4
    - name: Build with Maven
      run: mvn compile
```

**Purpose**: Validates code compilation
**Key Features**:
- Uses self-hosted runner
- Sets up Java 17 with Temurin distribution
- Enables Maven caching for performance

#### 2. Security Scan Stage
```yaml
security-scan:
  runs-on: self-hosted
  needs: compile
```

**Tools Used**:
- **Trivy**: Filesystem vulnerability scanning
- **Gitleaks**: Secret detection in code

**Security Benefits**:
- Early vulnerability detection
- Prevents secret leakage
- Generates security reports

#### 3. Unit Test Stage
```yaml
Unit-Test:
  runs-on: self-hosted
  needs: security-scan
```

**Purpose**: Executes unit tests using Maven
**Dependencies**: Runs after security scan passes

#### 4. Build & SonarQube Analysis
```yaml
Build_sonar:
  runs-on: self-hosted
  needs: Unit-Test
```

**Key Activities**:
- Maven package creation
- JAR artifact upload
- SonarQube code quality analysis
- Quality gate enforcement

#### 5. Docker Build & Push
```yaml
Build_Docker_Image_and_Push:
  runs-on: self-hosted
  needs: Build_sonar
```

**Features**:
- Multi-platform Docker builds
- Docker Hub integration
- Artifact reuse from previous stage

#### 6. Kubernetes Deployment
```yaml
deploy_to_kubernetes:
  runs-on: self-hosted
  needs: Build_Docker_Image_and_Push
```

**Deployment Process**:
- AWS CLI installation
- EKS cluster configuration
- Kubernetes deployment execution

## Best Practices

### 1. Workflow Organization
```yaml
# Use descriptive names
name: "CI/CD Pipeline - Banking App"

# Specify clear triggers
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
```

### 2. Job Dependencies
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
  
  deploy:
    needs: test  # Wait for test completion
    if: success()  # Only run if tests pass
```

### 3. Environment Variables
```yaml
env:
  JAVA_VERSION: '17'
  MAVEN_OPTS: '-Xmx1024m'

jobs:
  build:
    env:
      NODE_ENV: production
```

### 4. Conditional Execution
```yaml
steps:
  - name: Deploy to Production
    if: github.ref == 'refs/heads/main'
    run: ./deploy.sh
```

### 5. Matrix Builds
```yaml
strategy:
  matrix:
    java-version: [11, 17, 21]
    os: [ubuntu-latest, windows-latest]
```

## Common Patterns

### 1. Artifact Management
```yaml
# Upload artifacts
- name: Upload JAR
  uses: actions/upload-artifact@v4
  with:
    name: app-jar
    path: target/*.jar

# Download artifacts
- name: Download JAR
  uses: actions/download-artifact@v4
  with:
    name: app-jar
    path: ./app
```

### 2. Caching Dependencies
```yaml
- name: Cache Maven dependencies
  uses: actions/cache@v3
  with:
    path: ~/.m2
    key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
```

### 3. Multi-Stage Docker Builds
```yaml
- name: Build and push Docker image
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: |
      myapp:latest
      myapp:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### 4. Parallel Job Execution
```yaml
jobs:
  test-unit:
    runs-on: ubuntu-latest
  
  test-integration:
    runs-on: ubuntu-latest
  
  security-scan:
    runs-on: ubuntu-latest
  
  deploy:
    needs: [test-unit, test-integration, security-scan]
```

## Security Considerations

### 1. Secrets Management
```yaml
# Store sensitive data in GitHub Secrets
env:
  SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
```

**Current Secrets Used**:
- `SONAR_TOKEN`: SonarQube authentication
- `DOCKERHUB_TOKEN`: Docker Hub access
- `AWS_ACCESS_KEY_ID`: AWS authentication
- `AWS_SECRET_ACCESS_KEY`: AWS secret key
- `EKS_KUBECONFIG`: Kubernetes configuration

### 2. Variables for Non-Sensitive Data
```yaml
env:
  SONAR_HOST_URL: ${{ vars.SONAR_HOST_URL }}
  DOCKERHUB_USERNAME: ${{ vars.DOCKERHUB_USERNAME }}
```

### 3. Permission Restrictions
```yaml
permissions:
  contents: read
  security-events: write
  actions: read
```

### 4. OIDC Authentication (Recommended)
```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/GitHubActions
    aws-region: us-east-1
```

## Troubleshooting

### Common Issues

#### 1. Self-Hosted Runner Problems
```bash
# Check runner status
./run.sh --check

# Restart runner service
sudo systemctl restart actions.runner.service
```

#### 2. Maven Build Failures
```yaml
- name: Debug Maven
  run: |
    mvn --version
    mvn dependency:tree
    mvn clean compile -X  # Verbose output
```

#### 3. Docker Build Issues
```yaml
- name: Debug Docker
  run: |
    docker --version
    docker system df
    docker buildx ls
```

#### 4. Kubernetes Deployment Failures
```yaml
- name: Debug Kubernetes
  run: |
    kubectl cluster-info
    kubectl get nodes
    kubectl describe pod <pod-name>
```

### Monitoring and Logging

#### 1. Workflow Status Badges
```markdown
![CI/CD Pipeline](https://github.com/username/repo/workflows/CI%20CD%20pipeline/badge.svg)
```

#### 2. Notification Setup
```yaml
- name: Notify on failure
  if: failure()
  uses: 8398a7/action-slack@v3
  with:
    status: failure
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

## Optimization Recommendations

### 1. Improve Current Pipeline

#### Add Parallel Execution
```yaml
jobs:
  compile:
    runs-on: self-hosted
  
  security-scan:
    runs-on: self-hosted
    # Remove 'needs: compile' for parallel execution
  
  unit-test:
    runs-on: self-hosted
    needs: compile  # Only depends on compilation
```

#### Add Caching
```yaml
- name: Cache Maven dependencies
  uses: actions/cache@v3
  with:
    path: ~/.m2
    key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
    restore-keys: |
      ${{ runner.os }}-m2-
```

#### Environment-Specific Deployments
```yaml
deploy_to_staging:
  if: github.ref == 'refs/heads/develop'
  environment: staging

deploy_to_production:
  if: github.ref == 'refs/heads/main'
  environment: production
  needs: [deploy_to_staging]
```

### 2. Add Missing Features

#### Pull Request Workflow
```yaml
name: PR Validation
on:
  pull_request:
    branches: [main]

jobs:
  validate:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: mvn test
      - name: Security scan
        run: trivy fs .
```

#### Rollback Capability
```yaml
rollback:
  runs-on: self-hosted
  if: failure()
  steps:
    - name: Rollback deployment
      run: kubectl rollout undo deployment/bankapp
```

## Conclusion

Your current GitHub Actions pipeline demonstrates excellent DevSecOps practices with:
- Comprehensive security scanning
- Quality gates with SonarQube
- Containerized deployments
- Cloud-native deployment to EKS

Consider implementing the optimization recommendations to improve performance, add environment-specific deployments, and enhance monitoring capabilities.