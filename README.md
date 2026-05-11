<h1>This Projects shows how an end to end github actions workflow file looks like for a maven project:</h1>

The pipeliene includes building, testing, Semgrep(SAST, SCA,Secret Scanning) , trivy(container scanning), Dependency Track for sbom


Maven + AWS ECR + Trivy + Semgrep + Argo CD GitOps

This pipeline automates:

Build & test Java (Maven)<br>
Security scanning (SAST + container scanning)<br>
Docker image build & push to AWS ECR<br>
SBOM generation<br>
GitOps deployment via updated values.yaml files (triggering Argo CD)<br>
🌿 Branch Strategy Overview<br>
Branch	Purpose	What happens<br>
feat/**	Feature development branches	Full CI + Dev deployment<br>
main	Production branch	Full CI + Staging + Production pipeline<br>
Pull Requests → main	Code validation	Build + tests + security scans only<br>
🔄 High-Level Pipeline Flow<br>
Push / PR<br>
   ↓<br>
Build & Test (Maven)<br>
   ↓<br>
Semgrep SAST Scan<br>
   ↓<br>
Docker Build & Push (ECR)<br>
   ↓<br>
Trivy Container Scan + SBOM<br>
   ↓<br>
Dependency Track Upload (main only)<br>
   ↓<br>
Update values.yaml (GitOps trigger)<br>
   ↓<br>
Argo CD deploys to Kubernetes<br>


🧱 1. Build & Test Job

📌 Job: build-test<br>
Runs on:<br>
feat/** branches<br>
main<br>
Pull Requests targeting main<br>
What it does:<br>
Checks out code<br>
Caches Maven dependencies<br>
Runs:<br>
Unit tests<br>
Integration tests<br>
Uploads test reports as artifacts<br>
Output:<br>
Verified build artifacts<br>
Test reports -- unit test reports and Code coverage report using Jacoco plugin<br>

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

Uploads SBOM artifact<br>
📦 5. Dependency Tracking Upload<br>

Tool: OWASP Dependency-Track<br>

📌 Job: dependency_track<br>
Runs only on: main branch <br>
Depends on: trivy<br>
What it does: <br>
Downloads SBOM artifact<br>
Uploads it to Dependency Track server<br>
Registers project version as branch name<br>

☁️ 6. Dev Deployment (GitOps Trigger)

📌 Job: deploy-dev<br>
Runs on:<br>
❗ NOT main<br>
Only feature branches (feat/**)<br>
Depends on:docker-build-push<br>
What it does:<br>
Updates values-dev.yaml with new image tag<br>
Commits change to repo<br>
Pushes back to branch<br>
Result: Triggers Argo CD sync for Dev environment<br>

🧪 7. Staging Deployment

📌 Job: deploy-staging
Runs on:<br>
main branch only<br>
Depends on: docker-build-push <br>
What it does: <br>
Updates values-stage.yaml<br>
Commits and pushes change<br>
Triggers Argo CD staging deployment<br>
Requires:<br>
Typically manual approval in GitHub Environments<br>

🚀 8. Production Deployment

📌 Job: deploy-prod<br>
Runs on:<br>
main branch only<br>
Depends on: docker-build-push<br>
What it does:<br>
Updates values-prod.yaml<br>
Commits and pushes changes<br>
Triggers Argo CD production deployment <br>
Requires:<br>
Approval via GitHub Environments (recommended)<br>

Important Notes:<br>

1. GitOps model used here<br>

Instead of deploying directly to Kubernetes, this pipeline:<br>

updates YAML files<br>
commits them to Git<br>

Then:<br>

👉 Argo CD detects changes and deploys automatically.<br>

2. Branch behavior summary

🔹 Feature branches (feat/**)<br>
Build & test<br>
Security scan<br>
Docker build<br>
Dev deployment only<br>
🔹 Main branch (main)<br>
Full CI pipeline<br>
Security scans<br>
Docker build & push<br>
Staging deployment<br>
Production deployment<br>
🔹 Pull Requests → main<br>
Build & test only<br>
Semgrep scan<br>
No deployments<br>

🧠 3. Key Architecture Idea<br>

This pipeline follows:<br>

CI → Security → Build → Image → GitOps → Argo CD Deploy<br>

Developer Commit<br>
      ↓
GitHub Actions CI<br>
  ├─ Build + Test (Maven)<br>
  ├─ Semgrep SAST,SCA,Secret scanning<br>
  ├─ Docker Build<br>
  ├─ Push to ECR<br>
  ├─ Trivy Scan + SBOM<br>
      ↓<br>
Update values.yaml (GitOps repo state)<br>
      ↓<br>
Git Commit Push<br>
      ↓
Argo CD detects change<br>
      ↓<br>
Kubernetes Deployment (Dev / Stage / Prod)<br>

