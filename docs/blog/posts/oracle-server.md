---
date: 2025-02-26
categories:
  - devops
  - shell script
  - ssl
  - docker
  - oracle
---
# Setting Up Docker, SSL, and DuckDNS on Oracle Server

This guide will walk you through the process of setting up Docker, configuring SSL certificates, and setting up DuckDNS on an Oracle server (Oracle Linux 9).

![Oracle Server](../../assets/blog/oracle-server/banner.jpg)

<!-- more -->

## 1. Update and Install Docker

First, ensure your system is up-to-date and install Docker.

``` bash
sudo dnf update -y
sudo dnf install docker -y
```

Remove Podman and install docker

``` bash
sudo dnf remove -y podman
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io
```

Enable Docker to start on boot and start the service.

``` bash
sudo systemctl enable docker
sudo systemctl start docker
```

Add Current User to Docker Group

``` bash
sudo usermod -aG docker $USER
newgrp docker
```

Verify Docker Installation
``` bash
docker --version
```

## 2. Configure Firewall to Allow HTTP/HTTPS Ports

If you’re running a web server or using Docker containers exposed on HTTP/HTTPS, open the necessary ports.

``` bash
sudo firewall-cmd --add-port=80/tcp --permanent
sudo firewall-cmd --add-port=443/tcp --permanent
sudo firewall-cmd --reload
```

## 3. Set Up SSL Certificates Using Certbot

Install Certbot, which is a tool that helps you obtain SSL certificates for your domain.

``` bash
sudo dnf install epel-release -y
sudo dnf config-manager --set-enabled ol9_codeready_builder
sudo dnf install python3 python3-pip -y
sudo pip3 install certbot
```

Obtain SSL Certificate
Run Certbot to obtain an SSL certificate for your domain.

``` bash
sudo /usr/local/bin/certbot certonly --standalone -d jobwinner.duckdns.org
```

Add a cron job to automatically renew the SSL certificate every 90 days.
``` bash
(crontab -l ; echo "0 3 * * * certbot renew --quiet") | crontab -
```

## 4. Optionally generate PKCS12 file for Java App

``` bash
openssl pkcs12 -export -in your_certificate.crt -inkey your_private.key -out keystore.p12 -name selfsigned
```

It can also be automaed using certbot post hook with the following script (generate-p12.sh)

``` bash
#!/bin/bash

# Paths to the Let's Encrypt certificate and private key
CERT_PATH="/etc/letsencrypt/live/jobwinner.duckdns.org/fullchain.pem"
KEY_PATH="/etc/letsencrypt/live/jobwinner.duckdns.org/privkey.pem"
OUTPUT_PATH="/etc/letsencrypt/live/jobwinner.duckdns.org/keystore.p12"

# Ensure the output directory exists
mkdir -p $(dirname "$OUTPUT_PATH")

# Convert the certificate and key to PKCS12 format
openssl pkcs12 -export -in "$CERT_PATH" -inkey "$KEY_PATH" -out "$OUTPUT_PATH" -name selfsigned -password pass:yourpassword

echo "PKCS12 keystore created at $OUTPUT_PATH"
```

``` bash
chmod +x /usr/local/bin/generate-p12.sh
sudo certbot renew --deploy-hook "/usr/local/bin/generate-p12.sh"

```

## 5. Set Up DuckDNS for Dynamic DNS

If you are using DuckDNS for dynamic DNS (e.g., for a home server or a non-static IP), you will need to set up a script to update your IP regularly.

Install DuckDNS Script
Create a directory for DuckDNS and navigate to it.


``` bash
mkdir -p ~/duckdns && cd ~/duckdns
```

Create a script to update your DuckDNS record:
``` bash
vi duck.sh
echo url="https://www.duckdns.org/update?domains=yourname&token=your-duckdns-token&ip=" | curl -k -o ~/duckdns/duck.log -K -
chmod +x duck.sh
(crontab -l ; echo "*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1") | crontab -
```

## 5. Summary

- Docker is installed and configured.
- Ports 80 (HTTP) and 443 (HTTPS) are open for web traffic.
- SSL certificates are obtained via Certbot and set up to renew automatically.
- DuckDNS is set up to update your domain's IP address regularly.