A hands-on Privileged Access Management (PAM) lab built on Proxmox, implementing dynamic secret management, SSH OTP access, and Linux privilege hardening using HashiCorp Vault.

---

## Tech Stack

| Category          | Tech Stack                            |
| ----------------- | ------------------------------------- |
| Secret Management | HashiCorp Vault 2.0.3                 |
| Container Runtime | Docker + Docker Compose               |
| Database          | PostgreSQL 14                         |
| OS                | Ubuntu Server 22.04 LTS               |
| Hypervisor        | Proxmox VE                            |
| Auth Helper       | vault-ssh-helper                      |
| Linux PAM         | pam_faillock, pam_pwquality, pam_exec |
| Audit             | auditd, vault audit log               |

---

## VM Overview

|VM|Hostname|IP|Role|RAM|Disk|
|---|---|---|---|---|---|
|VM 1|vault-pam-server|192.168.1.110|Vault Server + Docker|2 GB|20 GB|
|VM 2|target-pam-server|192.168.1.120|Target Server (PostgreSQL + SSH)|2 GB|20 GB|

---
 
## Architecture

<img src="architecture-and-flow.png"></img>

---

## Phase 1 — Network Setup

### Static IP Configuration

**On both VMs**, check interface name first:

```bash
ip a
# Typically ens18 on Proxmox VMs
```

**VM 1 (vault-pam-server):**

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  ethernets:
    ens18:
      dhcp4: false
      addresses:
        - 192.168.1.110/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

```bash
sudo netplan apply
```

**VM 2 (target-pam-server):** repeat with IP `192.168.1.120`.

Verify connectivity from VM 1:

```bash
ping 192.168.1.120 -c 4
```

---

## Phase 2 — Linux PAM Hardening (VM 2)

All commands in this phase run on **VM 2 (target-pam-server)**.

### 2.1 — Install Dependencies

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y auditd audispd-plugins libpam-pwquality nginx
```

### 2.2 — Create Users with Different Roles

```bash
sudo adduser dev_user       # passwd: Developer123!
sudo adduser ops_user       # passwd: Operator123! 
sudo adduser readonly_user  # passwd: Readonly123!

sudo groupadd developers
sudo groupadd operations
sudo usermod -aG developers dev_user
sudo usermod -aG operations ops_user
```

### 2.3 — Sudoers Least Privilege Policy

```bash
sudo visudo -f /etc/sudoers.d/lab_policy
```

```bash
# dev_user: restart application services only
dev_user ALL=(ALL) NOPASSWD: /bin/systemctl restart nginx, \
                              /bin/systemctl status nginx

# ops_user: manage services and read logs
ops_user ALL=(ALL) NOPASSWD: /bin/systemctl *, \
                              /bin/journalctl *

# Enable sudo logging
Defaults logfile="/var/log/sudo.log"
Defaults log_input, log_output
Defaults iolog_dir="/var/log/sudo-io/%{user}"
```

Verify syntax:

```bash
sudo visudo -c -f /etc/sudoers.d/lab_policy
```

### 2.4 — PAM Password Policy

```bash
sudo nano /etc/pam.d/common-password
```

Find the line containing `pam_pwquality.so` and replace it with:

```
password requisite pam_pwquality.so retry=3 minlen=12 dcredit=-1 ucredit=-1 ocredit=-1 lcredit=-1
```

### 2.5 — Login Lockout Policy

```bash
sudo nano /etc/pam.d/common-auth
```

Add at the very top:

```
auth required pam_faillock.so preauth silent audit deny=5 unlock_time=900
```

### 2.6 — Auditd Configuration

```bash
sudo nano /etc/audit/rules.d/lab.rules
```

```bash
-D
-b 8192

