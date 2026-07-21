# Django Deployment & Infrastructure Playbook

Covers local Docker infrastructure, VPS provisioning, and CI/CD pipelines.

---

## Part 1: Local Docker Infrastructure

### 1.1 Dockerfile (API)

**Goal**: Provide a consistent, lean containerized environment for Django.

Uses `python:3.14-slim` for a balance between size and utility.

*Note*: The Dockerfile does not specify a default `CMD`. Command execution is delegated to the respective Docker Compose environments (`docker-compose.dev.yml` for development and `docker-compose.yml` for production).

```dockerfile
FROM python:3.14-slim AS builder

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --upgrade pip && \
    pip wheel --no-cache-dir --wheel-dir /app/wheels -r requirements.txt

FROM python:3.14-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    curl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /app/wheels /app/wheels
COPY --from=builder /app/requirements.txt .

RUN pip install --upgrade pip && \
    pip install --no-cache-dir --no-index --find-links=/app/wheels -r requirements.txt && \
    rm -rf /app/wheels

RUN groupadd -g 9999 appuser && useradd -u 9999 -g 9999 -m appuser
COPY --chown=appuser:appuser . .
USER appuser
EXPOSE 8000
```

*Important Note on `.dockerignore`:* Always define a `.dockerignore` file in your project root to prevent copying local virtual environments (`venv/`), Git logs (`.git/`), temporary cache files, and sensitive environment configs (`.env`) into the image build context.

*Note on Gunicorn Worker Count:* The standard formula for optimal Gunicorn worker count is `(2 × $num_cores) + 1`. For example, on a 2-core VPS, run `5` workers (configured via the `--workers` flag in the command override under the `web` service).

### 1.2 Docker Compose (Development Stack)

Orchestrates the local development database and the API service.

**Services Overview**

| Service | Purpose | Notes |
|---|---|---|
| `db` | PostgreSQL Database | Local development database |
| `redis` | Message broker & Cache | Used by Celery and Channels |
| `web` | Django API | Runs `runserver` with auto-reload |
| `celery` | Background worker | Runs tasks from `appX/tasks.py` |
| `ngrok` | External Tunnel | Exposes local API for webhooks/mobile |

**Development Configuration (`docker-compose.dev.yml`)**

```yaml
services:
  db:
    image: postgres:18-alpine
    volumes:
      - pgdata_dev:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
      PGDATA: /var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  redis:
    image: redis:7-alpine
    env_file:
      - .env
    command: ["redis-server", "--requirepass", "${REDIS_PASSWORD}"]
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  web:
    build: .
    image: myproject_dev
    command: sh -c "python manage.py migrate --noinput && python manage.py runserver 0.0.0.0:8000"
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    env_file:
      - .env
    environment:
      - DATABASE_URL=postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
      - DEBUG=1
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  celery:
    build: .
    command: celery -A config worker -l info
    volumes:
      - .:/app
    env_file:
      - .env
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  ngrok:
    image: ngrok/ngrok:latest
    ports:
      - "4040:4040"
    env_file:
      - .env
    command: http --url=${NGROK_DOMAIN} web:8000
    depends_on:
      - web
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

volumes:
  pgdata_dev:
```

### 1.3 Connectivity Rules

- **Internal Hostnames:** Services communicate with each other using service names (e.g., `REDIS_URL=redis://:password@redis:6379/0`).
- **Healthchecks:** Use `depends_on` with `service_healthy` to ensure the database and redis are up before Django/Celery start.
- **Port Conflicts:** Ensure local host ports (e.g., `8000`, `6379`, or `5432`) do not collide with other running local processes.

### 1.4 Development Workflow

Start the development stack:
```bash
docker compose -f docker-compose.dev.yml up --build
```
Start in detached mode:
```bash
docker compose -f docker-compose.dev.yml up -d
```

---

## Part 2: Preparing for Production (Codebase)

Before provisioning the server, prepare your repository with the necessary production configuration files.

### 2.1 Project Structure

`Dockerfile`

Uses the unified `Dockerfile` defined in **Part 1** as a reusable blueprint.

