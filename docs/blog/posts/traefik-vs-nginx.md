---
date: 2025-06-12
categories:
  - traefik
  - ssl
  - docker
  - devops
---
# Goodbye Nginx, Hello Traefik! Effortless HTTPS with Let's Encrypt and Docker

If you've struggled with Nginx reverse proxy configs, certbot timers, and `nginx -s reload`, it's time to meet Traefik — a modern reverse proxy built for dynamic containerized environments.

## Why Traefik over Nginx?

Unlike Nginx, which requires manual configuration updates and reloads, **Traefik auto-discovers services via Docker labels**, keeping your proxy config in sync with running containers. It also:

- Automatically obtains and renews **Let’s Encrypt certificates**
- Handles **HTTP/HTTPS routing**, path-based rules, load balancing, and more
- Supports **metrics, tracing**, and even **canary deployments** with Traefik Enterprise

For small setups or demos, it’s a powerful, drop-in Nginx replacement — with less boilerplate.

![Traefik vs Nginx](../../assets/blog/traefik-vs-nginx/banner.jpg)

## Old Way: Nginx + Certbot = Pain

Let’s say you want to proxy a backend running on port 8080 and a frontend on port 3000 — both behind HTTPS. With **Nginx**, you’d need something like:

### Nginx Config (static and fragile)

```nginx
server {
  listen 80;
  server_name example.com;
  location / {
    return 301 https://$host$request_uri;
  }
}

server {
  listen 443 ssl;
  server_name example.com;

  ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

  location / {
    proxy_pass http://frontend:3000;
  }

  location /graphql {
    proxy_pass http://backend:8080;
  }
}
```

Then you need to:

- Install Certbot
- Run it to issue the cert (possibly with a temporary port 80 server)
- Configure a cronjob for renewals
- Manually reload Nginx on changes
- Map all cert and config files into your container (if containerized)

It works — but it’s manual, brittle, and not fun to maintain in Docker-land.

## The New Way: Traefik + Docker Labels

Let's walk through setting up Traefik to:

- Automatically detect frontend and backend services
- Use Let's Encrypt for HTTPS
- Route traffic using Docker labels
- Auto-renew certificates

### Folder structure

```
project-root/
├── docker-compose.yml
├── traefik.yml
└── acme.json
```

### `traefik.yml`

```yaml
log:
  level: INFO

accessLog: {}

api:
  dashboard: false

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https

  websecure:
    address: ":443"

certificatesResolvers:
  letsencrypt:
    acme:
      email: your-email@example.com
      storage: /acme.json
      httpChallenge:
        entryPoint: web

providers:
  docker:
    exposedByDefault: false
```

### `docker-compose.yml`

```yaml
version: "3.9"

services:
  traefik:
    image: traefik:v2.11
    container_name: traefik
    command:
      - "--configFile=/traefik.yml"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./traefik.yml:/traefik.yml:ro
      - ./acme.json:/acme.json
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - app_network

  frontend:
    image: your-frontend-image
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.frontend.rule=Host(`example.com`)"
      - "traefik.http.routers.frontend.entrypoints=websecure"
      - "traefik.http.routers.frontend.tls.certresolver=letsencrypt"
      - "traefik.http.services.frontend.loadbalancer.server.port=80"
    networks:
      - app_network

  backend:
    image: your-backend-image
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.backend.rule=Host(`example.com`) && PathPrefix(`/graphql`)"
      - "traefik.http.routers.backend.entrypoints=websecure"
      - "traefik.http.routers.backend.tls.certresolver=letsencrypt"
      - "traefik.http.services.backend.loadbalancer.server.port=8080"
    networks:
      - app_network

networks:
  app_network:
    driver: bridge
```

### Setup Notes

- Create `acme.json` file and set permissions:

  ```bash
  touch acme.json
  chmod 600 acme.json
  ```

- Replace `example.com` with your domain.
- Replace image names with your real images.

## Result

Traefik discovers containers via Docker labels.
It fetches valid SSL certs from Let’s Encrypt automatically.
You can view the status and routes on https://<your-domain>:8080.
No more writing separate Nginx conf files or restarting on config changes.

## Final Thoughts

Traefik makes it *dead simple* to get up and running with HTTPS in Docker. You get automatic discovery, dynamic config reloads, and zero-downtime cert renewal — with one clean configuration file.

It's time to say goodbye to nginx.conf and embrace a modern, container-native proxy.
