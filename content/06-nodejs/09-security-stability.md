# Security & Stability: Threat Landscape and Defensive Architecture

## Conceptual Overview

Security in Node.js applications spans multiple dimensions: input validation, authentication, authorization, data protection, dependency management, and operational resilience. Security failures are often silent—credentials leaking, SQL injection succeeding, authentication bypassed—until a breach occurs. Understanding the threat landscape (OWASP Top 10), common vulnerabilities, and defensive architectural patterns is essential for building systems that resist attack and degrade gracefully under failure.

The challenge is balancing security rigor with development velocity. Over-engineering security slows feature delivery; under-engineering creates vulnerabilities that lead to breaches, compliance violations, and reputation damage.

---

## Internal Mechanics: Authentication and Authorization

### Session vs. Token-Based Authentication

**Session-based**:
```
1. User sends credentials to server
2. Server validates, creates session: { sessionId: "abc123", userId: 42 }
3. Server stores session in memory/cache: sessions = { "abc123": { userId: 42 } }
4. Server sends sessionId as cookie to client
5. Client includes cookie in each request
6. Server looks up session from sessionId
```

**Characteristics**:
- Stateful: Server maintains session store
- Cookies are automatically included by browser (CSRF risk if not protected)
- Session state can be shared across servers if stored in central cache (Redis)

**Token-based (JWT)**:
```
1. User sends credentials to server
2. Server validates, creates token: JWT { userId: 42, expiresAt: <future>, signature: <hmac> }
3. Server sends token to client (typically in Authorization header)
4. Client sends token in header with each request
5. Server verifies token signature (no server-side lookup needed)
```

**Characteristics**:
- Stateless: No session store required
- Server validates token cryptographically (no central state needed)
- Token expiration is embedded in token (client-enforced expiration)
- Revocation is complex (invalidating a token requires server-side state)

**Trade-off**: Sessions are simpler but stateful (harder to scale); tokens are stateless but revocation is complex.

### Authorization: Role-Based vs. Attribute-Based

**Role-Based Access Control (RBAC)**:
```javascript
const roles = {
  'admin': { permissions: ['read', 'write', 'delete'] },
  'editor': { permissions: ['read', 'write'] },
  'viewer': { permissions: ['read'] }
};

// User has role 'editor'
if (roles[user.role].permissions.includes('write')) {
  // Allowed
}
```

**Attribute-Based Access Control (ABAC)**:
```javascript
// Policy: Can edit posts if (author === user) OR (user.role === 'admin')
if (post.author === user.id || user.role === 'admin') {
  // Allowed
}
```

**RBAC is simpler; ABAC is more flexible.**

---

## Threat Landscape: OWASP Top 10 for Node.js

### 1. Injection (SQL, NoSQL, Command)

**The problem**: Unsanitized user input is concatenated into queries/commands.

```javascript
// Vulnerable
const query = `SELECT * FROM users WHERE email = '${userEmail}'`;
// If userEmail = "' OR '1'='1", query becomes:
// SELECT * FROM users WHERE email = '' OR '1'='1'  (returns all users)
```

**Solution**: Use parameterized queries or prepared statements.

```javascript
// Safe: Parameterized query
const query = 'SELECT * FROM users WHERE email = $1';
const result = await db.query(query, [userEmail]);
```

### 2. Broken Authentication

**The problem**: Weak password policies, credential exposure, session fixation.

```javascript
// Vulnerable: Weak password, no hashing
const hashedPassword = userPassword;  // Plaintext!
await user.update({ hashedPassword });

// Vulnerable: Session fixation
req.sessionID = userProvidedSessionID;  // Allows attacker to control session
```

**Solution**: Hash passwords with bcrypt; regenerate session ID after login.

```javascript
const bcrypt = require('bcrypt');
const hashedPassword = await bcrypt.hash(userPassword, 10);  // Hash with salt rounds

// Regenerate session after login
req.session.regenerate(() => {
  req.session.userId = user.id;
});
```

### 3. Sensitive Data Exposure

**The problem**: Encrypting/hashing incorrectly, logging sensitive data, transmitting over HTTP.

```javascript
// Vulnerable: Logging passwords
console.log(`Login attempt: ${email}:${password}`);  // Password in logs!

// Vulnerable: Storing plaintext credit card
user.creditCard = req.body.creditCard;  // Should never be stored

// Vulnerable: HTTP transmission
server.listen(3000);  // Should be HTTPS
```

**Solution**: Don't store sensitive data; encrypt if required; use HTTPS.

```javascript
// Never log sensitive data
console.log(`Login attempt: ${email}`);

// Hash credit card or use external payment processor
// Use HTTPS
spdy.createSecureServer(options, app).listen(3000);
```

### 4. XML External Entity (XXE)

