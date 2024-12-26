# CI/CD with Docker and GitHub Actions Guide

[← Back to Next.js Deployment](nextjs-docker-deploy.md) | [Back to Main Guide →](README.md)

---

## CI/CD Workflow

```mermaid
graph LR
    subgraph "Development"
        A[Developer] -->|push| B[GitHub Repository]
        B -->|trigger| C[GitHub Actions]
    end
    
    subgraph "CI Pipeline"
        C -->|build| D[Docker Build]
        D -->|test| E[Run Tests]
        E -->|scan| F[Security Scan]
        F -->|push| G[Docker Registry]
    end
    
    subgraph "CD Pipeline"
        G -->|pull| H[Staging Deploy]
        H -->|approve| I[Production Deploy]
    end
    
    style A fill:#90EE90
    style G fill:#FFB6C1
    style I fill:#87CEEB
```

## Deployment Process

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant Git as GitHub
    participant CI as GitHub Actions
    participant Reg as Docker Registry
    participant Prod as Production Server
    
    Dev->>Git: Push Code
    Git->>CI: Trigger Workflow
    CI->>CI: Build Docker Image
    CI->>CI: Run Tests
    CI->>Reg: Push Image
    
    alt Tests Pass
        CI->>Prod: Deploy to Production
        Prod->>Prod: Health Check
        Prod-->>CI: Deployment Status
        CI-->>Git: Update Status
    else Tests Fail
        CI-->>Git: Mark as Failed
        Git-->>Dev: Notification
    end
```

## Table of Contents
1. [Repository Setup](#repository-setup)
2. [GitHub Actions Configuration](#github-actions-configuration)
3. [Docker Registry Setup](#docker-registry-setup)
4. [Deployment Workflows](#deployment-workflows)
5. [Environment Management](#environment-management)
6. [Security Best Practices](#security-best-practices)

## Repository Setup

### Required Files Structure
```plaintext
your-project/
├── .github/
│   └── workflows/
│       ├── dev-deploy.yml
│       ├── staging-deploy.yml
│       └── prod-deploy.yml
├── scripts/
│   ├── deploy.sh
│   └── health-check.sh
├── Dockerfile
└── docker-compose.yml
```

### Environment Files
```plaintext
.github/
└── workflows/
    └── environments/
        ├── dev/
        │   └── .env.dev
        ├── staging/
        │   └── .env.staging
        └── prod/
            └── .env.prod
```

## GitHub Actions Configuration

### Development Workflow
```yaml
# .github/workflows/dev-deploy.yml
name: Development Deploy

on:
  push:
    branches: [ develop ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Log in to the Container registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and test
        uses: docker/build-push-action@v4
        with:
          context: .
          load: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:test
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Run tests
        run: |
          docker run --rm ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:test npm test

  deploy:
    needs: build-and-test
    runs-on: ubuntu-latest
    environment: development

    steps:
      - name: Deploy to Dev Server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.DEV_HOST }}
          username: ${{ secrets.DEV_USERNAME }}
          key: ${{ secrets.DEV_SSH_KEY }}
          script: |
            docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
            docker-compose -f docker-compose.dev.yml up -d
```

### Production Workflow
```yaml
# .github/workflows/prod-deploy.yml
name: Production Deploy

on:
  push:
    tags:
      - 'v*'

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    environment: production
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Log in to the Container registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}
            type=sha

      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Deploy to Production
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USERNAME }}
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            export TAG=${GITHUB_REF#refs/tags/}
            docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:$TAG
            docker-compose -f docker-compose.prod.yml up -d
```

## Docker Registry Setup

### GitHub Container Registry Setup
```bash
# Login to GitHub Container Registry
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Build and tag image
docker build -t ghcr.io/username/repo-name:tag .

# Push image
docker push ghcr.io/username/repo-name:tag
```

### Docker Hub Alternative
```yaml
# Add to GitHub Actions workflow
- name: Login to Docker Hub
  uses: docker/login-action@v2
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}
```

## Deployment Workflows

### Deployment Script
```bash
#!/bin/bash
# scripts/deploy.sh

set -e

# Get deployment environment
ENV=$1
TAG=$2

# Load environment variables
source .env.$ENV

# Pull latest image
docker pull $REGISTRY/$IMAGE_NAME:$TAG

# Deploy using docker-compose
docker-compose -f docker-compose.$ENV.yml up -d

# Run database migrations if needed
docker-compose -f docker-compose.$ENV.yml exec app npm run migrate

# Health check
./scripts/health-check.sh
```

### Health Check Script
```bash
#!/bin/bash
# scripts/health-check.sh

MAX_RETRIES=30
RETRY_INTERVAL=10

check_health() {
    curl -f http://localhost:3000/api/health
}

for i in $(seq 1 $MAX_RETRIES); do
    if check_health; then
        echo "Service is healthy"
        exit 0
    fi
    echo "Health check failed, retry $i/$MAX_RETRIES"
    sleep $RETRY_INTERVAL
done

