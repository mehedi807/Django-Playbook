# Django Deployment & Infrastructure Playbook

This comprehensive guide covers local Docker infrastructure, VPS provisioning, and CI/CD deployment pipelines.

---

## Part 1: Local Docker Infrastructure

### 1.1 Dockerfile (API & Workers)

**Goal:** Provide a consistent, production-ready environment for Django, Celery, and other Python-based services.

Use `python:3.12-slim` for a balance between size and utility.

*Note: The Dockerfile is built to be production-ready by default (using Gunicorn). Local development will override this command in `docker-compose.yml`.*

```dockerfile
FROM python:3.12-slim

# Prevents Python from writing pyc files to disc
ENV PYTHONDONTWRITEBYTECODE=1
# Prevents Python from buffering stdout and stderr
ENV PYTHONUNBUFFERED=1

WORKDIR /app

# Install system dependencies (build-essential for C extensions, libpq-dev for DB)
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Install dependencies first for better caching
COPY requirements.txt /app/
RUN pip install --upgrade pip && pip install -r requirements.txt

# Copy project files
COPY . /app/

# Collect static files for production
RUN python manage.py collectstatic --noinput

EXPOSE 8000

# Default command runs Gunicorn (production)
CMD ["gunicorn", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "3", \
     "--timeout", "60", \
     "myproject.wsgi:application"]
```

### 1.2 Docker Compose (Local Stack)

The `docker-compose.yml` orchestrates the API, background workers, and shared services (Redis).

**Services Overview**

| Service | Purpose | Notes |
|---|---|---|
| `db` | PostgreSQL Database | Local development database |
| `redis` | Message broker & Cache | Used by Celery and Channels |
| `web` | Django API | Runs `runserver` with auto-reload |
| `celery` | Background worker | Runs tasks from `appX/tasks.py` |
| `ngrok` | External Tunnel | Exposes local API for webhooks/mobile |

**Example Configuration**

```yaml
services:
  db:
    image: postgres:16-alpine
    env_file:
      - .env
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    env_file:
      - .env
    command: ["redis-server", "--requirepass", "${REDIS_PASSWORD}"]
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  web:
    build: .
    command: sh -c "python manage.py migrate && python manage.py runserver 0.0.0.0:8000"
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    env_file:
      - .env
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/')"]
      interval: 10s
      timeout: 5s
      retries: 3
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

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

volumes:
  postgres_data:
```

### 1.3 Connectivity Rules

- **Internal Hostnames:** Services talk to each other using service names (e.g., `REDIS_URL=redis://:password@redis:6379/0`).
- **Healthchecks:** Always use `depends_on` with `service_healthy` to ensure DB and Redis are up before Django/Celery start.
- **Port Conflicts:** Ensure `ports` mappings (e.g., `8001:8000`) don't collide with other local projects.

### 1.4 Ngrok Tunneling

For testing webhooks (like Stripe or Twilio) or local mobile development, use an integrated `ngrok` service.

```yaml
  ngrok:
    image: ngrok/ngrok:latest
    ports: 
      - "4040:4040"
    env_file:
      - .env
    command: http web:8000 --domain=${NGROK_DOMAIN}
    depends_on:
      - web
```

**Workflow:**
1. Set `NGROK_AUTHTOKEN` in `.env`.
2. Start the stack: `docker compose up`.
3. Visit the dashboard at `localhost:4040` to see traffic.

---

## Part 2: VPS Provisioning & Deployment

Ubuntu 22.04/24.04 LTS. Covers Docker and non-Docker paths. Note the following placeholders used throughout this guide: `deploy` (username), `yourdomain.com`, `myproject`, `mydb`/`myuser`/`strongpassword`.

### 2.1 Initial Server Setup

#### Login as Root
```bash
ssh -i secret.pem root@<SERVER_IP>
```

#### Create Deploy User
```bash
adduser deploy
usermod -aG sudo deploy
# Copy root's authorized_keys to new user (preserves key-based SSH access)
rsync --archive --chown=deploy:deploy ~/.ssh /home/deploy
```

#### SSH Hardening
⚠ It is highly recommended to keep the current session open and verify key-based login as `deploy` in a second terminal before proceeding.
```bash
sudo nano /etc/ssh/sshd_config
```
Set these values (uncomment if needed):
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
```bash
sudo sshd -t # validate config before restarting
sudo systemctl restart ssh
```
Tradeoff: Changing the SSH port (e.g. `Port 2222`) reduces automated scan noise but adds operational friction and offers no real security — a port scan finds it in seconds. Reasonable to skip unless noise is measurable.

