# SaaS Platform — Production Deployment Repository

This repository is the deployment-only configuration repository for the Quick Delivery SaaS platform (`omarmaarouf18/saas-core`).

Production cloud hosts pull from this repository to deploy pre-built Docker containers published to GitHub Container Registry (GHCR).

## Usage
1. Clone this repository on your production VPS:
   ```bash
   git clone https://github.com/omarmaarouf18/saas-core-deploy.git /opt/saas-platform
   cd /opt/saas-platform
   ```
2. Copy `.env.example` to `.env` and fill in production secrets:
   ```bash
   cp .env.example .env
   ```
3. Generate mTLS certificates (or copy internal certs):
   ```bash
   cd certs && ./generate-certs.sh && cd ..
   ```
4. Pull and run containers:
   ```bash
   docker compose pull && docker compose up -d
   ```

For full setup documentation, reverse proxy configuration, firewall rules, and source code, visit the primary development repository: [omarmaarouf18/saas-core](https://github.com/omarmaarouf18/saas-core).
