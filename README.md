

# **CI-DevSecOps: Secure GitHub Actions CI/CD with AWS ECR + OIDC, Trivy & SonarQube**

This repository demonstrates a **production-grade DevSecOps pipeline** using **GitHub Actions**, **AWS Elastic Container Registry (ECR)**, and **OpenID Connect (OIDC)** for secure authentication. The pipeline:

*   Builds a Docker image from your application.
*   Authenticates to AWS using **OIDC** (no long-lived keys).
*   Scans for vulnerabilities using **Trivy**.
*   Uploads SARIF results to **GitHub Code Scanning**.
*   Pushes the image to **AWS ECR** only if the scan passes.
*   Optionally runs **tests** and **SonarQube** analysis for code quality/security.

***

## ✅ **Why These Choices?**

### **Amazon ECR**

A managed, private container registry in AWS. It integrates with ECS/EKS and supports IAM-based access control, encryption, lifecycle policies, and private networking.

### **GitHub OIDC → AWS STS**

OIDC allows GitHub Actions to exchange a short-lived token for AWS STS credentials **at runtime**, eliminating hard-coded AWS keys and improving security.

### **Trivy + SARIF**

Trivy scans OS packages and libraries in your image. **Failing on HIGH/CRITICAL vulnerabilities** prevents insecure images from being published. SARIF uploads make results visible under **Security → Code scanning alerts**.

***

## ✅ **Prerequisites**

Before you run the CI/CD pipelines, make sure the following are set up:

### 1) AWS Account & ECR

*   An active **AWS account** (sandbox is fine).
*   An **ECR private repository** created:
    *   **Region**: `<REGION>` (e.g., `us-east-1`)
    *   **Repository name**: `<ECR_REPO>` (e.g., `fastapi-ci`)

### 2) GitHub OIDC → AWS IAM

*   **OIDC Identity Provider** in IAM:
    *   **Provider URL**: `https://token.actions.githubusercontent.com`
    *   **Audience**: `sts.amazonaws.com`
*   **IAM Role** (e.g., `GitHubActions_ECR_Push`) with **trust policy** scoped to your repo/branch:
    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {
            "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
          },
          "Action": "sts:AssumeRoleWithWebIdentity",
          "Condition": {
            "StringEquals": {
              "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
              "token.actions.githubusercontent.com:sub": "repo:<ORG>/<REPO>:ref:refs/heads/main"
            }
          }
        }
      ]
    }
    ```
*   **Permissions attached** to the role:
    *   Registry auth (**required**):
        ```json
        { "Effect": "Allow", "Action": ["ecr:GetAuthorizationToken"], "Resource": "*" }
        ```
    *   Repo-scoped **push** permissions:
        ```json
        {
          "Effect": "Allow",
          "Action": [
            "ecr:BatchCheckLayerAvailability",
            "ecr:InitiateLayerUpload",
            "ecr:UploadLayerPart",
            "ecr:CompleteLayerUpload",
            "ecr:PutImage",
            "ecr:BatchGetImage"
          ],
          "Resource": "arn:aws:ecr:<REGION>:<ACCOUNT_ID>:repository/<ECR_REPO>"
        }
        ```

### 3) GitHub Repository Settings

Go to **Settings → Secrets and variables → Actions**:

*   **Secrets** (encrypted):
    *   `AWS_ACCOUNT_ID` = `<ACCOUNT_ID>`
    *   *(Optional)* `SONARQUBE_TOKEN` if you use the SonarQube workflow
*   **Variables** (non-sensitive):
    *   `AWS_REGION` = `<REGION>`
    *   `ECR_REPOSITORY` = `<ECR_REPO>`

### 4) Code & Build Files

*   A valid **Dockerfile** at the repo root (or update workflow paths accordingly).
*   Your application code and any test configs (e.g., `requirements.txt`, `pytest` setup).
*   *(Optional)* `sonar-project.properties` if you run the SonarQube job.

### 5) Workflow Files

Create the following files:

*   `.github/workflows/ci.yml` → tests + SonarQube + Trivy (filesystem)
*   `.github/workflows/ecr.yml` → OIDC → build → Trivy → SARIF → push to ECR

***

## ✅ **Architecture Overview**

    Developer push → GitHub Actions (OIDC token)
        │
        ├─ aws-actions/configure-aws-credentials → STS AssumeRole (GitHubActions_ECR_Push)
        │        (scoped via trust policy to repo:<ORG>/<REPO>:ref:refs/heads/main)
        │
        ├─ Login to ECR (temporary creds)
        ├─ docker build
        ├─ trivy scan (gate)
        ├─ (always) trivy SARIF → Upload to GitHub Code Scanning
        └─ docker push (only if gate passes)

***

## ✅ **CI Pipelines**

### **Pipeline 1: Tests + SonarQube + Trivy (Filesystem Scan)**

Path: `.github/workflows/ci.yml`

*(Insert full YAML here — already provided in worflows.)*

***

### **Pipeline 2: ECR + OIDC: Build → Trivy → SARIF → Push**

Path: `.github/workflows/ecr.yml`

*(Insert full YAML here — already provided in worflows.)*

***

## ✅ **How to Run**

1.  Add secrets and variables in GitHub.
2.  Commit both workflow files under `.github/workflows/`.
3.  Push to `main` or trigger manually via **Actions → Run workflow**.

***

## ✅ **Verification**

*   Check **Actions logs** for:
    *   `Assumed role arn:aws:iam::<ACCOUNT_ID>:role/GitHubActions_ECR_Push`
    *   `Login Succeeded`
*   Check **AWS ECR** for image tag.
*   Check **GitHub Security → Code scanning alerts** for SARIF results.

***

## ✅ **Troubleshooting**

*   **OIDC errors**: Ensure `id-token: write` and trust policy matches repo/branch.
*   **Push denied**: Confirm ECR permissions and correct ARN.
*   **SARIF not visible**: Ensure `security-events: write` permission.

***




