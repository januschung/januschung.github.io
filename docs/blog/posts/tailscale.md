---
date: 2025-06-25
summary: "How we started replacing our VPN with Tailscale, improved onboarding and access control, and reduced costs—all in five days."
categories:
  - tailscale
  - vpn
  - kubernetes
  - networking
  - devops
---
# Swapping VPN for Tailscale: A Five-Day Internal Infra Upgrade
##### originally posted at [LinkedIn](https://www.linkedin.com/pulse/swapping-vpn-tailscale-five-day-internal-infra-upgrade-janus-chung-rnucc) at June 25, 2025

We recently started migrating away from our traditional VPN setup—and toward something simpler, faster, and cheaper: **Tailscale**.

This wasn’t a full rip-and-replace. In just five days, we moved a core set of internal Kubernetes services behind Tailscale, enough to start retiring our legacy VPN setup piece by piece.

The results?  
✅ Smoother developer workflows  
✅ Better access control  
✅ Significant cost savings  
✅ Self-serve onboarding  
✅ Fewer support headaches

![Enjoy Super Speeding in Private Network Tunnel](../../assets/blog/tailscale/banner.jpg)

<!-- more -->

## Why We Looked Beyond Our VPN

Our old VPN setup worked, but it came with pain:

- **Unreliable onboarding**: Getting new devs access to internal tools meant following tribal checklists.
- **Flaky performance**: DNS routing issues and random client disconnects were routine.
- **Scaling cost**: Per-user pricing added up quickly as the team grew.

We needed a way to:
- Securely expose internal services
- Manage access with identity, not IPs
- Spend less time maintaining tunnels

Tailscale had already been a favorite among engineers for peer-to-peer dev work. With their Kubernetes integration, we realized it could be the backbone of our internal network.


## The Tailscale Kubernetes Operator

We used the [Tailscale Kubernetes Operator](https://tailscale.com/kb/1236/kubernetes-operator/) to expose services inside our EKS cluster directly onto our Tailscale network.

That meant:
- No public ingress
- No legacy VPN tunnels
- Just a Tailscale IP and ACL-based access control

To make it work, we asked Tailscale to enable a beta feature. They did so the same day—unblocking us instantly.

> 💬 “Tailscale’s support was fast, helpful, and low-friction. That made a huge difference in moving quickly.”

## What We Did in 5 Days

This wasn’t a full migration—but it was a meaningful one.

### ✅ Day 1: Planning and Access
- Audited internal services for migration
- Requested access to the necessary beta feature
- Set up a test namespace in EKS

### ✅ Day 2: Deploy the Operator
- Installed the Tailscale operator in Kubernetes
- Verified access to a test service via tailnet

### ✅ Days 3–4: Migrate Key Services
- Moved internal dashboards and dev/test APIs
- Replaced ingress rules with tailnet-only access

### ✅ Day 5: Move GoLink Off a Laptop
Our internal GoLink (URL shortener) was still running on a MacBook.  
We containerized it, deployed it to EKS, and exposed it via Tailscale.

It’s now:
- Always available  
- Protected by SSO and ACLs  
- No longer a “single MacBook of failure”

## 📉 Cost Comparison: Traditional VPN vs. Tailscale

Here’s how costs shake out for teams of 10 to 50 users:

| Team Size | Traditional VPN ($10–15 PMPM) | Tailscale ($6 PMPM) | **Monthly Savings** |
|-----------|-------------------------------|----------------------|----------------------|
| 10 users  | $100–150                      | $60                  | $40–90               |
| 25 users  | $250–375                      | $150                 | $100–225             |
| 50 users  | $500–750                      | $300                 | $200–450             |

> With Tailscale, pricing is flat, predictable, and scales cleanly with the team.


## What We Gained So Far

Even with a partial rollout, the impact was immediate:

### 🧠 Effortless Onboarding  
New teammates just sign in with SSO and install the Tailscale app—on desktop **or mobile**.  
No setup guides. No support pings. No friction.

> One team member called it *“the smoothest internal access experience I’ve had.”*

### 🔐 Better Access Control  
We make use of ACL tagging right in the YAML config, and manage everything from one clean web UI.  
It's simple, auditable, and role-aware—without any subnet headaches.

### ⚡ Developer Happiness  
Internal tools are instantly reachable from anywhere via tailnet IPs.  
No more wrestling with VPN clients or wondering why DNS isn’t resolving.

### 💸 Real Cost Savings  
We’re already spending less than before—and the savings will grow as we migrate more.

## What’s Next

This rollout laid the groundwork for more:

- Expanding Tailscale coverage to more services  
- Introducing **exit nodes** for outbound traffic  
- Removing our legacy VPN entirely  
- Testing **Tailscale Funnel** for preview environments

## Final Thoughts

We didn’t replace everything in five days—but we replaced enough to see a clear difference.

With Tailscale, we simplified onboarding, secured access by identity, improved our internal network’s resilience, and lowered our costs. All without writing custom infra or managing new VPN servers.

If you're dealing with a brittle VPN setup or want something simpler, Tailscale’s worth exploring. Start small—see what happens.
