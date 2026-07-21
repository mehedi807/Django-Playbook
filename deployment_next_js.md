# Next.js Application Deployment Playbook

This playbook covers containerization, VPS hosting, host Nginx configuration, and CI/CD pipelines for Next.js applications deploying as isolated stacks behind a host-level Nginx reverse proxy.

### Key Security & Isolation Design Choices
1. **Host Nginx Ingress**: Nginx is the only entry point from the internet.
2. **Loopback Binding**: Container ports bind only to `127.0.0.1`.
3. **Isolated Directory Stacks**: Each app contains its own `docker-compose.yml`, `.env`, and deployment files under `/opt/your-app-name`.

---

## Part 1: Preparing the Codebase for Production

### 1.1 Next.js Configuration (`next.config.ts` / `next.config.js`)

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  output: "standalone",
};

export default nextConfig;
```


### 1.2 Multi-Stage Dockerfile

```dockerfile
FROM node:22-alpine AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci

FROM node:22-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs
COPY --from=builder /app/public ./public
RUN mkdir .next
RUN chown nextjs:nodejs .next
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
USER nextjs
EXPOSE 3000
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"
CMD ["node", "server.js"]
```


### 1.3 Docker Ignore Configuration (`.dockerignore`)

```text
.next
node_modules
out
build
.git
.github
README.md
```


### 1.4 Repository Docker Compose Configuration (`docker-compose.yml`)

Create this file in the root of the local Git repository (GitHub Actions will sync it to `/opt/your-app-name/docker-compose.yml` during deployment):

```yaml
services:
  web:
    image: ${IMAGE_REPO}:${IMAGE_TAG}
    restart: unless-stopped
    ports:
      - "127.0.0.1:${HOST_PORT}:3000"
    env_file:
      - .env
    healthcheck:
      test:
        - "CMD-SHELL"
        - "node -e \"const http = require('http'); http.get('http://localhost:3000', (res) => { if (res.statusCode === 200) { process.exit(0); } else { process.exit(1); } }).on('error', (err) => { process.exit(1); });\""
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 15s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```


---

## Part 2: Initial Server Preparation

### 2.1 VPS Directory Setup

```bash
sudo mkdir -p /opt/your-app-name
sudo chown -R $(whoami):$(whoami) /opt/your-app-name
```


### 2.2 VPS Environment File (`/opt/your-app-name/.env`)

Create this file manually in the application directory on the VPS:

```env
PORT=3000
HOST_PORT=3000
NEXT_PUBLIC_API_URL=https://api.yourdomain.com
```

---

---

## Part 3: CI/CD Pipeline & Initial Deployment

Push your code to GitHub to trigger the first deployment. This will start the Docker container so Nginx can proxy to it.

### 3.1 GitHub Repository Secrets

* `SERVER_IP`
* `SSH_USERNAME`
* `SSH_PRIVATE_KEY`
* `PRO_GHCR_TOKEN`


### 3.2 GitHub Actions Workflow (`.github/workflows/deploy.yml`)

```yaml
name: Build and Deploy Next.js App

on:
  push:
    branches:
      - main

concurrency:
  group: deploy-${{ github.workflow }}
  cancel-in-progress: false

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: yourusername/yourrepository
  VPS_DIR: /opt/your-app-name 

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image_tag: ${{ steps.meta.outputs.tag }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v6

      - name: Generate short SHA tag
        id: meta
        run: echo "tag=${GITHUB_SHA::12}" >> "$GITHUB_OUTPUT"

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: yourusername
          password: ${{ secrets.PRO_GHCR_TOKEN }}

      - name: Pull .env from VPS
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan -H ${{ secrets.SERVER_IP }} >> ~/.ssh/known_hosts
          scp -i ~/.ssh/id_rsa ${{ secrets.SSH_USERNAME }}@${{ secrets.SERVER_IP }}:${{ env.VPS_DIR }}/.env .env

      - name: Build and Push Docker Image
        uses: docker/build-push-action@v7
        with:
          context: .
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.tag }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: read
    steps:
      - name: Checkout code
        uses: actions/checkout@v6

      - name: Sync compose file to VPS via SCP
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_IP }}
          username: ${{ secrets.SSH_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "docker-compose.yml"
          target: ${{ env.VPS_DIR }}

      - name: Deploy via SSH (Zero-Downtime Rolling Update)
        uses: appleboy/ssh-action@v1.2.0
        env:
          IMAGE_TAG: ${{ needs.build-and-push.outputs.image_tag }}
          IMAGE_REPO: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          GHCR_USER: yourusername
          GHCR_TOKEN: ${{ secrets.PRO_GHCR_TOKEN }}
          VPS_DIR: ${{ env.VPS_DIR }}
        with:
          host: ${{ secrets.SERVER_IP }}
          username: ${{ secrets.SSH_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          envs: IMAGE_TAG,IMAGE_REPO,GHCR_USER,GHCR_TOKEN,VPS_DIR
          script: |
            bash -c '
            set -e
            cd "$VPS_DIR"

            echo "$GHCR_TOKEN" | docker login ghcr.io -u "$GHCR_USER" --password-stdin

            PREV_TAG=$(grep -oP "^IMAGE_TAG=\K.*" .deploy.env 2>/dev/null || echo "latest")

            cat > .deploy.env <<EOF
            IMAGE_REPO=$IMAGE_REPO
            IMAGE_TAG=$IMAGE_TAG
            EOF

            docker compose --env-file .env --env-file .deploy.env pull
            docker compose --env-file .env --env-file .deploy.env up -d --remove-orphans

            ATTEMPTS=0
            HEALTHY=false
            until [ "$HEALTHY" = true ]; do
              STATUS=$(docker inspect -f "{{.State.Health.Status}}" \
                $(docker compose --env-file .env --env-file .deploy.env ps -q) 2>/dev/null || echo "starting")
              
              if [ "$STATUS" = "healthy" ]; then
                echo "Deployment health check passed!"
                HEALTHY=true
              elif [ "$STATUS" = "unhealthy" ] || [ "$ATTEMPTS" -ge 12 ]; then
                echo "Deployment failed health check ($STATUS) — rolling back to $PREV_TAG"
                sed -i "s/^IMAGE_TAG=.*/IMAGE_TAG=$PREV_TAG/" .deploy.env
                docker compose --env-file .env --env-file .deploy.env up -d
                exit 1
              else
                echo "Waiting for container to become healthy (current status: $STATUS)..."
                ATTEMPTS=$((ATTEMPTS+1))
                sleep 5
              fi
            done

            docker image prune -af --filter "until=72h"
            '
```


---

## Part 4: Web Server & SSL Configuration

### 4.1 Nginx Server Block Configuration (`/etc/nginx/sites-available/your-app-name`)

```nginx
server {
    listen 80;
    server_name yourdomain.com;

    gzip on;
    gzip_proxied any;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
    gzip_vary on;

    location /_next/static/ {
        proxy_pass http://127.0.0.1:3000;
        access_log off;
    }

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
sudo ln -sf /etc/nginx/sites-available/your-app-name /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```


### 4.2 SSL with Certbot

```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

---

---

## Part 5: Deployment Operations Reference

### 5.1 Setting Shell Environment Properties

```bash
echo 'export COMPOSE_ENV_FILES=".env,.deploy.env"' >> ~/.bashrc
source ~/.bashrc
```

### 5.2 Checking Local VPS Listeners

```bash
sudo ss -tlnp
```
