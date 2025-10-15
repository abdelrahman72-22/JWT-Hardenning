# JWT Hardening

## Overview
This project hardens a vulnerable Node.js + Express + SQLite JWT authentication lab to follow secure authentication practices. Starting from the provided vulnerable demo, it moves configuration to a `.env` file, enforces `iss`/`aud` claims and short token lifetimes, removes hard-coded secrets, implements a refresh strategy with rotation and server-side storage, and demonstrates attacks (e.g., weak secret forgery, `alg:none` trick) that succeed on the vulnerable server but fail on the hardened one. Postman is used for requests, and Wireshark for traffic inspection.

The existing frontend is reused with minimal edits (e.g., `credentials: 'include'` for cookies). All requirements are met, plus all bonuses (A-E):
- **A:** Local HTTPS with self-signed certs.
- **B:** Helmet for secure headers (e.g., CSP, HSTS).
- **C:** Rate limiting on login (5 attempts/15min/IP).
- **D:** Persistent refresh tokens in SQLite.
- **E:** Logging for failures + `scan-logs.js` for detection.

**Token Lifetimes:** Access: 15 minutes. Refresh: 7 days (rotated on use).
##  Create self-signed cert (OpenSSL)

```bash
mkdir certs
cd certs
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr    # Common Name: http://localhost:1235
openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
```

Place server.key and server.crt in certs/.

---

## Trust certificate in Windows

Run mmc.exe → Add Snap-in → Certificates → Computer account → Trusted Root Certification Authorities → Import certs/server.crt.

---

## Setup Instructions
1. Clone or download the provided project to your local machine.
2. Ensure Node.js (v18+) and npm are installed.
3. Install dependencies:
   ```
   npm install
   ```
4. Initialize the sample database (if included):
   ```
   npm run init-db
   ```
   This creates `users.db` with sample users: alice/alicepass (role: user), admin/adminpass (role: admin).
5. Copy `.env.example` to `.env` and populate with secure values (see Environment Configuration below).
   
6. Start the vulnerable server (follow repo README; example):
   ```
   npm start
   ```
   Confirm it runs at http://localhost:1234/ and demo endpoints exist (adjust ports if your repo differs), e.g.:
   - POST /login (vulnerable or demo login)
   - POST /vuln-login (vulnerable demo)
   - POST /login and POST /refresh (secure flows if present)

## Environment Configuration (.env) (Required)
- Move all secrets and configuration out of source files into a `.env` file.
- Document in README how to copy/use `.env` and set values.

Copy `.env.example` to `.env`:
```
cp .env.example .env
```
Replace placeholders:
- `ACCESS_SECRET`: Strong random string for access tokens (e.g., 64-char hex).
- `REFRESH_SECRET`: Different strong random string for refresh tokens.
- `VULN_SECRET`: Weak secret for vulnerable server (DO NOT use in production).
- `JWT_ISSUER`: e.g., `jwt-lab`
- `JWT_AUDIENCE`: e.g., `web-app`

**Secret Generation:** Secrets generated using Node.js crypto (secure random bytes):
```
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```
Example: `14f11c5f93370d13d5ff747ee30c9da7a98d35cff2f82a4995fc07ad26efbc35`. Never commit `.env` to Git.

## Remove Hard-Coded / Weak Secrets (Required)
- Replace any hard-coded weak secret (e.g., `weak-secret`) with `process.env` values from `.env`.
- Explain how you generated the secrets (e.g., `crypto.randomBytes(...)` or a secure generator) in README.

All secrets (ACCESS_SECRET, REFRESH_SECRET, VULN_SECRET) are now loaded from `.env` via `dotenv`. No hard-coded values remain in source code. Generation method detailed above.

## Enforce Token Claims and Verification (Required)
- Issue tokens with `iss` (issuer) and `aud` (audience) claims when signing.
- Verify `iss` and `aud` (and `alg`) when accepting tokens.
- Shorten access token lifetime (e.g., 10–15 minutes) and document the chosen lifetimes.

Tokens are signed with `iss: JWT_ISSUER`, `aud: JWT_AUDIENCE`, `alg: 'HS256'`. Verification in middleware enforces these + `algorithms: ['HS256']`. Access: 15 minutes (`expiresIn: '15m'`); documented above.

