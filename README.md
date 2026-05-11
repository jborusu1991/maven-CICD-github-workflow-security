<h1>This Projects shows how an end to end github actions workflow file looks like for a maven project:</h1>

The pipeliene includes building, testing, Semgrep(SAST, SCA,Secret Scanning) , trivy(container scanning), Dependency Track for sbom


Maven + AWS ECR + Trivy + Semgrep + Argo CD GitOps

This pipeline automates:

Build & test Java (Maven)
Security scanning (SAST + container scanning)
Docker image build & push to AWS ECR
SBOM generation
GitOps deployment via updated values.yaml files (triggering Argo CD)
🌿 Branch Strategy Overview
Branch	Purpose	What happens
feat/**	Feature development branches	Full CI + Dev deployment
main	Production branch	Full CI + Staging + Production pipeline
Pull Requests → main	Code validation	Build + tests + security scans only
🔄 High-Level Pipeline Flow
Push / PR
   ↓
Build & Test (Maven)
   ↓
Semgrep SAST Scan
   ↓
Docker Build & Push (ECR)
   ↓
Trivy Container Scan + SBOM
   ↓
Dependency Track Upload (main only)
   ↓
Update values.yaml (GitOps trigger)
   ↓
Argo CD deploys to Kubernetes
🧱 1. Build & Test Job
📌 Job: build-test
Runs on:
feat/** branches
main
Pull Requests targeting main
What it does:
Checks out code
Caches Maven dependencies
Runs:
Unit tests
Integration tests
Uploads test reports as artifacts
Output:
Verified build artifacts
Test reports -- unit test reports and Code coverage report using Jacoco plugin

🔍 2. Semgrep Security Scan

Tool: Semgrep

📌 Job: semgrep
Runs on:
feat/** branches → full scan (via PR or push)
main branch → full scan

What it does:
SAST scan (code vulnerabilities)
Secret detection
Supply chain checks

Depends on:
build-test must pass first

🐳 3. Docker Build & Push to ECR

Tool: Amazon Elastic Container Registry

📌 Job: docker-build-push
Runs on:
feat/**
main

Depends on:
build-test
semgrep

What it does:
Logs into AWS using OIDC
Builds Docker image (multi-arch)
Tags image with short Git SHA
Pushes image to ECR

Output:
Docker image stored in ECR
Image URI exported for downstream jobs

🔐 4. Container Security Scan

Tool: Trivy

📌 Job: trivy
Runs on:
feat/**
main

Depends on:
docker-build-push

What it does:
Scans container image for vulnerabilities
Generates SBOM (CycloneDX format)
Enforces security gate:
Fails pipeline if HIGH/CRITICAL vulnerabilities found

Uploads SBOM artifact
📦 5. Dependency Tracking Upload

Tool: OWASP Dependency-Track

📌 Job: dependency_track
Runs only on:
main branch
Depends on:
trivy
What it does:
Downloads SBOM artifact
Uploads it to Dependency Track server
Registers project version as branch name

☁️ 6. Dev Deployment (GitOps Trigger)
📌 Job: deploy-dev
Runs on:
❗ NOT main
Only feature branches (feat/**)
Depends on:
docker-build-push
What it does:
Updates values-dev.yaml with new image tag
Commits change to repo
Pushes back to branch
Result:
Triggers Argo CD sync for Dev environment

🧪 7. Staging Deployment
📌 Job: deploy-staging
Runs on:
main branch only
Depends on:
docker-build-push
What it does:
Updates values-stage.yaml
Commits and pushes change
Triggers Argo CD staging deployment
Requires:
Typically manual approval in GitHub Environments

🚀 8. Production Deployment
📌 Job: deploy-prod
Runs on:
main branch only
Depends on:
docker-build-push
What it does:
Updates values-prod.yaml
Commits and pushes changes
Triggers Argo CD production deployment
Requires:
Approval via GitHub Environments (recommended)
⚠️ Important Notes (Beginner-friendly)
1. GitOps model used here

Instead of deploying directly to Kubernetes, this pipeline:

updates YAML files
commits them to Git

Then:

👉 Argo CD detects changes and deploys automatically.

2. Branch behavior summary
🔹 Feature branches (feat/**)
Build & test
Security scan
Docker build
Dev deployment only
🔹 Main branch (main)
Full CI pipeline
Security scans
Docker build & push
Staging deployment
Production deployment
🔹 Pull Requests → main
Build & test only
Semgrep scan
No deployments
🧠 3. Key Architecture Idea

This pipeline follows:

CI → Security → Build → Image → GitOps → Argo CD Deploy

Developer Commit
      ↓
GitHub Actions CI
  ├─ Build + Test (Maven)
  ├─ Semgrep SAST
  ├─ Docker Build
  ├─ Push to ECR
  ├─ Trivy Scan + SBOM
      ↓
Update values.yaml (GitOps repo state)
      ↓
Git Commit Push
      ↓
Argo CD detects change
      ↓
Kubernetes Deployment (Dev / Stage / Prod)

