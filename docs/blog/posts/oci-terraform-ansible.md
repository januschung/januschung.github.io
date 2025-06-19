---
title: "The DevOps Odyssey: Fully Automating OCI App Deployment with Terraform, Ansible, and Docker"
date: 2025-06-18
tags:
  - devops
  - oci
  - terraform
  - ansible
  - docker
  - cicd
  - github
summary: >
  Building a fully automated CI/CD pipeline on Oracle Cloud Infrastructure using Terraform, Ansible, Docker Compose, Traefik, DuckDNS, and GitHub Actions—all within the free tier.
---

## Introduction: The Engineer's Drive for Automation

As a DevOps engineer, I thrive on full‑stack automation—turning repetitive, error‑prone deployments into push‑button, ultra‑reliable workflows.  
I recently challenged myself to get **Job Winner**, an opensource full‑stack app (Spring Boot + React), live on **Oracle Cloud Infrastructure (OCI)** in less than **15 minutes** from a cold start.  
But the real goal wasn't speed alone—it was **idempotence**: every run of the pipeline should converge the system to the exact same, secure, HTTPS‑enabled state without manual touch‑points.

![OCI, Terraform, Ansible](../../assets/blog/oci-terraform-ansible/banner.jpg)

<!-- more -->

In this post you'll travel the entire odyssey—from a blank OCI account to a humming, production‑grade CI/CD pipeline. Expect:

* Detailed Terraform modules for free‑tier compute, networking, and remote state.
* Ansible roles that turn a vanilla Oracle Linux VM into a hardened Docker host.
* Traefik 3 as a smart edge router with automatic Let's Encrypt certificates.
* GitHub Actions tying it all together in a **single commit → live site** workflow.
* Hard‑won lessons, cost analysis, and future‑proofing tips.

**TL;DR**: Free cloud, real domain, zero click deploys. Grab a coffee—this is a long read.

---

## Why Oracle Cloud Free Tier?

OCI's Always‑Free resources give you an ARM‑based VM (4 OCPU, 24 GB RAM) *or* an AMD VM (1 OCPU, 1 GB RAM) **plus 10 TB egress**—an unbeatable budget playground.  
Compared to AWS or GCP free tiers, outbound bandwidth alone makes OCI ideal for side‑projects with media assets or heavy API traffic.

| Cloud | Monthly Free Egress | Notes |
|-------|--------------------|-------|
| **OCI** | **10 TB** | Always Free |
| AWS | 100 GB | Free tier year one |
| GCP | 1 GB | Always Free |

Beyond cost, OCI integrates cleanly with HashiCorp Terraform through an official provider and supports **Object Storage buckets** that double as free Terraform remote state back‑ends.

---

## Architecture at 10,000 Feet

```text
┌───────────────┐     GitHub Actions   ┌────────────────────────┐
│  GitHub Repo  │ ───────────────────▶ │ OCI: Object Storage    │
└───────────────┘  stores tfstate      └────────────────────────┘
       │ Push/Pull Requests                   ▲
       ▼                                      │
┌────────────────┐  terraform apply           │
│ Terraform      │ ───────────────────────────┘
│ (Infrastructure│
│  as Code)      │ ──▶ VCN, Subnet, Rules
└────────────────┘         │
                            ▼
                       ┌───────────────┐  ansible-playbook
                       │  OCI VM       │ ◀───────────────────── GitHub Runner
                       └───────────────┘
                         │  docker compose up
                         ▼
                  ┌────────────────────┐
                  │ Traefik Reverse    │
                  │ Proxy (HTTPS)      │
                  └────────────────────┘
                         │
                         ▼
                  ┌────────────────────┐
                  │ React Frontend     │
                  │ Spring Boot API    │
                  └────────────────────┘
```

---

## Phase 1: Terraform—Laying Down the Runway

### Module Design

