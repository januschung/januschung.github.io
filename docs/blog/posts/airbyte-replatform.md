---
date: 2025-07-25
categories:
  - tailscale
  - airbyte
  - kubernetes
  - networking
  - devops
  - etl
  - helm
---
# Replatforming Airbyte: From Developer Laptop to EKS
##### originally posted at [LinkedIn](https://www.linkedin.com/pulse/replatforming-airbyte-from-developer-laptop-eks-janus-chung-550sc/) at July 25, 2025

In early-stage engineering teams, it's natural for tools to start out simple — often running on a single developer machine, just to get things moving. That’s how our Airbyte setup began: quick to spin up, good enough for testing connectors, and easy to iterate on.

But as our team grew and data pipelines became more embedded in how we operated, we knew it was time to treat Airbyte like real infrastructure. That meant moving beyond local environments and into a scalable, secure, and repeatable deployment.

We migrated Airbyte OSS to Amazon EKS, using Helm and AWS-native services like S3 and IAM Roles for Service Accounts (IRSA). Our goal wasn’t to fix something broken, but to build on what was working and make it production-ready—without sacrificing developer velocity.

This post shares how we did it, what we learned, and what you might want to consider if you’re operationalizing Airbyte (or any similar open-source tool) in a small but growing cloud-native team.

![DevOps Clown sending laptop application to the Cloud](../../assets/blog/airbyte-replatform/banner.jpg)

<!-- more -->
## 📦 From Local Dev to Platform Service

The starting point was a typical OSS install: Airbyte running via Docker Compose, with internal Postgres and MinIO providing storage.

For a single user, this setup works. But as sync jobs became more important — and more cloud-integrated — the limitations grew obvious:
- Only one person could run or inspect jobs
- Jobs stopped if the machine rebooted or slept
- The setup didn’t integrate cleanly with AWS services like KMS or IAM

We needed to make Airbyte a first-class internal service — which meant replatforming it onto our Kubernetes infrastructure.

## 🚀 Helm-First on EKS

We deployed Airbyte using the official Helm chart on Amazon EKS, with all configuration defined in a single `values.yaml` file.

This made it easy to:
- Version control the deployment
- Propagate changes to dev, stage, or prod clusters
- Configure each Airbyte component with its own replica count, resources, and ingress

We enabled the core stack:
- `webapp`, `server`, and `worker` components
- Internal Postgres with persistent volumes
- An Ingress controller (in our case, Tailscale) for secure access

All of this is managed by Helm, making deployment and upgrades consistent and CI/CD-friendly.

## ☁️ Why We Abandoned MinIO

MinIO ships by default in Airbyte’s OSS Helm chart — and it’s fine if you're storing state internally.

But as soon as we configured an S3 destination connector, we ran into a hard blocker.

Airbyte’s connector expects **actual AWS S3** — not a MinIO alias. The authentication flows, endpoint discovery, and SDK behavior are incompatible. The result? Jobs fail, and you lose the ability to use one of Airbyte’s most valuable features.

So we turned off MinIO and went all-in on real S3:
```yaml
global:
  storage:
    type: s3
    bucket:
      log: <BUCKET_NAME>
      state: <BUCKET_NAME>
      workloadOutput: <BUCKET_NAME>
    s3:
      region: <AWS_REGION>
      authenticationType: instanceProfile
minio:
  enabled: false
```

This cleared the path for both internal object storage and S3 destination connectors — no hacks required.

## 🔐 Evolving IAM: From Node Role to IRSA

To integrate Airbyte with S3 securely, we started with the simplest approach: attaching S3 access to the EKS node role. This allowed Airbyte to access S3 without static credentials—a good start, but too broad.

Later, we switched to **IAM Roles for Service Accounts (IRSA)** to scope permissions only to Airbyte pods.

We configured:
- An IRSA role (`airbyte-irsa-role-prod`) in Terraform.
- A trust policy bound to the `airbyte-admin` service account.
- A Helm annotation to bind that IAM role:
```yaml
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<AWS_ACCOUNT>:role/airbyte-irsa-role-prod
```

This let us remove reliance on the NodeInstanceRole and grant least-privilege S3 + KMS access directly to Airbyte.

## ⚙️ Tuning Resources for Job Efficiency

One thing that stood out during testing: **Airbyte jobs default to higher resource allocations than expected** — likely to support heavy syncs across large datasets.

But for our workloads, that was overkill.

We explicitly set `jobs.resources` to more modest requests and limits:
```yaml
global:
  jobs:
    resources:
      requests:
        memory: 256Mi
        cpu: 250m
      limits:
        memory: 1Gi
        cpu: 200m
```

This helped reduce noise in our Kubernetes scheduler, especially when running multiple concurrent jobs. If you're deploying Airbyte in an environment with resource quotas or shared node pools, this is a worthwhile tweak.

## 🧱 Stable, Auditable, Cloud-Integrated

With the final setup:
- Airbyte OSS runs fully inside EKS.
- Storage is backed by real S3, with KMS support.
- IAM is scoped via IRSA — no static secrets anywhere.
- Ingress is handled through our standard controller (Tailscale).
- PostgreSQL is deployed in-cluster with persistent volume claims.

It’s reproducible, secure, and fits neatly into how we manage infrastructure.

## ✅ Key Lessons for Platform Teams

If you're supporting internal data or integration teams:
- **MinIO doesn't work with S3 destination connectors** — we learned this the hard way and switched to real S3
- **Start simple with IAM**, then evolve to IRSA as your use case grows
- **Use Helm to its full extent** — the Airbyte chart is surprisingly complete
- **Tune job resources** — the defaults may be more than you need

By replatforming Airbyte this way, we made it stable, scalable, and production-ready — not just for one user, but for the whole company.