`docker-compose.yml`

Deploys the production stack. Pulls pre-built images from GHCR and reads environment configurations from `.env` and `.deploy.env`.

```yaml
x-app-defaults: &app-defaults
  image: ${IMAGE_REPO}:${IMAGE_TAG}
  restart: unless-stopped
  env_file:
    - .env
  environment:
    - DATABASE_URL=postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
  depends_on:
    db:
      condition: service_healthy
    redis:
      condition: service_healthy
  logging:
    driver: "json-file"
    options:
      max-size: "10m"
      max-file: "3"

services:
  db:
    image: postgres:18-alpine
    restart: unless-stopped
    volumes:
      - pgdata:/var/lib/postgresql/data
    env_file:
      - .env
    environment:
      PGDATA: /var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    env_file:
      - .env
    command: ["redis-server", "--requirepass", "${REDIS_PASSWORD}"]
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  web:
    <<: *app-defaults
    command: >
      sh -c "python manage.py migrate --noinput &&
             python manage.py collectstatic --noinput &&
             gunicorn config.wsgi:application --bind 0.0.0.0:8000 --workers 3"
    ports:
      - "127.0.0.1:8000:8000"
    volumes:
      - /opt/myproject/staticfiles:/app/staticfiles
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/api/core/health/"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  celery:
    <<: *app-defaults
    command: celery -A config worker -l info

volumes:
  pgdata:
```

`.env` 
```env
DEBUG=False
ALLOWED_HOSTS=yourdomain.com,www.yourdomain.com
SECRET_KEY=replace-with-50+-char-random-string

POSTGRES_USER=task_management_user
POSTGRES_PASSWORD=strongpassword
POSTGRES_DB=task_management_db
REDIS_PASSWORD=strongredispassword
NGROK_AUTHTOKEN=your_ngrok_token
NGROK_DOMAIN=your-ngrok-domain.ngrok-free.app
```

`settings.py` — production-relevant section
```python
import environ

env = environ.Env(
    DEBUG=(bool, False)
)

SECRET_KEY = env('SECRET_KEY')
DEBUG = env('DEBUG')
ALLOWED_HOSTS = env.list('ALLOWED_HOSTS')

DATABASES = {
    'default': env.db('DATABASE_URL')
}

STATIC_URL = '/static/'
STATIC_ROOT = '/app/staticfiles'
MEDIA_URL = '/media/'
MEDIA_ROOT = '/app/media'
```

---

## Part 3: VPS Provisioning

Provision and configure on Ubuntu 22.04/24.04 LTS. Note the following placeholders used throughout this guide: `deploy` (username), `yourdomain.com`, `myproject`, `mydb`/`myuser`/`strongpassword`.

### 3.1 Initial Server Setup

#### Login as Root
```bash
ssh -i secret.pem root@<SERVER_IP>
```

#### Create Deploy User
Create a new user with sudo privileges and copy root's SSH keys:
```bash
adduser deploy
usermod -aG sudo deploy
rsync --archive --chown=deploy:deploy ~/.ssh /home/deploy
```

#### SSH Hardening
⚠ Keep the current session open and verify key-based login as `deploy` in a second terminal before proceeding.
```bash
sudo nano /etc/ssh/sshd_config
```
Set the following values:
```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
MaxSessions 3
ClientAliveInterval 300
ClientAliveCountMax 2
X11Forwarding no
```
Validate configuration and reload the service:
```bash
sudo sshd -t
sudo systemctl restart ssh
```
Tradeoff: Changing the SSH port reduces scanning noise but adds operational complexity without adding real security. Skip unless scan noise is high.

#### Update System
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ufw fail2ban unattended-upgrades
```

### 3.2 Firewall & Security

#### Firewall (UFW)
Configure UFW, ensuring SSH access is allowed before enabling:
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status verbose
```
⚠ Docker + UFW conflict: Docker bypasses UFW by modifying iptables directly. If binding container ports to `0.0.0.0`, UFW rules will not apply. Fix: bind to `127.0.0.1` in the docker-compose configurations and proxy ingress traffic through the host Nginx.

