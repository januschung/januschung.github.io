---
date: 2025-06-29
categories:
  - devops
  - oci
  - terraform
  - ansible
  - docker
  - cicd
  - github
summary: >
  Building a fully automated CI/CD pipeline on Oracle Cloud Infrastructure using Terraform, Ansible, K3s, Traefik, DuckDNS, and GitHub Actions—all within the free tier.
---
# The DevOps Odyssey Continues: Evolving from Docker to K3s with Ansible
##### originally posted at [LinkedIn](https://www.linkedin.com/pulse/devops-odyssey-continues-evolving-from-docker-k3s-ansible-janus-chung-2tqsc/) at July 25, 2025

In [Part 1](./2025/06/18/the-devops-odyssey-fully-automating-oci-app-deployment-with-terraform-ansible-and-docker/), I turned an OCI Free Tier VM into a fully automated, HTTPS-secured Docker host using Terraform, Ansible, Traefik, and GitHub Actions. That stack was great for monoliths or simple containers.

But containers want orchestration. And I want GitOps.

So this phase of the odyssey shifts gears: replacing Docker Compose with K3s — a lightweight Kubernetes distribution that fits beautifully in constrained environments like OCI free tier.

The goal? A production-grade Kubernetes control plane, fully bootstrapped with Ansible, ready for GitOps.

![Automation bots have evolved. What’s next?](../../assets/blog/oci-k3s-ansible/banner.jpeg)

<!-- more -->

## Why K3s?

K3s gives me:

* A certified Kubernetes API in ~100MB
* Zero hassle install (`curl | sh`)
* Embedded etcd, flannel, containerd — batteries included
* Low memory footprint (runs on 1GB RAM)
* Native ARM64 support (ideal for OCI’s Ampere A1 VMs)

---

## Replacing Docker with K3s

In the original setup, Ansible turned the vanilla OCI VM into a Docker host with:

* Docker + firewalld
* Traefik via Compose
* An exported acme.json for TLS

Now I’m swapping that out with:

* K3s with embedded Kubernetes components
* Proper kernel + sysctl config
* Secure kubectl access for the opc user

## Ansible Playbook: k3s.yml

Here’s the simplified playbook that bootstraps the cluster:

```hcl
- name: Disable swap
  command: swapoff -a
  when: ansible_swaptotal_mb > 0

- name: Remove swap entry from /etc/fstab
  replace:
    path: /etc/fstab
    regexp: '(^.*swap.*$)'
    replace: '# \1'

- name: Ensure br_netfilter module is loaded
  modprobe:
    name: br_netfilter
    state: present

- name: Ensure br_netfilter is loaded on boot
  copy:
    dest: /etc/modules-load.d/k8s.conf
    content: "br_netfilter\n"
    mode: '0644'

- name: Configure sysctl for Kubernetes networking
  sysctl:
    name: "{{ item.name }}"
    value: "{{ item.value }}"
    sysctl_set: yes
    reload: yes
  loop:
    - { name: net.bridge.bridge-nf-call-iptables, value: 1 }
    - { name: net.bridge.bridge-nf-call-ip6tables, value: 1 }
    - { name: net.ipv4.ip_forward, value: 1 }

- name: Install K3s
  shell: |
    curl -sfL https://get.k3s.io | sh -s - server --cluster-init
  args:
    creates: /usr/local/bin/k3s

- name: Wait for K3s service to be active
  systemd:
    name: k3s
    state: started
    enabled: yes

- name: Wait for node to be Ready
  shell: |
    export PATH=/usr/local/bin:$PATH
    kubectl get nodes --no-headers | grep ' Ready '
  register: node_ready_check
  retries: 10
  delay: 15
  until: node_ready_check.rc == 0
  environment:
    KUBECONFIG: /etc/rancher/k3s/k3s.yaml

- name: Ensure chrony time sync service is running
  package:
    name: chrony
    state: present

- name: Enable and start chronyd
  systemd:
    name: chronyd
    state: started
    enabled: yes
```

### After Ansible Runs

```bash
kubectl get nodes
```
You should see:
```bash
NAME STATUS ROLES AGE VERSION
k3s-master Ready control-plane,master 2m v1.29.3+k3s1
```

The VM is now a Kubernetes control plane.

## Why This Matters

This setup transforms the free-tier VM into a modern orchestrator:

* GitOps-ready (next post will wire up Argo CD)
* Secrets-ready (we’ll integrate Sealed Secrets)
* Future-proof (easily extend to multi-node or HA)

And it’s fully idempotent — re-run Ansible, and you land in the same good state.

## Coming Next

Next up, I’ll show how to:

* Install and expose Argo CD with TLS and DuckDNS
* Bootstrap workloads with GitOps from a self-managed repo
* Manage secrets with Bitnami Sealed Secrets

Thanks for following the journey — automation continues.