#### Update System
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ufw fail2ban unattended-upgrades
```

#### Firewall (UFW)
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp # SSH — do this FIRST
sudo ufw allow 80/tcp # HTTP
sudo ufw allow 443/tcp # HTTPS
sudo ufw enable
sudo ufw status verbose
```
⚠ Docker + UFW conflict: Docker bypasses UFW by modifying iptables directly. If you bind container ports to `0.0.0.0`, UFW rules won't apply. Fix: bind to `127.0.0.1` in `docker-compose` (covered in §3) and let host Nginx handle ingress.

#### Fail2Ban
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

#### Fish Shell
```bash
sudo apt install -y fish
chsh -s $(which fish) # applies on next login
```
Switch back anytime: `chsh -s $(which bash)`

### 2.2 Django Setup — With Docker

#### Install Docker
```bash
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker deploy
newgrp docker # apply group without logout
docker --version # verify
```

#### Project Structure

`Dockerfile`

Use the unified, production-ready `Dockerfile` defined in **Part 1 (§1.1)**. Since it includes the default production command (`gunicorn`) and collects static files, it is ready for deployment without modifications.

Workers formula: `(2 × CPU cores) + 1`. On a 2-core VPS = 5 workers. More workers ≠ better — they consume RAM. Monitor under load and adjust.

`docker-compose.yml`
```yaml
services:
  db:
    image: postgres:16-alpine
    restart: unless-stopped
    env_file: .env
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  web:
    build: .
    restart: unless-stopped
    env_file: .env
    expose:
      - "8000" # NOT ports — avoids UFW bypass
    ports:
      - "127.0.0.1:8000:8000" # host Nginx proxies to this
    volumes:
      - static_volume:/app/staticfiles
      - media_volume:/app/media
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/')"]
      interval: 30s
      timeout: 10s
      retries: 3
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-network

volumes:
  postgres_data:
  static_volume:
  media_volume:

networks:
  app-network:
    driver: bridge
```
Why no Nginx container here? Host Nginx handles SSL termination, manages multiple vhosts, and has direct access to certbot's `/etc/letsencrypt`. Containerized Nginx adds a layer for no gain in single-app setups.

`.env` 
```env
SECRET_KEY=replace-with-50+-char-random-string
DEBUG=False
ALLOWED_HOSTS=yourdomain.com,www.yourdomain.com
POSTGRES_DB=mydb
POSTGRES_USER=myuser
POSTGRES_PASSWORD=strongpassword
REDIS_PASSWORD=strongredispassword
NGROK_AUTHTOKEN=your_ngrok_token
NGROK_DOMAIN=your-ngrok-domain.ngrok-free.app
CSRF_TRUSTED_ORIGINS=https://yourdomain.com,https://www.yourdomain.com
CORS_ALLOWED_ORIGINS=https://yourdomain.com,https://yourfrontend.com
CORS_ALLOW_ALL_ORIGINS=False
```

`settings.py` — production-relevant section
```python
import os

SECRET_KEY = os.environ.get('SECRET_KEY')
DEBUG = os.environ.get('DEBUG', 'False') == 'True'
ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', '').split(',')

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('POSTGRES_DB'),
        'USER': os.environ.get('POSTGRES_USER'),
        'PASSWORD': os.environ.get('POSTGRES_PASSWORD'),
        'HOST': 'db', # Docker service name
        'PORT': '5432',
    }
}

STATIC_URL = '/static/'
STATIC_ROOT = '/app/staticfiles'
MEDIA_URL = '/media/'
MEDIA_ROOT = '/app/media'

CSRF_TRUSTED_ORIGINS = os.environ.get('CSRF_TRUSTED_ORIGINS', '').split(',')

# CORS Settings (Requires 'django-cors-headers' in INSTALLED_APPS & MIDDLEWARE)
CORS_ALLOWED_ORIGINS = [origin for origin in os.environ.get('CORS_ALLOWED_ORIGINS', '').split(',') if origin]
CORS_ALLOW_ALL_ORIGINS = os.environ.get('CORS_ALLOW_ALL_ORIGINS', 'False') == 'True'
```

### 2.3 Django Setup — Without Docker