#### Fail2Ban
Configure default settings:
```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```
Paste under `[DEFAULT]`:
```ini
bantime = 3600
findtime = 600
maxretry = 5
banaction = ufw

[sshd]
enabled = true
port = ssh
backend = systemd
maxretry = 3
bantime = 7200
```
```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

#### Auto Security Updates
```bash
sudo dpkg-reconfigure --priority=low unattended-upgrades
```


### 3.3 Shell Environment

```bash
sudo apt install -y fish
chsh -s $(which fish)
```

### 3.4 Install Core Dependencies (Docker, Nginx, Certbot)

#### Install Docker
```bash
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker deploy
newgrp docker
```

#### Install Nginx
```bash
sudo apt install -y nginx
sudo systemctl enable --now nginx
```

#### Install Certbot
Install Certbot using snap for the latest updates:
```bash
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

---

## Part 4: Initial Server Deployment & CI/CD


Covers automated deployment pipelines using GitHub Actions. Option 1 builds the container image locally on the VPS, while Option 2 builds the image using GitHub runners and hosts it on GHCR.

---

### 4.1 Setup SSH Credentials for GitHub Actions

Configure SSH key authentication to allow the GitHub Actions runner to connect to the VPS.

1. **Generate SSH key pair:**
   ```bash
   ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_actions -N ""
   ```
   This generates two files:
   - A private key: `~/.ssh/github_actions`
   - A public key: `~/.ssh/github_actions.pub`

2. **Authorize the key on the VPS**:
   ```bash
   mkdir -p /home/deploy/.ssh
   cat ~/.ssh/github_actions.pub >> /home/deploy/.ssh/authorized_keys
   chmod 700 /home/deploy/.ssh
   chmod 600 /home/deploy/.ssh/authorized_keys
   ```

3. **Configure GitHub Repository Secrets:**
   Add the following secrets under **Settings → Secrets and variables → Actions**:
   * `SERVER_IP`: The public IP of the VPS.
   * `SSH_USERNAME`: The deployment user (`deploy`).
   * `SSH_PRIVATE_KEY`: The contents of `~/.ssh/github_actions`.
   * `PRO_GHCR_TOKEN` *(Option 2 only)*: Personal Access Token (PAT) with package write permissions.

---

### 4.2 Option 1: Build Image on VPS

Deploys code directly to the VPS and compiles the container image locally.

* **Advantages**: Minimal external setup; no container registry credentials required.
* **Disadvantages**: Build compilation consumes VPS CPU and RAM resources.

#### 4.2.1 Setup GitHub Deploy Keys on VPS
Configure read-only repository access for the VPS:
1. **Generate deploy key:**
   ```bash
   ssh-keygen -t ed25519 -C "deploy" -f ~/.ssh/deploy_backend -N ""
   cat ~/.ssh/deploy_backend.pub
   ```
2. **Configure SSH mapping (`~/.ssh/config`):**
   ```text
   Host github.com
       HostName github.com
       User git
       IdentityFile ~/.ssh/deploy_backend
       IdentitiesOnly yes
   ```
3. **Register public key on GitHub:**
   Add the public key under **Repository Settings → Deploy keys** with read-only access.

#### 4.2.2 Initial VPS Setup
1. **Prepare directories:**
   ```bash
    sudo mkdir -p /opt/myproject/staticfiles /opt/myproject/media
    sudo chown -R deploy:deploy /opt/myproject
    sudo chown -R 9999:9999 /opt/myproject/staticfiles /opt/myproject/media
    cd /opt/myproject
   ```
2. **Clone repository:**
   ```bash
   git clone git@github.com:yourusername/yourrepository.git .
   ```
3. **Configure environment:**
   Create `/opt/myproject/.env` matching `.env.example`.
4. **Ensure `docker-compose.yml` has the build context:**
   ```yaml
   services:
     web:
       build: .
       image: myproject:latest
   ```
5. **Start services:**
   ```bash
   docker compose up -d --build
   ```