-a always,exit -F arch=b64 -S execve -F euid=0 -F auid!=4294967295 -k privileged_cmds
-w /etc/passwd -p wa -k user_changes
-w /etc/shadow -p wa -k user_changes
-w /etc/sudoers -p wa -k sudo_changes
-w /etc/sudoers.d/ -p wa -k sudo_changes
-w /etc/pam.d/ -p wa -k pam_changes
-w /etc/ssh/sshd_config -p wa -k ssh_config
-w /var/log/wtmp -p wa -k logins
-w /var/log/btmp -p wa -k failed_logins
```

```bash
sudo systemctl restart auditd
sudo augenrules --load
sudo systemctl enable auditd
sudo auditctl -l
```

### 2.7 — SSH Hardening

```bash
sudo nano /etc/ssh/sshd_config
```

Set or add:

```
PermitRootLogin no
MaxAuthTries 3
LogLevel VERBOSE
ClientAliveInterval 300
ClientAliveCountMax 2
KbdInteractiveAuthentication yes
UsePAM yes
```

```bash
sudo systemctl restart sshd
```

### 2.8 — Test PAM Policies

```bash
su dev_user
sudo systemctl restart nginx     # should succeed
sudo systemctl restart sshd      # should be denied
exit

su ops_user
sudo journalctl -n 20            # should succeed
exit

su readonly_user
sudo ls /root                    # should be denied
exit

sudo tail -30 /var/log/sudo.log
sudo tail -30 /var/log/auth.log
```

---

## Phase 3 — PostgreSQL Setup (VM 2)

```bash
sudo apt install -y postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

```bash
sudo -u postgres psql
```

```sql
CREATE DATABASE appdb;
CREATE USER vault_admin WITH PASSWORD 'VaultAdminPass2026!' CREATEROLE;
GRANT ALL PRIVILEGES ON DATABASE appdb TO vault_admin;

\c appdb
GRANT ALL ON SCHEMA public TO vault_admin;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO vault_admin;
ALTER DEFAULT PRIVILEGES FOR ROLE vault_admin IN SCHEMA public GRANT ALL ON TABLES TO PUBLIC;

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW()
);

ALTER TABLE users OWNER TO vault_admin;

INSERT INTO users (name, email) VALUES
    ('Alice', 'alice@example.com'),
    ('Bob', 'bob@example.com');

\q
```

Allow remote connections from Vault server:

```bash
sudo nano /etc/postgresql/14/main/postgresql.conf
```

```
listen_addresses = '*'
```

```bash
sudo nano /etc/postgresql/14/main/pg_hba.conf
```

Add at the bottom:

```
host    appdb    vault_admin    192.168.1.110/32    md5
host    appdb    all            192.168.1.110/32    md5
```

```bash
sudo systemctl restart postgresql
```

Test connection from VM 1:

```bash
sudo apt install -y postgresql-client
psql -h 192.168.1.120 -U vault_admin -d appdb
\q
```

---

## Phase 4 — Docker Installation (VM 1)

All commands in this phase run on **VM 1 (vault-pam-server)**.

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

sudo usermod -aG docker $USER
newgrp docker

docker --version
docker compose version
```

---

## Phase 5 — HashiCorp Vault Setup (VM 1)

### 5.1 — Directory Structure

```bash
mkdir -p ~/vault-lab/{config,data,logs,policies}
cd ~/vault-lab
```

### 5.2 — Vault Configuration

```bash
nano ~/vault-lab/config/vault.hcl
```

```hcl
disable_mlock = true

storage "file" {
  path = "/vault/data"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = true
}

ui = true

api_addr     = "http://192.168.1.110:8200"
cluster_addr = "http://192.168.1.110:8201"

log_level = "info"
log_file  = "/vault/logs/vault.log"
```

> **Note:** `disable_mlock = true` is required for Proxmox VMs as the hypervisor restricts the mlock syscall inside containers.

### 5.3 — Docker Compose

```bash
nano ~/vault-lab/docker-compose.yml
```

```yaml
services:
  vault:
    image: hashicorp/vault:latest
    container_name: vault-server
    restart: unless-stopped
    ports:
      - "8200:8200"
      - "8201:8201"
    volumes:
      - ./config:/vault/config
      - ./data:/vault/data
      - ./logs:/vault/logs
      - ./policies:/vault/policies
    environment:
      - VAULT_ADDR=http://0.0.0.0:8200
      - VAULT_API_ADDR=http://192.168.1.110:8200
    command: vault server -config=/vault/config/vault.hcl
    healthcheck:
      test: ["CMD", "vault", "status"]
      interval: 10s
      timeout: 5s
      retries: 5
