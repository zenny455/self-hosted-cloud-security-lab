# 🔐 Self-Hosted Cloud Security Lab

> A fully containerized private cloud stack with SSO, encryption, secrets management, and security monitoring.

---


## 📌 Table of Contents

- [Project Overview](#-project-overview)
- [Architecture](#architecture)
- [Technology Stack](#technology-stack)
- [Prerequisites](#-prerequisites)
- [Project Structure](#-project-structure)
- [Phase 1: Infrastructure Setup](#phase-1--infrastructure-setup)
- [Phase 2: Identity & Access Management](#phase-2--identity--access-management)
- [Phase 3: Data Security & Encryption](#phase-3--data-security--encryption)
- [IAM Policy Reference](#-iam-policy-reference)

---

## 📖 Project Overview

This project implements a fully self-hosted private cloud environment focused on security best practices across six phases:

1. **Infrastructure Setup** — Deploy all services via Docker Compose
2. **Identity & Access Management** — SSO, RBAC, and MFA via Keycloak
3. **Data Security & Encryption** — TLS termination, AES-256 at-rest encryption, and HashiCorp Vault for secrets management
4. **Security Monitoring** — Fail2ban, GoAccess, log analysis
5. **Vulnerability Scanning** — Trivy container image scanning
6. **Compliance Audit** — CIS Benchmark gap analysis

The stack is deployed directly on a Linux host using Docker (no VM required), making it reproducible on any Linux machine with Docker installed.

---

## Architecture

```
Browser (HTTPS)
      │
      ▼
┌─────────────┐
│    Nginx    │  ← Reverse proxy + TLS termination (self-signed cert)
│  Port 80/443│
└──────┬──────┘
       │
  ┌────┼──────────────────────────────────┐
  │    │                                  │
  ▼    ▼              ▼                   ▼
Nextcloud         Keycloak            MinIO         HashiCorp Vault
nextcloud.local   keycloak.local      minio.local   vault.local
   :80               :8080               :9001         :8200
  (OIDC SP)        (Identity Provider)  (Object Store) (Secrets Engine)
      │
      ▼
 PostgreSQL
 (DB backend)
```

All four services are accessible via `.local` hostnames mapped to `127.0.0.1` in `/etc/hosts`.

---

## Technology Stack

| Tool | Role |
|------|------|
| **Linux** | Host OS or VM |
| **Docker & Docker Compose** | Container runtime for all services |
| **Nextcloud** | Core self-hosted cloud storage platform |
| **PostgreSQL** | Database backend for Nextcloud |
| **MinIO** | S3-compatible local object storage |
| **Keycloak** | Identity Provider — SSO via OIDC, RBAC, MFA |
| **HashiCorp Vault** | Centralized secrets and encryption key management |
| **Nginx** | Reverse proxy with TLS termination |
| **Fail2ban** | Brute-force and intrusion prevention |
| **GoAccess** | Real-time Nginx log analysis |
| **Trivy** | Container image vulnerability scanner |
| **Timeshift** | Host-level snapshots (replaces VM snapshots) |

---

## ✅ Prerequisites

- Linux (Ubuntu 20.04+ or Debian)
- Docker CE 20.10+ and Docker Compose v2 plugin
- `openssl` (for TLS certificate generation)
- A browser (Chrome/Firefox recommended)
- ~4 GB RAM available for the full stack

### Installing Docker (Ubuntu/Mint)

```bash
# Add Docker's official GPT key and repo
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add yourself to the docker group (avoids needing sudo)
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker --version
docker compose version
```

---

## 📁 Project Structure

```
~/cloud-security-lab/
├── docker-compose.yml          # Full stack definition
├── nginx/
│   ├── certs/
│   │   ├── lab.crt             # Self-signed TLS certificate
│   │   └── lab.key             # Private key
│   └── conf/
│       └── default.conf        # Nginx reverse proxy config
├── nextcloud-data/             # Nextcloud persistent data volume
└── minio-data/                 # MinIO object storage volume
```

Create the folder structure before deploying:

```bash
mkdir -p ~/cloud-security-lab/nginx/{certs,conf}
mkdir -p ~/cloud-security-lab/{nextcloud-data,minio-data}
cd ~/cloud-security-lab
```

---

## Phase 1 — Infrastructure Setup

### 1. Local DNS Configuration

Add all four service hostnames to `/etc/hosts` so your browser resolves them to localhost:

```bash
sudo nano /etc/hosts
```

Add these lines:

```
127.0.0.1   nextcloud.local
127.0.0.1   keycloak.local
127.0.0.1   minio.local
127.0.0.1   vault.local
```

### 2. docker-compose.yml

Create `~/cloud-security-lab/docker-compose.yml`:

> ⚠️ **Set passwords** with strong, unique values before deploying.

### 3. Nginx Configuration (HTTP — initial)

Create `~/cloud-security-lab/nginx/conf/default.conf`:

```nginx
server {
    listen 80;
    server_name nextcloud.local;
    location / { proxy_pass http://nextcloud:80; proxy_set_header Host $host; }
}

server {
    listen 80;
    server_name keycloak.local;
    location / { proxy_pass http://keycloak:8080; proxy_set_header Host $host; }
}

server {
    listen 80;
    server_name minio.local;
    location / { proxy_pass http://minio:9001; proxy_set_header Host $host; }
}

server {
    listen 80;
    server_name vault.local;
    location / { proxy_pass http://vault:8200; proxy_set_header Host $host; }
}
```

### 4. Deploy the Stack

```bash
cd ~/cloud-security-lab
docker compose up -d

# Verify all containers are running
docker compose ps
```

Expected output: all six services showing `Up` status.

### 5. Fix Nextcloud Trusted Domain

```bash
docker compose exec nextcloud php occ config:system:set trusted_domains 1 --value=nextcloud.local
```

### 6. Verify Access

Open each URL in your browser:

| Service | URL |
|---------|-----|
| Nextcloud | http://nextcloud.local |
| Keycloak | http://keycloak.local |
| MinIO Console | http://minio.local |
| Vault UI | http://vault.local |

---

## Phase 2 — Identity & Access Management

### 1. Create the Keycloak Realm

1. Navigate to `http://keycloak.local` → sign in as admin
2. Click **Create Realm** → name it `cloud-lab` → Save

### 2. Create Roles

In the `cloud-lab` realm → **Realm roles** → Create role:

| Role | Purpose |
|------|---------|
| `admin` | Full administrative access |
| `editor` | File management + limited admin delegation |
| `viewer` | Read-only access |

### 3. Create Test Users

**Realm → Users → Add user** for each role. After creation:

1. Go to **Credentials** → Set password → turn off "Temporary"
2. Go to **Role mapping** → Assign the appropriate role

### 4. Register Nextcloud as a Keycloak Client

1. In `cloud-lab` realm → **Clients** → Create client
2. **Client ID:** `nextcloud`
3. **Client authentication:** ON (confidential client)
4. **Valid Redirect URIs:** `http://nextcloud.local/apps/sociallogin/custom_oidc/keycloak`
5. Save, then copy the **Client Secret** from the Credentials tab

### 5. Configure Social Login in Nextcloud

1. Log into Nextcloud as admin → **Apps** → search "Social Login" → Enable
2. Go to **Settings → Administration → Social login**
3. Add a **Custom OpenID Connect** provider:

| Field | Value |
|-------|-------|
| Internal name | `keycloak` |
| Title | `Keycloak` |
| Authorize URL | `http://keycloak.local/realms/cloud-lab/protocol/openid-connect/auth` |
| Token URL | `http://keycloak.local/realms/cloud-lab/protocol/openid-connect/token` |
| UserInfo URL | `http://keycloak.local/realms/cloud-lab/protocol/openid-connect/userinfo` |
| Logout URL | `http://keycloak.local/realms/cloud-lab/protocol/openid-connect/logout` |
| Client ID | `nextcloud` |
| Client Secret | *(from Keycloak Credentials tab)* |
| Scope | `openid profile email` |
| Groups claim | `groups` |

### 6. Fix Group/Role Sync (Critical)

By default Keycloak does not expose roles to the UserInfo endpoint. Add a custom mapper:

1. `cloud-lab` realm → **Client scopes** → `roles` → **Mappers** → Add mapper → By configuration → **User Realm Role**
2. Set **Token Claim Name:** `groups`
3. Enable **Add to userinfo**
4. Save

Verify with: **Clients → nextcloud → Client scopes → Evaluate** → select a user → **Generated user info** — confirm the `groups` array appears.

### 7. Enable MFA (TOTP)

1. `cloud-lab` realm → **Authentication** → **Required actions**
2. Enable **Configure OTP** and set as **Default action**

All users will be prompted to enroll an authenticator app (Google Authenticator, FreeOTP, etc.) on first login.

### 8. Delegate Admin Privileges

In Nextcloud → **Settings → Administration → Administration privileges**:

- `editor` group → grant only: **Background jobs**, **Theming**
- `viewer` group → no delegation

---

## Phase 3 — Data Security & Encryption

### 1. Generate a Self-Signed TLS Certificate

```bash
cd ~/cloud-security-lab/nginx/certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout lab.key \
  -out lab.crt \
  -subj "/CN=*.local/O=CloudSecLab/C=PK"
```

### 2. Update Nginx for HTTPS

Replace `nginx/conf/default.conf` with the HTTPS configuration:

```nginx
# HTTP → HTTPS redirect for all services
server {
    listen 80;
    server_name nextcloud.local keycloak.local minio.local vault.local;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name nextcloud.local;
    ssl_certificate     /etc/nginx/certs/lab.crt;
    ssl_certificate_key /etc/nginx/certs/lab.key;
    location / {
        proxy_pass http://nextcloud:80;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;
    }
}

server {
    listen 443 ssl;
    server_name keycloak.local;
    ssl_certificate     /etc/nginx/certs/lab.crt;
    ssl_certificate_key /etc/nginx/certs/lab.key;
    location / {
        proxy_pass http://keycloak:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;
    }
}

server {
    listen 443 ssl;
    server_name minio.local;
    ssl_certificate     /etc/nginx/certs/lab.crt;
    ssl_certificate_key /etc/nginx/certs/lab.key;
    location / {
        proxy_pass http://minio:9001;
        proxy_set_header Host $host;
    }
}

server {
    listen 443 ssl;
    server_name vault.local;
    ssl_certificate     /etc/nginx/certs/lab.crt;
    ssl_certificate_key /etc/nginx/certs/lab.key;
    location / {
        proxy_pass http://vault:8200;
        proxy_set_header Host $host;
    }
}
```

Reload Nginx:

```bash
docker compose restart nginx
```

> 💡 You will see a browser warning ("Your connection is not private") — this is expected for self-signed certificates. Click **Advanced → Proceed**.

### 3. Tell Nextcloud It's Behind HTTPS

```bash
docker compose exec nextcloud php occ config:system:set overwriteprotocol --value=https
```

### 4. Enable Server-Side Encryption in Nextcloud

1. **Settings → Administration → Security** → enable **Server-side encryption**
2. If uploads fail, set the encryption module explicitly:

```bash
docker compose exec nextcloud php occ encryption:set-default-module OC_DEFAULT_MODULE
```

Files are encrypted with **AES-256-CTR** at rest. You can verify by inspecting raw bytes inside the container:

```bash
docker compose exec nextcloud cat /var/www/html/data/<username>/files/<filename>
# Output will show: HBEGIN:oc_encryption_module:OC_DEFAULT_MODULE:cipher:AES-256-CTR...
```

### 5. HashiCorp Vault — Secrets Management

Vault runs in **dev mode** (pre-initialized, no unsealing required). Sign in at `https://vault.local` using the token `myroot`.

**Create a secret:**

1. Enable a **KV v2** secrets engine at `secret/`
2. Create a secret: `secret/database` → add key `password` → save

**Secret rotation** is built-in: every update creates a new version. All previous versions remain retrievable via **Version History**.

**Via CLI:**

```bash
# Exec into the vault container
docker compose exec vault sh

# Set vault address and token
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='myroot'

# Write a secret
vault kv put secret/database password="InitialPass123"

# Read a secret
vault kv get secret/database

# Update (creates version 2)
vault kv put secret/database password="RotatedPass456"

# View version history
vault kv metadata get secret/database
```

---

## 📋 IAM Policy Reference

**Realm:** `cloud-lab` | **Auth method:** OpenID Connect (Authorization Code flow)

### Roles and Permissions

| Role | Permissions |
|------|-------------|
| `admin` | Full Nextcloud administrative access — user management, security, encryption, apps, sharing, and all other settings |
| `editor` | Standard file operations (upload, edit, share, delete own files) + delegated access to **Background jobs** and **Theming** only |
| `viewer` | No administrative access; read-only access to shared content via standard file permissions |

### Authentication Policy

- All users authenticate through **Keycloak SSO** ("Sign in with Keycloak" button)
- The local Nextcloud `admin` account bypasses SSO — reserved as a **break-glass** credential only
- **TOTP MFA** is enforced **realm-wide** — all roles, not just admin
- Group membership is read from the `groups` claim in Keycloak's UserInfo response (populated by a custom "User Realm Role" mapper)

---


## 🔄 Useful Commands

```bash
# Start all services
docker compose up -d

# Stop all services
docker compose down

# View logs for a specific service
docker compose logs -f nextcloud

# Restart Nginx only (after config changes)
docker compose restart nginx

# Shell into a container
docker compose exec nextcloud bash
docker compose exec vault sh

# Check encryption status
docker compose exec nextcloud php occ encryption:status

# Run Trivy vulnerability scan on an image
trivy image nextcloud:27
```

---

## 📄 License

This project was created for academic purposes as part of CY464 — Cloud Security at the University of Management & Technology, Lahore.
