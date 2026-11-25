# Secrets Management Design

## Overview

All projects use a centralised HashiCorp Vault instance as the single source of truth for secrets. No secrets, hashes, or sensitive configuration are stored in application config files or on application machines. Each machine stores only a Vault authentication token in its OS credential store, retrieves secrets at startup, and holds them in memory.

## Infrastructure

### Vault Host

- **Hardware:** Dedicated Raspberry Pi (1GB RAM).
- **OS:** Raspberry Pi OS Lite (64-bit).
- **Network:** Static IP on the local network. All application machines reach Vault over LAN only.
- **Purpose:** Runs HashiCorp Vault community edition and nothing else.

### Storage Backend

Vault uses the **filesystem storage backend** on the Pi. This is the recommended backend for single-node deployments. No external database required.

```hcl
storage "file" {
  path = "/opt/vault/data"
}
```

### Listener

TLS-enabled listener bound to the Pi's static IP. Self-signed or internal CA certificate distributed to all application machines.

```hcl
listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_cert_file = "/opt/vault/tls/vault-cert.pem"
  tls_key_file  = "/opt/vault/tls/vault-key.pem"
}
```

### Auto-Unseal

Vault seals itself on restart and requires unsealing before it can serve secrets. Without a cloud KMS, the unseal key is stored in a restricted file on the Pi.

- **File:** `/opt/vault/unseal/key` owned by the `vault` user, permissions `600`.
- **Systemd unit:** A post-start script reads the unseal key and calls `vault operator unseal` automatically.
- **Physical security note:** Anyone with physical access to the Pi and root access can read this file. For a home LAN this is an accepted trade-off. If stronger guarantees are needed, a TPM HAT can be added to the Pi to hardware-bind the unseal key.

### Systemd Service

```ini
[Unit]
Description=HashiCorp Vault
After=network-online.target
Wants=network-online.target

[Service]
User=vault
Group=vault
ExecStart=/usr/bin/vault server -config=/etc/vault.d/vault.hcl
ExecStartPost=/opt/vault/scripts/auto-unseal.sh
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### Auto-Unseal Script

```bash
#!/bin/bash
export VAULT_ADDR="https://127.0.0.1:8200"
export VAULT_CACERT="/opt/vault/tls/vault-cert.pem"

until vault status 2>&1 | grep -q "Sealed.*true"; do
  sleep 1
done

vault operator unseal "$(cat /opt/vault/unseal/key)"
```

## Authentication

### AppRole Auth Method

Each application authenticates to Vault using the **AppRole** method. This provides machine-to-machine authentication without user interaction.

- **Role ID:** Identifies the application. Stored in the app's config file (not sensitive, acts like a username).
- **Secret ID:** Authenticates the application. Stored in the OS credential store on the application's machine.

#### Login Flow

1. Application reads its Role ID from config and Secret ID from the OS credential store.
2. Application calls `POST /v1/auth/approle/login` with both values.
3. Vault returns a short-lived token.
4. Application uses the token to read its secrets from Vault.
5. Token and secrets are held in memory only.

### OS Credential Store for Secret ID

Each application machine stores its Vault Secret ID in the OS-native credential store:

| OS | Store | Access Method |
|---|---|---|
| Windows | Windows Credential Manager (DPAPI) | .NET: `System.Security.Cryptography.ProtectedData` / `CredentialManager` package. Python: `keyring` library. |
| Linux | kernel keyring or libsecret | .NET: `keyring` via interop or file with `600` permissions owned by service user. Python: `keyring` library. |

## Secrets Engine

### KV Version 2

All secrets use the **KV v2 secrets engine**, which provides versioning and soft-delete.

```bash
vault secrets enable -path=secrets kv-v2
```

### Path Structure

```
secrets/
  hunter-industries-api/
    database        # SQLConnectionString, GitHubSQLConnectionString
    auth            # SecretKey, Control Panel credentials
  deployment-manager/
    auth            # Any deployment credentials
  server-status/
    auth            # Any API credentials
  wiki-site/
    auth            # Any API credentials
  quebecs-cave/
    twitch          # ClientId, ClientSecret, StreamerUserId
    database        # Connection string
  google-drive-sync/
    google          # OAuth credentials (credentials.json contents)
  database-blm-creator/
    database        # ConnectionString
    ftp             # FTPURL, FTPUsername, FTPPassword
  portfolio/
    api             # API_USERNAME, API_PASSWORD, SESSION_SECRET
  sbt-api/
    auth            # ClientId hash, ClientSecret hash, signing key
    webhook         # Webhook URL, webhook secret
    smtp            # Host, port, username, password
  sbt-installer/
    smtp            # Host, port, username, password
  archive-scanner/
    ao3             # Username, password
    smtp            # Host, port, username, password
  password-manager/
    jwt             # JWT signing key
    encryption      # Salt, Argon2id parameters
  blackbox/
    auth            # API key hashes, salts
    github          # GitHub API token
  self-hosted-llm/
    cloudflare      # Tunnel credentials, JWT validation config
    llm             # Model API keys (if cloud LLM used)
  virtual-assistant/
    jwt             # JWT signing key
    llm             # Cloud LLM API keys (if cloud LLM used)
    telegram        # Bot token
    smtp            # Host, port, username, password
  bibliotheca-fabulae/
    sources         # Per-source encrypted credentials
    smtp            # Host, port, username, password
  home-bills-tracker/
    jwt             # JWT signing key
    smtp            # Host, port, username, password
  droid-brain/
    llm             # Cloud LLM API keys
  bones-and-brew/
    database        # PostgreSQL connection string
    oauth           # OAuth2 client credentials
  lingua-ficta/
    github          # GitHub API token (issue submission)