```

> **Note:** The `version` field and `cap_add: IPC_LOCK` are intentionally omitted. The `version` field is obsolete in newer Docker Compose, and `IPC_LOCK` is unnecessary when `disable_mlock = true`.

### 5.4 — Fix Permissions and Start

```bash
sudo chown -R 100:1000 ~/vault-lab/logs
sudo chown -R 100:1000 ~/vault-lab/data
sudo chmod -R 755 ~/vault-lab/logs
sudo chmod -R 755 ~/vault-lab/data

docker compose up -d
docker compose logs -f vault
# Wait for "vault is sealed" or "seal configuration missing" — both are normal
# Ctrl+C to exit logs
```

### 5.5 — Install Vault CLI

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install -y vault
```

### 5.6 — Initialize Vault

```bash
export VAULT_ADDR='http://192.168.1.110:8200'

docker exec vault-server vault operator init -key-shares=3 -key-threshold=2
```

**IMPORTANT:** Save the 3 unseal keys and root token somewhere safe outside the VM. These cannot be recovered if lost.

### 5.7 — Unseal Vault

```bash
docker exec -it vault-server vault operator unseal
# Enter Unseal Key 1

docker exec -it vault-server vault operator unseal
# Enter Unseal Key 2

docker exec vault-server vault status
# Sealed should show: false
```

### 5.8 — Set Environment Variables

```bash
nano ~/.bashrc
```

Add at the bottom:

```bash
export VAULT_ADDR='http://192.168.1.110:8200'
export VAULT_TOKEN='hvs.your-root-token-here'
```

```bash
source ~/.bashrc

vault status
```

---

## Phase 6 — Enable Audit Log

```bash
vault audit enable file file_path=/vault/logs/audit.log
vault audit list
```

View real-time audit log:

```bash
tail -f ~/vault-lab/logs/audit.log | python3 -c "
import sys, json
for line in sys.stdin:
    try:
        log = json.loads(line)
        ts = log['time'][:19]
        op = log.get('request',{}).get('operation','')
        path = log.get('request',{}).get('path','')
        print(f'{ts} | {op:<8} | {path}')
    except:
        pass
"
```

---

## Phase 7 — KV Secrets (Web UI)

Open browser: `http://192.168.1.110:8200/` → login with root token.

**Enable KV Engine:**

1. Secrets → Enable new engine → KV → Next
2. Path: `secret`, Version: 2 → Enable engine

**Create secrets:**

1. Click `secret/` → Create secret
2. Path: `myapp/database`, add key-value pairs:
    - `host` → `192.168.1.120`
    - `port` → `5432`
    - `dbname` → `appdb`
    - `username` → `appuser`
    - `password` → `AppPass2026!`
3. Save

Repeat for `myapp/api`:

- `stripe_key` → `sk_test_dummy123`
- `sendgrid_key` → `SG.dummy456`

**Test versioning:**

1. Open `myapp/database` → click **Create new version**
2. Change `password` value → Save
3. Click tab **Version History** to see both versions

---

## Phase 8 — Policies (Web UI)

On the sidebar, Access Control → ACL Policies → Create ACL policy

**developer policy:**

```hcl
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}
path "secret/data/servers/*" {
  capabilities = ["deny"]
}
path "database/*" {
  capabilities = ["deny"]
}
```

**operations policy:**

```hcl
path "secret/data/myapp/*" {
  capabilities = ["create", "read", "update", "list"]
}
path "secret/data/servers/*" {
  capabilities = ["read", "list"]
}
path "database/creds/readonly-role" {
  capabilities = ["read"]
}
path "database/creds/readwrite-role" {
  capabilities = ["read"]
}
```

