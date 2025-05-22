---
date: 2025-05-17
categories:
  - shell script
  - sso
  - kubernetes
  - aws
  - devops
---
# Automatically Renew AWS SSO Session and Refresh Kubeconfig for EKS Access

Working with AWS EKS clusters via AWS SSO is secure but sometimes frustrating.  
If your session expires, `kubectl` commands will fail until you manually renew the session and update your kubeconfig.

Let's automate that with a small Bash script.

![Auto Renew Kubeconfig](../../assets/blog/auto-renew-kubeconfig/banner.jpg)

<!-- more -->

## Problem

- AWS SSO sessions expire every 8–12 hours by default.
- `kubectl` will throw `ExpiredTokenException` or connection errors.
- You need to manually run `aws sso login` and `aws eks update-kubeconfig`.

!!! danger "Frustration"
    Manually logging back in slows you down and interrupts your work.

---

## Solution: Bash Script

Create a script called `aws_sso_kubeconfig.sh`:

```bash
#!/bin/bash

PROFILE="prod"
CLUSTER_NAME="your-eks-cluster-name"
REGION="your-eks-cluster-region"

# Check if AWS SSO session is valid
if ! aws sts get-caller-identity --profile "$PROFILE" > /dev/null 2>&1; then
  echo "🔒 AWS SSO session expired. Logging in again..."
  aws sso login --profile "$PROFILE"
else
  echo "✅ AWS SSO session still valid."
fi

# Always refresh kubeconfig
aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$REGION" --profile "$PROFILE"
echo "✅ Kubeconfig updated."
```

## How to Use
Make it executable:

``` bash
chmod +x aws_sso_kubeconfig.sh
```

Run it before using kubectl:

``` bash
./aws_sso_kubeconfig.sh
```

!!! tip "Quick Tip" 
    Add an alias to your .bashrc or .zshrc for faster use: bash alias krenew="~/path/to/aws_sso_kubeconfig.sh"

Then just type:
```bash
krenew
```

## Bonus: Automate it with a Cron Job

You can set up a cron job to run the script every 4–6 hours automatically, keeping your session fresh.

Edit your crontab:

``` bash
crontab -e
```

Add this line (every 6 hours):

``` bash
0 */6 * * * /path/to/aws_sso_kubeconfig.sh >> /tmp/aws_sso_renew.log 2>&1
```

This:

- Runs the script every 6 hours
- Redirects output to /tmp/aws_sso_renew.log for easy debugging if needed

!!! note "Important" 
    Make sure your environment variables ($PATH) are properly available inside cron.
    Sometimes you need to load your AWS credentials manually inside the script if your environment is not loaded.

## Why It Matters

- ⏳ No more session expiration surprises.
- 🔒 Maintain secure access to your Kubernetes cluster.
- 🚀 Speed up your daily development workflow.

## Final Thoughts

AWS SSO is great for security, but it can disrupt your Kubernetes operations without automation.
This small script saves time, reduces frustration, and helps you maintain a smooth EKS workflow without manual steps.

Happy Kubernetes hacking! 🚀