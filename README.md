# 🏠 Homelab — Self-Hosted Infrastructure on OCI

![Architecture](https://img.shields.io/badge/Architecture-ARM64-blue)
![OS](https://img.shields.io/badge/OS-Ubuntu%2024.04-orange)
![Proxy](https://img.shields.io/badge/Proxy-Caddy-green)
![Auth](https://img.shields.io/badge/Auth-Authelia%20SSO%20%2B%202FA-purple)

A zero-trust, containerized home lab running **8+ services** behind **Caddy + Authelia SSO** on a single Oracle Cloud ARM64 instance. Encrypted backups. Automated scheduling. No SSH into individual containers — ever.

---

## 🧱 Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                          Internet                                │
│                      :443 (HTTPS only)                           │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Caddy Reverse Proxy                            │
│                    dg.linkpc.net (*.dg.linkpc.net)                 │
│                                                                   │
│   ┌────────────────────┐   ┌──────────────────────────────────┐  │
│   │  Auth-Protected     │   │  Direct Proxy (own auth)         │  │
│   │  ┌─────────────┐   │   │  ┌──────────────┐               │  │
│   │  │ auth.       │   │   │  │ nextcloud.   │               │  │
│   │  │ dg.linkpc   │   │   │  │ dg.linkpc    │               │  │
│   │  │ .net        │   │   │  │ .net         │               │  │
│   │  └──────┬──────┘   │   │  └──────┬───────┘               │  │
│   │         │          │   │         │                        │  │
│   │         ▼          │   │         ▼                        │  │
│   │  ┌──────────┐      │   │  ┌──────────┐                   │  │
│   │  │ Authelia │◄────►│   │  │ Nextcloud│                   │  │
│   │  │  SSO     │      │   │  │ (own     │                   │  │
│   │  │  2FA     │      │   │  │  auth)   │                   │  │
│   │  └──────────┘      │   │  └──────────┘                   │  │
│   │                    │   │  ┌──────────────┐               │  │
│   │  ┌──────────┐      │   │  │ notes.       │               │  │
│   │  │ QwenPaw  │◄─────┤   │  │ dg.linkpc    │               │  │
│   │  │ Dockhand │      │   │  │ .net         │               │  │
│   │  │ OpenChamber│    │   │  └──────┬───────┘               │  │
│   │  └──────────┘      │   │         │                        │  │
│   └────────────────────┘   │         ▼                        │  │
│                            │  ┌──────────┐                   │  │
│                            │  │ Trilium  │                   │  │
│                            │  │ (own     │                   │  │
│                            │  │  auth)   │                   │  │
│                            │  └──────────┘                   │  │
│                            └──────────────────────────────────┘  │
│                                                                   │
│    ┌──────────────────────────────────────────────────┐          │
│    │         Isolated Docker Bridge Networks           │          │
│    │  (Each service sandboxed, only web-exposed       │          │
│    │   services on the shared proxy network)          │          │
│    └──────────────────────────────────────────────────┘          │
│                                                                   │
│    ┌──────────────────────────────────────────────────┐          │
│    │              Borg Backup Engine                   │          │
│    │         ┌──────────────────────────┐             │          │
│    │         │ OCI Block Volume (/backups)│            │          │
│    │         │   Encrypted backups       │            │          │
│    │         │   4 retention tiers       │            │          │
│    │         └──────────────────────────┘             │          │
│    └──────────────────────────────────────────────────┘          │
│                                                                   │
│    ┌──────────────────────────────────────────────────┐          │
│    │             Cronmaster Scheduler                  │          │
│    │  • Borg backup orchestration                     │          │
│    │  • Health checks                                 │          │
│    │  • Automated maintenance                          │          │
│    └──────────────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────────────────┘
```

### Traffic Flow

**Auth-protected services** (QwenPaw, Dockhand, OpenChamber):
1. **User** → `https://<service>.dg.linkpc.net` → **Caddy**
2. **Caddy** intercepts and forwards to **Authelia** for authentication
3. **Authelia** validates session (SSO + 2FA if configured)
4. On success → **Caddy** proxies to the backend container

**Direct-proxy services** (Nextcloud, Trilium):
1. **User** → `https://<service>.dg.linkpc.net` → **Caddy**
2. **Caddy** proxies directly to the backend
3. Authentication handled by the service itself (for desktop/mobile app connectivity)

All web-exposed services live on a shared Docker network (`proxy`). Each backend stack is isolated in its own Docker bridge network.

### Security Rules (Oracle Cloud Firewall)

| Direction | Source | Port | Action |
|-----------|--------|------|--------|
| Inbound  | 0.0.0.0/0 | 80  | Allow (redirect to 443) |
| Inbound  | 0.0.0.0/0 | 443 | Allow |
| Inbound  | 0.0.0.0/0 | Any | **Deny** (default-deny) |

All internal service traffic is routed exclusively through Caddy. No container ports are exposed directly to the internet.

---

## 📦 Services

| Service | Subdomain | Auth | Description |
|---------|-----------|------|-------------|
| [**Caddy**](https://caddyserver.com) | `*.dg.linkpc.net` | — | Reverse proxy with automatic HTTPS via Let's Encrypt |
| [**Authelia**](https://www.authelia.com) | `auth.dg.linkpc.net` | SSO + 2FA | Authentication portal for protected services |
| [**Nextcloud**](https://nextcloud.com) | `nextcloud.dg.linkpc.net` | Own auth | File sync, share, and collaboration |
| [**Trilium Notes**](https://github.com/zadam/trilium) | `notes.dg.linkpc.net` | Own auth | Self-hosted knowledge base / note-taking |
| [**QwenPaw**](https://github.com/agentscope-ai/QwenPaw) | — | Authelia | Open-source AI agent framework by AgentScope / Qwen Lab |
| [**Dockhand**](https://github.com/Finsys/dockhand) | — | Authelia | Modern Docker management UI (containers, compose stacks, file browser) |
| [**OpenChamber**](https://github.com/agentscope-ai/QwenPaw) | — | Authelia | QwenPaw agent runtime (uses **OpenCode** internally) |
| [**Cronmaster**](https://github.com/fccview/cronmaster) | — | None (internal) | Cron job management UI with human-readable syntax and live logging |
| [**Borg Backup**](https://www.borgbackup.org) | — | None (internal) | Encrypted, deduplicated backups |

---

## 📂 Repository Structure

```
homelab/
├── README.md
├── caddy/
│   └── docker-compose.yml
├── authelia/
│   ├── docker-compose.yml
│   └── configuration/
│       ├── configuration.yml
│       └── users_database.yml
├── nextcloud/
│   ├── docker-compose.yml
│   ├── Dockerfile              # (custom image if needed)
│   └── config/
├── trilium/
│   └── docker-compose.yml
├── qwenpaw/
│   └── docker-compose.yml
├── dockhand/
│   └── docker-compose.yml
├── openchamber/
│   ├── docker-compose.yml
│   └── Dockerfile
├── borg/
│   └── docker-compose.yml
└── cronmaster/
    └── docker-compose.yml
```

Each service has its own directory containing:
- `docker-compose.yml` — service definition
- `Dockerfile` — only when the image requires custom builds (e.g., cross-compiling for ARM64)

---

## 🔒 Security Model

| Principle | Implementation |
|-----------|---------------|
| **Zero Trust** | Every request authenticated before reaching any service |
| **Default-Deny** | OCI security group blocks everything except ports 80/443 |
| **Network Isolation** | Each service stack on its own Docker bridge network |
| **Minimum Exposure** | Only web-exposed services join the shared `proxy` network |
| **Encrypted Backups** | Borg repositories encrypted with a passphrase/keyfile |
| **No Direct Container Access** | All management through Caddy, Authelia, or Dockhand UI |

### Auth-Protected vs Direct-Proxy Services

| Type | Services | Auth Method | Rationale |
|------|----------|-------------|-----------|
| **Auth-protected** | QwenPaw, Dockhand, OpenChamber | Authelia SSO + 2FA | Web-only access, no client app needed |
| **Direct-proxy** | Nextcloud, Trilium | Service's own auth (username/password) | Desktop/mobile apps need direct API auth |

### Caddy + Authelia Auth Flow (for protected services)

```
Request → Caddy → [unauthenticated] → Authelia login page → SSO → Caddy → Service
```

Authelia sits in front of protected services. Once authenticated, Caddy forwards requests with the user's identity headers. Unauthenticated requests are redirected to `auth.dg.linkpc.net` for login.

Nextcloud and Trilium bypass Authelia entirely — Caddy proxies them directly so desktop and mobile clients can authenticate using their own mechanisms.

---

## 💾 Backup Strategy

| Tier | Frequency | Retention |
|------|-----------|-----------|
| Daily | Every day | 7 days |
| Weekly | Every Sunday | 4 weeks |
| Monthly | 1st of month | 6 months |
| Yearly | 1st of year | 2 years |

- **Engine:** Borg Backup (encrypted + deduplicated)
- **Storage:** Dedicated OCI block volume, mounted at `/backups`
- **Integrity:** Automated consistency checks after every backup run
- **Schedule:** Managed by Cronmaster

---

## ⏰ Automation (Cronmaster)

[Cronmaster](https://github.com/fccview/cronmaster) orchestrates all scheduled tasks:

| Job | Schedule | Description |
|-----|----------|-------------|
| Borg backup | Daily at 02:00 | Full encrypted backup of all service data |
| Health check | Every 15 min | Ping each service endpoint, alert on failure |
| Backup integrity | After every backup | Validate backup consistency |
| Log rotation | Weekly | Rotate and archive container logs |

---

## 🚀 Getting Started (Fresh Deploy)

1. **Provision an OCI ARM instance** (Ubuntu 24.04, minimum 4 GB RAM)
2. **Open ports 80 and 443** in the security list (default-deny everything else)
3. **Clone this repo** on the instance
4. **Set up Caddy** — configure your domain `dg.linkpc.net` and email for Let's Encrypt
5. **Configure Authelia** — set `jwt_secret`, `storage_encryption_key`, user accounts
6. **Mount backup volume** to `/backups` (OCI block volume)
7. **Initialize Borg repo** — `borg init --encryption=repokey-blake2 /backups/homelab`
8. **Deploy all stacks** — `for d in */; do (cd "$d" && docker compose up -d); done`
9. **Verify** — hit `https://auth.dg.linkpc.net` and confirm Authelia loads

---

## 🛠 Requirements

- **OS:** Ubuntu 24.04 LTS (ARM64)
- **Kernel:** Linux 6.8+
- **Docker:** 24.0+
- **Docker Compose:** v2.20+
- **Domain:** `*.dg.linkpc.net` pointing to the instance's public IP
- **Storage:** OCI block volume for backups (min 50 GB recommended)

---

## 📜 License

Each service is governed by its own license. This repository's compose files and documentation are provided as-is — no warranty, use at your own risk.

---

*Built by [Dhrubajyoti Giri](https://github.com/dhrubajyoti-giri)*