**admin policy:**

```hcl
path "*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
path "sys/audit" {
  capabilities = ["read", "list"]
}
```

---

## Phase 9 — Dynamic Database Secrets

### 9.1 — Enable Database Engine (Web UI)

Secrets → Enable new engine → Databases → path: `database` → Enable

### 9.2 — Configure PostgreSQL Connection (Web UI)

1. Secrets → database → Create connection
2. Plugin: PostgreSQL
3. Connection name: `appdb`
4. Connection URL: `postgresql://{{username}}:{{password}}@192.168.1.120:5432/appdb?sslmode=disable`
5. Username: `vault_admin`, Password: your vault_admin password
6. Allowed roles: `readonly-role,readwrite-role`
7. Save → when prompted to rotate root credentials, click **Enable**

### 9.3 — Create Roles (CLI)

```bash
vault write database/roles/readonly-role \
    db_name=appdb \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
                         GRANT CONNECT ON DATABASE appdb TO \"{{name}}\"; \
                         GRANT USAGE ON SCHEMA public TO \"{{name}}\"; \
                         GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"

vault write database/roles/readwrite-role \
    db_name=appdb \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
                         GRANT CONNECT ON DATABASE appdb TO \"{{name}}\"; \
                         GRANT USAGE ON SCHEMA public TO \"{{name}}\"; \
                         GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="30m" \
    max_ttl="4h"
```

### 9.4 — Generate Dynamic Credentials (Web UI)

Secrets → database → click a role → Generate credentials → copy username and password

Test the credential:

```bash
psql -h 192.168.1.120 -U <generated-username> -d appdb
SELECT * FROM users;   # should work for readonly-role
INSERT INTO users (name, email) VALUES ('Test', 'test@test.com');  # denied for readonly-role
\q
```

### 9.5 — View and Revoke Leases

```bash
vault list sys/leases/lookup/database/creds/readonly-role
vault lease revoke -prefix database/creds/readonly-role
```

Via UI: Access → Leases → navigate to `database/creds/readonly-role/` → Revoke

---

## Phase 10 — Secret Rotation

### 10.1 — Rotate Root Credential (Web UI)

Secrets → database → click `⋯` next to `appdb` → Rotate root credentials → Rotate

After this, only Vault knows the new `vault_admin` password. This is intentional.

> **Note:** Rotating root credentials may cause existing active leases to disappear from the lease tracker, as Vault refreshes its internal connection pool. This is expected behavior — generate new credentials after rotation.

### 10.2 — Tune SSH Secret TTL (CLI)

```bash
vault secrets tune -default-lease-ttl=8h -max-lease-ttl=8h ssh/
```

---

## Phase 11 — Auth Methods (Web UI)

### 11.1 — Enable Userpass

Access Control → Authentication Methods → Enable new method → Username & Password → path: `userpass` → Enable

### 11.2 — Create Users

Access Control → Authentication Methods → userpass → Create user

| Username | Password       | Policy     |
| -------- | -------------- | ---------- |
| alice    | AlicePass2026! | developer  |
| bob      | BobPass2026!   | operations |

> **Note:** In Vault 2.0.3 UI, the policy field may be labeled **Generated Token's Policies** — expand that section if not visible. Alternatively, assign policies via CLI:

```bash
vault write auth/userpass/users/alice password="AlicePass2024!" token_policies="developer"
vault write auth/userpass/users/bob password="BobPass2024!" token_policies="operations"
```

### 11.3 — Test Access Control

Sign out → login as `alice`:

- Secrets → secret → myapp → database → **accessible**
- Secrets → database → **denied** (developer policy)

Login as `bob`:

- Secrets → database → readonly-role → Generate → **accessible** (operations policy)

---

## Phase 12 — SSH OTP Access

### 12.1 — Install vault-ssh-helper (VM 2)