6. **Create Django superuser:**
   ```bash
   docker compose exec web python manage.py createsuperuser
   ```

#### 4.2.3 Manual Deploy & Updates
```bash
cd /opt/myproject
git pull origin main
docker compose up -d --build --remove-orphans
docker compose exec web python manage.py migrate --noinput
```

#### 4.2.4 Automated GitHub Actions Workflow (`.github/workflows/deploy.yml`)
```yaml
name: Deploy Server to VPS (Build on VPS)

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.2.0
        with:
          host: ${{ secrets.SERVER_IP }}
          username: ${{ secrets.SSH_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/myproject
            git pull origin main
            docker compose build
            docker compose up -d --remove-orphans
            docker compose exec -T web python manage.py migrate --noinput
```

---

### 4.3 Option 2: Use GHCR (GitHub Container Registry)

Recommended production workflow. Compiles the image using GitHub Actions runners, pushes it to GHCR, and triggers the VPS to pull the pre-built image.

* **Advantages**: Zero VPS CPU compilation overhead; built-in health checks and automatic rollback on failure.

#### 4.3.1 Initial VPS Setup & First Deploy
1. **Prepare directories:**
   ```bash
    sudo mkdir -p /opt/myproject/staticfiles /opt/myproject/media
    sudo chown -R deploy:deploy /opt/myproject
    sudo chown -R 9999:9999 /opt/myproject/staticfiles /opt/myproject/media
    cd /opt/myproject
   ```
2. **Configure environment:**
   Create `/opt/myproject/.env` matching `.env.example`.
3. **Trigger deployment:**
   Push a commit to the `main` branch to trigger the automated CI/CD pipeline.
4. **Verify container status:**
   ```bash
   cd /opt/myproject
   docker compose --env-file .env --env-file .deploy.env ps
   ```


#### 4.3.2 Automated GitHub Actions Workflow (`.github/workflows/deploy.yml`)
```yaml
name: Deploy Server to VPS

on:
  push:
    branches:
      - main

concurrency:
  group: production-deploy
  cancel-in-progress: false

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: yourusername/yourrepository

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image_tag: ${{ steps.meta.outputs.tag }}
    steps:
      - uses: actions/checkout@v6

      - id: meta
        run: echo "tag=${GITHUB_SHA::12}" >> "$GITHUB_OUTPUT"

      - uses: docker/setup-buildx-action@v4

      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: yourusername
          password: ${{ secrets.PRO_GHCR_TOKEN }}

      - uses: docker/build-push-action@v7
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
      - uses: actions/checkout@v6

      - name: Sync compose file to VPS
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_IP }}
          username: ${{ secrets.SSH_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "docker-compose.yml"
          target: "/opt/myproject"

      - name: Deploy on VPS
        uses: appleboy/ssh-action@v1.2.0
        env:
          IMAGE_TAG: ${{ needs.build-and-push.outputs.image_tag }}
          IMAGE_REPO: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          GHCR_USER: yourusername
          GHCR_TOKEN: ${{ secrets.PRO_GHCR_TOKEN }}
        with:
          host: ${{ secrets.SERVER_IP }}
          username: ${{ secrets.SSH_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          envs: IMAGE_TAG,IMAGE_REPO,GHCR_USER,GHCR_TOKEN
          script: |
            bash -c '
            set -e
            mkdir -p /opt/myproject
            cd /opt/myproject

            echo "$GHCR_TOKEN" | docker login ghcr.io -u "$GHCR_USER" --password-stdin

            PREV_TAG=$(grep -oP "^IMAGE_TAG=\K.*" .deploy.env 2>/dev/null || \
               echo "latest")

            cat > .deploy.env <<EOF
            IMAGE_REPO=$IMAGE_REPO
            IMAGE_TAG=$IMAGE_TAG
            EOF

            docker compose \
              --env-file .env --env-file .deploy.env pull

            docker compose \
              --env-file .env --env-file .deploy.env up -d --remove-orphans

            ATTEMPTS=0
            until [ "$(docker inspect -f "{{.State.Health.Status}}" \
              $(docker compose --env-file .env --env-file .deploy.env ps -q web) \
              2>/dev/null)" = "healthy" ]; do
              ATTEMPTS=$((ATTEMPTS+1))
              if [ "$ATTEMPTS" -ge 12 ]; then
                echo "Health check failed — rolling back to $PREV_TAG"
                sed -i \
                  "s/^IMAGE_TAG=.*/IMAGE_TAG=$PREV_TAG/" .deploy.env
                docker compose \
                  --env-file .env --env-file .deploy.env up -d
                exit 1
              fi
              sleep 5
            done

            docker image prune -af --filter "until=72h"
            '
```

