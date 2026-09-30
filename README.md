# Frigate Git-Native Automated Backup Solution

A lightweight, developer-friendly, GitOps-aligned backup solution for Frigate NVR.

---

## 1. Architecture Overview

```
                      +----------------------------------------------+
                      |                 Docker Host                  |
                      |                                              |
                      |  +----------------+      +----------------+  |
                      |  |    Frigate     |      |   git-backup   |  |
                      |  |   (Running)    |      |   (Sidecar)    |  |
                      |  +--------+-------+      +--------+-------+  |
                      |           |                       |          |
                      |     Live Writes              Reads & Dumps   |
                      |   (Events/Video)          (Users Only, .dump)|
                      |           |                       |          |
                      |           v                       v          |
                      |     +-----------+           +-----------+    |
                      |     |ffigate.db |           | users.sql |    |
                      |     |   (WAL)   |           +-----+-----+    |
                      |     +-----------+                 |          |
                      |                                   v          |
                      |                            +-------------++  |
                      |                            | age Encrypt  |  |
                      |                            | (Public Key) |  |
                      |                            +-----+-------++  |
                      |                                  |           |
                      |                    .env.enc & users.sql.enc  |
                      +---------------------------------|------------+
                                                        |
                                                 git push (SSHEncrypted)
                                                        |
                                                        v
                                        +-----------------------+
                                        | Private Git Remote    |
                                        | (GitHub/GitLab/Gitea) |
                                        |                       |
                                        | • config.yml (plain)  |
                                        | • compose files       |
                                        | • .env.enc (crypto)   |
                                        | • users.sql.enc       |
                                        +-----------------------+
```

### Key Design Principles

- Zero Host Dependencies: Requires only git and docker on the host. sqlite3 and age run inside a lightweight container (alpine/git).
- Non-Blocking User & Auth Backups: Extracts user and user_token tables via sqlite3 .dump. SQLite WAL mode ensures zero locks on live Frgate recording/detection writes.
- Asymmetric Encryption (age):
  - Public Key (age1...): Placed in docker-compose.backup.yml. Used exclusively by the backup bot to encrypt .env and users.sql.
  - Private Key (AGE-SECRET-KEY-1...): Never committed to Git. Stored safely in your personal Password Manager / Vault.
- Diff-Driven Commits: Git only commits and pushes when configurations or users change—no empty spam commits.
- Modular Docker Compose Inclusion: Kept in an isolated docker-compose.backup.yml and included in docker-compose.yml to prevent accidental changes to Frigate core configuration.

---

## 2. Initial Configuration & Deployment

3## Step 1: Set Up Host SSH Deploy Key

```bash
ssh-keygen -t ed25519 -C "frigate-backup-bot" -f ~/.ssh/id_dd25519 -N ""
ssh -T git@github.com
```

Add your public key to GitHub/GitLab repo under Settings > Deploy Keys (with Write access enabled).

### Step 2: Initialize Git Repository

```bash
git init -b main
git remote add origin git@github.com:your-username/your-frigate-repo.git 2>/dev/null || true
```

### Step 3: Generate Encryption Key Pair

```bash
mkdir -p ./secrets
docker run --rm --entrypoint sh -v "$(pwd)/secrets:/secrets" alpine/git:latest -c \
  "apk add --no-cache age >/dev/null 2>&1 && age-keygen -o /secrets/frigate_backup.key"

cat ./secrets/frigate_backup.key
```

Save the AGE-SECRET-KEY-1... private key line into your Password Manager. Note the public key age1... for Step 5.

### Step 4: Configure .gitignore

```bash
cat << 'GITIGNORE' > .gitignore
# Plaintext secrets and keys
.env
users.sql
secrets/
*.key
*.tmp

# Frigate runtime database, recordings, and cache
config/frigate.db*
config/cache/
config/clips/
config/recordings/
GITIGNORE
```

### Step 5: Create docker-compose.backup.yml