```bash
wget https://releases.hashicorp.com/vault-ssh-helper/0.2.1/vault-ssh-helper_0.2.1_linux_amd64.zip
sudo apt install -y unzip
unzip vault-ssh-helper_0.2.1_linux_amd64.zip
sudo mv vault-ssh-helper /usr/local/bin/
sudo chmod 0755 /usr/local/bin/vault-ssh-helper
```

### 12.2 — Configure Helper

```bash
sudo mkdir -p /etc/vault-ssh-helper.d
sudo nano /etc/vault-ssh-helper.d/config.hcl
```

```hcl
vault_addr      = "http://192.168.1.110:8200"
ssh_mount_point = "ssh"
tls_skip_verify = true
allowed_roles   = "*"
```

### 12.3 — Configure PAM for SSH

```bash
sudo nano /etc/pam.d/sshd
```

Replace the top lines so the file starts with:

```
auth requisite pam_exec.so quiet expose_authtok log=/var/log/vault-ssh.log /usr/local/bin/vault-ssh-helper -dev -config=/etc/vault-ssh-helper.d/config.hcl
auth required pam_permit.so
# PAM configuration for the Secure Shell service
# Standard Un*x authentication.
#@include common-auth
```

> **Important:** Comment out `@include common-auth` to prevent double password prompts. Replace `pam_unix.so` with `pam_permit.so` so SSH session proceeds immediately after OTP is validated by Vault.

### 12.4 — Configure sshd_config (VM 2)

```bash
sudo nano /etc/ssh/sshd_config
```

Set:

```
KbdInteractiveAuthentication yes
UsePAM yes
PasswordAuthentication no
```

> **Note:** On Ubuntu 22.04 with newer OpenSSH, use `KbdInteractiveAuthentication` instead of the deprecated `ChallengeResponseAuthentication`.

```bash
sudo systemctl restart sshd
```

### 12.5 — Enable SSH Secret Engine (VM 1)

```bash
vault secrets enable ssh

vault write ssh/roles/dev-role \
    key_type=otp \
    default_user=dev_user \
    cidr_list=192.168.1.120/32

vault write ssh/roles/ops-role \
    key_type=otp \
    default_user=ops_user \
    cidr_list=192.168.1.120/32

vault write ssh/roles/readonly-role-ssh \
    key_type=otp \
    default_user=readonly_user \
    cidr_list=192.168.1.120/32
```

> **Note:** Role name `readonly-role-ssh` is intentionally different from `readonly-role` (database) to avoid conflict.

### 12.6 — Generate OTP and Connect

```bash
# Generate OTP
vault write -f ssh/creds/dev-role ip=192.168.1.120
```

Copy the `key` value from output, then **immediately** SSH:

```bash
ssh dev_user@192.168.1.120
# Paste OTP when prompted for password
```

> **Important:** Use the OTP immediately after generating — do not delay. OTP is single-use and will be invalidated if submitted more than once or if the lease expires.

### 12.7 — Verify Least Privilege After Login

Once logged in as `dev_user`:

```bash
sudo systemctl restart nginx   # should succeed (sudoers policy)
sudo cat /etc/shadow            # should be denied
```

This demonstrates **two layers of PAM working simultaneously**: Vault controls _who can SSH and for how long_, while Linux sudoers controls _what they can do once inside_.

### 12.8 — Set Session Auto-Logout (Optional)

For a hard 8-hour session limit per user:

```bash
# On VM 2, for each user
sudo nano /home/dev_user/.bashrc
```

Add at the bottom:

```bash
TMOUT=28800
readonly TMOUT
export TMOUT
```

Repeat for `ops_user` and `readonly_user`.

---
## Key Concepts Demonstrated

- **Ephemeral credentials** — Database and SSH credentials are never static; all auto-expire
- **No standing privilege** — No always-on privileged accounts; all access must be explicitly requested
- **Least privilege** — Role-based access at both Vault policy level and Linux sudoers level
- **Root credential rotation** — Database admin password is managed and rotated entirely by Vault
- **Dual-layer audit trail** — Both Vault audit log and Linux auditd provide full forensic trail
- **Zero knowledge** — After root rotation, no human knows the database admin password