#### Install System Dependencies
```bash
sudo apt install -y python3 python3-venv python3-pip python3-dev \
postgresql postgresql-contrib libpq-dev nginx git
```

#### Configure PostgreSQL
```bash
sudo -u postgres psql
```
```sql
CREATE DATABASE mydb;
CREATE USER myuser WITH PASSWORD 'strongpassword';
ALTER ROLE myuser SET client_encoding TO 'utf8';
ALTER ROLE myuser SET default_transaction_isolation TO 'read committed';
ALTER ROLE myuser SET timezone TO 'UTC';
GRANT ALL PRIVILEGES ON DATABASE mydb TO myuser;
\q
```

#### Gunicorn Config
Create `/opt/myproject/gunicorn_config.py`:
```python
import multiprocessing

bind = "unix:/run/gunicorn.sock"
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = "gthread"
threads = 2
timeout = 60
keepalive = 5
loglevel = "info"

# Logs
accesslog = "-" # stdout → journald
errorlog = "-"

# Graceful restart
max_requests = 1000
max_requests_jitter = 50
```
Unix socket vs TCP: Unix socket is faster for same-host Nginx↔Gunicorn communication. Use TCP (`127.0.0.1:8000`) only if they'll ever live on different machines.

#### Systemd Services

`/etc/systemd/system/gunicorn.socket`
```ini
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock
SocketUser=www-data

[Install]
WantedBy=sockets.target
```

`/etc/systemd/system/gunicorn.service`
```ini
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
User=deploy
Group=www-data
WorkingDirectory=/opt/myproject
EnvironmentFile=/opt/myproject/.env
ExecStart=/opt/myproject/venv/bin/gunicorn -c gunicorn_config.py myproject.wsgi:application
ExecReload=/bin/kill -s HUP $MAINPID
Restart=on-failure
KillMode=mixed

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now gunicorn.socket
sudo systemctl enable --now gunicorn
sudo systemctl status gunicorn
```

Verify socket exists:
```bash
file /run/gunicorn.sock
# should output: /run/gunicorn.sock: socket
```

### 2.4 Nginx Setup

#### Install Nginx
```bash
sudo apt install -y nginx
sudo systemctl enable --now nginx
```

#### Configure Server Block

Create `/etc/nginx/sites-available/myproject`:

Docker path (proxy to `127.0.0.1:8000`):
```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    # Static files served directly (mounted Docker volume)
    location /static/ {
        alias /opt/myproject/staticfiles/; # or your volume mount path
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

Non-Docker path (proxy to Unix socket):
```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    location = /favicon.ico { access_log off; log_not_found off; }

    location /static/ {
        root /opt/myproject;
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    location /media/ {
        root /opt/myproject;
        access_log off;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
        proxy_read_timeout 90;
    }
    client_max_body_size 20M;
}
```

```bash
sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default # remove default
sudo nginx -t # ALWAYS validate before reload
sudo systemctl reload nginx
```

### 2.5 SSL with Certbot

#### Prerequisites
- DNS A record pointing `yourdomain.com` → your VPS IP
- Port 80 accessible (Certbot uses HTTP-01 challenge)

#### Install Certbot
```bash
# snap version is recommended — gets updates fastest
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

#### Obtain Certificate
```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```
Certbot will:
- Verify domain ownership via HTTP-01 challenge
- Issue the certificate to `/etc/letsencrypt/live/yourdomain.com/`
- Automatically edit your Nginx server block to add TLS config
- Set up HTTP → HTTPS redirect

#### Harden TLS Config
After certbot modifies your config, add/verify these in the `listen 443` block:
```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers on;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;
ssl_session_tickets off;
ssl_stapling on;
ssl_stapling_verify on;
resolver 1.1.1.1 8.8.8.8 valid=300s;
resolver_timeout 5s;

# Security headers
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header X-Frame-Options "DENY" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header X-XSS-Protection "1; mode=block" always;
```
⚠ Don't add `preload` to HSTS until you're confident HTTPS is stable across all subdomains — it's hard to undo.

```bash
sudo nginx -t && sudo systemctl reload nginx
```

#### Verify Auto-Renewal
Certbot registers a systemd timer on snap installs:
```bash
# Check timer is active
systemctl list-timers | grep certbot

# Dry-run renewal (no certs replaced, confirms the process works)
sudo certbot renew --dry-run
```
Renewal runs twice daily and only triggers when cert is within 30 days of expiry.

