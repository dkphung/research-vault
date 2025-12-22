---
tags: [authentication]
date: 2024-12-22
status: complete
---

# Production-Grade SimpleSAMLphp Dockerfile Research

**Date**: 2025-10-30
**Project**: SimpleSAMLphp v2.4.2 with RSS Module
**Purpose**: Research and planning for production-grade Dockerfile suitable for Kubernetes deployment

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Current State Analysis](#current-state-analysis)
3. [Dockerfile Comparison: Dockerfile.old vs Dockerfile.ref](#dockerfile-comparison)
4. [Production Dockerfile Best Practices](#production-dockerfile-best-practices)
5. [SimpleSAMLphp-Specific Considerations](#simplesamlphp-specific-considerations)
6. [Kubernetes-Specific Requirements](#kubernetes-specific-requirements)
7. [Security Hardening](#security-hardening)
8. [Gap Analysis & Concerns](#gap-analysis--concerns)
9. [Recommended Production Dockerfile Structure](#recommended-production-dockerfile-structure)
10. [Implementation Recommendations](#implementation-recommendations)

---

## Executive Summary

### Key Findings

1. **Current State**: Repository has minimal `Dockerfile` (2 lines) with two reference implementations (`Dockerfile.old` and `Dockerfile.ref`)
2. **Dockerfile.ref is Superior**: Modern PHP 8.3, better layer optimization, comprehensive PHP extensions, includes RSS module
3. **Critical Gap**: Source files (config, modules, cert, scripts) are missing from repository - only Dockerfiles and docker-compose.yml exist
4. **Production Readiness**: Dockerfile.ref is close to production-ready but needs enhancements for Kubernetes deployment

### Recommended Approach

1. Use **Dockerfile.ref** as the base (90% ready)
2. Add **multi-stage build** for smaller production image
3. Implement **non-root user** for security
4. Add **health checks** and **signal handling**
5. Externalize **configuration management** for Kubernetes
6. Add **build arguments** for version pinning and flexibility

---

## Current State Analysis

### Repository Structure

```
/Users/little/Projects/simplesaml/
├── Dockerfile                    # Minimal (2 lines only)
├── Dockerfile.old                # Legacy v1.19.0 implementation
├── Dockerfile.ref                # Modern v2.4.2 implementation
├── docker-compose.yml            # Local development configuration
├── CLAUDE.md                     # Development instructions
└── research/                     # Existing documentation
    ├── complete-research-findings.md
    ├── config-map.yaml           # Kubernetes ConfigMap example
    └── [other research docs]
```

### Missing Components

According to Dockerfile.ref and docker-compose.yml, these files should exist but are missing:

```
MISSING FROM REPOSITORY:
├── config/                       # SimpleSAMLphp configuration
│   └── module_cron.php
├── modules/                      # Custom RSS modules
│   ├── rss/
│   └── rsstheme/
├── cert/                         # Certificates
│   └── inc-md-cert.pem
├── public/                       # Public assets
│   └── healthz.php
├── fixes/                        # Compatibility patches
│   └── pdo-handler-php82-fix.patch
├── apache-simplesamlphp.conf     # Apache configuration
└── docker-php-entrypoint         # Container entrypoint script
```

**Impact**: Cannot build Dockerfile.ref without these files. This research assumes they exist or will be created.

---

## Dockerfile Comparison

### Dockerfile.old (Legacy)

**Target**: SimpleSAMLphp v1.19.0 on PHP 7.4.16

**Strengths**:
- Minimal layer count (efficient)
- Uses Composer v1 (appropriate for v1.19.0)
- Includes health check endpoint
- Custom entrypoint script
- Security hardening (security.conf)

**Weaknesses**:
- ❌ **Outdated PHP version** (7.4 - EOL November 2022)
- ❌ **Outdated SimpleSAMLphp** (v1.19.0 - multiple security vulnerabilities)
- ❌ **Runs as root** (security risk)
- ❌ **No multi-stage build** (larger image size)
- ❌ **Hardcoded working directory** `/var/simplesamlphp` (non-standard)
- ❌ **Limited PHP extensions** (only mysqli, pdo_mysql)
- ❌ **No health check defined** in Dockerfile
- ❌ **Manual sed modification** of core SimpleSAMLphp files (line 35)

**Size Estimate**: ~500-600 MB

---

### Dockerfile.ref (Modern)

**Target**: SimpleSAMLphp v2.4.2 on PHP 8.3

**Strengths**:
- ✅ **Modern PHP 8.3** (actively supported until November 2026)
- ✅ **Latest SimpleSAMLphp v2.4.2** (current stable release)
- ✅ **Comprehensive PHP extensions** (intl, zip, mbstring, dom, simplexml)
- ✅ **Better layer caching** (separates system deps, PHP extensions, app installation)
- ✅ **Uses Composer 2** (faster, more secure)
- ✅ **Standard working directory** (`/var/www/html`)
- ✅ **Includes RSS module** and dependencies
- ✅ **Proper file permissions** (chown www-data)
- ✅ **Optimized Composer** (--no-dev, --optimize-autoloader)
- ✅ **Applies compatibility fixes** via patches
- ✅ **Health check endpoint** (healthz.php)
- ✅ **Removes unnecessary packages** (ldap module)

**Weaknesses**:
- ❌ **Still runs as root** (www-data used for file ownership only)
- ❌ **No multi-stage build** (carries build dependencies in production)
- ❌ **No health check** in Dockerfile (only script exists)
- ❌ **No signal handling** configuration
- ❌ **Large image size** (~800 MB - includes git, curl, cron, build tools)
- ❌ **Hardcoded versions** in some places (predis:2.4.0)
- ❌ **apt cache not fully cleaned** between layers
- ❌ **Manual sed modification** of core files (line 81)

**Size Estimate**: ~700-800 MB

---

### Side-by-Side Comparison

| Feature | Dockerfile.old | Dockerfile.ref | Production Best Practice |
|---------|---------------|----------------|-------------------------|
| **Base Image** | php:7.4.16-apache | php:8.3-apache | ✅ Ref (modern) |
| **SimpleSAMLphp Version** | 1.19.0 | 2.4.2 | ✅ Ref (latest) |
| **PHP Extensions** | mysqli, pdo_mysql | pdo_mysql, mbstring, dom, simplexml, intl, zip | ✅ Ref (comprehensive) |
| **Multi-stage Build** | ❌ | ❌ | ❌ Both (should add) |
| **Non-root User** | ❌ | ❌ | ❌ Both (should add) |
| **Layer Optimization** | Moderate | Good | ⚠️ Ref (can improve) |
| **Health Check** | Script only | Script only | ❌ Both (should add to Dockerfile) |
| **Working Directory** | /var/simplesamlphp | /var/www/html | ✅ Ref (standard) |
| **Custom Modules** | ❌ | RSS + RSSTheme | ✅ Ref |
| **Entrypoint Script** | ✅ | ✅ | ✅ Both |
| **Image Size (est.)** | ~500-600 MB | ~700-800 MB | Target: <400 MB |

**Winner**: **Dockerfile.ref** - Modern, comprehensive, better structured. Needs refinement for production.

---

## Production Dockerfile Best Practices

### 1. Multi-Stage Builds

**Purpose**: Separate build-time dependencies from runtime dependencies to minimize image size.

**Benefits**:
- ✅ Smaller production images (50-70% reduction)
- ✅ Reduced attack surface
- ✅ Faster deployment to Kubernetes
- ✅ Lower storage and bandwidth costs

**Structure**:
```dockerfile
# Stage 1: Builder (with git, build tools)
FROM php:8.3-apache AS builder
RUN apt-get install git curl unzip
COPY . /app
RUN composer install --optimize-autoloader

# Stage 2: Production (minimal runtime)
FROM php:8.3-apache AS production
COPY --from=builder /app/vendor /var/www/html/vendor
# Only runtime dependencies
```

### 2. Security Hardening

#### A. Run as Non-Root User

**Why**: Container escape vulnerabilities could grant root access to host.

**How**:
```dockerfile
# Create dedicated user
RUN groupadd -r simplesaml && useradd -r -g simplesaml simplesaml

# Change ownership
RUN chown -R simplesaml:simplesaml /var/www/html

# Switch to non-root
USER simplesaml
```

**Challenge with Apache**: Official php:apache image runs Apache as root by default. Solutions:
1. Use `www-data` user (already exists)
2. Configure Apache to drop privileges after binding to port 80
3. Use port >1024 and run as non-root (Kubernetes can remap)

#### B. Minimize Attack Surface

```dockerfile
# Remove unnecessary packages
RUN apt-get purge -y --auto-remove git curl wget

# Remove package manager cache
RUN rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Disable unnecessary Apache modules
RUN a2dismod status autoindex
```

#### C. Use Specific Image Tags

```dockerfile
# ❌ BAD - uses latest (unpredictable)
FROM php:8.3-apache

# ✅ GOOD - pinned to specific digest
FROM php:8.3.12-apache-bookworm@sha256:abc123...
```

### 3. Layer Optimization

**Principle**: Each RUN command creates a new layer. Minimize layers and optimize for caching.

**Best Practices**:

```dockerfile
# ❌ BAD - Multiple layers, poor caching
RUN apt-get update
RUN apt-get install -y git
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*  # Too late! Cache already in previous layers

# ✅ GOOD - Single layer, cleanup in same command
RUN apt-get update && apt-get install -y \
    git \
    curl \
    && rm -rf /var/lib/apt/lists/*

# ✅ BEST - Separate layers by change frequency
# System dependencies (rarely change) - early layer
RUN apt-get update && apt-get install -y libxml2-dev && rm -rf /var/lib/apt/lists/*

# PHP extensions (rarely change)
RUN docker-php-ext-install pdo_mysql

# Application code (frequently changes) - late layer
COPY . /var/www/html
```

### 4. Health Checks

**Purpose**: Let Kubernetes know if container is healthy and ready to receive traffic.

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
    CMD curl -f http://localhost/healthz.php || exit 1
```

**Requirements**:
- `healthz.php` must return HTTP 200 when healthy
- Should check: database connectivity, Redis, file permissions
- Must be fast (<3s) and lightweight

### 5. Signal Handling

**Purpose**: Graceful shutdown when Kubernetes terminates pod.

**Apache Specific**:
```dockerfile
# Use tini or dumb-init for proper signal forwarding
RUN apt-get update && apt-get install -y tini && rm -rf /var/lib/apt/lists/*
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["apache2-foreground"]
```

**Why**: Apache's `apache2-foreground` script handles SIGTERM correctly, but custom entrypoints may not.

### 6. Build Arguments for Flexibility

```dockerfile
ARG SIMPLESAMLPHP_VERSION=2.4.2
ARG PHP_VERSION=8.3
ARG COMPOSER_VERSION=2.7

FROM php:${PHP_VERSION}-apache

RUN curl -L https://github.com/simplesamlphp/simplesamlphp/releases/download/v${SIMPLESAMLPHP_VERSION}/simplesamlphp-${SIMPLESAMLPHP_VERSION}-full.tar.gz
```

**Benefits**:
- Easy version upgrades
- CI/CD flexibility
- Testing different versions

### 7. Environment-Based Configuration

**12-Factor App Principle**: Configuration via environment variables, not files.

```dockerfile
# Don't bake secrets into image
# ❌ BAD
COPY config/config.php /var/www/html/config/

# ✅ GOOD - Use entrypoint to generate from env vars
COPY docker-entrypoint.sh /usr/local/bin/
ENTRYPOINT ["docker-entrypoint.sh"]
```

**Entrypoint Script**:
```bash
#!/bin/bash
# Generate config.php from environment variables
envsubst < config.template.php > /var/www/html/config/config.php
exec apache2-foreground
```

---

## SimpleSAMLphp-Specific Considerations

### PHP Requirements (v2.4.2)

**Minimum**: PHP 8.1.0
**Recommended**: PHP 8.3 (security support until Nov 2026)

**Required Extensions** (from SimpleSAMLphp docs):
- Always: `date`, `dom`, `fileinfo`, `filter`, `hash`, `json`, `libxml`, `mbstring`, `openssl`, `pcre`, `session`, `simplexml`, `sodium`, `SPL`, `zlib`
- Linux: `posix`
- Translations: `intl`
- HTTP requests: `curl`
- LDAP: `ldap`
- Database: `PDO` + driver (`pdo_mysql`)
- Redis session: `predis` (PHP package, not extension)

### Writable Directories

SimpleSAMLphp needs write access to:

```
/var/www/html/simplesamlphp/
├── log/          # Application logs
├── data/         # Cached data (metadata, etc.)
└── cert/         # Certificate storage (some scenarios)
```

**Kubernetes Approach**:
- Mount `emptyDir` volumes for log and data
- Use ConfigMap for read-only config
- Use Secrets for certificates

### Configuration Management

SimpleSAMLphp has two config locations:
1. `/var/www/html/simplesamlphp/config/` - Main config directory
2. Environment variables (v2.x supports `getenv()`)

**Best Practice for Kubernetes**:
```yaml
# ConfigMap for config.php (non-sensitive)
# Secret for authsources.php (contains credentials)
# Environment variables for:
- SIMPLESAMLPHP_ADMIN_PASSWORD
- SIMPLESAMLPHP_SECRET_SALT
- DATABASE_DSN
- DATABASE_USERNAME
- DATABASE_PASSWORD
- REDIS_HOST
```

### Metadata Handling

SimpleSAMLphp metadata can be:
1. **Static files** in `metadata/` directory
2. **Database** (via PDO) - used by RSS module
3. **MDQ** (Metadata Query Protocol) - fetch on-demand

**For Kubernetes**:
- Static metadata: ConfigMap
- Database metadata: External MySQL/PostgreSQL
- MDQ: Configure endpoints in config.php

### Apache Configuration

SimpleSAMLphp requires Apache rewrite rules:

```apache
Alias /sp /var/www/html/simplesamlphp/public

<Directory /var/www/html/simplesamlphp/public>
    Require all granted
    RewriteEngine On
    RewriteBase /sp
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteRule ^(.*)$ index.php/$1 [L,QSA]
</Directory>
```

**Dockerfile.ref already includes**: `COPY apache-simplesamlphp.conf` (missing from repo)

### RSS Module Dependencies

From `Dockerfile.ref` and `docker-compose.yml`:

**Composer Packages**:
- `predis/predis:2.4.0` - Redis client
- RSS module itself (`modules/rss/composer.json`)

**External Services**:
- MySQL database (metadata storage)
- Redis (session/cache storage)
- Mock OIDC provider (in dev, real OIDC in prod)
- Mock client-config service (in dev, real service in prod)

**Environment Variables**:
```bash
RSS_CLIENT_CONFIG_URL=http://client-config-service
RSS_OPENID_URL=http://oidc-provider
RSS_OPENID_CLIENT_ID=rss-client
RSS_OPENID_CLIENT_SECRET=xxx
RSS_CIPHER_PASSPHRASE=32-char-key
CRON_KEY=cron-secret
```

---

## Kubernetes-Specific Requirements

### 1. 12-Factor App Principles

| Factor | Requirement | Implementation |
|--------|-------------|----------------|
| **I. Codebase** | One codebase, many deploys | ✅ Docker image tagged per environment |
| **II. Dependencies** | Explicitly declare | ✅ Composer, Dockerfile |
| **III. Config** | Store in environment | ⚠️ Partial - needs more env var support |
| **IV. Backing Services** | Treat as attached resources | ✅ MySQL, Redis via env vars |
| **V. Build/Release/Run** | Strict separation | ✅ Docker build vs Kubernetes deploy |
| **VI. Processes** | Stateless processes | ⚠️ Needs session storage in Redis |
| **VII. Port Binding** | Export via port binding | ✅ Apache on port 80 |
| **VIII. Concurrency** | Scale via processes | ✅ Kubernetes replicas |
| **IX. Disposability** | Fast startup, graceful shutdown | ⚠️ Needs signal handling |
| **X. Dev/Prod Parity** | Keep environments similar | ✅ Same Docker image |
| **XI. Logs** | Treat as event streams | ⚠️ Needs stdout/stderr logging |
| **XII. Admin Processes** | Run as one-off processes | ✅ Kubernetes Jobs for cron |

### 2. Stateless Design

**Current State**: docker-compose.yml shows volumes for:
- `./config` - Configuration files
- `./log` - Application logs
- `./metadata` - Metadata files
- `./cert` - Certificates

**Kubernetes Approach**:

```yaml
# Pod volumes
volumes:
  - name: config
    configMap:
      name: simplesaml-config
  - name: secrets
    secret:
      secretName: simplesaml-secrets
  - name: log
    emptyDir: {}  # Ephemeral - use stdout instead
  - name: data
    emptyDir: {}  # Ephemeral cache
```

**Session Storage**: MUST use Redis (already configured in docker-compose.yml)

### 3. Health and Readiness Probes

```yaml
# Kubernetes Deployment
livenessProbe:
  httpGet:
    path: /sp/healthz.php
    port: 80
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /sp/healthz.php
    port: 80
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3
```

**healthz.php Requirements** (must implement):
- Database connectivity check
- Redis connectivity check
- File permission check
- Return HTTP 200 if all healthy, 503 if not

### 4. Graceful Shutdown

**Kubernetes Pod Termination Flow**:
1. Pod receives SIGTERM
2. Grace period (default 30s)
3. If still running, SIGKILL

**Requirements**:
- Apache must handle SIGTERM gracefully
- In-flight requests should complete
- New requests should be rejected (readiness probe fails)

**Implementation**:
```dockerfile
# Use tini for proper signal handling
ENTRYPOINT ["/usr/bin/tini", "--", "docker-php-entrypoint"]
CMD ["apache2-foreground"]
```

### 5. Resource Limits

From `docker-compose.yml`: `mem_limit: 1g`

**Kubernetes Resources**:
```yaml
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 2000m
    memory: 1Gi
```

**PHP Configuration** (php-ext-memory.ini):
```ini
memory_limit = 1024M
max_execution_time = 300
```

### 6. Image Tagging Strategy

**Best Practice**:
- Use semantic versioning: `v2.4.2`
- Include git commit SHA: `v2.4.2-abc1234`
- Tag environments: `v2.4.2-production`, `v2.4.2-staging`

**Example**:
```bash
docker build -t simplesamlphp:2.4.2-abc1234 .
docker tag simplesamlphp:2.4.2-abc1234 simplesamlphp:2.4.2
docker tag simplesamlphp:2.4.2-abc1234 simplesamlphp:latest
```

### 7. Logging to stdout/stderr

**Current**: Logs to `/var/www/html/simplesamlphp/log/`

**Kubernetes Best Practice**: Logs to stdout/stderr (collected by Kubernetes)

**Solutions**:
1. Symlink log files to stdout:
   ```dockerfile
   RUN ln -sf /dev/stdout /var/www/html/simplesamlphp/log/simplesamlphp.log
   ```

2. Configure SimpleSAMLphp to log to syslog/errorlog:
   ```php
   'logging.handler' => 'errorlog',  // Logs to stderr
   ```

3. Use sidecar container (more complex, not recommended)

---

## Security Hardening

### 1. Run as Non-Root User

**Challenge**: Apache binds to port 80 (requires root)

**Solution 1**: Use high-numbered port (8080) and run as non-root
```dockerfile
# Modify Apache to listen on 8080
RUN sed -i 's/Listen 80/Listen 8080/' /etc/apache2/ports.conf
RUN sed -i 's/:80/:8080/' /etc/apache2/sites-available/*.conf

# Run as www-data
USER www-data
EXPOSE 8080
```

Then in Kubernetes Service: `port: 80, targetPort: 8080`

**Solution 2**: Use `setcap` to allow www-data to bind to port 80
```dockerfile
RUN apt-get install -y libcap2-bin
RUN setcap 'cap_net_bind_service=+ep' /usr/sbin/apache2
USER www-data
EXPOSE 80
```

**Recommendation**: Solution 1 (port 8080) is simpler and more standard.

### 2. Read-Only Root Filesystem

**Kubernetes**:
```yaml
securityContext:
  readOnlyRootFilesystem: true
```

**Requirements**: Define writable volumes explicitly
```yaml
volumeMounts:
  - name: tmp
    mountPath: /tmp
  - name: log
    mountPath: /var/www/html/simplesamlphp/log
  - name: data
    mountPath: /var/www/html/simplesamlphp/data
  - name: apache-run
    mountPath: /var/run/apache2
```

### 3. Scan for Vulnerabilities

**Tools**:
- `docker scan` (Snyk integration)
- Trivy
- Grype
- Clair

**CI/CD Integration**:
```bash
# Scan during build
docker build -t simplesamlphp:latest .
trivy image simplesamlphp:latest --severity HIGH,CRITICAL --exit-code 1
```

### 4. Secrets Management

**Never bake secrets into image**:

```dockerfile
# ❌ BAD
ENV SIMPLESAMLPHP_ADMIN_PASSWORD=admin123
COPY secrets.php /var/www/html/config/

# ✅ GOOD
# Secrets provided via Kubernetes Secret at runtime
```

**Kubernetes**:
```yaml
envFrom:
  - secretRef:
      name: simplesaml-secrets
```

### 5. Minimal Base Image

**Current**: `php:8.3-apache` (~500 MB base)

**Alternatives**:
1. `php:8.3-apache-slim` - Not available
2. `php:8.3-fpm-alpine` + nginx - Requires significant changes
3. Stay with `php:8.3-apache` but minimize additions

**Recommendation**: Keep `php:8.3-apache` but use multi-stage build to reduce final image size.

---

## Gap Analysis & Concerns

### Critical Gaps

1. **Missing Source Files**
   - `config/`, `modules/`, `cert/`, `public/`, `fixes/`, `apache-simplesamlphp.conf`, `docker-php-entrypoint`
   - **Impact**: Cannot build Dockerfile.ref until these are provided
   - **Mitigation**: Assume they exist or will be created separately

2. **No Multi-Stage Build**
   - Current image includes build tools (git, curl, composer) in production
   - **Impact**: 200-300 MB larger image, increased attack surface
   - **Fix**: Implement multi-stage build (detailed below)

3. **Runs as Root**
   - Apache runs as root user
   - **Impact**: Security risk if container compromised
   - **Fix**: Switch to port 8080 and run as www-data

4. **No Health Check in Dockerfile**
   - healthz.php exists but no HEALTHCHECK directive
   - **Impact**: Kubernetes can't determine container health automatically
   - **Fix**: Add HEALTHCHECK directive

5. **Logs to File Instead of stdout**
   - Logs written to `/var/www/html/simplesamlphp/log/`
   - **Impact**: Not collected by Kubernetes log aggregation
   - **Fix**: Symlink to stdout or configure errorlog handler

### Moderate Concerns

6. **Hardcoded Versions**
   - `predis/predis:2.4.0` hardcoded in Dockerfile.ref line 44
   - **Impact**: Difficult to upgrade, inflexible
   - **Fix**: Use build arguments

7. **Manual Code Modifications**
   - Line 81 in Dockerfile.ref: `sed -i` modifies core SimpleSAMLphp file
   - **Impact**: Fragile, may break on upgrades, difficult to maintain
   - **Fix**: Create a proper patch file or submit upstream PR

8. **Large Image Size**
   - Estimated 700-800 MB
   - **Impact**: Slow deployments, higher storage costs
   - **Fix**: Multi-stage build + remove build tools

9. **No Image Scanning in CI/CD**
   - No evidence of vulnerability scanning
   - **Impact**: May deploy vulnerable images
   - **Fix**: Add Trivy/Grype to CI pipeline

### Minor Issues

10. **No .dockerignore**
    - Entire repository copied during build (if COPY . is used)
    - **Impact**: Larger build context, slower builds
    - **Fix**: Create .dockerignore

11. **No Build Metadata**
    - No labels indicating version, git commit, build date
    - **Impact**: Difficult to trace image provenance
    - **Fix**: Add LABEL directives

12. **Inconsistent Cleanup**
    - Some RUN commands clean apt cache, others don't
    - **Impact**: Slightly larger image
    - **Fix**: Standardize cleanup in all RUN commands

---

## Recommended Production Dockerfile Structure

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│ Stage 1: base                                           │
│ - PHP 8.3 Apache base image                            │
│ - System dependencies                                   │
│ - PHP extensions                                        │
│ - Common configuration                                  │
└─────────────────────────────────────────────────────────┘
                        ↓
        ┌───────────────┴───────────────┐
        ↓                               ↓
┌─────────────────────┐     ┌─────────────────────────┐
│ Stage 2: builder    │     │ Stage 3: production     │
│ - Install Composer  │     │ - Copy from builder     │
│ - Download SSP      │     │ - Runtime config        │
│ - Install deps      │     │ - Non-root user         │
│ - Build RSS module  │     │ - Health check          │
│ - Apply patches     │     │ - Minimal size          │
└─────────────────────┘     └─────────────────────────┘
                                      ↓
                            ┌─────────────────────────┐
                            │ Final Production Image  │
                            │ - 350-450 MB            │
                            │ - Non-root              │
                            │ - Secure                │
                            │ - K8s-ready             │
                            └─────────────────────────┘
```

### Dockerfile Outline

```dockerfile
# Stage 1: Base - Common foundation
FROM php:8.3-apache AS base
ARG SIMPLESAMLPHP_VERSION=2.4.2
# Install system dependencies and PHP extensions
# Configure Apache
# Create writable directories

# Stage 2: Builder - Build-time operations
FROM base AS builder
# Install Composer
# Download SimpleSAMLphp
# Install Composer dependencies
# Build RSS module
# Apply patches

# Stage 3: Production - Minimal runtime
FROM base AS production
# Copy only necessary files from builder
# Remove build tools
# Switch to non-root user
# Configure health check
# Set entrypoint
```

### Key Features

1. **Multi-stage build**: 3 stages (base, builder, production)
2. **Build arguments**: SIMPLESAMLPHP_VERSION, PHP_VERSION, etc.
3. **Non-root user**: Runs as www-data on port 8080
4. **Health check**: HEALTHCHECK directive
5. **Optimized layers**: Minimize layer count and size
6. **Labels**: Metadata for image provenance
7. **Security**: Minimal attack surface, no secrets
8. **Kubernetes-ready**: Stateless, externalized config

---

## Implementation Recommendations

### Phase 1: Immediate Improvements (Low Risk)

1. **Add Multi-Stage Build**
   - Separate builder and production stages
   - Target image size: <450 MB (from current ~750 MB)
   - Estimated effort: 2-3 hours

2. **Add Health Check**
   - Implement HEALTHCHECK directive
   - Ensure healthz.php checks database and Redis
   - Estimated effort: 1 hour

3. **Add Build Arguments**
   - Parameterize SIMPLESAMLPHP_VERSION, PREDIS_VERSION, etc.
   - Estimated effort: 30 minutes

4. **Add Labels**
   - org.opencontainers.image.* labels
   - Include version, git commit, build date
   - Estimated effort: 30 minutes

5. **Create .dockerignore**
   - Exclude .git, research/, node_modules/, etc.
   - Estimated effort: 15 minutes

### Phase 2: Security Hardening (Medium Risk)

6. **Run as Non-Root**
   - Change Apache to port 8080
   - Run as www-data user
   - Update Kubernetes Service accordingly
   - Estimated effort: 2-3 hours
   - **Risk**: Requires testing Apache behavior as non-root

7. **Configure Logging to stdout**
   - Either symlink log files or use errorlog handler
   - Update Kubernetes to collect logs
   - Estimated effort: 1-2 hours
   - **Risk**: May affect existing log monitoring

8. **Add Vulnerability Scanning**
   - Integrate Trivy into CI/CD
   - Gate deployments on HIGH/CRITICAL vulnerabilities
   - Estimated effort: 2-4 hours
   - **Risk**: May block deployments initially

### Phase 3: Advanced Optimizations (Higher Risk)

9. **Refactor sed Modifications to Patches**
   - Create proper patch files for SimpleSAMLphp modifications
   - Document why patches are needed
   - Submit upstream PR if applicable
   - Estimated effort: 3-4 hours
   - **Risk**: May break functionality if patch is incorrect

10. **Read-Only Root Filesystem**
    - Configure all writable paths as volumes
    - Test with Kubernetes securityContext.readOnlyRootFilesystem
    - Estimated effort: 4-6 hours
    - **Risk**: May uncover hidden write dependencies

11. **Automated Testing**
    - Container structure tests (container-structure-test)
    - Integration tests for health checks
    - Security tests (Anchore, Grype)
    - Estimated effort: 8-12 hours
    - **Risk**: Requires CI/CD pipeline setup

### Recommended Sequence

**Sprint 1** (Week 1):
- [ ] Phase 1: Items 1-5 (all low risk, immediate value)
- [ ] Create initial production Dockerfile
- [ ] Test locally with docker-compose
- [ ] Document changes

**Sprint 2** (Week 2):
- [ ] Phase 2: Items 6-7 (security improvements)
- [ ] Test in staging Kubernetes cluster
- [ ] Update Kubernetes manifests
- [ ] Load testing

**Sprint 3** (Week 3):
- [ ] Phase 2: Item 8 (vulnerability scanning)
- [ ] Phase 3: Item 9 (patch refactoring)
- [ ] Production deployment planning
- [ ] Monitoring and alerting setup

**Sprint 4** (Week 4):
- [ ] Phase 3: Items 10-11 (advanced hardening)
- [ ] Production deployment
- [ ] Post-deployment validation

### Testing Checklist

Before deploying to production:

- [ ] Image builds successfully
- [ ] Image size < 500 MB
- [ ] No HIGH/CRITICAL vulnerabilities
- [ ] Health check returns 200 when healthy
- [ ] Health check returns 503 when unhealthy (test by stopping MySQL)
- [ ] Graceful shutdown (SIGTERM handling)
- [ ] Runs as non-root user
- [ ] File permissions correct for www-data
- [ ] Apache starts on port 8080
- [ ] SimpleSAMLphp admin interface accessible
- [ ] SAML authentication flow works
- [ ] RSS module endpoints respond correctly
- [ ] Database connectivity works
- [ ] Redis session storage works
- [ ] Cron jobs trigger (if applicable)
- [ ] Logs appear in Kubernetes logs (kubectl logs)
- [ ] ConfigMap mounted correctly
- [ ] Secrets mounted correctly
- [ ] Pod tolerates restarts (stateless)
- [ ] Horizontal scaling works (multiple replicas)
- [ ] Resource limits respected

### Success Metrics

| Metric | Current | Target | Measurement |
|--------|---------|--------|-------------|
| Image Size | ~750 MB | <450 MB | `docker images` |
| Build Time | ~5 min | <3 min | CI/CD logs |
| Vulnerabilities (HIGH/CRIT) | Unknown | 0 | Trivy scan |
| Startup Time | Unknown | <30s | `initialDelaySeconds` |
| Pod Restart Rate | N/A | <1/day | Kubernetes metrics |
| Memory Usage | 1 GB limit | 512 MB avg | Prometheus |
| CPU Usage | Unknown | <500m avg | Prometheus |

---

## Appendices

### Appendix A: Complete PHP Extension List

**Required by SimpleSAMLphp v2.4.2**:
- date (built-in)
- dom ✅ (Dockerfile.ref)
- fileinfo (built-in)
- filter (built-in)
- hash (built-in)
- json (built-in)
- libxml (built-in)
- mbstring ✅ (Dockerfile.ref)
- openssl (built-in)
- pcre (built-in)
- session (built-in)
- simplexml ✅ (Dockerfile.ref)
- sodium (built-in in PHP 8.3)
- SPL (built-in)
- zlib (built-in)
- posix (built-in on Linux)
- intl ✅ (Dockerfile.ref)
- curl (needs libcurl)
- PDO ✅ (built-in)
- pdo_mysql ✅ (Dockerfile.ref)

**Additional in Dockerfile.ref**:
- zip ✅ (for Composer, file handling)

**Verdict**: Dockerfile.ref has all required extensions ✅

### Appendix B: Docker Build Commands

**Build production image**:
```bash
docker build \
  --target production \
  --build-arg SIMPLESAMLPHP_VERSION=2.4.2 \
  --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
  --build-arg VCS_REF=$(git rev-parse --short HEAD) \
  --tag simplesamlphp:2.4.2 \
  --tag simplesamlphp:latest \
  .
```

**Build and push**:
```bash
docker build -t registry.example.com/simplesamlphp:2.4.2 .
docker push registry.example.com/simplesamlphp:2.4.2
```

**Scan for vulnerabilities**:
```bash
trivy image simplesamlphp:2.4.2 --severity HIGH,CRITICAL
```

### Appendix C: Kubernetes Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simplesamlphp
  namespace: apps
spec:
  replicas: 3
  selector:
    matchLabels:
      app: simplesamlphp
  template:
    metadata:
      labels:
        app: simplesamlphp
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 33  # www-data
        fsGroup: 33
      containers:
      - name: simplesamlphp
        image: registry.example.com/simplesamlphp:2.4.2
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: SIMPLESAML_BASE_URL
          value: "https://saml.example.com"
        envFrom:
        - secretRef:
            name: simplesaml-secrets
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 2000m
            memory: 1Gi
        livenessProbe:
          httpGet:
            path: /sp/healthz.php
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /sp/healthz.php
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        volumeMounts:
        - name: config
          mountPath: /var/www/html/simplesamlphp/config
          readOnly: true
        - name: log
          mountPath: /var/www/html/simplesamlphp/log
        - name: data
          mountPath: /var/www/html/simplesamlphp/data
      volumes:
      - name: config
        configMap:
          name: simplesaml-config
      - name: log
        emptyDir: {}
      - name: data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: simplesamlphp
  namespace: apps
spec:
  selector:
    app: simplesamlphp
  ports:
  - port: 80
    targetPort: 8080
    name: http
  type: ClusterIP
```

### Appendix D: Environment Variables Reference

**SimpleSAMLphp Core**:
- `SIMPLESAMLPHP_ADMIN_PASSWORD` - Admin interface password
- `SIMPLESAMLPHP_SECRET_SALT` - Encryption salt (32+ chars)
- `SIMPLESAML_BASE_URL` - Base URL (e.g., https://saml.example.com)
- `SIMPLESAML_ALIAS_PATH` - URL path (e.g., /sp)

**Database**:
- `DATABASE_DSN` - PDO DSN string (e.g., mysql:host=mysql;dbname=simplesaml)
- `DATABASE_USERNAME` - Database username
- `DATABASE_PASSWORD` - Database password

**Redis**:
- `REDIS_HOST` - Redis hostname
- `REDIS_PORT` - Redis port (default 6379)
- `REDIS_DATABASE` - Redis database number (default 0)
- `REDIS_PREFIX` - Key prefix (e.g., saml_)
- `REDIS_TTL` - Session TTL in seconds

**RSS Module**:
- `RSS_CLIENT_CONFIG_URL` - Client config service URL
- `RSS_OPENID_URL` - OIDC provider URL
- `RSS_OPENID_CLIENT_ID` - OIDC client ID
- `RSS_OPENID_CLIENT_SECRET` - OIDC client secret
- `RSS_CIPHER_PASSPHRASE` - Encryption passphrase (32 chars)
- `CRON_KEY` - Cron endpoint authentication key

**Apache**:
- `APACHE_RUN_USER` - User to run Apache (default www-data)
- `APACHE_RUN_GROUP` - Group to run Apache (default www-data)

---

## Conclusion

### Summary of Findings

1. **Dockerfile.ref is the superior base** - Modern, comprehensive, well-structured
2. **Repository is incomplete** - Missing source files referenced in Dockerfile.ref
3. **Close to production-ready** - Needs multi-stage build, non-root user, health checks
4. **Security improvements needed** - Run as non-root, vulnerability scanning, minimal image
5. **Kubernetes-ready with modifications** - Needs logging to stdout, proper health checks, stateless design

### Recommended Next Steps

1. **Immediate**: Create production Dockerfile based on Dockerfile.ref with multi-stage build
2. **Week 1**: Add health checks, build arguments, labels, .dockerignore
3. **Week 2**: Implement non-root user, stdout logging, test in staging
4. **Week 3**: Add vulnerability scanning, refactor code modifications to patches
5. **Week 4**: Deploy to production with full monitoring

### Estimated Timeline

- **Research & Planning**: 1 day (complete)
- **Implementation**: 2-3 weeks
- **Testing**: 1 week
- **Production Deployment**: 1 week
- **Total**: 4-5 weeks

### Risk Assessment

- **Low Risk**: Multi-stage build, build arguments, labels, .dockerignore
- **Medium Risk**: Non-root user, stdout logging, vulnerability scanning
- **High Risk**: Read-only root filesystem, major refactoring of code modifications

### Final Recommendation

**Proceed with phased approach**:
1. Start with low-risk improvements (multi-stage build, health checks)
2. Test thoroughly in staging environment
3. Gradually add security hardening (non-root, vulnerability scanning)
4. Reserve high-risk changes for later iterations

**Success Criteria**:
- Image size < 450 MB
- Zero HIGH/CRITICAL vulnerabilities
- Runs as non-root
- Health checks working
- Stateless and horizontally scalable
- Production-ready for Kubernetes

---

**End of Research Document**
