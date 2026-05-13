# Vprofile GitOps — EKS CI/CD Pipeline

End-to-end GitOps project that provisions an EKS cluster with Terraform and deploys a Java web application to it via a GitHub Actions CI/CD pipeline.

The application source is based on [hkhcoder/vprofile-project](https://github.com/hkhcoder/vprofile-project).

---

## Repositories

| Repo | Purpose |
|---|---|
| [`iac-vprofile`](https://github.com/SalmaEasa/iac-vprofile) | Terraform IaC — provisions VPC, EKS cluster, and nginx ingress |
| [`vprofile-action`](https://github.com/SalmaEasa/vprofile-action) | App CI/CD — test, build Docker image, push to ECR, deploy via Helm |

---

## Architecture Overview

```
GitHub Actions (iac-vprofile)
        │
        ▼
  Terraform ──► AWS VPC (172.20.0.0/16)
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

### S3 Terraform State Backend
![S3 bucket storing Terraform remote state](./demo%26screenshots/s3.png)

### VPC & Subnets
![VPC created with public and private subnets](./demo%26screenshots/vpc%20created%20and%20subnets.png)

### NAT Gateway
![NAT gateway for private subnet outbound traffic](./demo%26screenshots/nat%20gateway.png)

### EKS Cluster
![EKS cluster provisioned and active](./demo%26screenshots/eks%20cluster.png)

### Node Groups
![EKS managed node groups](./demo%26screenshots/node%20groups.png)

### Nodes
![Worker nodes running in the cluster](./demo%26screenshots/nodes.png)

### Load Balancer
![AWS Load Balancer created by nginx ingress controller](./demo%26screenshots/load_balancer.png)

### Route 53 Record
![Route 53 DNS record pointing to the load balancer](./demo%26screenshots/route53%20record.png)

### SonarCloud Analysis
![SonarCloud code quality scan results](./demo%26screenshots/sonarcloud.png)
