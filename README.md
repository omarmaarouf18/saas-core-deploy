# SaaS Platform — Production Deployment Repository

This repository is the deployment-only configuration repository for the Quick Delivery SaaS platform (`omarmaarouf18/saas-core`).

Production cloud hosts pull from this repository to deploy pre-built Docker containers published to GitHub Container Registry (GHCR).

## Automated Continuous Deployment (Current Setup)

This repository has an automated CD pipeline (`.github/workflows/deploy.yml`) that runs
on a self-hosted GitHub Actions runner installed directly on the production VM
(`quickdelivery-vm`, runner name `saas-vm-runner`). On every push to `main` (typically
triggered by `saas-core`'s build-and-publish pipeline updating this repo's
docker-compose.yml with new image tags), the runner automatically:
1. Checks out the latest docker-compose.yml
2. Ensures a `.env` file exists (copying from `/opt/saas-platform/.env` if missing)
3. Validates the compose configuration
4. Pulls the new container images
5. Runs `docker compose up -d --remove-orphans`
6. Dumps container logs automatically on any failure for fast diagnosis

Manual deployment (below) is still useful for first-time server setup or disaster
recovery, but routine releases no longer require any manual SSH step.

**Important: volume naming is pinned.** The `mongo_data` and `redis_data` volumes are
explicitly named `saas_platform_mongo_data` / `saas_platform_redis_data` in
docker-compose.yml (rather than left to Docker Compose's default directory-based naming).
This was fixed after a real incident where the CD pipeline running from a different
working directory than the original manual setup created a *second*, empty MongoDB
volume — causing `docker compose up` to authenticate against the wrong (empty) volume
and fail with `AuthenticationFailed: SCRAM authentication failed, storedKey mismatch`.
If you ever change where this repo is checked out from, the pinned names guarantee the
same physical volume is reused rather than a new one being silently created.

## Image Registry Authentication & Setup Options

By default, Docker images published to GHCR (`ghcr.io`) from private repositories require authentication to pull.

### Option A (RECOMMENDED): Public GHCR Packages (Zero Auth on Host)
If the GHCR packages are set to **Public** in GitHub Package Settings (or via `gh api -X PATCH /user/packages/container/<package>/visibility -f visibility=public`), no host authentication is required. You can immediately run `docker compose pull`.

### Option B: Private GHCR Packages (Host PAT Authentication)
If the packages are kept **Private**, authenticate your production host to GHCR once before running `docker compose pull`:

1. Generate a Personal Access Token (PAT) with `read:packages` scope in GitHub settings (e.g. `GHCR_READ_TOKEN`).
   > [!NOTE]
   > `DEPLOY_REPO_PAT` (used by GitHub Actions with `repo` write scope) is separate from `GHCR_READ_TOKEN`. While a single PAT can combine scopes, using a dedicated read-only PAT on the production host is best practice.
2. Authenticate on the production server:
   ```bash
   echo "<GHCR_READ_TOKEN>" | docker login ghcr.io -u <github-username> --password-stdin
   ```

## Deployment Steps
1. Clone this repository on your production VPS:
   ```bash
   git clone https://github.com/omarmaarouf18/saas-core-deploy.git /opt/saas-platform
   cd /opt/saas-platform
   ```
2. Copy `.env.example` to `.env` and fill in production secrets:
   ```bash
   cp .env.example .env
   ```
3. Generate mTLS certificates:
   ```bash
   mkdir -p certs && cd certs
   openssl genrsa -out ca.key 4096
   openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt -subj "/CN=SaaS-Platform-Prod-Root-CA"
   SERVICES=("api-gateway" "auth-service" "chat-service" "notification-service" "user-service")
   for service in "${SERVICES[@]}"; do
     openssl genrsa -out "${service}.key" 2048
     openssl req -new -key "${service}.key" -out "${service}.csr" -subj "/CN=${service}" -addext "subjectAltName = DNS:${service}, DNS:localhost, IP:127.0.0.1"
     cat <<EOF > "${service}.ext"
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = DNS:${service}, DNS:localhost, IP:127.0.0.1
EOF
     openssl x509 -req -in "${service}.csr" -CA ca.crt -CAkey ca.key -CAcreateserial -out "${service}.crt" -days 3650 -sha256 -extfile "${service}.ext"
     rm -f "${service}.csr" "${service}.ext"
   done
   openssl req -x509 -newkey rsa:2048 -nodes -keyout api-gateway-external.key -out api-gateway-external.crt -days 3650 -subj "/CN=localhost" -addext "subjectAltName = DNS:localhost, IP:127.0.0.1"
   chmod 600 *.key && chmod 644 *.crt && cd ..
   ```
4. Pull and start containers:
   ```bash
   docker compose pull && docker compose up -d
   ```

For full setup documentation, reverse proxy configuration, firewall rules, and source code, visit the primary development repository: [omarmaarouf18/saas-core](https://github.com/omarmaarouf18/saas-core).