echo "Service failed to become healthy"
exit 1
```

## Environment Management

### GitHub Secrets Configuration
```yaml
# Required secrets for each environment
GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
DEV_HOST: "dev.example.com"
DEV_USERNAME: "deploy"
DEV_SSH_KEY: ${{ secrets.DEV_SSH_KEY }}
STAGING_HOST: "staging.example.com"
STAGING_USERNAME: "deploy"
STAGING_SSH_KEY: ${{ secrets.STAGING_SSH_KEY }}
PROD_HOST: "prod.example.com"
PROD_USERNAME: "deploy"
PROD_SSH_KEY: ${{ secrets.PROD_SSH_KEY }}
```

### Environment Variables
```yaml
# .github/workflows/environments/prod/.env.prod
NODE_ENV=production
DATABASE_URL=postgresql://user:pass@host:5432/db
REDIS_URL=redis://redis:6379
API_KEY=your-api-key
```

## Security Best Practices

### Docker Security
```dockerfile
# Dockerfile security best practices
FROM node:18-alpine AS builder
# Use non-root user
USER node
# Set proper permissions
RUN chown -R node:node /app
# Use specific versions
COPY --chown=node:node package*.json ./
# Scan for vulnerabilities
RUN npm audit
```

### GitHub Actions Security
```yaml
# Add security scanning
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - name: Run Snyk to check for vulnerabilities
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          vuln-type: 'os,library'
          severity: 'CRITICAL,HIGH'
```

### Access Control
```yaml
# Branch protection rules
branches:
  - name: main
    protection:
      required_status_checks:
        strict: true
        contexts: ["build", "test", "security"]
      required_pull_request_reviews:
        required_approving_review_count: 2
      enforce_admins: true
```

## Monitoring and Notifications

### Slack Notifications
```yaml
# Add to workflows
- name: Notify Slack
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    fields: repo,message,commit,author,action,eventName,ref,workflow
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
  if: always()
```

### Deployment Tracking
```yaml
# Add deployment tracking
- name: Create GitHub Deployment
  uses: chrnorm/deployment-action@v2
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
    environment: production
    ref: ${{ github.ref }}
```

## Next Steps
- Set up automated rollbacks
- Implement blue-green deployments
- Configure performance monitoring
- Set up automated testing
- Implement security scanning

## Complete CI/CD Pipeline

```mermaid
graph TB
    subgraph "Source Control"
        C[Commit] -->|trigger| PR[Pull Request]
        PR -->|merge| M[Main Branch]
    end
    
    subgraph "CI Pipeline"
        M -->|trigger| L[Lint Code]
        L -->|pass| T[Unit Tests]
        T -->|pass| S[Security Scan]
        S -->|pass| B[Build Image]
        
        B -->|success| PT[Performance Tests]
        PT -->|pass| IT[Integration Tests]
        IT -->|pass| ST[Staging Deploy]
    end
    
    subgraph "CD Pipeline"
        ST -->|manual approve| PD[Production Deploy]
        PD -->|deploy| BG[Blue/Green Deploy]
        
        BG -->|switch| BLUE[Blue Environment]
        BG -->|switch| GREEN[Green Environment]
        
        BLUE -->|health check| HC1[Health Check]
        GREEN -->|health check| HC2[Health Check]
        
        HC1 -->|pass| DONE1[Complete]
        HC2 -->|pass| DONE2[Complete]
    end
    
    style C fill:#f96,stroke:#333
    style M fill:#9cf,stroke:#333
    style B fill:#9f9,stroke:#333
    style PD fill:#ff9,stroke:#333
    style BLUE fill:#9cf,stroke:#333
    style GREEN fill:#9f9,stroke:#333
```

## Environment Promotion

```mermaid
stateDiagram-v2
    [*] --> Development
    Development --> Testing: Automated Tests Pass
    Testing --> Staging: Integration Tests Pass
    Staging --> Production: Manual Approval
    
    state Testing {
        [*] --> UnitTests
        UnitTests --> IntegrationTests
        IntegrationTests --> SecurityTests
        SecurityTests --> [*]
    }
    
    state Staging {
        [*] --> Deploy
        Deploy --> Smoke
        Smoke --> Performance
        Performance --> [*]
    }
    
    state Production {
        [*] --> BlueGreen
        BlueGreen --> HealthCheck
        HealthCheck --> Monitoring
        Monitoring --> [*]
    }
```

## Rollback Process

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant Git as GitHub
    participant Reg as Registry
    participant Prod as Production
    participant Mon as Monitoring
    
    Mon->>Prod: Detect Issue
    Prod->>Dev: Alert Team
    Dev->>Git: Trigger Rollback
    Git->>Reg: Pull Previous Image
    Reg->>Prod: Deploy Previous Version
    
    alt Rollback Successful
        Prod->>Mon: Verify Health
        Mon->>Dev: Confirm Recovery
    else Rollback Failed
        Prod->>Dev: Emergency Alert
        Dev->>Prod: Manual Intervention
    end
```

---
*This guide will be updated with more detailed sections as we progress.* 