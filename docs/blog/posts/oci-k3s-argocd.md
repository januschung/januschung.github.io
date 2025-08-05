---
date: 2025-07-31
categories:
  - devops
  - oci
  - terraform
  - argocd
  - ansible
  - docker
  - cicd
  - github
---
# The DevOps Odyssey, Part 3: GitOps on K3s with Argo CD — Self-Managing Infrastructure from a Single Commit
##### originally posted at [LinkedIn](https://www.linkedin.com/pulse/devops-odyssey-part-3-gitops-k3s-argo-cd-from-single-commit-chung-emmlc/) at July 31, 2025

In [Part 1](./2025/06/18/the-devops-odyssey-fully-automating-oci-app-deployment-with-terraform-ansible-and-docker/), we bootstrapped a zero-click deployment pipeline on OCI using Terraform, Ansible, and Docker Compose — complete with HTTPS, DNS, and CI/CD.

[Part 2](./2025/06/29/the-devops-odyssey-continues-evolving-from-docker-to-k3s-with-ansible/) evolved that foundation into a Kubernetes-native architecture, replacing Docker with K3s. That gave us a declarative control plane and a better foundation for future growth — without sacrificing simplicity or resource constraints.

Now, in **Part 3**, we finally bring in GitOps: managing the entire cluster from a Git repository using **Argo CD**. This marks the transition from automation to self-reconciliation — and sets the stage for horizontal scaling and federated identity in the next phase.

![Automation bots have evolved. What’s next?](../../assets/blog/oci-k3s-argocd/banner.jpg)

<!-- more -->

## A Quick Look Back: Where We Are

- ✅ One OCI Free Tier VM running K3s  
- ✅ Public-facing DNS via DuckDNS  
- ✅ Traefik 3 routing HTTPS traffic with automatic Let's Encrypt certs  
- ✅ Ansible roles to automate K3s provisioning

But until now, workloads still required Ansible to deploy. Changes were pushed, not pulled. That meant some infrastructure lived in code, but some drifted with time.

It was fast, but not fully convergent.


## Why GitOps?

GitOps introduces a shift in control: from pushing changes to letting the cluster **pull** and reconcile them.

That means:

- Cluster state always matches Git  
- Drift is detected and corrected automatically  
- Rollbacks are just Git reverts  
- CI becomes a pure code pipeline — no secrets, no cloud credentials  
- Disaster recovery is repeatable and fast  

For small teams or solo builders, it replaces tribal knowledge with source of truth. For larger efforts, it enforces consistency and control.

## Installing Argo CD with Helm (via Ansible)

We install Argo CD directly with Helm using an Ansible role, so we can keep cluster provisioning clean, repeatable, and decoupled from GitOps bootstrapping.

This avoids the "chicken and egg" problem: GitOps can’t install Argo CD if Argo CD isn’t already present. So the bootstrap is handled by Ansible — but only once.

```yaml
# ansible
- name: Install/Upgrade ArgoCD via Helm
  kubernetes.core.helm:
    name: argocd
    chart_ref: argo/argo-cd
    release_namespace: argocd
    create_namespace: true
    kubeconfig: "{{ kubeconfig }}"
    values_files:
      - /tmp/argocd-values.yaml
    wait: true
    wait_timeout: 600s
    update_repo_cache: true
```

```yaml
# argocd-values.yaml
configs:
  params:
    server.insecure: true
    server.rootpath: /argocd
    server.basehref: /argocd/

server:
  ingress:
    extraArgs:
      - --insecure
      - --rootpath=/argocd
      - --basehref=/argocd/
    enabled: true
    ingressClassName: traefik
    hostname: janusc.duckdns.org
    path: /argocd
    pathType: Prefix
    annotations:
      traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
      traefik.ingress.kubernetes.io/router.tls.certresolver: le
    tls: true
```

This also configures a public HTTPS endpoint, fronted by Traefik using DNS-validated certificates from Let's Encrypt.

## The Bootstrap Pattern: One Root Application to Rule Them All

Once Argo CD is up, everything else is declared in Git — starting with a **root application**.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bootstrap-argocd
  namespace: argocd
spec:
  project: default
  sources:
    - path: bootstrap
      repoURL: https://github.com/januschung/argocd
      targetRevision: HEAD
    - path: infra
      repoURL: https://github.com/januschung/argocd
      targetRevision: HEAD
      directory:
        recurse: true
        include: "{*application.yaml,*application-set.yaml}"
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

This single manifest creates a **fan-out** effect: every other service — including Argo CD itself — is defined as a child application within this repo.

The moment root-app syncs, it recursively applies the rest of the cluster configuration:

```
root/
├── ansible/                    # Infrastructure provisioning scripts (e.g. Argo CD bootstrap)
├── bootstrap/                 
│   └── root-application.yml   # The entry-point Argo CD Application that syncs everything else
├── infra/
│   ├── argocd/                # Helm chart and values for Argo CD (self-managed)
│   └── sealed-secrets/        # Placeholder for Sealed Secrets controller (to be utilitzed in Part 4)
└── future/                    # Staging ground for upcoming apps or services under GitOps control
```

Even new workloads or platform services follow the same pattern — just add a new folder and application YAML, commit, and walk away.

## Self-Management: Argo CD Manages Argo CD

We declare Argo CD’s own Helm chart as a child application — inside Git — so that any changes to its configuration are now tracked, versioned, and deployed via GitOps.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/januschung/argocd
    targetRevision: HEAD
    path: infra/argocd
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

What this enables:

- Changes to Argo CD config (e.g., enabling DEX or setting custom timeouts) happen in Git  
- Upgrades are triggered via Helm value bumps in a PR  
- Everything becomes testable, reviewable, and roll-backable  

Even Argo CD’s RBAC, repo credentials, and UI branding can be automated this way.

## Operational Reality: Zero-Drift, Zero-Touch

Once the bootstrap completes:

- The Argo CD UI becomes a live view of your cluster’s alignment with Git  
- Drifted resources are auto-corrected (or highlighted)  
- Every change has an audit trail  
- Disaster recovery is just: reprovision → let Argo CD sync from Git  

This is infrastructure with feedback loops, not just automation.

## Current Architecture

```
┌────────────────────────────┐
│ Git Repository             │
│ - root/                    │
│   - infra/                 │
│     - argocd/              │
│     - sealed-secret/       │
│   - applications/          │
└────────────┬───────────────┘
             │ GitOps sync
             ▼
     ┌─────────────┐
     │ Argo CD     │
     └─────┬───────┘
           │
     ┌─────▼───────┐
     │ K3s Cluster │
     │ (OCI Free)  │
     └─────────────┘
```

## What’s Ahead: Scale, Identity, Secrets

The current setup is production-worthy for single-node workloads — but it’s not the end of the journey.

Next, I’ll introduce:

- **A new K3s worker node**, joining the same cluster  
- **DEX integration** for GitHub and Google login — so platform access is federated, not static  
- **Bitnami Sealed Secrets**, finally put to use for managing encrypted secrets in Git  

Each of these steps is already staged in Git. When the time comes, syncing the root app will roll them out — cleanly, safely, and declaratively.

## Final Takeaways

This phase of the odyssey marks a shift from automation to autonomy.

It’s no longer about writing scripts to install services — it’s about modeling intent in Git and letting the system enforce it. This Git-first design minimizes guesswork, eliminates drift, and enables confident change — even at scale.

You don’t need a fleet of VMs or a Kubernetes team to get here.

You just need the right foundation, the right tools, and a commitment to source-driven infrastructure.

> From a blank cloud account to a Git-managed cluster, self-correcting and TLS-secured — the journey continues.
