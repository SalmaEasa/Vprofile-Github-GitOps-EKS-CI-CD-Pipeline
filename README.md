# Vprofile GitOps — EKS CI/CD Pipeline

End-to-end GitOps project that provisions an EKS cluster with Terraform and deploys a Java web application to it via a GitHub Actions CI/CD pipeline.

The application source is based on [hkhcoder/vprofile-project](https://github.com/hkhcoder/vprofile-project).

---

## Architecture Diagram

![Architecture Diagram](./demo%26screenshots/arch_diagram.png)

---

## Architecture Overview

```
GitHub Actions (iac-vprofile)
        │
        ▼
  Terraform ──► AWS VPC (172.20.0.0/16)
                    │
                    ├── Public Subnets  ──► NAT Gateway + Load Balancer
                    └── Private Subnets ──► EKS Worker Nodes
                              │
                              ▼
                        EKS Cluster (vprofile-eks)
                        ├── node-group-1 (t3.small, 1–3 nodes)
                        └── node-group-2 (t3.small, 1–2 nodes)

GitHub Actions (vprofile-action)
        │
        ├── Maven Test + Checkstyle + SonarQube Scan
        ├── Docker Build ──► Amazon ECR
        └── Helm Deploy ──► EKS (nginx ingress → vprofile.salma-app.site)
```

**Application stack:** Spring MVC · Spring Security · MySQL · Memcached · RabbitMQ · Elasticsearch  
**Exposed via:** nginx ingress controller → AWS Load Balancer → Route 53

---

## Repositories

| Repo | Purpose |
|---|---|
| [`iac-vprofile`](https://github.com/SalmaEasa/iac-vprofile) | Terraform IaC — provisions VPC, EKS cluster, and nginx ingress |
| [`vprofile-action`](https://github.com/SalmaEasa/vprofile-action) | App CI/CD — test, build Docker image, push to ECR, deploy via Helm |

---

## How It Works

### 1. Infrastructure (iac-vprofile)

Pushing to the `main` branch triggers the Terraform workflow which:

1. Initialises Terraform with an S3 remote state backend
2. Runs `fmt`, `validate`, and `plan`
3. Applies the plan — creates the VPC, subnets, NAT gateway, and EKS cluster
4. Updates `kubeconfig` for the new cluster
5. Installs the nginx ingress controller via `kubectl`

### 2. Application (vprofile-action)

Triggered manually (or on push). Three sequential jobs:

| Job | Steps |
|---|---|
| `test` | Maven unit tests → Checkstyle → SonarQube scan |
| `build-and-publish` | Docker multi-stage build → push to ECR (`latest` + run number tag) |
| `deploy` | Update kubeconfig → create ECR pull secret → Helm deploy to EKS |

---

## Secrets Required

Set these in each repo's GitHub Actions secrets:

| Secret | Repo |
|---|---|
| `AWS_ACCESS_KEY_ID` | Both |
| `AWS_SECRET_ACCESS_KEY` | Both |
| `BUCKET_TF_STATE` | `iac-vprofile` |
| `REGISTRY` | `vprofile-action` |
| `SONAR_URL` | `vprofile-action` |
| `SONAR_TOKEN` | `vprofile-action` |
| `SONAR_ORGANIZATION` | `vprofile-action` |
| `SONAR_PROJECT_KEY` | `vprofile-action` |

---

## Demo & Screenshots

### Live App Demo
![Vprofile app demo](./demo%26screenshots/demo.gif)

---

### S3 Terraform State Backend
Terraform state is stored remotely in S3, enabling safe concurrent runs and state locking.

![S3 bucket storing Terraform remote state](./demo%26screenshots/s3.png)

---

### VPC & Subnets
The VPC (`172.20.0.0/16`) is split into public and private subnets across 3 availability zones. Worker nodes live in private subnets and are never directly exposed to the internet. The public subnets host the NAT Gateway and the AWS Load Balancer.

![VPC created with public and private subnets](./demo%26screenshots/vpc%20created%20and%20subnets.png)

---

### NAT Gateway
A single NAT Gateway in the public subnet gives worker nodes outbound internet access (for pulling images, reaching the EKS control plane, etc.) without exposing them inbound.

![NAT gateway for private subnet outbound traffic](./demo%26screenshots/nat%20gateway.png)

---

### EKS Cluster
The `vprofile-eks` cluster running Kubernetes 1.32, provisioned via the `terraform-aws-modules/eks` module.

![EKS cluster provisioned and active](./demo%26screenshots/eks%20cluster.png)

---

### Node Groups
Two managed node groups using `t3.small` instances. Node Group 1 scales 1–3 nodes, Node Group 2 scales 1–2 nodes.

![EKS managed node groups](./demo%26screenshots/node%20groups.png)

---

### Nodes
Worker nodes running inside the private subnets, registered and ready in the cluster.

![Worker nodes running in the cluster](./demo%26screenshots/nodes.png)

---

### Load Balancer
The nginx ingress controller automatically provisions an AWS Load Balancer in the public subnet to handle inbound traffic and route it to the worker nodes.

![AWS Load Balancer created by nginx ingress controller](./demo%26screenshots/load_balancer.png)

---

### Route 53 Record
A Route 53 DNS record points `vprofile.salma-app.site` to the Load Balancer, making the app publicly accessible.

![Route 53 DNS record pointing to the load balancer](./demo%26screenshots/route53%20record.png)

---

### SonarCloud Analysis
Every pipeline run triggers a SonarCloud scan for code quality, test coverage, and Checkstyle violations.

![SonarCloud code quality scan results](./demo%26screenshots/sonarcloud.png)
