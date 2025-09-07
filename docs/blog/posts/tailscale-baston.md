---
date: 2025-09-06
categories:
  - tailscale
  - vpn
  - kubernetes
  - networking
  - devops
  - terraform
---
# Extending Our Tailscale Setup with a Terraform-Managed Bastion
##### originally posted at [LinkedIn](https://www.linkedin.com/pulse/extending-our-tailscale-setup-terraform-managed-bastion-janus-chung-hxbqc) at Sept 06, 2025

In [my previous post](./2025/06/25/swapping-vpn-for-tailscale-a-five-day-internal-infra-upgrade/), I wrote about how we replaced a traditional VPN with Tailscale to connect engineers to Kubernetes services. That solved a big piece of the puzzle: cluster access was now simple, secure, and reliable.  

But as always, not everything lives in Kubernetes. We still had private databases, legacy services, and tools running in our VPC that engineers needed to reach. That’s where a bastion came in.

![Flying with Super Speed in the New Bastion Tunnel](../../assets/blog/tailscale-bastion/banner.jpg)

<!-- more -->

## Why Not Run the Bastion in EKS?  

The first idea was straightforward: just run the bastion inside the Kubernetes cluster. We already had the Tailscale Kubernetes controller working, so why not extend it?  

The catch is resilience. If the cluster itself is unavailable, the bastion inside it disappears too. That leaves no clear way back in when you need it most.  

So we decided to place the bastion outside the cluster, where it could act as an independent entry point.  

## A Simple EC2 Router  

The design is uncomplicated:  

1. Provision a small EC2 instance.  
2. Install Tailscale.  
3. Enable subnet routing with `--advertise-routes` so it can expose private ranges from the VPC.  
4. Approve the advertised routes in the Tailscale admin console (or via Terraform).  

With that, the Tailnet extends beyond Kubernetes and into the rest of our infrastructure. The bastion becomes a consistent path to reach internal resources, no matter where they live.  

## Automating with Terraform  

We didn’t want to maintain this manually. The entire setup — instance, networking, routing — is described in Terraform. A minimal example looks like this:  

```hcl
resource "aws_instance" "bastion" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro" # Free tier eligible
  subnet_id     = aws_subnet.private.id
  vpc_security_group_ids = [aws_security_group.bastion.id]

  user_data = <<-EOT
    #!/bin/bash
    curl -fsSL https://tailscale.com/install.sh | sh
    tailscale up \\
      --authkey=${var.tailscale_auth_key} \\
      --advertise-routes=10.0.0.0/16 \\
      --ssh
  EOT

  tags = {
    Name = "tailscale-bastion"
  }
}
```  

If it ever needs to be replaced, we can rebuild it from code. No one is SSHing in to "fix" things by hand.  

## Scaling and Redundancy  

One of the pleasant surprises was how easy it is to scale this setup:  

- **Horizontal scaling**: We can deploy multiple bastion instances across availability zones, each advertising the same routes. Tailscale handles the routing logic automatically, so clients don’t need to care which router they connect to.  
- **Failover support**: If one EC2 router goes down, traffic shifts to another. No manual intervention is required.  
- **Flexibility**: If we need more throughput, we can scale up the instance size or add more routers. Terraform makes this a one-line change.  

This setup gives us a clear and resilient path back into our infrastructure, independent of Kubernetes.  

## The Architecture

Here’s a simplified view of how it all fits together:

```
+-------------------+          +---------------------+
|   Developers      |          |   Tailscale Admin   |
| (Tailscale client)|--------->|   Coordination      |
+-------------------+          +---------------------+
              |                        
              | Tailnet connection
              v
+-----------------------------------+
|     EC2 Bastion (Tailscale)       |
|   --advertise-routes=10.0.0.0/16  |
+-----------------------------------+
              |
              v
   +-------------------+      +-------------------+
   |    VPC Subnets    |----->|  Kubernetes (EKS) |
   | (private ranges)  |      +-------------------+
   |                   |----->|  Databases        |
   |                   |----->|  Legacy Services  |
   +-------------------+.     +-------------------+
```

## What We Gained  

- A single, consistent way to reach Kubernetes and non-Kubernetes systems.  
- Independence from the cluster itself — the bastion remains available even if EKS has issues.  
- Everything defined in Terraform, so we can recreate or scale it quickly.  
- Options to add redundancy without much overhead.  

## Closing Thoughts  

Over the course of this series, we went from relying on a traditional VPN, to adopting Tailscale for Kubernetes access, and finally to extending that same approach with a Terraform-managed bastion. What we have now is a network model that:

- Works across both Kubernetes and non-Kubernetes resources.
- Gives us a reliable way back into our infrastructure, even when the cluster itself is unhealthy.
- Is fully expressed in code, so we can reproduce or extend it without guesswork.
- Scales naturally when we need to add capacity or redundancy.

The outcome is not just a technical improvement — it’s a shift in how we think about connectivity. Instead of designing networks around hardware appliances and static tunnels, we treat secure access as something we can declare in code, roll out quickly, and evolve as our infrastructure changes.

That foundation will serve us well as we continue to grow, and it’s one less operational burden we need to carry.
