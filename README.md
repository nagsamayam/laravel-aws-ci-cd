# EC2 Setup & Production Runtime Documentation

## 1. EC2 Provisioning

Created an AWS EC2 Free Tier instance with:

- OS: Ubuntu 26.04 LTS
- Architecture: x86_64 / AMD64
- CPU: 2 vCPU
- RAM: ~1 GB
- Root EBS storage: expanded from 8 GB to 20 GB

Verification:

```bash
uname -m
lsb_release -a
df -h
free -h
```

---

## 2. Configure Swap

Because the EC2 instance has limited RAM, created a 2 GB swap file:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

Persisted swap across reboots:

```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Verification:

```bash
free -h
swapon --show
```

---

## 3. Install Git

Installed Git and verified:

```bash
git --version
```

Important:

> The Laravel application source is not built on EC2. GitHub Actions performs the application build.

---

## 4. Install Docker Engine

Installed Docker Engine from the official Docker repository.

Verification:

```bash
docker --version
docker info
```

Configured the current user to run Docker without `sudo`:

```bash
sudo usermod -aG docker $USER
```

After reconnecting to SSH:

```bash
docker ps
```

---

## 5. Install Docker Compose

Installed Docker Compose v2.

Verification:

```bash
docker compose version
```

---

## 6. Verify Docker Buildx

Verification:

```bash
docker buildx version
```

Builds are performed in GitHub Actions rather than on the EC2 host.

---

## 7. Verify Docker Runtime

Checked Docker configuration:

```bash
docker info
```

Verified the expected runtime characteristics, including:

- x86_64 architecture
- overlay2 storage driver
- systemd cgroup driver
- cgroup v2

---

## 8. Create Production Runtime Directory

Created:

```text
/opt/laravel-aws-ci-cd/
├── docker-compose.yml
└── .env
```

The directory contains production runtime configuration only.

The Laravel source code is not stored here.

---

## 9. Production Docker Compose

The production Compose file defines:

```text
Nginx
   ↓
Laravel PHP-FPM
   ↓
MySQL
```

Services:

- `nginx`
- `app`
- `mysql`

Application images are pulled from GitHub Container Registry:

```text
ghcr.io/<owner>/<repository>:<IMAGE_TAG>
ghcr.io/<owner>/<repository>-nginx:<IMAGE_TAG>
```

MySQL data is stored in a persistent Docker volume:

```text
mysql_data:/var/lib/mysql
```

---

## 10. Production Environment Variables

Created:

```text
/opt/laravel-aws-ci-cd/.env
```

Contains production runtime configuration such as:

```dotenv
APP_ENV=production
APP_DEBUG=false
APP_KEY=...

DB_DATABASE=laravel
DB_USERNAME=laravel
DB_PASSWORD=...
DB_ROOT_PASSWORD=...
```

Important:

> The production `.env` is never committed to GitHub.

---

## 11. Laravel APP_KEY

Generated the Laravel application key separately and placed it in the EC2 production `.env`.

The application key is not regenerated during deployment.

Important:

> Changing `APP_KEY` between deployments can invalidate encrypted Laravel data and cookies.

---

## 12. MySQL Health Check

Docker Compose waits for MySQL to become healthy before starting the Laravel application.

Flow:

```text
MySQL starts
    ↓
Health check
    ↓
MySQL healthy
    ↓
Laravel app starts
```

The Compose health check uses escaped variable interpolation:

```yaml
-p$${DB_ROOT_PASSWORD}
```

so the password is evaluated inside the container rather than by Compose itself.

---

## 13. GHCR Authentication

EC2 was authenticated against GitHub Container Registry:

```text
ghcr.io
```

This allows production to pull private application images.

Production images use immutable Git SHA tags:

```text
ghcr.io/<owner>/<repository>:<SHA>
ghcr.io/<owner>/<repository>-nginx:<SHA>
```

This is particularly important after making the GitHub repository/package private.

---

## 14. GitHub Actions SSH Deployment

Created a dedicated SSH key for GitHub Actions deployment.

Architecture:

```text
GitHub Actions
      |
      | SSH
      v
     EC2
