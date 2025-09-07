---
date: 2025-08-31
categories:
  - devops
  - oci
  - terraform
  - argocd
  - cicd
  - github
  - sealedsecret
  - kubernetes
  - dex
---

# The DevOps Odyssey, Part 4: Secrets, GitHub Auth, and Scaling Out
##### originally posted at [LinkedIn](https://www.linkedin.com/pulse/devops-odyssey-part-4-secrets-github-auth-scaling-out-janus-chung-afzgc/) at Aug 31, 2025

In [Part 1](./2025/06/18/the-devops-odyssey-fully-automating-oci-app-deployment-with-terraform-ansible-and-docker/), I bootstrapped a zero-click deployment pipeline on OCI with Terraform, Ansible, and Docker Compose — complete with HTTPS, DNS, and CI/CD.

[Part 2](./2025/06/29/the-devops-odyssey-continues-evolving-from-docker-to-k3s-with-ansible/) evolved that into a Kubernetes-native architecture, replacing Docker with K3s for a declarative control plane.

[Part 3](./2025/07/31/the-devops-odyssey-part-3-gitops-on-k3s-with-argo-cd--self-managing-infrastructure-from-a-single-commit/) brought in GitOps with Argo CD, letting the cluster manage itself from a single commit.  

Now, in **Part 4**, I pushed the setup toward something that looks and feels much closer to production. Three key steps made that happen: 

1. **Sealing secrets** so I could finally commit them to Git safely.  
2. **Adding GitHub authentication with Dex**, making the Argo CD UI open (read-only) to anyone with a GitHub account.  
3. **Expanding the cluster with a proper worker node** — and replacing my ill-fated “master as NAT” shortcut with OCI’s managed NAT Gateway.  

![Autobot master cloned a worker self to prepare for the upcoming battle.](../../assets/blog/oci-k3s-worker/banner.jpg)

<!-- more -->

## 1. Sealed Secrets: Unlocking GitOps for Real  

Everything in GitOps lives in Git. That’s the promise, but it’s also the challenge: Kubernetes `Secret` objects are just base64-encoded strings. Put them in Git as-is, and you’ve basically published your passwords.  

That’s why I started Part 4 by deploying **Bitnami’s Sealed Secrets** operator via Argo CD. Once it was running, I could safely commit sealed versions of my sensitive credentials:  

- **My GitHub Container Registry pull secret** (to pull images).  
- **My GitHub OAuth client secret** (to connect Argo CD to GitHub for login).  

Here’s what the sealed Dex secret looks like:

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: dex-github-secret
  namespace: argocd
spec:
  encryptedData:
    dex.github.clientSecret: AgCBlxRncrOcJeIdZ8+8oCr...
  template:
    metadata:
      labels:
        app.kubernetes.io/part-of: argocd
      name: dex-github-secret
      namespace: argocd
    type: Opaque
```  

By sealing secrets first, I laid the foundation for the next step: GitHub authentication.  

## 2. GitHub Auth via Dex: A Live Demo for Everyone  

With secrets secure, I moved on to **authentication for Argo CD**.  

Argo CD ships with a local admin account, but I wanted more than that. This cluster is both a hobby lab and a **public demo of my DevOps skills**. I wanted curious folks to experience GitOps first-hand, without me handing out cluster credentials.  

So I set up **Dex with GitHub as the identity provider**. My Argo CD installation is self-managed via Helm and Argo CD itself. In my `values.yaml`, I configured Dex and GitHub like this:  

```yaml
argo-cd:
  configs:
    cm:
      url: https://janusc.duckdns.org/argocd
      dex.config: |
        connectors:
        - type: github
          id: github
          name: GitHub
          config:
            clientID: <my-client-id>
            clientSecret: $dex-github-secret:dex.github.clientSecret
            redirectURI: https://janusc.duckdns.org/argocd/api/dex/callback
            scopes:
            - read:org
            - user:email
            useLoginAsID: true
    rbac:
      policy.csv: |
        g, januschung, role:admin
      policy.default: role:readonly
      scopes: '[profile, email]'
```  

A few things to note here:  

- The `clientSecret` points to the **sealed secret** I committed earlier, so it never appears in plaintext.  
- RBAC is configured so my own GitHub user (`januschung`) is an admin.  
- Everyone else defaults to **role:readonly**.  

That last part is the point: **anyone with a GitHub account can log into my Argo CD UI, but only in read-only mode**.  

Why? Because this project is my **open DevOps showcase**:  

- Developers and tech enthusiasts can log in, click around, and see real GitOps in action.  
- There’s no risk — the read-only role prevents anyone from changing deployments.  
- It turns Argo CD into a live demo environment, always available for anyone curious about GitOps.  

## 3. Scaling Out: Adding a Worker Node  

The final step in this phase was **adding a worker node**. Until now, everything ran on my single K3s master. That was fine for testing, but not the kind of setup you’d take to production.  

Here’s the truth: I almost didn’t do it right. At first, I tried to let the master act as a **NAT gateway** so that my private worker could reach the internet. It felt clever, but it was actually a liability:  

- The master is supposed to run the control plane, not forward traffic.  
- If the master died, the entire cluster’s egress would go dark.  
- My Terraform started looking like a spaghetti of conditional routing rules.  

Then I realized something obvious: **OCI already provides a managed NAT Gateway — and it comes with a generous free monthly quota.**  

I tore out my DIY NAT setup, dropped in the managed gateway, and my network instantly became simpler and more reliable. The master node went back to doing what it does best, the worker had clean egress, and I had one less thing to babysit.  

## 4. The Cluster Grows Up  

With these changes in place, my setup made a leap:  

- **Sealed Secrets** gave me safe, GitOps-friendly secret management.  
- **Dex with GitHub** made Argo CD a transparent demo portal — open to anyone with a GitHub account, but read-only by default.  
- **Worker nodes + managed NAT** gave me a scalable, production-grade topology.  

This is the moment where my hobby cluster stopped being a single-VM experiment and started looking like a real platform.  

## 5. What’s Next: Running Real Apps  

Infrastructure is great, but the real payoff comes from running apps that matter. That’s what’s next on this journey.  

- First up: **migrating Jobwinner**, my existing app that’s been living in Docker Compose on a separate VM, into the Argo CD-managed K3s cluster. This consolidates everything into one pipeline and one platform.  
- And then, something more personal: I want to build a **photo album app** — a kind of virtual bookshelf — to showcase my past life as a photographer. Over the years, I’ve taken portraits of the **beautiful ladies I crossed paths with**, and I want a polished gallery to present that work. This cluster gives me the perfect home for it.  

## Wrapping Up  

Part 4 was about **responsibility and trust**. I learned not to cut corners with networking, I locked down secrets properly, and I opened up my GitOps UI in a way that’s transparent but safe.  

Now, the cluster isn’t just a backend experiment — it’s becoming a **stage**. One that can run my real projects, show off my skills, and even host my photography.  

In **Part 5**, we’ll step onto that stage: deploying Jobwinner and my photo album app with Helm + Argo CD.  
