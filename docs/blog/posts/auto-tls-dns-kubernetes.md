---
date: 2025-05-30
categories:
  - networking
  - ssl
  - kubernetes
  - aws
  - eks
  - devops
---
# Automated TLS and DNS in Kubernetes with ExternalDNS, Ingress, and Let's Encrypt

Managing DNS and TLS certificates for Kubernetes applications can be tedious and error-prone. Thankfully, tools like **ExternalDNS**, **Ingress**, and **Cert-Manager** automate the entire process — from setting DNS records to provisioning Let's Encrypt certificates.

In this guide, we'll walk through how to:

- Use **ExternalDNS** to automatically create DNS records.
- Annotate **Ingress** resources to request a Let's Encrypt TLS cert.
- Get HTTPS with minimal manual intervention.
- Understand how these components interact.

![Auto TLS and DNS with ExternalDNS, Ingress, and Let's Encrypt](../../assets/blog/auto-tls-dns-kubernetes/banner.jpg)

<!-- more -->

## Prerequisites

- A Kubernetes cluster (EKS, GKE, K3s, etc.)
- A domain name (e.g., `example.com`) with access to its DNS provider
- A public DNS provider supported by ExternalDNS (e.g., Route53, Cloudflare, Google DNS)
- Helm 3 installed locally
- `kubectl` configured for your cluster

## Step 1: Set Up ExternalDNS

ExternalDNS watches your Kubernetes resources (like `Ingress`) and creates the corresponding DNS records in your provider.

### Install ExternalDNS via Helm

For Route53 (AWS example):

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install external-dns bitnami/external-dns \
  --set provider=aws \
  --set aws.zoneType=public \
  --set domainFilters[0]=example.com \
  --set policy=sync \
  --set registry=txt \
  --set txtOwnerId=my-cluster \
  --set serviceAccount.create=true \
  --set serviceAccount.name=external-dns
```

> ⚠️ IAM permissions for Route53 are required on the node or pod role.


### IAM Policy for ExternalDNS (Route53 Example)

If you are using AWS Route53, ExternalDNS needs permission to manage DNS records.

Here is a sample IAM policy for managing **multiple domains**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/ZONE_ID_1",
        "arn:aws:route53:::hostedzone/ZONE_ID_2",
        "arn:aws:route53:::hostedzone/ZONE_ID_3"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets"
      ],
      "Resource": ["*"]
    }
  ]
}
```

> 🛠 Replace `ZONE_ID_X` with your actual Route53 hosted zone IDs (you can find them in the AWS Console under Route53).

If you're using **EKS**, create an IAM role with this policy and bind it to the Kubernetes ServiceAccount used by ExternalDNS using **IRSA**:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/ExternalDNSRole
```

## Step 2: Set Up Cert-Manager for Let's Encrypt

Cert-Manager issues and renews TLS certs from Let’s Encrypt.

### Install Cert-Manager via Helm

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

### Create Let's Encrypt Issuer

Create a production Issuer or Staging one (for testing) with HTTP-01 challenge:

**`issuer.yaml`**
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-http
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-http-private-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

Apply it:

```bash
kubectl apply -f issuer.yaml
```

## Step 3: Create Ingress with DNS and TLS Annotations

Now that DNS and Cert-Manager are configured, create an Ingress resource.

**`ingress.yaml`**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-http"
    external-dns.alpha.kubernetes.io/hostname: app.example.com
spec:
  tls:
    - hosts:
        - app.example.com
      secretName: my-app-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-service
                port:
                  number: 80
```

Apply it:

```bash
kubectl apply -f ingress.yaml
```

## How It All Works Together

1. **ExternalDNS** sees the Ingress annotation and creates a DNS A or CNAME record for `app.example.com` pointing to your Ingress controller's IP or hostname.
2. **Cert-Manager** notices the `cert-manager.io/cluster-issuer` annotation and requests a cert from Let's Encrypt using the `ClusterIssuer`.
3. It completes the HTTP-01 challenge via your NGINX ingress controller.
4. If successful, a TLS secret is created (`my-app-tls`) and used by the Ingress to serve HTTPS.

## Additional Tips

- You can use **wildcard DNS** with Let's Encrypt using **DNS-01 challenge**, which requires API access to your DNS provider.
- For custom Ingress controllers (like Traefik or HAProxy), make sure their class matches the annotation.
- Always start with **Let's Encrypt Staging** to avoid rate limits.
- Monitor logs of `cert-manager`, `external-dns`, and the ingress controller for debugging.

## Sample DNS Record

After everything is set up, you should see a record like this in your DNS provider:

```
app.example.com  ->  <Ingress LoadBalancer IP or hostname>
```

And your app should be accessible via:

```
https://app.example.com
```

with a valid Let's Encrypt certificate.

## Conclusion

By combining **ExternalDNS**, **Ingress**, and **Cert-Manager**, you can fully automate the setup of public DNS and TLS for Kubernetes workloads. This reduces manual effort and increases reliability — all with a simple set of annotations.