For this project, I crafted a single, comprehensive Terraform module that provisions everything needed for the deployment. This module takes care of spinning up the Virtual Cloud Network (VCN), public subnet, Internet Gateway, security lists, and the free-tier Oracle Linux VM—all in one go. By bundling the infrastructure logic together, setup becomes straightforward and easy to reuse for future experiments or side projects.


```hcl
module "web_server" {
  source                = "./modules/web-server"
  compartment_ocid      = var.compartment_ocid
  availability_domain   = var.availability_domain
  shape                 = "VM.Standard.A1.Flex"
  vm_ocpus              = 1
  vm_memory             = 6
  ssh_public_key        = file("~/.ssh/id_rsa.pub")
  # ...other variables...
}
```

### Remote State Back‑End

```hcl
terraform {
  backend "oci" {
    bucket    = "your-state-bucket"
    namespace = "your-namespace"
    region    = "your-region"
    key       = "terraform.tfstate"
  }
}
```

Remote state unlocks `terraform plan` diffs inside GitHub Actions so pull‑request reviewers can preview infra changes before merging.

---


## Phase 2 – Ansible: Turning VMs into Cattle

### Cloud-Init: The Initial Approach

When I first attempted to automate post-provisioning on my OCI VM, I turned to **Cloud-Init**. It’s a native mechanism that Oracle Cloud Infrastructure supports out of the box, letting you pass a shell script through the `user_data` field in Terraform:

```hcl
resource "oci_core_instance" "jobwinner" {
  # ... other config ...
  metadata = {
    ssh_authorized_keys = file("~/.ssh/id_rsa.pub")
    user_data           = base64encode(file("cloud-init/jobwinner-init.sh"))
  }
}
```

This script (`jobwinner-init.sh`) aimed to bootstrap the whole stack—installing Docker, creating the network, pulling images, writing configs, and launching containers. It was a decent first shot.

But the problems were immediate:

- No visibility into when or if the script failed.
- Troubleshooting meant SSHing into the VM and running:
  ```bash
  sudo tail -n 50 /var/log/cloud-init.log
  ```
- Debugging Bash templates is like diffing spaghetti—hard to maintain, fragile to extend.

I needed a better approach. One with structure. One with feedback. One that was *actually fun to use*.

### Switching to Ansible: Structure, Idempotence, Sanity

That’s when I pivoted to **Ansible**—and it completely changed the game.

Instead of wrestling with cloud-init logs, I now have a clean split between **infrastructure provisioning (Terraform)** and **configuration management (Ansible)**.

The Ansible setup revolves around **two focused roles**:

#### 1. `common` – System Setup
This role handles the core VM preparation:

- Updates all packages using `dnf`
- Installs `epel-release`, `firewalld`, and `docker`
- Starts and enables `firewalld`
- Opens ports **80** and **443**

These steps are run through the `setup-docker.yml` playbook, making the VM Docker-ready with a secure firewall configuration.

#### 2. `deploy-jobwinner` – Application Stack & DNS
Once the VM is ready, this role takes over. Here’s what it does:

- **Updates DuckDNS** with the VM’s current IP via an HTTP call
- **Creates a secured `acme.json` file** for Traefik to use with Let’s Encrypt
- **Templates out `traefik.yml`** from a Jinja2 file
- **Bootstraps Docker networks** using a simple shell command
- **Spins up Traefik and the app stack** using `docker-compose`

All of this happens in a structured, readable, and repeatable manner. No surprises.

### Jinja2 Templating: One Config to Rule Them All

Instead of hardcoded configs, everything lives as a template:

```jinja
# docker-compose.yml.j2
version: "3.9"
services:
  frontend:
    image: "{{ frontend_image }}"
    labels:
      - "traefik.http.routers.frontend.rule=Host(`{{ traefik_host_domain }}`)"
```

All variables are centralized in `group_vars/all.yml`, meaning I can swap images, update domains, or add ports with a single change. It's clean, version-controlled, and predictable.

### Reusable Compose Logic

One of my favorite touches is the shared `setup-compose-app.yml` task file. Both the app and Traefik use it. It performs three simple tasks:

