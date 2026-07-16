# Password Manager for Raspberry Pi - Implementation Plan

## Context

Build a self-hosted password manager that runs on a Raspberry Pi with encrypted storage, JWT auth, and device registration. Passwords are only decryptable when a user is authenticated — the encryption key is derived from the master password at login and held in server memory, never stored on disk or in the JWT.

---

## Tech Stack

| Layer | Choice | Why |
|-------|--------|-----|
| Backend | .NET Core 8 Web API (C#) | User preference; mature auth/middleware pipeline; `AesGcm` is built-in with hardware acceleration |
| Database | SQLite via EF Core (`Microsoft.EntityFrameworkCore.Sqlite`) | Zero-daemon, single-file DB, trivially backed up, EF Core handles migrations |
| Frontend | React + Vite + TypeScript + MUI | Vite produces static assets served by Nginx — zero runtime overhead on the Pi |
| Encryption | `System.Security.Cryptography.AesGcm` + `Konscious.Security.Cryptography.Argon2id` | See recommendation below |

### Encryption Library Recommendation

**`System.Security.Cryptography.AesGcm` (built-in) + `Konscious.Security.Cryptography` (NuGet)**

- **AES-256-GCM**: Built into .NET since Core 3.0 via `AesGcm` class. Wraps platform crypto (OpenSSL on Linux/ARM64), so it leverages ARMv8 hardware AES instructions on the RPi 5. Zero external dependency.
- **Argon2id**: `Konscious.Security.Cryptography` is the most widely used .NET Argon2 implementation. Pure managed code, works on ARM64 without native compilation issues. Argon2id is the OWASP-recommended KDF.
- **Why not alternatives:**
  - `libsodium-net` / `NSec`: Good libraries but add unnecessary native dependencies. .NET's built-in `AesGcm` is sufficient and simpler.
  - `BCrypt` / `PBKDF2` for KDF: PBKDF2 is available built-in (`Rfc2898DeriveBytes`) but Argon2id is significantly more resistant to GPU/ASIC attacks due to its memory-hard design.

---

## Encryption Architecture

### Key Derivation: Two Outputs from One Master Password

The master password derives **two independent values** using different salts. The auth hash is stored in the DB, so it must never double as the encryption key.

```
Master Password
      |
      +--[Argon2id + salt_auth]--> auth_hash    (stored in DB for login verification)
      |
      +--[Argon2id + salt_enc]---> encryption_key (256-bit, NEVER stored anywhere)
```

- On account creation: generate 32 random bytes, split into `salt_auth` (first 16) and `salt_enc` (last 16)
- Argon2id params: `Iterations: 3, MemorySize: 65536 (64MB), DegreeOfParallelism: 4, OutputLength: 32` — tuned for RPi 5, ~0.5-1s per derivation

### Where the Encryption Key Lives

**In-memory server-side session store** (`ConcurrentDictionary<string, EncryptionSession>`).

- At login: derive key → store in dictionary keyed by a random `sessionId` (GUID)
- The JWT contains `sessionId` (NOT the key)
- On each request: middleware reads `sessionId` from JWT claims → looks up key in dictionary → sets `HttpContext.Items["EncryptionKey"]`
- On logout: zero-fill the byte array (`CryptographicOperations.ZeroMemory(key)`), remove from dictionary, invalidate refresh tokens
- On server restart: all keys purged — users must re-enter master password (this is a feature)
- Session timeout: 30 min inactivity, 4 hour absolute max
- Background `IHostedService` runs every 60s to purge expired sessions

### Encrypted Fields (AES-256-GCM)

Only **username** and **password** fields are encrypted. Title, URL, notes, and category are stored in plaintext for searchability and list display.

Each encrypted field gets its own random 12-byte nonce. Stored as a self-contained blob:

```
base64( 12-byte nonce || 16-byte authTag || ciphertext )
```

**Encrypt (C#):**
```csharp
byte[] nonce = RandomNumberGenerator.GetBytes(12);
byte[] tag = new byte[16];
byte[] ciphertext = new byte[plaintext.Length];
using var aes = new AesGcm(encryptionKey, tagSizeInBytes: 16);
aes.Encrypt(nonce, plaintextBytes, ciphertext, tag);
// Store: Convert.ToBase64String(nonce.Concat(tag).Concat(ciphertext))
```

**Decrypt (C#):**
```csharp
byte[] blob = Convert.FromBase64String(stored);
byte[] nonce = blob[..12];
byte[] tag = blob[12..28];
byte[] ciphertext = blob[28..];
byte[] plaintext = new byte[ciphertext.Length];
using var aes = new AesGcm(encryptionKey, tagSizeInBytes: 16);
aes.Decrypt(nonce, ciphertext, tag, plaintext);
```

---

## Database Schema (SQLite via EF Core)

```sql
CREATE TABLE Users (
    Id          INTEGER PRIMARY KEY AUTOINCREMENT,
    Username    TEXT NOT NULL UNIQUE,
    AuthHash    TEXT NOT NULL,
    Salt        TEXT NOT NULL,
    CreatedAt   TEXT NOT NULL DEFAULT (datetime('now')),
    UpdatedAt   TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE Devices (
    Id                 INTEGER PRIMARY KEY AUTOINCREMENT,
    UserId             INTEGER NOT NULL REFERENCES Users(Id),
    DeviceName         TEXT NOT NULL,
    DeviceFingerprint  TEXT NOT NULL UNIQUE,
    DeviceToken        TEXT NOT NULL UNIQUE,
    IsApproved         INTEGER NOT NULL DEFAULT 0,
    IsRevoked          INTEGER NOT NULL DEFAULT 0,
    LastUsedAt         TEXT,
    CreatedAt          TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE RefreshTokens (
    Id         INTEGER PRIMARY KEY AUTOINCREMENT,
    UserId     INTEGER NOT NULL REFERENCES Users(Id),
    DeviceId   INTEGER NOT NULL REFERENCES Devices(Id),
    TokenHash  TEXT NOT NULL UNIQUE,
    ExpiresAt  TEXT NOT NULL,
    CreatedAt  TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE PasswordEntries (
    Id            INTEGER PRIMARY KEY AUTOINCREMENT,
    UserId        INTEGER NOT NULL REFERENCES Users(Id),
    Title         TEXT NOT NULL,
    UsernameEnc   TEXT NOT NULL,
    PasswordEnc   TEXT NOT NULL,
    Url           TEXT,
    Notes         TEXT,
    Category      TEXT,
    CreatedAt     TEXT NOT NULL DEFAULT (datetime('now')),
    UpdatedAt     TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE AuditLog (
    Id         INTEGER PRIMARY KEY AUTOINCREMENT,
    UserId     INTEGER,
    DeviceId   INTEGER,
    Action     TEXT NOT NULL,
    IpAddress  TEXT NOT NULL,
    Details    TEXT,
    CreatedAt  TEXT NOT NULL DEFAULT (datetime('now'))
);
```

---

## API Endpoints

```
Public:
  GET    /api/health
  GET    /api/auth/status              -- { initialized: bool }
  POST   /api/auth/setup               -- one-time: create account + auto-approve first device
  POST   /api/auth/login               -- returns accessToken + refreshToken + deviceToken
  POST   /api/auth/refresh             -- rotate refresh token, issue new JWT

Authenticated + Device-Checked:
  POST   /api/auth/logout
  GET    /api/passwords                -- list all (title, url, category — no decryption needed)
  GET    /api/passwords/{id}           -- single entry with decrypted username + password
  POST   /api/passwords
  PUT    /api/passwords/{id}
  DELETE /api/passwords/{id}
  GET    /api/devices
  POST   /api/devices/{id}/approve
  POST   /api/devices/{id}/revoke
  DELETE /api/devices/{id}
  GET    /api/audit
```

Note: `GET /api/passwords` returns the list **without decrypting** username/password — only title, URL, category, and timestamps. Decryption happens only when fetching a single entry.

---

## Device Registration Flow

1. **First-time setup** (no users exist): POST `/api/auth/setup` creates the account and auto-approves the first device
2. **New device**: Login succeeds (correct password) but no device token → server creates a pending device, returns 202
3. **Approval**: From an already-approved device, user approves the pending device via the Devices page
4. **Identification**: Client generates a fingerprint from User-Agent + screen size + timezone + a random `localStorage` value, hashed to SHA-256
5. **Revocation**: Sets `IsRevoked = 1`, deletes refresh tokens, purges session keys — immediate lockout

---

## Auth Flow Summary

- **JWT access token**: 15-minute expiry, contains `{ userId, sessionId, deviceId }`, signed HS256
- **Refresh token**: 7-day expiry, rotated on each use, SHA-256 stored in DB
- **Device token**: Sent in `X-Device-Token` header, validated on every authenticated request
- **Session expired** (server restart): Refresh fails with `SESSION_EXPIRED` → frontend prompts re-entry of master password

---

## Project Structure

```
password-manager/
  PasswordManager.Api/
    Controllers/
      AuthController.cs
      PasswordsController.cs
      DevicesController.cs
      AuditController.cs
    Data/
      AppDbContext.cs
      Entities/
        User.cs
        Device.cs
        RefreshToken.cs
        PasswordEntry.cs
        AuditLogEntry.cs
    DTOs/
      Auth/
        LoginRequest.cs, LoginResponse.cs
        SetupRequest.cs, RefreshRequest.cs
      Passwords/
        PasswordListItem.cs, PasswordDetail.cs
        CreatePasswordRequest.cs, UpdatePasswordRequest.cs
      Devices/
        DeviceListItem.cs
    Middleware/
      DeviceCheckMiddleware.cs
    Services/
      ICryptoService.cs / CryptoService.cs
      ISessionStore.cs / SessionStore.cs
      IAuthService.cs / AuthService.cs
      IPasswordService.cs / PasswordService.cs
      IDeviceService.cs / DeviceService.cs
      IAuditService.cs / AuditService.cs
      SessionCleanupService.cs
    Program.cs
    appsettings.json
    appsettings.Development.json
  PasswordManager.Tests/
    Services/
      CryptoServiceTests.cs
      AuthServiceTests.cs
      PasswordServiceTests.cs
    Controllers/
      AuthControllerTests.cs
      PasswordsControllerTests.cs
  frontend/
    src/
      api/
      components/
      contexts/
      hooks/
      pages/
      App.tsx, main.tsx
    vite.config.ts
  deploy/
    nginx/
    systemd/
    setup.sh
  PasswordManager.sln
```

---

## Security Hardening

- **HTTPS**: Via Cloudflare Tunnel or Nginx with certs (not in Kestrel)
- **Rate limiting**: .NET 8 built-in `RateLimiter` middleware — 5 login attempts / 15 min per IP, 100 general / 15 min
- **CORS**: Specific origin only via `builder.Services.AddCors()`
- **Input validation**: FluentValidation or data annotations on DTOs
- **SQL injection**: Prevented by EF Core parameterized queries
- **Security headers**: Custom middleware or a response header library for CSP, X-Frame-Options, etc.
- **Audit logging**: All auth events and password access logged with IP and device

---

## Implementation Phases

### Phase 1: Backend Foundation
- Create .NET 8 Web API project + solution
- EF Core setup with SQLite, entity models, initial migration
- `CryptoService` — Argon2id KDF + AES-256-GCM encrypt/decrypt
- `SessionStore` — `ConcurrentDictionary` with expiry
- Unit tests for CryptoService (roundtrip, distinct keys, tamper detection)

### Phase 2: Authentication
- `AuthController` with setup, login, refresh, logout endpoints
- JWT configuration in `Program.cs` (`AddAuthentication` + `AddJwtBearer`)
- `DeviceCheckMiddleware`
- `SessionCleanupService` (IHostedService)
- Integration tests for full auth cycle

### Phase 3: Device Registration
- `DevicesController` with list, approve, revoke endpoints
- Device creation logic in `AuthService` (auto-approve on setup, pending otherwise)
- Tests

### Phase 4: Password CRUD
- `PasswordsController` with list (no decrypt), get (decrypt), create, update, delete
- `PasswordService` — encrypt on write, decrypt on read using session key
- DTO validation
- Tests

### Phase 5: Frontend
- Vite + React + TypeScript + MUI setup
- Axios client with JWT interceptor + auto-refresh
- Login page (device-registration aware)
- Password list (plaintext titles) / detail view (decrypted credentials)
- Password create/edit forms with password generator
- Device management page
- Auth context with session-expired modal

### Phase 6: Security & Polish
- Rate limiting, CORS, security headers
- Audit logging middleware
- Error handling middleware (no stack traces in production)

### Phase 7: RPi Deployment
- `dotnet publish -r linux-arm64 --self-contained` for RPi
- systemd service with `ProtectSystem=strict`, `ReadWritePaths` for DB dir
- Nginx config (static frontend + reverse proxy to Kestrel)
- Cloudflare Tunnel ingress entry
- `setup.sh` deployment script
- Backup cron (daily copy of vault.db, 30-day retention)

---

## Verification

1. **CryptoService unit tests**: encrypt→decrypt roundtrip, verify distinct auth/encryption keys from same password, verify tamper detection (flip a ciphertext bit → `CryptographicException` thrown)
2. **Auth integration tests**: full login→access→refresh→logout cycle, verify session purge on logout, verify 401 after simulated restart (clear session store)
3. **Device tests**: register device→pending→approve→access works, revoke→immediate 401
4. **E2E on RPi**: deploy, create account from browser, add passwords, verify they survive a service restart (data persists but session expires), login from second device and test approval flow

---

## Key NuGet Packages

```
Microsoft.EntityFrameworkCore.Sqlite
Microsoft.AspNetCore.Authentication.JwtBearer
Konscious.Security.Cryptography.Argon2
FluentValidation.AspNetCore (or use data annotations)
xunit + Moq (testing)
```

## Key npm Packages (Frontend)

```
react, react-dom, react-router-dom
@mui/material, @mui/icons-material, @emotion/react, @emotion/styled
axios
vite, @vitejs/plugin-react, typescript
vitest, @testing-library/react
```