**Less common in Node.js (JSON is default) but relevant for XML parsing.**

```javascript
// Vulnerable: Parses XML without disabling external entities
const xml = require('xml2js');
const parser = new xml.Parser();  // Default: external entities enabled
parser.parseString(userXml, callback);

// Attacker can reference external files:
// <!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
// <foo>&xxe;</foo>
```

### 5. Broken Access Control

**The problem**: Authorization checks are missing or bypassable.

```javascript
// Vulnerable: No authorization check
app.delete('/users/:id', async (req, res) => {
  await User.destroy({ where: { id: req.params.id } });
  res.send('User deleted');
});
// Any authenticated user can delete any user

// Better: Check ownership
app.delete('/users/:id', async (req, res) => {
  if (req.user.id !== parseInt(req.params.id) && req.user.role !== 'admin') {
    return res.status(403).send('Forbidden');
  }
  await User.destroy({ where: { id: req.params.id } });
  res.send('User deleted');
});
```

### 6. Security Misconfiguration

**The problem**: Default credentials, unneeded features enabled, outdated dependencies.

```javascript
// Vulnerable: Default MongoDB credentials
mongoose.connect('mongodb://localhost:27017/myapp');  // No auth; if exposed, attackable

// Better: Use credentials
mongoose.connect('mongodb://${process.env.MONGO_USER}:${process.env.MONGO_PASSWORD}@....');
```

### 7. Cross-Site Scripting (XSS)

**The problem**: User input rendered in HTML without escaping.

```html
<!-- Vulnerable: User input in template -->
<h1><%= user.comment %></h1>
<!-- If comment = "<script>alert('hacked')</script>", script executes -->

<!-- Better: Escape output -->
<h1><%= escapeHtml(user.comment) %></h1>
```

### 8. Insecure Deserialization

**The problem**: Deserializing untrusted data (pickle, Node.js `eval`).

```javascript
// Vulnerable: Deserializing arbitrary data
const user = eval(jsonString);  // Code execution!

// Better: Use safe JSON parsing
const user = JSON.parse(jsonString);
```

### 9. Using Components with Known Vulnerabilities

**The problem**: Dependencies with disclosed vulnerabilities not updated.

```bash
npm audit --production  # Identifies vulnerable transitive dependencies
npm audit fix           # Auto-fixes where possible
```

### 10. Insufficient Logging & Monitoring

**The problem**: Security events not logged; breaches not detected.

```javascript
// Vulnerable: No security logging
app.post('/login', (req, res) => {
  // User login succeeds/fails but no record
});

// Better: Log security events
app.post('/login', (req, res) => {
  if (user) {
    securityLogger.info(`Login successful: ${user.id}`);
  } else {
    securityLogger.warn(`Login failed: ${email}`);
  }
});
```

---

## Problems & Challenges

### 1. Dependency Vulnerability Management

**The problem**: Transitive dependencies introduce vulnerabilities. Updating can break compatibility.

### 2. Credential Management

**The problem**: Hardcoding credentials in code or config files exposes them in version control.

```javascript
// Vulnerable: Credentials in code
const db = connect('postgresql://user:password@localhost/db');
```

### 3. Token Revocation Complexity

**The problem**: JWT tokens can't be revoked without server-side state.

### 4. CORS Misconfiguration

**The problem**: Overly permissive CORS allows cross-origin requests from any domain.

```javascript
// Vulnerable: Allows requests from any origin
app.use(cors());  // Equivalent to: Access-Control-Allow-Origin: *
```

### 5. Prototype Pollution

**The problem**: Merging user objects pollutes Object.prototype.

```javascript
// Vulnerable: Uncontrolled merge
const defaults = { role: 'user' };
const user = Object.assign(defaults, userInput);
// If userInput = { "__proto__": { role: "admin" } }, all objects get role: admin
```

### 6. Child Process Injection

**The problem**: Spawning processes with user-controlled arguments.

```javascript
// Vulnerable: Shell injection
const { execSync } = require('child_process');
const filename = req.query.file;
const result = execSync(`cat ${filename}`);  // If filename = "file.txt; rm -rf /", disaster
```

---

## Solutions & Architectural Approaches

### 1. Input Validation: Schema-Based Validation

```javascript
const { body, validationResult } = require('express-validator');

app.post('/users', [
  body('email').isEmail().normalizeEmail(),
  body('password').isLength({ min: 8 }),
  body('age').isInt({ min: 18 })
], (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({ errors: errors.array() });
  }
  // All inputs validated
});

// Or use Zod for TypeScript:
const userSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  age: z.number().min(18)
});

const user = userSchema.parse(req.body);  // Throws if invalid
```

### 2. Password Hashing and Verification