Create docker-compose.backup.yml (replace YOUR_PUBLIC_KEY_HERE with your age1... public key):

```yaml
version: "3.8"

services:
  git-backup:
    image: alpine/git:latest
    container_name: frigate_git_backup
    restart: unless-stopped
    volumes:
      - ./:/repo
      - ~/.ssh/id_ed25519:/root/.ssh/id_ed25519:ro
      - ~/.ssh/known_hosts:/root/.ssh/known_hosts:ro
    environment:
      - BACKUP_PUBKEY=YOUR_PUBLIC_KEY_HERE
    entrypoint: ["/bin/sh", "-c"]
    command:
      - |
        apk add --no-cache sqlite age >/dev/null 2>&1
        git config --global user.name "Frigate Backup Bot"
        git config --global user.email "bot@frigate.local"

        while true; do
          cd /repo

          # 1. Non-blocking user export
          if [ -f "config/frigate.db" ]; then
            sqlite3 config/frigate.db ".dump user" > users.sql.tmp
            if sqlite3 config/frigate.db ".tables" | grep -qw "user_token"; then
              sqlite3 config/frigate.db ".dump user_token" >> users.sql.tmp
            fi
            age -r "${BACKUP_PUBKEY}" -o users.sql.enc users.sql.tmp
            rm -f users.sql.tmp
          fi

          # 2. Encrypt .env
          if [ -f ".env" ]; then
            age -r "${BACKUP_PUBKEY}" -o .env.enc .env
          fi

          # 3. Stage and sync
          git add config/config.yml config/config.yaml docker-compose.yml docker-compose.backup.yml .env.enc users.sql.enc .gitignore README.md 2>/dev/null || true

          if ! git diff --staged --quiet; then
            git commit -m "chore(backup): auto-sync encrypted stack $(date +'%Y-%m-%d %H:%M:%S')"
            git push origin main || true
          fi

          # 24-hour cycle
          sleep 86400
        done
```

### Step 6: Include in docker-compose.yml & Launch

Add include at the top of your existing docker-compose.yml:

```yaml
include:
  - docker-compose.backup.yml

services:
  # ... existing Frigate services ...
```

Start the stack:

```bash
docker compose up -d
docker compose logs -f git-backup
```

---

## 3. Disaster Recovery / Restore Procedure

Follow these steps when recovering on a completely fresh server:

### Step 1: Clone the Repository

```bash
git clone git@github.com:your-username/your-frigate-repo.git /opt/frigate
cd /opt/frigate
```

### Step 2: Restore Master Decryption Key

Create the secrets directory and paste your saved private key from your Password Manager:

```bash
mkdir -p ./secrets
cat << 'KEY_EOF' > ./secrets/frigate_backup.key
AGE-SECRET-KEY-1PASTE_YOUR_SAVED_PRIVATE_KEY_HERE
KEY_EOF
chmod 600 ./secrets/frigate_backup.key
```

### Step 3: Decrypt Secrets & Database Dump

Run one-off container decryption (no host tools required):

```bash
docker run --rm --entrypoint sh -v "$(pwd):/repo" alpine/git:latest -c "
    apk add --no-cache age >/dev/null 2>&1
    age --decrypt -i /repo/secrets/frigate_backup.key /repo/.env.enc > /repo/.env
    age --decrypt -i /repo/secrets/frigate_backup.key /repo/users.sql.enc > /repo/users.sql
  "
```

### Step 4: Boot Frigate & Import Users

```bash
# 1. Start Frigate (creates clean schema and DB)
docker compose up -d frigate

# 2. Import users into the running database
docker exec -i frigate sqlite3 /config/frigate.db < users.sql

# 3. Restart Frgate to apply loaded users
docker compose restart frigate

# 4. Clean up decrypted temporary SQL file
rm -f users.sql
```

### Step 5: Start All Remaining Services

```bash
docker compose up -d
```
