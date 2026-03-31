# Level 1 — Simple CI/CD Pipeline: Deploy to S3

## Overview

This repository demonstrates a **Level 1 simple CI/CD pipeline** using **GitHub Actions** and **AWS**.  
The pipeline automatically deploys the contents of this repository to an **S3 bucket** whenever changes are pushed to the `main` branch.

We also upgraded the pipeline to use **OIDC (OpenID Connect)** for AWS access instead of storing AWS keys as secrets. This is considered a **best practice** for production-ready pipelines.

---

## What We Have Done

1. **Created an IAM Role in AWS for GitHub OIDC**
   - Trusted entity: **Web Identity (OIDC)**
   - Trust policy restricted to:
     - Our GitHub repository
     - Branch `main`
   - Attached an S3 access policy for deployment.

2. **Configured GitHub Repository**
   - Enabled **OIDC tokens** for GitHub Actions:
     - Settings → Actions → General → Workflow permissions  
     - Enabled `Read and write permissions` and `Allow GitHub Actions to create and approve OIDC tokens`
   - Added GitHub Secrets:
     - `AWS_ROLE_ARN` → ARN of the IAM Role
     - `AWS_REGION` → AWS region for the S3 bucket
     - `S3_BUCKET_NAME` → Target bucket name

3. **Created GitHub Actions Workflow**
   - Path: `.github/workflows/deploy.yml`
   - The workflow triggers **on push to main branch**
   - Workflow Steps:
     1. **Checkout repository code**
        ```yaml
        - uses: actions/checkout@v3
        ```
     2. **Configure AWS via OIDC**
        ```yaml
        - uses: aws-actions/configure-aws-credentials@v2
          with:
            role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
            aws-region: ${{ secrets.AWS_REGION }}
        ```
        - GitHub automatically requests a short-lived OIDC token
        - AWS validates the token and allows the workflow to assume the IAM Role
     3. **Deploy repository contents to S3**
        ```yaml
        - run: aws s3 sync . s3://${{ secrets.S3_BUCKET_NAME }} --delete
        ```
        - Syncs all files from the repo to the S3 bucket
        - Deletes files in the bucket that no longer exist in the repo

---

## How to Use

1. Clone this repository.
2. Push changes to the `main` branch.
3. The workflow will run automatically and deploy the files to the configured S3 bucket.
4. Monitor the progress in **GitHub Actions → deploy job**.

---

## Key Notes

- **Security**: No AWS access keys are stored in GitHub. Only a short-lived OIDC token is used.
- **Branch restriction**: Only pushes to `main` can trigger AWS role assumption.
- **Next steps**: Add testing layers or additional environments (staging/production) in higher-level pipelines.

---

## Pipeline Summary

```text
Push → GitHub Actions (Checkout → AWS OIDC → Deploy to S3)


## Pipeline Flow Diagram

```mermaid
flowchart TD
    A[Push to main branch] --> B[GitHub Actions Workflow Starts]
    B --> C[Checkout Repository Code]
    C --> D[Request OIDC Token from GitHub]
    D --> E[AWS IAM Role validates OIDC Token]
    E --> F[Temporary AWS Credentials Granted]
    F --> G[Deploy Repository Files to S3 Bucket]
    G --> H[Deployment Complete]



### **Explanation of the Diagram**

1. **Push to main branch**  
   - Triggers the workflow in `.github/workflows/deploy.yml`.

2. **GitHub Actions Workflow Starts**  
   - Runner provisions an environment (`ubuntu-latest`) to run jobs.

3. **Checkout Repository Code**  
   - `actions/checkout@v3` retrieves your code so it can be deployed.

4. **Request OIDC Token from GitHub**  
   - GitHub generates a **short-lived OIDC token** for this workflow.

5. **AWS IAM Role validates OIDC Token**  
   - AWS checks the token against the IAM Role trust policy.
   - Only your repo + branch is allowed.

6. **Temporary AWS Credentials Granted**  
   - Workflow receives temporary credentials to access AWS services.

7. **Deploy Repository Files to S3 Bucket**  
   - `aws s3 sync` uploads files from the repo to the S3 bucket.

8. **Deployment Complete**  
   - Pipeline finishes; files in S3 are updated automatically.

---

This diagram clearly shows **the security flow** — GitHub issues a token, AWS verifies it, and no static secrets are needed.  

---
