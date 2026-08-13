# EKS Platform with GitOps Automation

A production-ready Kubernetes cluster on AWS EKS with complete automation, GitOps, monitoring, and security scanning.

## Overview

This project deploys a containerized 2048 game application on AWS EKS with:

- ✅ Fully automated infrastructure deployment (~15 minutes)
- ✅ GitOps with ArgoCD
- ✅ Automatic SSL certificates (Let's Encrypt)
- ✅ Automatic DNS management (External-DNS)
- ✅ Prometheus + Grafana monitoring
- ✅ Security scanning (Checkov + Trivy) with results published to the GitHub Security tab
- ✅ OIDC federation for CI/CD and IRSA for workloads — no long-lived AWS credentials anywhere

**Ephemeral by design.** The platform stands up from one workflow and tears down from another, so it runs only when needed rather than sitting idle. The public endpoints below are usually down for that reason.

- Application: https://eks.ahmedmhcodelab.click
- ArgoCD: https://argocd.ahmedmhcodelab.click
- Grafana: https://grafana.ahmedmhcodelab.click
- Prometheus: https://prometheus.ahmedmhcodelab.click

## Architecture

![EKS Architecture Diagram](meta/EKS%20Diagram.png)

**Infrastructure:**
- AWS EKS
- 2 worker nodes (t3.medium) across 2 availability zones
- VPC with public/private subnets
- Route53 DNS + ECR registry
- Terraform state in S3 + DynamoDB

**Platform Services:**
- NGINX Ingress Controller
- Cert-Manager (SSL certificates)
- External-DNS (Route53 automation)
- ArgoCD (GitOps)
- Prometheus + Grafana (monitoring)

## Authentication model

No AWS access keys exist in this repository, in GitHub Secrets, or in Terraform state.

**CI/CD → AWS.** GitHub Actions requests a short-lived token from GitHub's OIDC issuer and exchanges it for temporary credentials by assuming an IAM role. The role's trust policy is scoped to this repository through the `sub` claim, so no other repo can assume it. Provider and role are defined in `Terraform/github-oidc.tf`.

**Workloads → AWS.** In-cluster workloads use IRSA. A service account is annotated with an IAM role and the pod receives a projected service account token, exchanged for temporary credentials at runtime. Cert-Manager, External-DNS, and Cluster Autoscaler each get their own role rather than sharing node permissions. No credentials are mounted into containers.

## Quick Start

### Prerequisites
- AWS CLI v2
- Terraform >= 1.0
- kubectl
- Helm >= 3.0
- AWS account with Route53 hosted zone

### Setup

1. **Configure Terraform backend:**
   ```bash
   ./setup-backend.sh
   # Update Terraform/backend.tf with output values
   ```

2. **Bootstrap the OIDC role:**

   `github-oidc.tf` creates the role that CI later assumes, so the first apply runs locally with your own credentials. Every run after that authenticates through OIDC.

   ```bash
   cd Terraform
   terraform init
   terraform apply -target=aws_iam_openid_connect_provider.github_actions \
                   -target=aws_iam_role.github_actions
   terraform output github_actions_role_arn
   ```

   Store the ARN as the repository secret `AWS_ROLE_ARN`.

3. **Set variables in `Terraform/terraform.tfvars`:**

   Copy from the example and fill in your own values. This file is gitignored and should never be committed — it carries your account ID and hosted zone ID.

   ```hcl
   project_name    = "EKSDeployment"
   aws_account_id  = "YOUR_ACCOUNT_ID"
   route53_zone_id = "YOUR_HOSTED_ZONE_ID"

   # Restrict the EKS API endpoint to your own address.
   # 0.0.0.0/0 exposes the control plane to the internet.
   cluster_endpoint_public_access_cidrs = ["YOUR_IP/32"]
   ```

4. **Deploy via GitHub Actions:**
   - Run "Setup Infrastructure" workflow
   - Input: `CREATE`
   - Wait ~15 minutes

5. **Access services:**
   - Credentials will be displayed in workflow output

## Project Structure

```
EKS-Deployment/
├── .github/workflows/    # CI/CD pipelines
├── Terraform/            # Infrastructure as Code
│   └── modules/          # VPC, EKS, ECR modules
├── helm-values/          # Helm chart configurations
├── k8s/                  # Kubernetes manifests
├── argocd-apps/          # ArgoCD applications
└── Dockerfile            # Application container
```

## GitOps Workflow

1. Push code to GitHub
2. GitHub Actions builds and scans Docker image
3. Image pushed to ECR with new tag
4. Kubernetes manifests updated
5. ArgoCD detects changes and syncs cluster
6. Rolling update applied to pods

![ArgoCD Dashboard](meta/argocd.png)
*ArgoCD automatically syncs and deploys application changes*

## Cleanup

**Via GitHub Actions (recommended):**
```
Run "Cleanup" workflow → Input: DESTROY → Wait ~10 minutes
```

**Manually:**
```bash
cd Terraform
terraform destroy -auto-approve
```

## Common Commands

```bash
# Cluster status
kubectl get nodes
kubectl get pods -A

# Get ArgoCD password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Get Grafana password
kubectl get secret -n monitoring prometheus-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d

# Check certificates
kubectl get certificate -A

# View logs
kubectl logs -f <pod-name> -n <namespace>
```

## Troubleshooting

- **Pods stuck pending:** Check node capacity with `kubectl describe node`
- **Certificates not issuing:** Verify cert-manager logs and DNS records
- **Ingress 503 errors:** Check service endpoints with `kubectl get endpoints`
- **Terraform destroy fails:** Manually delete load balancers first
- **`AssumeRoleWithWebIdentity` denied in CI:** confirm the workflow sets `permissions: id-token: write` and that the role trust policy's `sub` condition matches this repository

## Documentation

For detailed documentation, see:
- [Detailed Documentation](https://brassy-coast-e1c.notion.site/Documentation-EKS-Project-2a1ae71e63408099b477f0d5c4ce3594)

## Built By

[Ahmed](https://github.com/AhmedMHCodeLab)