```

## Vault Policies

Each application gets a policy granting read-only access to its own secrets path and nothing else.

### Example: Archive Scanner Policy

```hcl
path "secrets/data/archive-scanner/*" {
  capabilities = ["read"]
}
```

### Admin Policy

A single admin policy for managing all secrets. Used only for initial setup and rotation.

```hcl
path "secrets/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "auth/approle/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "sys/policies/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
```

## Application Integration

### .NET Core / .NET 5+ Projects (appsettings.json)

A shared `VaultClient` utility retrieves secrets at startup and injects them into the configuration pipeline.

```csharp
public static class VaultExtensions
{
    public static IConfigurationBuilder AddVault(
        this IConfigurationBuilder builder,
        string vaultAddr,
        string roleId,
        string secretPath)
    {
        // 1. Read Secret ID from OS credential store
        // 2. Authenticate via AppRole
        // 3. Read secrets from secretPath
        // 4. Add as in-memory configuration source
    }
}
```

**Usage in `Program.cs`:**

```csharp
builder.Configuration.AddVault(
    vaultAddr: builder.Configuration["Vault:Address"],
    roleId: builder.Configuration["Vault:RoleId"],
    secretPath: "secrets/data/sbt-api/auth"
);
```

**Remaining in `appsettings.json`** (non-sensitive only):

```json
{
  "Vault": {
    "Address": "https://192.168.x.x:8200",
    "RoleId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  },
  "Database": {
    "Path": "sbt.db"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  }
}
```

### .NET Framework Projects (web.config / App.config)

.NET Framework does not have the `IConfigurationBuilder` pipeline. Instead, a static helper reads from Vault at startup and holds secrets in a singleton dictionary. The app reads from this dictionary instead of `ConfigurationManager`.

```csharp
public static class VaultSecrets
{
    private static Dictionary<string, string> _secrets = new();

    public static void Initialise()
    {
        var vaultAddr = ConfigurationManager.AppSettings["Vault:Address"];
        var roleId = ConfigurationManager.AppSettings["Vault:RoleId"];

        if (string.IsNullOrEmpty(vaultAddr) || string.IsNullOrEmpty(roleId))
            return; // No Vault config — use web.config/app.config values

        try
        {
            // 1. Read Secret ID from Windows Credential Manager (DPAPI)
            // 2. Authenticate via AppRole
            // 3. Read secrets and populate _secrets dictionary
        }
        catch
        {
            // Vault unreachable — fall back to config file values
            // Log a warning
        }
    }

    public static string Get(string key, string fallback = null)
    {
        return _secrets.TryGetValue(key, out var value) ? value : fallback;
    }
}
```

**Usage in `Global.asax.cs` (ASP.NET Web API):**

```csharp
protected void Application_Start()
{
    VaultSecrets.Initialise();
    // ...existing startup code
}
```

**Usage at point of need:**

```csharp
var connectionString = VaultSecrets.Get("SQLConnectionString",
    fallback: ConfigurationManager.ConnectionStrings["SQLConnectionString"]?.ConnectionString);
```

**Remaining in `web.config` / `App.config`** (non-sensitive only):

```xml
<appSettings>
  <add key="Vault:Address" value="https://192.168.x.x:8200" />
  <add key="Vault:RoleId" value="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" />
</appSettings>
```

Sensitive values (`connectionStrings`, passwords, API keys) are removed from the XML once Vault is live. The `VaultSecrets.Get()` fallback parameter allows backwards compatibility — if Vault is not configured, the app reads from the config file as before.

### Node.js Projects (.env)

A shared module reads from Vault at startup and merges secrets into `process.env`.

```javascript
const vault = require("node-vault")({
  endpoint: process.env.VAULT_ADDR,
  requestOptions: {
    ca: fs.readFileSync(process.env.VAULT_CA_CERT || "/path/to/vault-ca.pem"),
  },
});

async function loadSecrets(secretPath) {
  const roleId = process.env.VAULT_ROLE_ID;
  const secretId = process.env.VAULT_SECRET_ID;

  if (!roleId || !secretId) return; // No Vault config — use .env values

  try {
    const loginResult = await vault.approleLogin({
      role_id: roleId,
      secret_id: secretId,
    });
    vault.token = loginResult.auth.client_token;

    const { data } = await vault.read(`secrets/data/${secretPath}`);
    Object.entries(data.data).forEach(([key, value]) => {
      process.env[key] = value; // Overrides .env values
    });
  } catch {
    // Vault unreachable — fall back to .env values
    // Log a warning
  }
}
```

**Usage at startup (before other imports that use `process.env`):**

```javascript
require("dotenv").config();
await loadSecrets("portfolio/api");
```

**Remaining in `.env`** (non-sensitive only):

```
VAULT_ADDR=https://192.168.x.x:8200
VAULT_ROLE_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
VAULT_SECRET_ID=stored-in-os-credential-store-or-docker-secret
VITE_API_URL=http://localhost:3001
```

### Python Projects

A shared `vault_client` module retrieves secrets at startup.

```python
import hvac
import keyring

def get_secrets(vault_addr: str, role_id: str, secret_path: str) -> dict:
    secret_id = keyring.get_password("vault", role_id)

    client = hvac.Client(url=vault_addr, verify="/path/to/vault-ca.pem")
    client.auth.approle.login(role_id=role_id, secret_id=secret_id)

    response = client.secrets.kv.v2.read_secret_version(path=secret_path)
    return response["data"]["data"]
```

**Usage:**

```python
secrets = get_secrets(
    vault_addr="https://192.168.x.x:8200",
    role_id=config["vault"]["role_id"],
    secret_path="archive-scanner/ao3"
)

ao3_username = secrets["username"]
ao3_password = secrets["password"]
```

**Remaining in `config.json`** (non-sensitive only):

```json
{
  "vault": {
    "address": "https://192.168.x.x:8200",
    "role_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  },
  "relationships": [
    {
      "site": "ao3",
      "tag": "Character A/Character B",
      "track_from": "2026-11-18T10:30:00Z"
    }
  ],
  "storage": {
    "mode": "json"
  }
}
```

### Docker Containers

For containerised applications, the Secret ID is passed as an environment variable at container start rather than using the OS credential store.

**Docker Compose example:**

```yaml
services:
  home-bills-tracker:
    image: home-bills-tracker:latest
    environment:
      VAULT_ADDR: "https://192.168.x.x:8200"
      VAULT_ROLE_ID: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
      VAULT_SECRET_ID_FILE: "/run/secrets/vault_secret_id"
    secrets:
      - vault_secret_id
    volumes:
      - ./vault-ca.pem:/etc/ssl/certs/vault-ca.pem:ro

secrets:
  vault_secret_id:
    file: ./secrets/vault_secret_id.txt
```

The `vault_secret_id.txt` file is stored with `600` permissions, owned by root, and excluded from version control.

## Secret Rotation

### Application Secrets

1. Update the secret in Vault (KV v2 creates a new version automatically).
2. Restart the application. It pulls the new value on startup.
3. No config file changes needed on application machines.

### AppRole Secret IDs

Secret IDs can be configured with a TTL and rotation policy in Vault.

```bash
vault write auth/approle/role/archive-scanner \
  secret_id_ttl=720h \
  token_ttl=1h \
  token_max_ttl=4h \
  token_policies="archive-scanner"
```

When a Secret ID expires, generate a new one and update the OS credential store on the target machine.

### Vault Unseal Key

Rotate by rekeying Vault:

```bash
vault operator rekey -init -key-shares=1 -key-threshold=1
vault operator rekey -nonce=<nonce> <current-unseal-key>
```

Update `/opt/vault/unseal/key` with the new key.

## Pi Network Hardening

The Vault Pi is a LAN-only device. It must never be reachable from the internet.

### Firewall (iptables)

Lock down to only the Vault port and SSH, accepting connections only from the local subnet.

```bash
#!/bin/bash
LAN_SUBNET="192.168.x.0/24"

# Flush existing rules
iptables -F
iptables -X

# Default policy: drop everything
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT

# Allow established/related connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow Vault (8200) from LAN only
iptables -A INPUT -p tcp --dport 8200 -s "$LAN_SUBNET" -j ACCEPT

# Allow SSH (22) from LAN only
iptables -A INPUT -p tcp --dport 22 -s "$LAN_SUBNET" -j ACCEPT

# Allow ICMP (ping) from LAN only
iptables -A INPUT -p icmp -s "$LAN_SUBNET" -j ACCEPT

# Drop everything else (default policy handles this, but be explicit)
iptables -A INPUT -j DROP
```

Persist the rules so they survive reboot:

```bash
apt install iptables-persistent
netfilter-persistent save
```

### SSH Hardening

Edit `/etc/ssh/sshd_config`:

```
ListenAddress 192.168.x.x
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
AllowUsers vault-admin
```

- SSH listens only on the LAN IP, not `0.0.0.0`.
- Password auth disabled; key-based only.
- Only a dedicated `vault-admin` user can SSH in (not `pi`, not `root`).

### Disable Unnecessary Services

```bash
# Disable Wi-Fi and Bluetooth (use wired ethernet only)
echo "dtoverlay=disable-wifi" >> /boot/firmware/config.txt
echo "dtoverlay=disable-bt" >> /boot/firmware/config.txt

# Disable Avahi/mDNS (broadcasts presence on the network)
systemctl disable avahi-daemon
systemctl stop avahi-daemon
```

### Vault Listener Bind

Vault is already configured to listen on `0.0.0.0:8200` with TLS. As a belt-and-braces measure, bind it to the LAN IP only so it cannot listen on any future interface:

```hcl
listener "tcp" {
  address       = "192.168.x.x:8200"
  tls_cert_file = "/opt/vault/tls/vault-cert.pem"
  tls_key_file  = "/opt/vault/tls/vault-key.pem"
}
```

### Router-Level Protection

As an additional layer, configure the network router to:

- **Not port-forward** anything to the Vault Pi's IP.
- **Assign a static DHCP reservation** (or static IP on the Pi itself) so the IP never changes.
- **Optionally isolate** the Pi in a VLAN with only the application machines, if the router supports it.

### Automatic Security Updates

```bash
apt install unattended-upgrades
dpkg-reconfigure -plow unattended-upgrades
```

This ensures the Pi receives security patches automatically without manual intervention.

## TLS / Certificate Management

### Internal CA

Generate a self-signed CA for Vault TLS. Distribute the CA certificate to all application machines.

```bash
# Generate CA key and cert
openssl genrsa -out vault-ca-key.pem 4096
openssl req -x509 -new -nodes -key vault-ca-key.pem -sha256 -days 3650 \
  -out vault-ca.pem -subj "/CN=Vault Internal CA"

# Generate Vault server cert
openssl genrsa -out vault-key.pem 4096
openssl req -new -key vault-key.pem -out vault.csr \
  -subj "/CN=vault.local" \
  -addext "subjectAltName=IP:192.168.x.x,DNS:vault.local"
openssl x509 -req -in vault.csr -CA vault-ca.pem -CAkey vault-ca-key.pem \
  -CAcreateserial -out vault-cert.pem -days 3650 -sha256 \
  -extfile <(echo "subjectAltName=IP:192.168.x.x,DNS:vault.local")
```

### Certificate Distribution

- **Windows:** Import `vault-ca.pem` into the Trusted Root Certification Authorities store.
- **Linux:** Copy to `/usr/local/share/ca-certificates/` and run `update-ca-certificates`.
- **Docker:** Mount as a read-only volume.

## Backup

### Vault Data

Daily backup of `/opt/vault/data` to an external location (USB drive, NAS, or another machine).

```bash
#!/bin/bash
BACKUP_DIR="/mnt/backup/vault"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

systemctl stop vault
tar czf "$BACKUP_DIR/vault-data-$TIMESTAMP.tar.gz" /opt/vault/data
systemctl start vault

# Retain last 30 backups
ls -t "$BACKUP_DIR"/vault-data-*.tar.gz | tail -n +31 | xargs rm -f
```

### Unseal Key

The unseal key and CA private key should be backed up separately and stored offline (e.g. printed, USB drive in a safe). These are the only secrets that, if lost, would require full Vault reinitialization.

## Disaster Recovery

### Vault Pi Failure

1. Set up a new Pi with the same static IP.
2. Install Vault.
3. Restore `/opt/vault/data` from backup.
4. Restore TLS certificates.
5. Restore the unseal key to `/opt/vault/unseal/key`.
6. Start Vault. The auto-unseal script handles the rest.
7. All application machines reconnect automatically.

### Application Machine Failure

1. Rebuild the machine.
2. Install the application.
3. Store the application's Secret ID in the OS credential store.
4. Start the application. It authenticates to Vault and pulls its secrets.

## Per-Project Migration

Each project must support two modes: **Vault mode** (secrets from Vault) and **local mode** (secrets from config files as originally designed). The app checks for Vault configuration at startup. If present and reachable, it uses Vault. If not, it falls back to local config. This allows incremental adoption and ensures the projects remain usable without Vault infrastructure.

### Fallback Strategy

#### .NET Projects

The `AddVault` configuration extension is replaced with a `AddSecrets` method that handles both modes. Vault configuration is optional in `appsettings.json` — if the `Vault` section is absent or Vault is unreachable, the app continues with whatever is in the config file.

```csharp
public static IConfigurationBuilder AddSecrets(
    this IConfigurationBuilder builder,
    string[] secretPaths)
{
    var config = builder.Build();
    var vaultAddr = config["Vault:Address"];
    var roleId = config["Vault:RoleId"];

    if (string.IsNullOrEmpty(vaultAddr) || string.IsNullOrEmpty(roleId))
        return builder; // No Vault config — use local config as-is

    try
    {
        // 1. Read Secret ID from OS credential store
        // 2. Authenticate via AppRole
        // 3. Read each secret path
        // 4. Add as in-memory configuration source (overrides local values)
    }
    catch
    {
        // Vault unreachable — fall back to local config
        // Log a warning so the operator knows Vault isn't being used
    }

    return builder;
}
```

When Vault is active, its values are added as an in-memory configuration source that **overrides** local config values. This means `appsettings.json` can still contain fallback/default values for local development.

#### Python Projects

A `get_secrets` function attempts Vault first, then falls back to reading from the local config dict.

```python
def get_secrets(config: dict, secret_path: str) -> dict | None:
    vault_config = config.get("vault")
    if not vault_config or not vault_config.get("address"):
        return None  # No Vault config — caller uses local config

    try:
        secret_id = keyring.get_password("vault", vault_config["role_id"])
        if not secret_id:
            return None

        client = hvac.Client(
            url=vault_config["address"],
            verify=vault_config.get("ca_cert", True)
        )
        client.auth.approle.login(
            role_id=vault_config["role_id"],
            secret_id=secret_id
        )
        response = client.secrets.kv.v2.read_secret_version(path=secret_path)
        return response["data"]["data"]
    except Exception:
        # Log warning — Vault unreachable, falling back to local config
        return None
```

Usage with fallback:

```python
ao3_secrets = get_secrets(config, "archive-scanner/ao3")
ao3_username = ao3_secrets["username"] if ao3_secrets else config["ao3"]["username"]
ao3_password = ao3_secrets["password"] if ao3_secrets else config["ao3"]["password"]
```

### Project-Specific Changes

#### Existing Projects

---

#### Hunter Industries API (.NET Framework 4.7.2 — web.config)

**Vault paths:** `secrets/hunter-industries-api/database`, `secrets/hunter-industries-api/auth`

**What moves to Vault:**
- `SQLConnectionString` (SQL Server connection string with credentials)
- `GitHubSQLConnectionString`
- `SecretKey` (authentication signing key)
- Control Panel credentials (Base64-encoded username/password)

**What stays in `web.config`:**
- `Vault:Address`, `Vault:RoleId` (added to `<appSettings>`)
- Non-sensitive `<system.web>` and `<system.webServer>` configuration
- Route configuration, CORS settings, logging

**Changes required:**
1. Add the shared `VaultSecrets` class (from the .NET Framework integration pattern above).
2. Call `VaultSecrets.Initialise()` in `Application_Start`.
3. Replace direct `ConfigurationManager.ConnectionStrings[...]` reads with `VaultSecrets.Get(...)` with fallback to the config value.
4. Add `Vault:Address` and `Vault:RoleId` to `<appSettings>`.
5. Remove connection strings and credentials from `web.config` once Vault is live.

**Note:** The committed `Web.config` has empty values. Real credentials are local only — no rotation required.

---

#### Hunter Industries API Control Panel (.NET 10.0 — appsettings.json)

**Vault path:** `secrets/hunter-industries-api/auth`

**What moves to Vault:**
- Base64-encoded API credentials
- `AuthPayloadLocation` path (if sensitive)

**What stays in `appsettings.json`:**
- `Vault:Address`, `Vault:RoleId`
- `BaseURL` (if pointing to localhost/internal)
- Logging, UI configuration

**Changes required:**
1. Add the shared `AddSecrets` extension.
2. Call `builder.Configuration.AddSecrets(...)` in `Program.cs`.
3. Reads the same `secrets/hunter-industries-api/auth` path as the API itself — shared credentials.
4. Remove credentials from `appsettings.json` once Vault is live.

---

#### DeploymentManager (.NET 10.0 — appsettings.json)

**Vault path:** `secrets/deployment-manager/auth`

**What moves to Vault:**
- Any deployment credentials or tokens added during development

**What stays in `appsettings.json`:**
- `Vault:Address`, `Vault:RoleId`
- Non-sensitive configuration

**Changes required:**
1. Add the shared `AddSecrets` extension.
2. Call `builder.Configuration.AddSecrets(...)` in `Program.cs`.
3. Currently has empty/template config — integrate Vault from the start before credentials are added.

---

#### ServerStatus (.NET 8.0 — appsettings.json)

**Vault path:** `secrets/server-status/auth`

**What moves to Vault:**
- Any API credentials or tokens added during development

**What stays in `appsettings.json`:**
- `Vault:Address`, `Vault:RoleId`
- Non-sensitive configuration

**Changes required:**
1. Add the shared `AddSecrets` extension.
2. Call `builder.Configuration.AddSecrets(...)` in `Program.cs`.
3. Currently has empty/template config — integrate Vault from the start.

**Note:** Applies to both ServerStatus and ServerStatusSite repos.

---

#### Wiki-Site (.NET 8.0 — appsettings.json)

**Vault path:** `secrets/wiki-site/auth`

**What moves to Vault:**
- Any API credentials or tokens added during development

**What stays in `appsettings.json`:**
- `Vault:Address`, `Vault:RoleId`
- Non-sensitive configuration

**Changes required:**
1. Add the shared `AddSecrets` extension.
2. Call `builder.Configuration.AddSecrets(...)` in `Program.cs`.
3. Currently has empty/template config — integrate Vault from the start.

---

#### Quebecs-Cave (.NET 10.0 — appsettings.json)

**Vault paths:** `secrets/quebecs-cave/twitch`, `secrets/quebecs-cave/database`

**What moves to Vault:**
- Twitch `ClientId`, `ClientSecret`, `StreamerUserId`
- Database connection string (when populated)

**What stays in `appsettings.json`:**
- `Vault:Address`, `Vault:RoleId`
- `VideoRoot` storage path
- Non-sensitive configuration

**Changes required:**
1. Add the shared `AddSecrets` extension to both the Web and Downloader projects.
2. Call `builder.Configuration.AddSecrets(...)` in `Program.cs` for each.
3. Currently has empty placeholders — populate in Vault instead of the config file.

---

#### GoogleDriveSync (.NET Framework 4.7.2 — App.config)

**Vault path:** `secrets/google-drive-sync/google`

**What moves to Vault:**
- Google OAuth credentials (currently in external `credentials.json`)

**What stays in `App.config`:**
- `Vault:Address`, `Vault:RoleId` (added to `<appSettings>`)
- Non-sensitive application settings

**Changes required:**
1. Add the shared `VaultSecrets` class (from the .NET Framework integration pattern).
2. Call `VaultSecrets.Initialise()` at application startup.
3. Read Google OAuth credentials from Vault instead of the external `credentials.json` file.
4. Fall back to reading `credentials.json` if Vault is unavailable.

---

#### Database-BLM-Creator (.NET 6.0 — App.config)

**Vault paths:** `secrets/database-blm-creator/database`, `secrets/database-blm-creator/ftp`

**What moves to Vault:**
- `ConnectionString`
- `FTPURL`, `FTPUsername`, `FTPPassword`

**What stays in `App.config`:**
- `Vault:Address`, `Vault:RoleId` (added to `<appSettings>`)
- Non-sensitive settings

**Changes required:**
1. .NET 6.0 console app with `App.config` — can use either the .NET Core `IConfiguration` pipeline or the .NET Framework `VaultSecrets` pattern depending on how config is currently read.
2. Load database and FTP credentials from Vault at startup. Fall back to `App.config` values if Vault is unavailable.
3. Currently has empty placeholders — populate in Vault instead.

---

#### Portfolio (Node.js — .env)

**Vault path:** `secrets/portfolio/api`

**What moves to Vault:**
- `API_USERNAME`, `API_PASSWORD`
- `SESSION_SECRET`

**What stays in `.env`:**
- `VAULT_ADDR`, `VAULT_ROLE_ID`, `VAULT_SECRET_ID`
- `VITE_API_URL` (non-sensitive, used by frontend build)

**Changes required:**
1. Add `node-vault` to dependencies.
2. Add the shared Vault loader module (from the Node.js integration pattern above).
3. Call `loadSecrets("portfolio/api")` at startup before other initialisation.
4. Remove `API_USERNAME`, `API_PASSWORD`, `SESSION_SECRET` from `.env` once Vault is live.

**Note:** The `.env` file is not committed to source control — credentials are local only, no rotation required.

---

#### PythonApps / Archive Scanner (Python — config.json)

This is the existing deployed Archive Scanner. The migration steps in the "Archive Scanner" section under Upcoming Projects apply here. The committed file is `config.example.json` with placeholder values only — real credentials in `config.json` are local and not in source control. No rotation required.

---

#### Upcoming Projects

---

#### SBT API (.NET)

**Vault paths:** `secrets/sbt-api/auth`, `secrets/sbt-api/webhook`, `secrets/sbt-api/smtp`

**What moves to Vault:**
- `Authentication:ClientId` (SHA-512 hash)
- `Authentication:ClientSecret` (SHA-512 hash)
- Authentication signing key
- `Webhook:Url` and webhook secret
- SMTP host, port, username, password

**What stays in `appsettings.json`:**
- `Vault:Address`, `Vault:RoleId`
- `Database:Path`
- Logging configuration
- Non-sensitive webhook settings (`TimeoutSeconds`, `RetryCount`)

**Changes required:**
1. Add the shared `AddSecrets` extension (NuGet package or project reference).
2. Add optional `Vault` section to `appsettings.json`.
3. Call `builder.Configuration.AddSecrets(...)` in `Program.cs`.
4. Remove the `Authentication` and `Webhook:Url` sections from `appsettings.json` once Vault is live (optional — Vault values override them).

---

#### SBT Installer (.NET TUI)

**Vault path:** `secrets/sbt-installer/smtp`

**What moves to Vault:**
- SMTP host, port, username, password

**What stays in config:**
- `Vault:Address`, `Vault:RoleId`
- All non-SMTP installer configuration

**Changes required:**
1. Add the shared `AddSecrets` extension.
2. During the installer wizard, if Vault is configured, store SMTP credentials in Vault instead of writing them to the local config file. If Vault is not configured, write to local config as before.
3. At runtime (when sending notifications), read SMTP credentials from Vault if available, otherwise from local config.

---

#### Archive Scanner (Python)

**Vault paths:** `secrets/archive-scanner/ao3`, `secrets/archive-scanner/smtp`

**What moves to Vault:**
- `ao3.username`, `ao3.password`
- `smtp.host`, `smtp.port`, `smtp.username`, `smtp.password`

**What stays in `config.json`:**
- `vault.address`, `vault.role_id` (optional section)
- `relationships` list
- `storage` settings
- `smtp.use_tls` (non-sensitive)

**Changes required:**
1. Add `hvac` and `keyring` to dependencies.
2. Add the shared `get_secrets` function.
3. At startup, attempt to load AO3 and SMTP secrets from Vault. Fall back to `config.json` values if Vault is unavailable.
4. Remove `ao3.username`, `ao3.password`, `smtp.username`, `smtp.password` from `config.json` once Vault is live (optional).

---

#### Password Manager (.NET + React)

**Vault paths:** `secrets/password-manager/jwt`, `secrets/password-manager/encryption`

**What moves to Vault:**
- JWT signing key
- Argon2id salt and parameters

**What stays in `appsettings.json`:**
- `Vault:Address`, `Vault:RoleId`
- `Database:ConnectionString`
- JWT expiry settings (`AccessTokenExpiryMinutes`, `RefreshTokenExpiryDays`)
- Logging, CORS configuration

**Changes required:**
1. Add the shared `AddSecrets` extension.
2. Call `builder.Configuration.AddSecrets(...)` in `Program.cs`.
3. The master password → Argon2id → AES-256-GCM flow is unchanged. Only the static parameters (salt, iteration count) and the JWT key come from Vault instead of config.

**Note:** The user's master password and derived encryption key are never stored anywhere — that design is unchanged.

---

#### Blackbox (.NET)

**Vault paths:** `secrets/blackbox/auth`, `secrets/blackbox/github`

**What moves to Vault:**
- API key hashes and salts
- GitHub API token

**What stays in `appsettings.json`:**
- `Vault:Address`, `Vault:RoleId`
- Database path
- Logging, rate limiting configuration
- Non-sensitive API settings

**Changes required:**
1. Add the shared `AddSecrets` extension.
2. Call `builder.Configuration.AddSecrets(...)` in `Program.cs`.
3. API key registration flow: when a new installation registers, the hash and salt are written to Vault instead of `appsettings.json`. Requires write access to `secrets/data/blackbox/auth` for the registration endpoint only — add a separate policy or use a short-lived token with write capability.
4. Fall back to reading hashes from config if Vault is unavailable.

---

#### Self-Hosted LLM (.NET + React)

**Vault paths:** `secrets/self-hosted-llm/cloudflare`, `secrets/self-hosted-llm/llm`

**What moves to Vault:**
- Cloudflare tunnel credentials and JWT validation config
- Cloud LLM API keys (if cloud LLM mode is used)

**What stays in `appsettings.json`:**
- `Vault:Address`, `Vault:RoleId`
- Model configuration, rate limits, token budgets
- Logging

**Changes required:**
1. Add the shared `AddSecrets` extension.
2. Call `builder.Configuration.AddSecrets(...)` in `Program.cs`.
3. Cloudflare JWT validation public cert endpoint URL can remain in config (it's public). The tunnel token and any private config moves to Vault.

**Note:** This project already delegates auth to Cloudflare Zero Trust. The main secrets are the tunnel credentials and optional cloud LLM keys.

---

#### Virtual Assistant (.NET + Blazor Hybrid)

**Vault paths:** `secrets/virtual-assistant/jwt`, `secrets/virtual-assistant/llm`, `secrets/virtual-assistant/telegram`, `secrets/virtual-assistant/smtp`

**What moves to Vault:**
- JWT signing key
- Cloud LLM API keys (Anthropic, OpenAI — if cloud LLM enabled)
- Telegram bot token
- SMTP host, port, username, password

**What stays in `appsettings.json`:**
- `Vault:Address`, `Vault:RoleId`
- Database path
- JWT expiry settings
- Skill router configuration
- Logging

**Changes required:**
1. Add the shared `AddSecrets` extension.
2. Call `builder.Configuration.AddSecrets(...)` in `Program.cs`.
3. Features that depend on optional secrets (cloud LLM, Telegram) should check whether the secret was loaded and disable the feature gracefully if not. This is already the design intent — Vault just changes where the check happens.

---

#### Bibliotheca Fabulae (Python + React)

**Vault paths:** `secrets/bibliotheca-fabulae/sources`, `secrets/bibliotheca-fabulae/smtp`

**What moves to Vault:**
- Per-source credentials (AO3, Fimfiction, etc.)
- SMTP host, port, username, password

**What stays in `config.json`:**
- `vault.address`, `vault.role_id`
- Source site URLs and non-sensitive config
- Sync schedule, storage settings

**Changes required:**
1. Add `hvac` and `keyring` to dependencies.
2. Add the shared `get_secrets` function.
3. Remove the AES-256 encryption-at-rest layer for source credentials — Vault handles encryption now. If Vault is unavailable, fall back to the original AES-256 encrypted local storage.
4. Each source's credentials are a separate key under `secrets/bibliotheca-fabulae/sources` (e.g. `sources/ao3`, `sources/fimfiction`).

---

#### HomeBillsTracker (Python + React)

**Vault paths:** `secrets/home-bills-tracker/jwt`, `secrets/home-bills-tracker/smtp`

**What moves to Vault:**
- JWT signing key
- SMTP host, port, username, password

**What stays in config / environment:**
- `vault.address`, `vault.role_id`
- Database path (SQLite, local file)
- JWT expiry settings

**Changes required:**
1. Add `hvac` and `keyring` to dependencies.
2. Add the shared `get_secrets` function.
3. At startup, load JWT and SMTP secrets from Vault if configured. Fall back to environment variables or config file if not.
4. Docker Compose: add Vault environment variables and Docker secret for the Secret ID as shown in the Docker Containers section above.

---

#### Droid Brain (Python / .NET)

**Vault path:** `secrets/droid-brain/llm`

**What moves to Vault:**
- Cloud LLM API keys (Anthropic, OpenAI)

**What stays in config:**
- `vault.address`, `vault.role_id`
- Model selection, prompt templates, BLE configuration

**Changes required:**
1. Add `hvac` and `keyring` to dependencies (Python) or the shared `AddSecrets` extension (.NET).
2. Load LLM API keys from Vault at startup. Fall back to environment variables if Vault is unavailable.

---

#### Bones & Brew (Godot + .NET)

**Vault paths:** `secrets/bones-and-brew/database`, `secrets/bones-and-brew/oauth`

**What moves to Vault:**
- PostgreSQL connection string
- OAuth2 client ID and client secret

**What stays in `appsettings.json`:**
- `Vault:Address`, `Vault:RoleId`
- Game configuration, matchmaking settings
- Logging

**Changes required:**
1. Add the shared `AddSecrets` extension to the .NET Web API backend.
2. Call `builder.Configuration.AddSecrets(...)` in `Program.cs`.
3. The Godot client does not access Vault directly — it authenticates to the .NET backend, which holds the OAuth and database secrets.

---

#### Lingua Ficta (Python + React)

**Vault path:** `secrets/lingua-ficta/github`

**What moves to Vault:**
- GitHub API token (issue submission)

**What stays in config:**
- `vault.address`, `vault.role_id`
- Language rules, grammar definitions

**Changes required:**
1. Add `hvac` and `keyring` to dependencies.
2. Add the shared `get_secrets` function.
3. Load the GitHub token from Vault at startup. Fall back to config or environment variable if Vault is unavailable.
4. If no GitHub token is available from either source, disable the issue submission feature gracefully.

---

### Shared Libraries

To avoid duplicating the Vault integration logic across every project:

#### .NET Core / .NET 5+: `HunterIndustries.Vault` NuGet Package

A shared class library published as a local NuGet package (or Git submodule) containing:
- `AddSecrets` configuration extension method with Vault-then-local fallback
- OS credential store helper for Secret ID retrieval
- Vault health check for ASP.NET Core health endpoints

Used by: SBT API, SBT Installer, Password Manager, Blackbox, Self-Hosted LLM, Virtual Assistant, Bones & Brew, Control Panel, DeploymentManager, ServerStatus, ServerStatusSite, Wiki-Site, Quebecs-Cave.

#### .NET Framework: `HunterIndustries.Vault.Framework` NuGet Package

A separate class library targeting .NET Framework 4.7.2+ containing:
- `VaultSecrets` static class with `Initialise()` and `Get()` methods
- Windows Credential Manager (DPAPI) helper for Secret ID retrieval
- Fallback to `ConfigurationManager` values when Vault is unavailable

Used by: Hunter Industries API, GoogleDriveSync. Also usable by Database-BLM-Creator if it reads config via `ConfigurationManager`.

#### Python: `hunter_vault` Package

A shared Python package (installable via `pip install -e ./hunter-vault` or a local PyPI) containing:
- `get_secrets` function with fallback
- `keyring` wrapper for Secret ID retrieval
- Logging integration

Used by: Archive Scanner, Bibliotheca Fabulae, HomeBillsTracker, Droid Brain, Lingua Ficta.

#### Node.js: `hunter-vault` Package

A shared module (local package or copied into projects) containing:
- `loadSecrets` function that merges Vault values into `process.env`
- Fallback to `.env` values when Vault is unavailable
- TLS CA cert configuration

Used by: Portfolio.

## Setup Checklist

### One-Time Vault Setup

1. [ ] Flash Raspberry Pi OS Lite to the dedicated Pi.
2. [ ] Assign static IP (and static DHCP reservation on router).
3. [ ] Create `vault-admin` user, disable `pi` user.
4. [ ] Configure SSH: key-only auth, LAN-only listen, `AllowUsers vault-admin`.
5. [ ] Disable Wi-Fi, Bluetooth, and Avahi.
6. [ ] Configure iptables firewall (allow only ports 8200 and 22 from LAN subnet).
7. [ ] Persist firewall rules with `iptables-persistent`.
8. [ ] Enable unattended security upgrades.
9. [ ] Verify no port forwarding to this Pi on the router.
10. [ ] Install HashiCorp Vault.
11. [ ] Generate internal CA and Vault TLS certificates.
12. [ ] Configure Vault (`/etc/vault.d/vault.hcl`) — bind listener to LAN IP only.
13. [ ] Initialise Vault (`vault operator init -key-shares=1 -key-threshold=1`).
14. [ ] Store the unseal key in `/opt/vault/unseal/key` (permissions `600`, owner `vault`).
15. [ ] Back up the unseal key and root token offline.
16. [ ] Enable the AppRole auth method (`vault auth enable approle`).
17. [ ] Enable KV v2 secrets engine (`vault secrets enable -path=secrets kv-v2`).
18. [ ] Configure the systemd service with auto-unseal.
19. [ ] Set up the daily backup cron job.

### Per-Application Setup

1. [ ] Create a Vault policy for the application.
2. [ ] Create an AppRole role for the application.
3. [ ] Generate a Secret ID for the application.
4. [ ] Store the Secret ID in the OS credential store on the application machine (or Docker secret for containers).
5. [ ] Distribute the Vault CA certificate to the application machine.
6. [ ] Add Vault address and Role ID to the application's config file.
7. [ ] Populate the application's secrets in Vault.
8. [ ] Test: restart the application and verify it retrieves secrets from Vault.
