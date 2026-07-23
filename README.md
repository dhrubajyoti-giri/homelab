# 🏠 Homelab — Self-Hosted Infrastructure on OCI

![Architecture](https://img.shields.io/badge/Architecture-ARM64-blue)
![OS](https://img.shields.io/badge/OS-Ubuntu%2024.04-orange)
![Proxy](https://img.shields.io/badge/Proxy-Caddy-green)
![Auth](https://img.shields.io/badge/Auth-Authelia%20SSO-purple)

A containerized homelab running behind **Caddy + Authelia SSO** on a single Oracle Cloud ARM64 instance. Encrypted backups via Borg. Each service in its own Docker bridge network.

---

## 📍 Quick Links

| Service           | URL                                                        | Auth       |
| ----------------- | ---------------------------------------------------------- | ---------- |
| **Caddy**         | `*.dg.linkpc.net`                                          | —          |
| **Authelia**      | [auth.dg.linkpc.net](https://auth.dg.linkpc.net)           | SSO portal |
| **Nextcloud**     | [nextcloud.dg.linkpc.net](https://nextcloud.dg.linkpc.net) | Own auth   |
| **Trilium Notes** | [notes.dg.linkpc.net](https://notes.dg.linkpc.net)         | Own auth   |
| **BorgUI**        | [borg.dg.linkpc.net](https://borg.dg.linkpc.net)           | Authelia   |
| **Dockhand**      | [docker.dg.linkpc.net](https://docker.dg.linkpc.net)       | Authelia   |
| **QwenPaw**       | [qwenpaw.dg.linkpc.net](https://qwenpaw.dg.linkpc.net)     | Authelia   |
| **Cronmaster**    | [cron.dg.linkpc.net](https://cron.dg.linkpc.net)           | Authelia   |
| **OpenChamber**   | [code.dg.linkpc.net](https://code.dg.linkpc.net)           | Own auth   |

---

## 🧱 Architecture

```
                             ┌──────────────┐
                             │   Internet    │
                             │  :443 (HTTPS) │
                             └──────┬───────┘
                                    │
                                    ▼
                    ┌──────────────────────────────────┐
                    │         Caddy Reverse Proxy       │
                    │        *.dg.linkpc.net            │
                    │                                   │
                    │  ┌──────────┐  ┌───────────────┐  │
                    │  │ Protected│  │  Direct Proxy │  │
                    │  │ via      │  │  (own auth)   │  │
                    │  │ Authelia │  │               │  │
                    │  └─────┬────┘  └──────┬────────┘  │
                    └────────┼──────────────┼───────────┘
                             │              │
                  ┌──────────┼──────┐  ┌────┼──────────┐
                  │          │      │  │    │          │
                  ▼          ▼      ▼  ▼    ▼          ▼
             ┌────────┐ ┌────────┐  ┌──────────┐ ┌──────────┐
             │ QwenPaw│ │Dockhand│  │ Nextcloud│ │  Trilium │
             │Authelia│ │Authelia│  │ Own auth │ │ Own auth │
             └────────┘ └────────┘  └──────────┘ └──────────┘
             ┌────────┐ ┌────────┐                  ┌──────────┐
             │ BorgUI │ │Cronmaster│                │OpenChamber│
             │Authelia│ │Authelia │                 │ Own auth │
             └────────┘ └─────────┘                 └──────────┘

  ┌─────────────────────────────────────────────────────────────┐
  │         Isolated Docker Bridge Networks                     │
  │   Each service in its own network. Web-exposed services     │
  │   join a shared proxy network to reach Caddy.               │
  └─────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────┐
  │              BorgUI (schedules + manages backups)            │
  │         ┌─────────────────────────────────────┐             │
  │         │  OCI Block Volume (/backups)         │            │
  │         │  Encrypted + deduplicated backups    │            │
  │         │  Scheduled directly in BorgUI        │            │
  │         └─────────────────────────────────────┘             │
  └─────────────────────────────────────────────────────────────┘
```

### Traffic Flow

**Authelia-protected services** (QwenPaw, Dockhand, BorgUI, Cronmaster):

```
User → https://<svc>.dg.linkpc.net → Caddy → [unauth] → Authelia → SSO → Caddy → Backend
```

**Direct-proxy services** (Nextcloud, Trilium, OpenChamber):

```
User → https://<svc>.dg.linkpc.net → Caddy → Backend (handles own auth)
```

---

## 📦 Services

| Service                                                     | Subdomain                 | Auth      | Purpose                                           |
| ----------------------------------------------------------- | ------------------------- | --------- | ------------------------------------------------- |
| [**Caddy**](https://caddyserver.com)                        | `*.dg.linkpc.net`         | —         | Reverse proxy, automatic HTTPS (Let's Encrypt)    |
| [**Authelia**](https://www.authelia.com)                    | `auth.dg.linkpc.net`      | SSO + 2FA | Centralized authentication portal                 |
| [**Nextcloud**](https://nextcloud.com)                      | `nextcloud.dg.linkpc.net` | Own auth  | File sync, share, calendar, contacts              |
| [**Trilium Notes**](https://github.com/zadam/trilium)       | `notes.dg.linkpc.net`     | Own auth  | Self-hosted knowledge base / note-taking          |
| [**BorgUI**](https://github.com/karanhudia/borg-ui)         | `borg.dg.linkpc.net`      | Authelia  | Borg backup management with built-in scheduler    |
| [**Dockhand**](https://github.com/Finsys/dockhand)          | `docker.dg.linkpc.net`    | Authelia  | Docker management UI (containers, compose, files) |
| [**QwenPaw**](https://github.com/agentscope-ai/QwenPaw)     | `qwenpaw.dg.linkpc.net`   | Authelia  | AI agent framework by AgentScope / Qwen Lab       |
| [**Cronmaster**](https://github.com/fccview/cronmaster)     | `cron.dg.linkpc.net`      | Authelia  | Cron job management UI with live logs             |
| [**OpenChamber**](https://github.com/agentscope-ai/QwenPaw) | `code.dg.linkpc.net`      | Own auth  | QwenPaw agent runtime (with desktop app)          |

### Auth Split

| Behind Authelia | Own Auth    |
| --------------- | ----------- |
| QwenPaw         | Nextcloud   |
| Dockhand        | Trilium     |
| BorgUI          | OpenChamber |
| Cronmaster      |             |

Services behind Authelia are web-only and don't need client app auth. Nextcloud, Trilium, and OpenChamber use their own auth so desktop/mobile apps can connect directly.

---

## 🔒 Security Model

| Principle                     | Implementation                                          |
| ----------------------------- | ------------------------------------------------------- |
| **Default-Deny**              | OCI security group — only ports 80 (→443) and 443 open  |
| **Network Isolation**         | Each service on its own Docker bridge network           |
| **Minimum Exposure**          | Only web-exposed services join the shared proxy network |
| **Encrypted Backups**         | Borg repos encrypted with passphrase/keyfile            |
| **No Direct Container Ports** | Everything routed through Caddy proxy                   |

### Oracle Cloud Firewall

| Direction | Source    | Port | Action                |
| --------- | --------- | ---- | --------------------- |
| Inbound   | 0.0.0.0/0 | 80   | Allow (redirect →443) |
| Inbound   | 0.0.0.0/0 | 443  | Allow                 |
| Inbound   | 0.0.0.0/0 | Any  | **Deny**              |

---

## 💾 Backup (BorgUI)

[BorgUI](https://github.com/karanhudia/borg-ui) at [borg.dg.linkpc.net](https://borg.dg.linkpc.net) handles everything:

- **Dashboard** — repository health, activity, storage usage
- **Scheduling** — built-in scheduler for automated backup runs
- **Archive browsing** — browse and restore files from any archive
- **Repositories** — manage local, SSH, and SFTP destinations
- **Remote machines** — SSH key deployment and storage visibility

**Storage:** OCI block volume mounted at `/backups`

---

## ⏰ Automation (Cronmaster)

[Cronmaster](https://github.com/fccview/cronmaster) at [cron.dg.linkpc.net](https://cron.dg.linkpc.net) handles general scheduled tasks (health checks, log rotation, etc.). Borg backup scheduling is managed directly inside BorgUI.

---

## 📂 Repository Structure

```
homelab/
├── README.md
├── .gitignore
│
├── authelia/        → compose.yaml
├── backrest/        → compose.yaml
├── borg-ui/         → compose.yaml
├── caddy/           → compose.yaml
├── cronmaster/      → compose.yaml
├── dockmesh/        → compose.yaml, dockmesh (binary)
├── flame/           → compose.yaml
├── hermes/          → compose.yaml
├── hermes-web-ui/   → compose.yaml
├── homarr/          → compose.yaml
├── joplin/          → compose.yaml
├── mcp/             → compose.yaml
├── nextcloud/       → compose.yaml
├── notesnook/app/   → compose.yaml
├── ollama/          → compose.yaml
├── openchamber/     → compose.yaml, Dockerfile, openchamber (binary)
├── open-webui/      → compose.yaml
├── opencloud/       → compose.yaml
├── paseo/           → compose.yaml
├── qwenpaw/         → compose.yaml
├── trilium/         → compose.yaml
├── ubuntu/          → compose.yaml
├── vikunja/         → compose.yaml
│
├── arcane/          → compose.yaml
└── nse-stock-analyzer  → single file
```

> The `.gitignore` ignores everything **except** `compose.yaml`/`docker-compose.yaml`, `Dockerfile`, `.gitignore`, and `README.md`. Only container definition files are tracked — no `.env`, no config dumps, no source code.

---

## 🚀 Fresh Deploy Steps

1. **Provision** OCI ARM instance (Ubuntu 24.04, 4 GB+ RAM)
2. **Open ports 80 + 443** in security list (default-deny rest)
3. **Clone** this repo on the instance
4. **Configure Caddy** — domain `*.dg.linkpc.net`, Let's Encrypt email
5. **Setup Authelia** — `jwt_secret`, `storage_encryption_key`, users
6. **Mount backup volume** to `/backups`
7. **Init Borg repo** — `borg init --encryption=repokey-blake2 /backups/homelab`
8. **Deploy** — `for d in */; do (cd "$d" && docker compose up -d); done`
9. **Verify** — visit each URL and confirm auth works

---

## 🛠 Requirements

| Component      | Requirement                            |
| -------------- | -------------------------------------- |
| OS             | Ubuntu 24.04 LTS (ARM64)               |
| Kernel         | Linux 6.8+                             |
| Docker         | 24.0+                                  |
| Docker Compose | v2.20+                                 |
| Domain         | `*.dg.linkpc.net` → instance public IP |
| Backup storage | OCI block volume (50 GB+ recommended)  |

---

## 📜 License

Each service is governed by its own license. This repository's compose files and documentation are provided as-is — no warranty.

---

_Built by [Dhrubajyoti Giri](https://github.com/dhrubajyoti-giri)_