#### 4.3.3 Sourcing Default Environment Files
Use the `COMPOSE_ENV_FILES` variable (requires Docker Compose v2.25+) to avoid passing multiple `--env-file` arguments during manual troubleshooting:

1. **Configure shell profile:**
   ```bash
   echo 'export COMPOSE_ENV_FILES=".env,.deploy.env"' >> ~/.bashrc
   source ~/.bashrc
   ```
2. **Test configuration:**
   Verify standard docker compose commands (e.g. `docker compose ps`) run without environment warnings.

---

## Part 5: Web Server & SSL

Now that the application is deployed and the containers are running, configure the web server and secure it.

### 5.1 Configure Nginx Server Block

#### Configure Server Block
Create `/etc/nginx/sites-available/myproject` to proxy traffic to the Django container:
```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    location /static/ {
        alias /opt/myproject/staticfiles/;
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    location /media/ {
        alias /opt/myproject/media/;
        access_log off;
    }

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
        proxy_read_timeout 90;
    }
    client_max_body_size 20M;
}
```
Enable the configuration by creating a symlink, delete the default site configuration, validate the syntax, and reload Nginx:
```bash
sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

### 5.2 SSL with Certbot

#### Prerequisites
- DNS A record pointing `yourdomain.com` → your VPS IP
- Port 80 accessible (Certbot uses HTTP-01 challenge)


#### Obtain Certificate
```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```
Certbot execution handles:
- Domain ownership verification via HTTP-01 challenge.
- Certificate generation to `/etc/letsencrypt/live/yourdomain.com/`.
- Automatic Nginx server block configuration for TLS.
- HTTP to HTTPS redirection.


#### Harden TLS Config
Add or verify the following inside the Nginx `listen 443` server block:
```nginx

# OCSP Stapling
ssl_stapling on;
ssl_stapling_verify on;
resolver 1.1.1.1 8.8.8.8 valid=300s;
resolver_timeout 5s;

# Security Headers
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header X-Frame-Options "DENY" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```
⚠ Do not add `preload` to HSTS until HTTPS is stable across all subdomains.

```bash
sudo nginx -t && sudo systemctl reload nginx
```

#### Verify Auto-Renewal
Certbot registers a systemd timer on snap installs. Verify that the timer is active:
```bash
systemctl list-timers | grep certbot
```
Test the dry-run renewal process to confirm verification paths work:
```bash
sudo certbot renew --dry-run
```
Renewal runs twice daily and only triggers when a certificate is within 30 days of expiry.

Optional: Create a post-renewal hook to reload Nginx automatically:
Create `/etc/letsencrypt/renewal-hooks/post/reload-nginx.sh`:
```bash
#!/bin/bash
systemctl reload nginx
```
```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh
```


---

## Part 6: Quick Reference — Useful Commands

| Task | Command |
|---|---|
| Nginx config test | `sudo nginx -t` |
| Reload Nginx | `sudo systemctl reload nginx` |
| View Nginx errors | `sudo tail -f /var/log/nginx/error.log` |
| Docker: view logs | `docker compose logs -f web` |
| Docker: restart app | `docker compose restart web` |
| Check SSL expiry | `sudo certbot certificates` |
| Check open ports | `sudo ss -tlnp` |
| UFW status | `sudo ufw status numbered` |
| Fail2Ban SSH bans | `sudo fail2ban-client status sshd` |