```javascript
const bcrypt = require('bcrypt');

// Hashing
const hashedPassword = await bcrypt.hash(userPassword, 10);

// Verification
const isValid = await bcrypt.compare(userPassword, hashedPassword);
```

### 3. Content Security Policy (CSP)

```javascript
const helmet = require('helmet');

app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'", "trusted-cdn.com"],
    styleSrc: ["'self'", "'unsafe-inline'"]  // unsafe-inline allows inline styles
  }
}));
```

### 4. Rate Limiting

```javascript
const rateLimit = require('express-rate-limit');

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 5  // 5 attempts
});

app.post('/login', loginLimiter, (req, res) => {
  // Protected endpoint
});
```

### 5. CORS Configuration

```javascript
// Restrictive CORS
app.use(cors({
  origin: ['https://app.example.com', 'https://admin.example.com'],
  credentials: true
}));
```

### 6. Environment Variables for Secrets

```bash
# .env (not in version control)
DATABASE_URL=postgresql://user:password@localhost/db
JWT_SECRET=your-secret-key
```

```javascript
require('dotenv').config();
const dbUrl = process.env.DATABASE_URL;
```

### 7. Helmet: Security Headers

```javascript
const helmet = require('helmet');

app.use(helmet());
// Sets security headers:
// - X-Content-Type-Options: nosniff
// - X-Frame-Options: DENY
// - Strict-Transport-Security: max-age=...
```

### 8. Graceful Error Handling

```javascript
// Vulnerable: Leaks stack trace
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.stack });  // Stack trace exposed!
});

// Better: Generic error message in production
app.use((err, req, res, next) => {
  logger.error(err);  // Log for debugging
  res.status(500).json({ error: 'Internal server error' });  // Generic to client
});
```

---

## Trade-offs & Limitations

### Session vs. Token

| Aspect | Session | Token |
|--------|---------|-------|
| **Scalability** | Requires shared storage | Fully distributed |
| **Revocation** | Instant | Requires blacklist |
| **CSRF protection** | Built-in | Requires CSRF tokens |
| **Complexity** | Simpler | More setup |

### RBAC vs. ABAC

| Aspect | RBAC | ABAC |
|--------|------|------|
| **Simplicity** | Simple | Complex |
| **Flexibility** | Limited | Highly flexible |
| **Performance** | Fast | Slower (policy evaluation) |

---

## Common Pitfalls

### 1. Storing Sensitive Data in Plaintext

```javascript
// Anti-pattern
user.creditCard = creditCard;  // Store plaintext
user.ssn = ssn;

// Better: Don't store sensitive data; use third-party services
// Or encrypt if must store
user.creditCard = await encrypt(creditCard, encryptionKey);
```

### 2. SQL Injection via String Concatenation

```javascript
// Anti-pattern
const query = `SELECT * FROM users WHERE id = ${userId}`;

// Better: Use parameterized queries
const query = 'SELECT * FROM users WHERE id = $1';
await db.query(query, [userId]);
```

### 3. CORS Wildcard Origin

```javascript
// Anti-pattern
app.use(cors());  // Allows requests from ANY origin

// Better: Whitelist origins
app.use(cors({ origin: ['https://trusted.com'] }));
```

### 4. Missing HTTPS in Production

```javascript
// Anti-pattern: HTTP in production
server.listen(3000);

// Better: Use HTTPS; redirect HTTP to HTTPS
app.use((req, res, next) => {
  if (req.header('x-forwarded-proto') !== 'https') {
    res.redirect(`https://${req.header('host')}${req.url}`);
  } else {
    next();
  }
});
```

### 5. Logging Credentials

```javascript
// Anti-pattern
logger.info(`User: ${email}, Password: ${password}`);

// Better: Never log passwords
logger.info(`User login attempt: ${email}`);
```

---

## How This Affects System Architecture

Security choices influence architecture:

- **Authentication**: Session vs. token affects scalability and deployment options.
- **Authorization**: RBAC simplifies initial design; ABAC enables complex policies.
- **Data protection**: Encryption/hashing requirements affect performance.
- **Audit trails**: Security logging requirements increase storage and performance monitoring.
- **Resilience**: Rate limiting and circuit breakers prevent abuse and cascade failures.

---

## Key Takeaways

1. **Always use parameterized queries; string concatenation is vulnerable to SQL injection.**
2. **Hash passwords with bcrypt; never store plaintext or use weak hashing (MD5, SHA1).**
3. **Validate all user input; don't trust data from clients.**
4. **Use HTTPS in production; transmit credentials only over encrypted channels.**
5. **Audit dependencies regularly; `npm audit` catches known vulnerabilities.**
6. **Log security events; don't log sensitive data (passwords, tokens).**
7. **Implement rate limiting and CORS restrictively; prevent abuse and unauthorized access.**