1. Ensures the target directory exists (`/opt/jobwinner`, `/opt/traefik`, etc.)
2. Templates the appropriate `docker-compose.yml`
3. Runs `docker compose up -d` to launch the service

This small abstraction keeps things **DRY** and easy to extend.

### Wait for Readiness

Before deploying the app stack, Ansible waits for Traefik to be up using a simple health check:

```yaml
- name: Wait for Traefik to be ready
  uri:
    url: http://localhost:80
    status_code: 404
    retries: 10
    delay: 5
    until: traefik_check is succeeded
```

This ensures we don’t start routing app traffic before Traefik is ready to accept connections.

### Traefik, DuckDNS, and HTTPS—All Baked In

Once Traefik is live:

- It listens on **port 80 and 443**
- It pulls **Let’s Encrypt certs** using DuckDNS DNS validation
- It binds the ACME data to a secured `acme.json` file
- It dynamically routes frontend and backend containers based on Docker labels

And thanks to Ansible, all of this happens automatically—no need to SSH into the machine, touch configs, or run manual cert renewals.

> **Pro-Tip**: Don't forget to bind-mount `acme.json` with `0600` permissions. Traefik won’t start if it’s too permissive.

## Phase 3 – GitHub Actions CI/CD (Reality Edition)

My pipeline is split into **three purpose‑built workflows**, each with a single job.  
This keeps logs short and lets me re‑run only the layer I need.

| Workflow | What it Does |
|----------|---------|
| **terraform.yml** | Provisions or updates all OCI resources—VCN, subnet, free‑tier VM, and Object Storage state bucket |
| **setup-docker.yml** | Runs the **common** Ansible role to update packages, install `docker`, enable firewalld, and open ports 80/443 |
| **deploy-jobwinner.yml** | Runs the **deploy‑jobwinner** role to refresh DuckDNS, start Traefik, wait for it, and launch the app stack |

### Workflow 1: Provision with Terraform
Triggered on every push or PR to main, this workflow initializes Terraform, runs plan on PRs, and apply on merges.
It handles OCI creds from secrets and writes the backend config for remote state.

### Workflow 2: Make the VM Docker-Ready
Manually triggered once the VM is up. It SSHes into the instance and runs the common Ansible role:

- Updates packages
- Installs Docker
- Enables firewalld and opens ports 80/443

This takes only couple minutes and gets the system production-ready.

### Workflow 3: Deploy Job Winner
Also manually triggered, this one runs the deploy-jobwinner Ansible role:

- Updates DuckDNS with the current public IP
- Templates and boots Traefik
- Waits for Traefik to come online
- Starts the Job Winner frontend and backend using Docker Compose

### Why This Split?
Keeping infra, setup, and deploy separate gives me:

🔄 Faster app iteration

🛡️ Safer infra changes

📋 Cleaner logs and simpler debugging

Push code for infra, run two workflows for software, and your HTTPS app is live—no hand-holding needed.

## Cost Breakdown

| Component | Monthly Cost |
|-----------|--------------|
| OCI VM (Free Tier) | $0 |
| Object Storage ≤ 20 GB | $0 |
| DuckDNS Domain | $0 |
| Let's Encrypt Certs | $0 |
| *Total* | **$0.00** |

## Lessons Learned & Final Thoughts

- Avoid using cloud-init for multi-step app bootstrapping—debugging is painful.
- Ansible + templates + modular playbooks = power and clarity.
- Keep each component loosely coupled.
- Use retries and until in Ansible to ensure readiness (e.g., waiting for Traefik).
- Remote Terraform state on OCI Free Tier is fully achievable and highly recommended.
- Integrating GitHub Actions for CI/CD transforms deployment speed and reliability.
- Even free-tier cloud resources are enough for a clean, HTTPS-protected, full-stack app, deployed in minutes.

## Conclusion

With a strategic mix of Terraform, Ansible, Docker Compose, and GitHub Actions, it's possible to reach **enterprise‑grade DevOps** on a **coffee‑budget**.  