Optional: Post-renewal hook — create `/etc/letsencrypt/renewal-hooks/post/reload-nginx.sh`:
```bash
#!/bin/bash
systemctl reload nginx
```
```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh
```

### 2.6 Quick Reference — Useful Commands

| Task | Command |
|---|---|
| Check Gunicorn | `sudo systemctl status gunicorn` |
| Reload Gunicorn (zerodowntime) | `sudo systemctl reload gunicorn` |
| Nginx config test | `sudo nginx -t` |
| Reload Nginx | `sudo systemctl reload nginx` |
| View Nginx errors | `sudo tail -f /var/log/nginx/error.log` |
| Docker: view logs | `docker compose logs -f web` |
| Docker: restart app | `docker compose restart web` |
| Check SSL expiry | `sudo certbot certificates` |
| Check open ports | `sudo ss -tlnp` |
| UFW status | `sudo ufw status numbered` |
| Fail2Ban SSH bans | `sudo fail2ban-client status sshd` |

---

## Part 3: CI/CD & Deployment Workflow

This section covers setting up deployment keys for pulling code from GitHub, and the workflow for deploying updates to your VPS, both for Docker and non-Docker setups.

### 3.1 Setup GitHub Deploy Keys

This allows your VPS to pull code from a private GitHub repository securely.

#### Generate Deploy keys on VPS:
```bash
ssh-keygen -t ed25519 -C "deploy" -f ~/.ssh/deploy_backend -N ""
cat ~/.ssh/deploy_backend.pub
```

#### Config SSH on VPS (`~/.ssh/config`):
```text
Host github-backend
    HostName github.com
    User git
    IdentityFile ~/.ssh/deploy_backend
    IdentitiesOnly yes
```

#### Add Deploy keys to GitHub:
Go to your repository on GitHub → Settings → Deploy keys → Add deploy key, and paste the output of the public key.

### 3.2 Initial Application Deployment

#### With Docker:
```bash
cd /opt
sudo git clone https://github.com/yourrepo/myproject.git
sudo chown -R deploy:deploy /opt/myproject
cd /opt/myproject
cp .env.example .env
nano .env # fill production values
docker compose up -d --build

# Run once after first deploy
docker compose exec web python manage.py migrate
docker compose exec web python manage.py createsuperuser

# Verify
docker compose ps
docker compose logs -f web
```

#### Without Docker:
```bash
cd /opt
sudo git clone https://github.com/yourrepo/myproject.git
sudo chown -R deploy:www-data /opt/myproject
cd /opt/myproject
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
nano .env # fill production values
python manage.py migrate
python manage.py collectstatic --noinput
python manage.py createsuperuser
deactivate
```

### 3.3 Deploy Updates

When you push new code to your repository, follow these steps to update your application.

#### With Docker:
```bash
cd /opt/myproject
git pull
docker compose up -d --build
docker compose exec web python manage.py migrate # if schema changed
```

#### Without Docker:
```bash
cd /opt/myproject
git pull
source venv/bin/activate
pip install -r requirements.txt
python manage.py migrate
python manage.py collectstatic --noinput
deactivate
sudo systemctl reload gunicorn # zero-downtime: sends HUP, workers restart gradually
```

### 3.4 Automated CI/CD

To automate the deployment, configure a GitHub Action that connects to your server via SSH and runs the update commands automatically on push to `main`.

Here is a basic example `.github/workflows/deploy.yml` (Django with Docker):
```yaml
name: Deploy to VPS

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SERVER_IP }}
          username: ${{ secrets.SSH_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/myproject
            git pull origin main

            docker compose down
            docker compose build
            docker compose up -d
            docker compose exec -T web python manage.py migrate
```

Here is a basic example `.github/workflows/deploy.yml` (Next.js with PM2):
```yaml
name: Deploy Next.js to VPS (PM2)

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SERVER_IP }}
          username: ${{ secrets.SSH_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /var/www/nextjs-app
            git pull origin main
            npm ci
            npm run build
            pm2 reload nextjs-app
```
#### Get the SSH Private Key for GitHub Actions

1. **Generate a new SSH key pair** (on your local machine or VPS):
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
   chown -R deploy:deploy /home/deploy/.ssh
   ```
  
*(Make sure to add `SERVER_IP`, `SSH_USERNAME`, and `SSH_PRIVATE_KEY` to repository's GitHub Action Secrets.)*
