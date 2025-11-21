---
date: 2025-11-21
categories:
  - devops
  - github
  - kubernetes
  - cicd
  - security
---

# Developing and Testing K3s Apps Locally

When building Kubernetes-aware tools — whether a CLI, dashboard, or internal Python service — you often need your local environment to talk directly to the cluster API.  
But exposing the K3s server’s port `6443` to the internet is never a good idea.

Here’s how to make your **local machine behave like it’s inside the cluster**, safely, using an SSH tunnel.

![Remote accessing via the secret tunnel securely.](../../assets/blog/remote-secret-tunnel/banner.jpg)

<!-- more -->

## Why This Setup

K3s generates its own API certificate that only includes internal addresses like:

```
10.0.x.x, 127.0.0.1, ::1
```

If you connect directly using your public IP, the TLS validation fails.  
You could rebuild the certificates with `--tls-san`, but that’s overkill for development.  
Instead, we’ll tunnel the API securely through SSH and reuse the `127.0.0.1` identity that K3s already trusts.


## Step 1 — Copy the Cluster Configuration

On your K3s server, fetch the kubeconfig:

```bash
sudo cat /etc/rancher/k3s/k3s.yaml
```

Copy the full content to your local machine as:

```bash
mkdir -p ~/.kube
nano ~/.kube/k3s_config
```

Paste and save.

## Step 2 — Create a Secure Tunnel

On your local machine:

```bash
ssh -L 8001:localhost:6443 <your-ssh-user>@<your-server-ip>
```

This command forwards:
- Local port **8001** → remote K3s API port **6443**
- Over a fully encrypted SSH connection

Keep that terminal open. As long as it’s running, your laptop behaves like it’s inside the cluster.


## Step 3 — Update Your Kubeconfig

Edit `~/.kube/k3s_config` and replace:

```yaml
server: https://127.0.0.1:6443
```

with:

```yaml
server: https://127.0.0.1:8001
```

Save the file.


## Step 4 — Test with `kubectl`

Now, in a new terminal window:

```bash
kubectl get nodes --kubeconfig ~/.kube/k3s_config
```

You should see something like:

```
NAME            STATUS   ROLES                  AGE   VERSION
k3s-master      Ready    control-plane,master   2d    v1.30.0+k3s1
```

You can also verify connectivity with:

```bash
curl -k https://127.0.0.1:8001/version
```


## Step 5 — Test with Python

Install the official Kubernetes Python client:

```bash
pip install kubernetes
```

Then create a quick test script:

```python
from kubernetes import client, config

config.load_kube_config("~/.kube/k3s_config")
v1 = client.CoreV1Api()

for node in v1.list_node().items:
    print(node.metadata.name)
```

Run it locally while your SSH tunnel is open.  
Your Python code now talks directly to the remote K3s cluster — as if it were local.


## Step 6 — Optional Helper Script

You can wrap the tunnel in a small helper file:

```bash
#!/usr/bin/env bash
# connect-k3s.sh — open SSH tunnel to K3s API
set -e
ssh -L 8001:localhost:6443 <your-ssh-user>@<your-server-ip>
```

Run it when you need cluster access:

```bash
./connect-k3s.sh &
python my_k3s_app.py
```


## Why It’s Perfect for Local Development

This approach gives you the best of both worlds:

- Full `kubectl` and Python SDK access
- No need to regenerate TLS certificates
- No need to open port 6443 to the public internet
- Works anywhere you have SSH access

It’s effectively a **developer VPN** for Kubernetes — lightweight, built into SSH, and secure by design.


## Closing Thoughts

When you’re building K3s-native tools or testing automation, you don’t need to open the cluster to the world.  
You just need a secure tunnel home.

Somewhere between `localhost` and the cloud, your code talks to K3s as if it were running inside the cluster itself.