```

The public key was added to:

```text
~/.ssh/authorized_keys
```

Permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

GitHub Secrets contain:

```text
EC2_HOST
EC2_USER
EC2_SSH_PRIVATE_KEY
```

The private key is never committed to the repository.

---

## 15. Deployment IMAGE_TAG

The deployment workflow determines the exact image version from the GitHub Actions workflow run:

```bash
export IMAGE_TAG=<Git SHA>
```

Example:

```text
IMAGE_TAG=55b4659f4c0511ddb032aa059c539d69be073f04
```

EC2 then pulls that exact version.

Important:

> `IMAGE_TAG` is deployment-time configuration. It is not stored in the EC2 `.env`.

---

## 16. Pull Exact Production Images

Deployment runs:

```bash
docker compose pull
```

The Compose file resolves the exact `IMAGE_TAG`.

This means EC2 does not build Docker images.

---

## 17. Database Migration

Deployment runs:

```bash
docker compose run --rm app php artisan migrate --force
```

The migration executes using the same application image being deployed.

---

## 18. Start Application

Deployment runs:

```bash
docker compose up -d --remove-orphans
```

The runtime becomes:

```text
Nginx
  ↓
Laravel PHP-FPM
  ↓
MySQL
```

---

## 19. Laravel Optimization

After starting the containers:

```bash
docker compose exec -T app php artisan optimize
```

This prepares Laravel's production caches.

---

## 20. Production Health Check

The deployment checks Laravel's health endpoint:

```text
http://localhost/up
```

using:

```bash
curl -fsS http://localhost/up
```

The workflow retries the check before declaring deployment successful.

Flow:

```text
Deploy
  ↓
Start containers
  ↓
Wait
  ↓
/up
  ↓
HTTP 200
  ↓
Deployment successful
```

---

## 21. Deployment Rollback

Before deployment, the workflow records the currently running application image/tag.

If the new deployment fails its health check:

```text
New deployment
      ↓
Health check fails
      ↓
Read previous image SHA
      ↓
Pull previous image
      ↓
Start previous version
      ↓
Health check
```

The rollback currently handles application/container images.

Important:

> Database migrations are not automatically rolled back. Database rollback is intentionally treated as a separate concern because migrations can be destructive or non-reversible.

---

# Final EC2 Architecture

```text
AWS EC2
│
├── Ubuntu 26.04
├── Docker Engine
├── Docker Compose
├── Docker Buildx
├── Git
├── 2 GB Swap
│
└── /opt/laravel-aws-ci-cd/
    ├── docker-compose.yml
    └── .env
```

Runtime:

```text
                 GitHub Actions
                       │
                       │ SSH
                       ▼
                  ┌─────────┐
                  │   EC2   │
                  └────┬────┘
                       │
                Docker Compose
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
        Nginx         App          MySQL
          │            │             │
          └───────► PHP-FPM          │
                                     │
                               mysql_data
```

# Complete CI/CD Flow

```text
Developer
   ↓
GitHub
   ↓
GitHub Actions CI
   ↓
Tests / Pint / PHPStan
   ↓
Docker Build
   ↓
Trivy Security Scan
   ↓
GHCR
   ↓
SSH
   ↓
EC2
   ↓
docker compose pull <SHA>
   ↓
Laravel migration
   ↓
docker compose up
   ↓
Laravel optimization
   ↓
/up health check
   ↓
Deployment successful
```

## Key Architecture Principle

**EC2 is the container runtime, not the build environment.**

GitHub Actions is responsible for:

- Testing
- Quality checks
- Docker image builds
- Security scanning
- Publishing images
- Deployment orchestration

EC2 is responsible for:

- Running Docker
- Running Nginx
- Running Laravel PHP-FPM
- Running MySQL
- Persisting database data
- Running the deployed immutable image

This separation keeps the production host simple and makes deployments reproducible.
