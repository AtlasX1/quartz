# API Security & Authentication: Building Secure Distributed Systems

## Introduction & Purpose

API security is fundamentally about answering: **"Who are you?" (Authentication) and "What are you allowed to do?" (Authorization)**

These are architectural concerns affecting every layer of an application. Poor security choices at the API level don't just expose data—they undermine the entire system. Understanding security requires understanding **threat models, cryptographic principles, and how distributed systems exchange trust**.

This document explores the architectural foundations of API security: authentication protocols, authorization models, input validation, and the infrastructure that makes secure systems feasible at scale.

## Core Concepts & Internal Architecture

### Authentication Fundamentals

**Authentication** answers: "Who are you?"

#### Password-Based Authentication

**Flow:**
```
User submits: username + password
    ↓
Server hashes password, compares to stored hash
    ↓
If match, user authenticated
    ↓
Server creates session (or token)
```

**Architectural decisions:**

1. **Password hashing:** Never store plain text. Use bcrypt, argon2, or PBKDF2
   - Hashing is one-way (can't reverse)
   - Salting prevents rainbow tables
   - Intentionally slow (milliseconds) to prevent brute force

2. **Session vs. Token:**
   - **Session:** Server maintains state in database; client receives session ID
   - **Token:** Server encodes state in token; client stores and sends token

**Session-based architecture:**
```
Client submits credentials
    ↓
Server creates session entry in database
    ↓
Server sends Set-Cookie: sessionId
    ↓
Client sends cookie with each request
    ↓
Server validates session exists
```

**Pros:** Simple, server controls revocation
**Cons:** Not stateless; doesn't scale across servers; requires shared session store

**Token-based architecture:**
```
Client submits credentials
    ↓
Server generates JWT token (signed)
    ↓
Server sends token to client
    ↓
Client sends token with each request (Authorization header)
    ↓
Server validates token signature
```

**Pros:** Stateless, scales across servers, works with SPAs and APIs
**Cons:** Can't immediately revoke tokens; must use token blacklist or short expiry

### JWT (JSON Web Tokens)

JWT is a **standard for encoding claims** in a cryptographically signed token:

```
JWT = Header.Payload.Signature
```

**Header:**
```json
{
    "alg": "HS256",  // Algorithm (HMAC + SHA256)
    "typ": "JWT"
}
```

**Payload (claims):**
```json
{
    "sub": "1234567890",   // Subject (user ID)
    "name": "John Doe",
    "iat": 1516239022,     // Issued at (timestamp)
    "exp": 1516242622      // Expiration (timestamp)
}
```

**Signature:**
```
HMACSHA256(
    base64(Header) + "." + base64(Payload),
    secret_key
)
```

**Verification process:**
```
Server receives JWT
    ↓
Extract header and payload
    ↓
Recalculate signature using secret key
    ↓
Compare received signature with calculated
    ↓
If match, payload is authentic (not tampered)
```

**Architectural consequence:** JWT enables **stateless authentication**. Server doesn't store anything; the token itself contains all needed information, cryptographically signed.

### OAuth2 and Authorization Code Flow

OAuth2 enables **delegated authentication** — users log in via third-party provider (Google, GitHub).

**Flow:**

```
1. User clicks "Login with Google"
    ↓
2. Redirects to: https://accounts.google.com/auth?client_id=xxx&redirect_uri=https://myapp.com/callback
    ↓
3. User logs in to Google (if not already)
    ↓
4. Google redirects to: https://myapp.com/callback?code=auth_code_xyz
    ↓
5. Server exchanges code for access token (backend to backend)
    ↓
6. Server uses token to fetch user info from Google
    ↓
7. Server creates session/token for user
    ↓
8. User logged into my app
```

**Architectural layers:**

```
Client App (frontend)
    ↓
Identity Provider (Google, GitHub, Auth0)
    ↓
Application Backend (validates tokens)
```

**OAuth2 grants:**

1. **Authorization Code** — Users login via browser (most secure)
2. **Implicit** — Deprecated (too insecure)
3. **Password** — Username/password directly to app (less secure)
4. **Client Credentials** — Service-to-service (no user involved)

**Architectural consequence:** OAuth2 delegates authentication to specialized providers, reducing burden on application.

### OpenID Connect (OIDC)

OIDC is a **layer on top of OAuth2** that adds identity information:

OAuth2 provides **access tokens** (for authorization). OIDC adds **ID tokens** (for authentication):

```
ID Token (JWT):
{
    "iss": "https://accounts.google.com",
    "sub": "1234567890",
    "email": "user@example.com",
    "email_verified": true,
    "aud": "my-app-client-id"
}
```

**Architectural consequence:** OIDC is the standard for authentication across applications; OAuth2 for authorization.

### Authorization Models

**Authorization** answers: "What are you allowed to do?"

#### Role-Based Access Control (RBAC)

Users have roles; roles have permissions:

```
User "Alice"
    ↓
Roles: ["admin", "moderator"]
    ↓
admin role has permissions: ["read:users", "write:users", "delete:users"]
moderator role has permissions: ["read:users", "read:posts"]
    ↓
When Alice tries action: check if her roles have permission
```

**Implementation:**

```sql
CREATE TABLE roles (
    id INT PRIMARY KEY,
    name VARCHAR(50)  -- "admin", "user", "guest"
)

CREATE TABLE permissions (
    id INT PRIMARY KEY,
    action VARCHAR(50)  -- "read:posts", "write:posts"
)

CREATE TABLE role_permissions (
    role_id INT,
    permission_id INT,
    PRIMARY KEY (role_id, permission_id)
)

CREATE TABLE user_roles (
    user_id INT,
    role_id INT,
    PRIMARY KEY (user_id, role_id)
)
```

**Architectural simplicity:** Easy to understand; scales to thousands of users.

**Limitation:** Coarse-grained. Can't express "user can read their own posts but not others'."

#### Attribute-Based Access Control (ABAC)

Access decisions based on attributes (user attributes, resource attributes, environment):

```
Request: Alice attempts to read post #123
Attributes:
    User: { id: "alice", role: "user", department: "sales", created_at: "2020-01-01" }
    Resource: { id: 123, author_id: "alice", status: "draft", classification: "internal" }
    Environment: { time: "09:00", ip: "192.168.1.1", is_vpn: false }

Policy:
    IF user.id == resource.author_id THEN allow
    IF user.role == "admin" THEN allow
    IF resource.status == "public" AND user.role != "banned" THEN allow
    ELSE deny
```

**Architectural complexity:** Very expressive but computationally complex to evaluate.

**Use case:** Fine-grained access control for sensitive data.

#### Scopes

Scopes limit token capabilities:

```
access_token = "eyJhbGc..." with scopes: ["read:profile", "write:posts"]

User requests with this token:
    GET /profile → Allowed (read:profile scope)
    POST /posts → Allowed (write:posts scope)
    DELETE /posts/123 → Denied (no delete:posts scope)
```

**Architectural consequence:** Tokens can be **least-privilege** — each token has minimum permissions needed.

### Input Validation and Sanitization

APIs must validate inputs to prevent injection attacks:

#### SQL Injection

**Vulnerable:**
```sql
SELECT * FROM users WHERE email = '" + req.body.email + "'"
// If email = "'; DROP TABLE users; --"
// Becomes: SELECT * FROM users WHERE email = ''; DROP TABLE users; --'
// Database executes DROP TABLE!
```

**Secure (parameterized queries):**
```sql
const result = await db.query(
    "SELECT * FROM users WHERE email = ?",
    [req.body.email]  // Parameter is escaped by driver
)
```

#### XSS (Cross-Site Scripting)

**Vulnerable:**
```html
<div>Welcome, <%= user.name %></div>
<!-- If name = "<script>alert('xss')</script>"
     Then JavaScript runs in browser -->
```

**Secure (escaping/encoding):**
```html
<div>Welcome, <%= escapeHTML(user.name) %></div>
<!-- Special characters encoded; script tags rendered as text -->
```

#### Command Injection

**Vulnerable:**
```javascript
exec(`ffmpeg -i ${req.body.filename} output.mp4`)
// If filename = "input.mp4 && rm -rf /"
// Both ffmpeg AND rm execute
```

**Secure:**
```javascript
execFile('ffmpeg', ['-i', req.body.filename, 'output.mp4'])
// Arguments passed separately; shell doesn't interpret
```

**Architectural consequence:** Every input is **untrusted by default**. Validation and encoding must happen at API boundary.

### HTTPS and Transport Security

HTTPS (HTTP over TLS) encrypts data in transit:

```
Client ←→ Server
Unencrypted (anyone on network can see)

Client ←TLS/SSL→ Server
Encrypted (only client and server can decrypt)
```

**Architectural layers:**

```
Application Layer (HTTP request/response)
    ↓
Presentation Layer (TLS encryption)
    ↓
Transport Layer (TCP)
    ↓
Network Layer (IP)
```

**TLS handshake:**
```
Client → Server: ClientHello (supported versions, cipher suites)
Server → Client: ServerHello (chosen version, cipher suite, certificate)
Client → Server: KeyExchange, ChangeCipherSpec
Server → Client: ChangeCipherSpec
    ↓
Connection encrypted; can exchange data
```

**Certificate verification:**
```
Server presents certificate signed by Certificate Authority
    ↓
Client has CA's public key (built into browser/OS)
    ↓
Client verifies certificate is signed by trusted CA
    ↓
Client trusts server's identity
```

**Architectural consequence:** HTTPS is **not just privacy**; it's identity verification. APIs should enforce HTTPS only (HTTP should redirect).

### Secrets Management

APIs need secrets (database passwords, API keys, JWT signing keys):

**Anti-pattern:**
```javascript
const dbPassword = "super_secret_123"  // In source code
const apiKey = "abc123xyz"  // Hardcoded
```

**Better (environment variables):**
```bash
# .env file (NOT in git)
DB_PASSWORD=super_secret_123
API_KEY=abc123xyz

# In code
const dbPassword = process.env.DB_PASSWORD
```

**Best (secrets manager):**
```javascript
// AWS Secrets Manager
const secretsClient = new SecretsManager()
const secret = await secretsClient.getSecretValue('db-password')
const dbPassword = secret.SecretString
```

**Architectural layers:**

```
Application Code (doesn't know secrets)
    ↓
Secrets Manager (stores encrypted secrets)
    ↓
Infrastructure (keys rotated automatically)
```

**Architectural consequence:** Secrets are **centralized, encrypted, audited, and rotated** automatically.

## Common Problems & Failure Scenarios

### Problem 1: Token Expiration and Revocation

**Scenario:** User logs out, but JWT token is still valid until expiration.

```
User logs out → Client deletes token
    ↓
Client loses token (can't use anymore)
    ↓
But server doesn't know user logged out
    ↓
If someone steals the token, they can use it until expiration
```

**Root cause:** JWT doesn't require server validation; token itself is authority.

**Solutions:**

1. **Short expiration times** (minutes) + refresh tokens
2. **Token blacklist** — maintain list of revoked tokens (defeats stateless benefit)
3. **Logout endpoint** — invalidate session (requires state)

### Problem 2: JWT Secret Compromise

**Scenario:** JWT secret (signing key) is exposed in production:

```
Attacker has secret
    ↓
Attacker can forge arbitrary JWT tokens
    ↓
Attacker impersonates any user
```

**Root cause:** Secrets not properly secured.

**Mitigation:**
```
1. Never commit secrets to git
2. Use secrets manager (AWS Secrets Manager, Vault)
3. Rotate secrets regularly
4. Audit who has access
5. Log all secret access
```

### Problem 3: Authorization Bypass

**Scenario:**

```
URL: GET /api/users/123/profile
Authorization check: Is user authenticated?
    ↓ Yes, allow

Attacker changes to: GET /api/users/456/profile
Same check passes (still authenticated)
    ↓
Attacker can read any user's profile
```

**Root cause:** Authentication ≠ Authorization. Authenticating that user exists doesn't mean they should access that resource.

**Solution:** Explicit authorization checks:

```javascript
app.get('/api/users/:id/profile', async (req, res) => {
    // 1. Authentication (verified above)
    // 2. Authorization (explicit check)
    if (req.user.id !== parseInt(req.params.id) && req.user.role !== 'admin') {
        return res.status(403).json({ error: 'Not authorized' })
    }
    // 3. Return resource
})
```

### Problem 4: CORS Misconfiguration

**Scenario:**

```
API allows: Access-Control-Allow-Origin: *
    ↓
Attacker website can call API from user's browser
    ↓
User's browser automatically includes cookies
    ↓
Attacker steals user's data
```

**Root cause:** Overly permissive CORS headers.

**Solution:** Explicit allowed origins:

```javascript
app.use(cors({
    origin: ['https://example.com', 'https://app.example.com'],
    credentials: true  // Allow cookies only for allowed origins
}))
```

## Design Decisions & Trade-offs

### Trade-off 1: Simplicity vs. Security

**Simple (less secure):**
```javascript
const token = user.id + ":" + Date.now()
// Anyone can forge token; no crypto
```

**Secure (more complex):**
```javascript
const token = jwt.sign({ sub: user.id }, secret, { expiresIn: '1h' })
// Cryptographically signed; expires; verifiable
```

**Trade-off:** Security requires complexity (crypto libraries, secret management, auditing).

### Trade-off 2: Stateless (JWT) vs. Stateful (Sessions)

**JWT (stateless):**
- Pros: Scalable, no server state, works across servers
- Cons: Can't immediately revoke, larger tokens, vulnerable to secret compromise

**Sessions (stateful):**
- Pros: Simple, can revoke immediately, smaller cookies
- Cons: Not scalable, requires shared store (Redis), state to maintain

**Trade-off:** Most modern systems choose JWT for APIs; sessions for traditional web apps.

### Trade-off 3: Strict Authorization vs. User Experience

**Very strict:**
```javascript
// Every action requires explicit permission check
// Slow, complex, but very secure
```

**User-friendly:**
```javascript
// Assume users can do most things
// Fast, simple, but potential for bugs
```

**Trade-off:** Security should not compromise usability; need balance.

## When to Use Different Approaches

### Simple Internal APIs

Use: Basic API key + IP whitelisting

```javascript
if (req.headers['x-api-key'] !== process.env.API_KEY) {
    return res.status(401).json({ error: 'Unauthorized' })
}
```

### Single Organization (SPA + Backend)

Use: Session-based or JWT with same-origin requests

```javascript
// Session-based
app.post('/login', (req, res) => {
    // Verify credentials
    req.session.userId = user.id
})

// Or JWT
app.post('/login', (req, res) => {
    const token = jwt.sign({ sub: user.id }, secret)
    res.json({ token })
})
```

### Multiple Organizations (SaaS)

Use: OAuth2/OIDC with role-based access control

```javascript
// Users log in with their identity providers
// App enforces RBAC on resources
```

### Public APIs with Mobile Clients

Use: OAuth2 authorization code flow + JWT access tokens

```javascript
// Minimize data exposure; use scopes
// Short-lived tokens + refresh tokens
```

### Microservices Internal Communication

Use: mTLS (mutual TLS) + service-to-service authentication

```javascript
// Each service has certificate
// Services authenticate each other via certs
```

---

## Conclusion

API security is **layered**: authentication establishes identity, authorization enforces permissions, and transport security protects data in transit.

Key principles:

1. **Never trust user input** — validate and sanitize everything
2. **Defense in depth** — multiple security layers
3. **Least privilege** — users/tokens get minimum permissions needed
4. **Secrets management** — centralized, encrypted, rotated
5. **Audit trails** — log all sensitive operations
6. **HTTPS everywhere** — encrypted communication
7. **Strong authentication** — passwords hashed, tokens signed
8. **Explicit authorization** — don't assume authenticated users should access data

Security is not a feature; it's a fundamental architectural requirement affecting design decisions across the entire system.
