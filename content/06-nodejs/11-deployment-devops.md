# Deployment & DevOps: Containerization, CI/CD, and Operational Excellence

## Conceptual Overview

Deployment and DevOps practices bridge the gap between application development and production operations. Containerization (Docker) provides consistency across environments; CI/CD automates testing and deployment; monitoring and logging enable observability; orchestration platforms (Kubernetes) enable scaling and resilience. Understanding the full deployment pipeline—from commit to running production instance—is essential for delivering reliable, maintainable applications at scale.

The challenge is balancing automation (reducing manual errors) with operational visibility (understanding what's happening in production).

---

## Internal Mechanics: Container Architecture and Runtime Isolation

### Docker: Image and Container Model

**Image**: Immutable blueprint; contains application code, runtime, dependencies, and configuration.

**Container**: Running instance of image; isolated process with dedicated resources (filesystem, network, process space).

**Layering**:
```
FROM node:18-alpine
RUN npm install
COPY . /app
WORKDIR /app
EXPOSE 3000
CMD ["node", "app.js"]
```

Each line creates a layer. Layers are cached and reused:

```
Image layers (cached):
  Layer 1: Base OS (node:18-alpine)
  Layer 2: npm install (cached from dependencies)
  Layer 3: Application code (usually changes frequently)
  
When you rebuild:
  - Layers 1-2 reused from cache (fast)
  - Layer 3 rebuilt (application changed)
```

**Efficiency**: Order matters. Dependencies (rarely change) before application code (frequently changes) maximizes cache hit rate.

### Multi-Stage Builds: Optimizing Image Size

```dockerfile
# Build stage
FROM node:18 AS builder
WORKDIR /app
COPY package.json .
RUN npm install
COPY . .
RUN npm run build

# Runtime stage
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

**Result**:
- Build image: Contains compiler, TypeScript, dev dependencies (large)
- Runtime image: Only production dependencies and compiled code (small)

Final image size: ~200MB instead of 1GB.

---

## CI/CD Pipeline Architecture

### Typical Pipeline: Commit → Test → Build → Deploy

```
1. Developer commits code
      ↓
2. Webhook triggers CI/CD
      ↓
3. Checkout code
      ↓
4. Run tests (unit, integration, E2E)
      ↓
5. Build Docker image
      ↓
6. Push image to registry
      ↓
7. Deploy to staging
      ↓
8. Run smoke tests
      ↓
9. Approve for production
      ↓
10. Deploy to production
```

**Early failure principle**: Fail fast; stop pipeline if tests fail (don't build image, don't deploy).

### GitHub Actions Example

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_PASSWORD: password
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v2
      
      - name: Set up Node.js
        uses: actions/setup-node@v2
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
        env:
          DATABASE_URL: postgresql://postgres:password@postgres:5432/testdb
      
      - name: Build Docker image
        if: github.ref == 'refs/heads/main'
        run: docker build -t myapp:${{ github.sha }} .
      
      - name: Push to registry
        if: github.ref == 'refs/heads/main'
        run: docker push myapp:${{ github.sha }}
```

---

## Environment Configuration Management

### Configuration Hierarchy

```
Defaults (code)
    ↓
Environment-specific (config/production.js)
    ↓
Environment variables (process.env)
    ↓
Secrets (encrypted key management)
```

### Handling Secrets Securely

**Vulnerable**:
```javascript
const DB_PASSWORD = 'secret123';  // Hardcoded! Exposed in version control.
```

**Better: Environment variables**:
```bash
# .env (not in version control; local development only)
DB_PASSWORD=secret123

# .gitignore
.env
```

```javascript
require('dotenv').config();
const dbPassword = process.env.DB_PASSWORD;
```

**Best: Secret management service**:
```bash
# Vault, AWS Secrets Manager, Google Secret Manager
vault kv get secret/database/password

# In container runtime
ENV DB_PASSWORD=<fetched from vault>
```

---

## Monitoring and Observability

### The Three Pillars: Logs, Metrics, Traces

**Logs**: Event-based records (request received, error occurred).

**Metrics**: Time-series data (CPU%, requests/sec, latency percentiles).

**Traces**: Execution flow across distributed systems (request spans multiple microservices).

### Logging: Winston, Pino

```javascript
const logger = require('pino')();

app.use((req, res, next) => {
  logger.info({
    method: req.method,
    path: req.path,
    ip: req.ip
  });
  next();
});

app.get('/api/users/:id', async (req, res) => {
  try {
    const user = await getUser(req.params.id);
    res.json(user);
  } catch (error) {
    logger.error({ error, userId: req.params.id }, 'Failed to get user');
    res.status(500).json({ error: 'Internal server error' });
  }
});
```

### Metrics: Prometheus

```javascript
const prometheus = require('prom-client');

const httpRequests = new prometheus.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'status']
});

const httpDuration = new prometheus.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'route']
});

app.use((req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    httpRequests.inc({ method: req.method, status: res.statusCode });
    httpDuration.observe({ method: req.method, route: req.route?.path }, (Date.now() - start) / 1000);
  });
  next();
});

// Prometheus scrapes /metrics endpoint
app.get('/metrics', (req, res) => {
  res.set('Content-Type', prometheus.register.contentType);
  res.end(prometheus.register.metrics());
});
```

### Distributed Tracing: OpenTelemetry

```javascript
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');

const sdk = new NodeSDK({
  traceExporter: new JaegerExporter({ serviceName: 'my-app' }),
  instrumentations: [getNodeAutoInstrumentations()]
});

sdk.start();
```

---

## Problems & Challenges

### 1. Configuration Drift Across Environments

**The problem**: Dev, staging, and production have different configurations. A change that works in staging might fail in production.

### 2. Secrets Exposure in Container Images

**The problem**: Secrets accidentally included in Docker image (hardcoded, `.env` file committed).

### 3. Deployment Downtime and Failed Rollouts

**The problem**: Deploying new version causes brief downtime. If deployment fails, customers experience service interruption.

### 4. Log Explosion and Retention Costs

**The problem**: Logging everything produces terabytes of data; storage becomes expensive; searching becomes slow.

### 5. Monitoring Blind Spots

**The problem**: Application works locally; fails in production with specific data or load pattern not tested.

### 6. Orchestration Complexity

**The problem**: Kubernetes is powerful but complex; requires deep understanding of networking, volumes, service discovery, etc.

---

## Solutions & Architectural Approaches

### 1. Infrastructure as Code (IaC)

Define infrastructure in code; version control it like application code.

```hcl
# Terraform
resource "docker_image" "app" {
  name = "myapp:${var.version}"
  build {
    context = "${path.module}/."
  }
}

resource "docker_container" "app" {
  image = docker_image.app.image_id
  ports {
    internal = 3000
    external = 80
  }
  env = [
    "DATABASE_URL=${var.database_url}",
    "NODE_ENV=production"
  ]
}
```

**Benefits**: Repeatable deployments; auditability; version history.

### 2. Blue-Green Deployment: Zero-Downtime Rollouts

Maintain two production environments (Blue and Green). Switch traffic between them:

```
Blue (v1):  10.0.0.1
Green (v2): 10.0.0.2

Load balancer routes traffic to Blue.

Deploy to Green:
  1. Build and deploy v2 to Green
  2. Run smoke tests on Green
  3. Switch load balancer to Green (instant switch; no downtime)
  4. If issues, revert to Blue
```

**Benefits**: Zero downtime; instant rollback.

**Trade-off**: Requires duplicating infrastructure (double the cost).

### 3. Graceful Shutdown and Connection Draining

```javascript
const server = app.listen(3000);

process.on('SIGTERM', async () => {
  console.log('Shutdown signal received');
  
  // Stop accepting new connections
  server.close(async () => {
    // Wait for in-flight requests to complete
    await db.close();
    await cache.close();
    process.exit(0);
  });
  
  // Force shutdown after timeout
  setTimeout(() => {
    console.log('Forced shutdown');
    process.exit(1);
  }, 30 * 1000);
});
```

### 4. Structured Logging with Correlation IDs

```javascript
const correlationId = uuid.v4();

// Log with correlation ID
logger.info({ correlationId, userId: user.id }, 'User logged in');

// Trace through distributed system
// Request → ServiceA (correlationId) → ServiceB (correlationId) → ServiceC
// All logs linked by correlationId; easy to trace request flow
```

### 5. Canary Deployments: Gradual Traffic Shift

```
v1: 100% traffic
    ↓
v1: 95% traffic → v2: 5% traffic (canary; monitors for errors)
    ↓
v1: 50% traffic → v2: 50% traffic
    ↓
v1: 0% traffic → v2: 100% traffic
```

**Benefits**: Catch issues affecting small percentage of traffic before full rollout.

**Tools**: Istio, ArgoCD, Flagger (automatic canary promotion).

### 6. Health Checks and Readiness Probes

```javascript
// Liveness probe: Is the application running?
app.get('/health/live', (req, res) => {
  res.json({ status: 'alive' });
});

// Readiness probe: Can the application handle requests?
app.get('/health/ready', async (req, res) => {
  try {
    await db.query('SELECT 1');  // Check database connection
    await cache.get('test');     // Check cache connection
    res.json({ status: 'ready' });
  } catch (error) {
    res.status(503).json({ status: 'not ready', error: error.message });
  }
});
```

### 7. Log Aggregation and Querying

```javascript
// Send logs to centralized service
const ElasticsearchTransport = require('pino-elasticsearch');

const logger = require('pino')(
  ElasticsearchTransport({
    node: 'http://elasticsearch:9200',
    index: 'logs-%d'
  })
);

// Query in Kibana
GET logs-*/_search
{
  "query": {
    "match": { "correlationId": "abc123" }
  }
}
```

---

## Trade-offs & Limitations

### Single Environment vs. Multiple Environments

| Aspect | Single | Multiple |
|--------|--------|----------|
| **Cost** | Low | High |
| **Safety** | Risky; prod issues impact dev | Safe; isolated |
| **Debugging** | Easy; prod data available | Harder; prod data restricted |

### Container Size

| Approach | Size | Build Time |
|----------|------|-----------|
| **Single-stage** | Large (1GB) | Fast |
| **Multi-stage** | Small (200MB) | Slightly slower |
| **Distroless** | Tiny (50MB) | Slowest |

---

## Common Pitfalls

### 1. Hardcoded Configuration

```javascript
// Anti-pattern
const DB_URL = 'postgresql://localhost:5432/db';
const API_KEY = 'secret123';

// Better: Use environment variables
const DB_URL = process.env.DATABASE_URL;
const API_KEY = process.env.API_KEY;
```

### 2. Secrets in Docker Image

```dockerfile
# Anti-pattern: Secrets baked into image
RUN echo "API_KEY=secret123" >> .env

# Better: Pass at runtime
CMD ["node", "app.js"]
# docker run -e API_KEY=secret123 myapp
```

### 3. No Graceful Shutdown

```javascript
// Anti-pattern: Immediate exit
process.on('SIGTERM', () => {
  process.exit(0);
});

// Better: Drain connections before exiting
process.on('SIGTERM', async () => {
  server.close();
  await db.close();
  process.exit(0);
});
```

### 4. Logging Everything

```javascript
// Anti-pattern: Logs every operation
logger.info(`User ${user.id} viewed page ${page}`);
logger.info(`Database query took ${duration}ms`);

// Better: Log only important events
logger.info({ userId: user.id }, 'User created');
logger.warn({ duration }, 'Slow query detected');
```

### 5. No Health Checks

```javascript
// Anti-pattern: Kubernetes sends traffic to crashed pod
// kubectl auto-restarts pod but traffic is lost

// Better: Implement readiness probe
app.get('/health/ready', async (req, res) => {
  // Check dependencies
});
```

---

## How This Affects System Architecture

DevOps decisions influence architecture:

- **Deployment frequency**: CI/CD enables frequent deployments; architectural changes needed to support safe deployments.
- **Scalability**: Container orchestration (Kubernetes) enables dynamic scaling.
- **Resilience**: Health checks and monitoring enable automatic failover.
- **Observability**: Centralized logging and tracing enable debugging production issues.
- **Cost**: Container efficiency and resource allocation directly impact infrastructure costs.

---

## Key Takeaways

1. **Docker multi-stage builds reduce image size by 5-10x; optimize layer caching by placing dependencies before code.**
2. **CI/CD automates testing and deployment; fail early (tests before build, build before deploy).**
3. **Blue-green and canary deployments enable zero-downtime updates.**
4. **Never hardcode secrets; use environment variables or secret management services.**
5. **Graceful shutdown prevents data loss; drain connections before exiting.**
6. **Health checks and readiness probes enable automatic failover and load distribution.**
7. **Structured logging with correlation IDs enables tracing requests across distributed systems.**