## Implement a Refresh Strategy (Required)
- Provide a separate refresh mechanism (different secret or rotation). Describe how refresh tokens are handled (rotation or server-side store).

Uses separate `REFRESH_SECRET`. Tokens stored in SQLite (`refresh_tokens` table) with `token_id`, `username`, timestamps. On refresh: Verify JWT + DB match, delete old, issue new (rotation). Expires in 7 days; supports revocation (delete from DB).

## Keep/Reuse the Existing Frontend (Required)
- Do not replace the supplied front-end. You may make small edits to client fetch options (for example, `credentials: 'include'` if you use cookies), but do not rebuild the UI.

Frontend (`public/index.html`) unchanged. Minor edit: Add `credentials: 'include'` to fetches for HttpOnly refresh cookies.

## Running the Project
### Vulnerable Server (HTTP, Port 1234)
```
node vuln-server.js  # Assumes original vuln-server.js
```
- URL: http://localhost:1234

### Hardened Server (HTTPS, Port 1235)
```
node secure-server.js
```
- URL: https://localhost:1235 (accept self-signed cert)
- Logs: `node secure-server.js > server.log 2>&1` for scanning.

## Demonstrate Attacks & Protections (Required)
- Reproduce at least two attack scenarios locally on the vulnerable server (examples below). Use Postman to craft/replay requests for both the vulnerable and hardened server.
- Show the same attack(s) against your hardened server and confirm they fail (server returns 401/403 or rejects token).

Postman collections: `postman/jwt-vuln.json` (vulnerable), `postman/jwt-secure.json` (hardened). Or use `forge_attack.js`.

### Vulnerable Demo (Attacks Succeed)
1. Start: `node vuln-server.js`.
2. Login: POST http://localhost:1234/login `{ "username": "admin", "password": "adminpass" }` → Get token.
3. **Attack 1: Forge with Known/Weak Secret**
   - Decode at jwt.io, re-sign with `VULN_SECRET` (HS256).
   - Or: `node forge_attack.js` (targets 1234).
   - GET http://localhost:1234/admin `Authorization: Bearer <forged>` → 200 (success).
4. **Attack 2: alg:none Header Trick**
   - Craft: `{"alg":"none","typ":"JWT"}.<payload>.` (empty sig).
   - GET /admin with Bearer → 200 (accepts unsigned).
5. **Attack 3: Simulate Token Theft**
   - Copy token from login (Network tab).
   - Replay GET /admin → 200 (no checks).

### Hardened Demo (Attacks Fail)
1. Start: `node secure-server.js`.
2. Login: POST https://localhost:1235/login (same body) → Token + cookie.
3. Valid Refresh: POST /refresh (include cookie) → New tokens.
4. **Attack 1:** Same forge → GET /admin: 401 (secret/claims mismatch).
5. **Attack 2:** alg:none → 401 (alg enforced).
6. **Attack 3:** Replay expired/stolen → 401 (lifetime/DB rotation).

**Bonus C Test:** 6+ logins/IP/15min → 429.

## Record Traffic and Show What’s Visible (Required)
- Use Wireshark to capture local traffic during your demos. Show:
  - On HTTP: where the token is visible in requests (headers/body).
  - If you complete the HTTPS bonus, show how traffic is encrypted (TLS) and token payloads are not visible in cleartext.
- Include a short note in README describing how you captured (interface or loopback), and suggested filters used (e.g., http or tcp.port == 1234).

**Capture How:** Interface: Loopback (lo0 macOS/Linux; Loopback Adapter Windows). Start capture, perform requests (login + attack), stop, filter/export screenshots.

**Filters:**
- HTTP (Vuln): `http or tcp.port == 1234`
- HTTPS (Hardened): `tcp.port == 1235 or tls`

**Visibility:**
- HTTP: Token in cleartext Authorization header (follow stream → decode payload/user/role).
- HTTPS (Bonus A): TLS-encrypted; no cleartext (follow stream → obfuscated bytes).

**Assumptions & Limitations:** Assumes original `vuln-server.js`; single-server (no distributed sessions); tested Node 20/macOS. No production use—strengthen keys further.
