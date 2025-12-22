---
tags: [authentication]
date: 2024-12-22
status: complete
---

# RSS Module - Complete Onboarding Guide

**Date**: 2025-10-29
**Purpose**: Comprehensive guide for understanding and maintaining the RSS module
**Audience**: New developers onboarding to the SimpleSAMLphp RSS module

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [What is the RSS Module?](#what-is-the-rss-module)
3. [Architecture Deep Dive](#architecture-deep-dive)
4. [Core Components Explained](#core-components-explained)
5. [How It Works: Request Flows](#how-it-works-request-flows)
6. [Configuration Guide](#configuration-guide)
7. [Database & Storage](#database--storage)
8. [Authentication & Security](#authentication--security)
9. [Testing Framework](#testing-framework)
10. [Common Maintenance Tasks](#common-maintenance-tasks)
11. [Troubleshooting Guide](#troubleshooting-guide)
12. [Development Workflow](#development-workflow)

---

## Executive Summary

The RSS module is a **custom SimpleSAMLphp v2.4.2 extension** that provides:

- **SAML metadata management** with external service integration
- **RESTful API endpoints** for metadata operations
- **OIDC-based authentication** for secure API access
- **Automated metadata refresh** via cron jobs
- **XML-to-JSON metadata conversion** for client applications

**Key Technologies:**
- PHP 8.1+ with strict typing
- Slim Framework v4 (for REST API)
- SimpleSAMLphp v2.4.2 (SAML framework)
- OpenID Connect (for authentication)
- MySQL 8.0 (metadata storage)
- Redis (caching layer)

---

## What is the RSS Module?

### Business Problem

SimpleSAMLphp needs to dynamically manage SAML Identity Provider (IdP) metadata from external sources (the "client-config" service) and provide API endpoints for:

1. **Parsing** SAML metadata XML into JSON format
2. **Refreshing** metadata from external sources
3. **Updating** metadata in the database
4. **Automating** metadata synchronization via cron

### Solution Overview

The RSS module bridges SimpleSAMLphp with an external "client-config" service by:

```
┌─────────────────┐        ┌──────────────┐        ┌─────────────────┐
│  Client-Config  │◄──────►│  RSS Module  │◄──────►│  SimpleSAMLphp  │
│    Service      │  OIDC  │   (Bridge)   │  Hooks │   Core System   │
└─────────────────┘        └──────────────┘        └─────────────────┘
         │                         │                         │
         │                         ▼                         │
         │                  ┌──────────────┐                │
         │                  │  MySQL DB    │                │
         │                  │  (Metadata)  │                │
         │                  └──────────────┘                │
         │                         │                         │
         └─────────────────────────┴─────────────────────────┘
                          Redis Cache (Tokens/Data)
```

---

## Architecture Deep Dive

### Directory Structure

```
modules/rss/
├── config-templates/          # Configuration templates
│   └── module_rss.php        # Module configuration with env vars
├── hooks/                     # SimpleSAMLphp integration hooks
│   ├── hook_client_config.php     # External service integration
│   ├── hook_cron_refresh.php      # Metadata sync automation
│   ├── hook_metadata_parse.php    # XML → JSON conversion
│   └── hook_metadata_refresh.php  # Database metadata updates
├── public/                    # API layer (Slim Framework)
│   ├── index.php             # API router (entry point)
│   ├── handlers.php          # Request handlers (controllers)
│   └── middlewares.php       # Auth & JSON middleware
├── src/                       # Business logic (PSR-4)
│   ├── Logger.php            # Centralized logging
│   ├── LoggerTrait.php       # Logging mixin for classes
│   ├── OpenIDClient.php      # OIDC authentication
│   ├── Cron.php              # Cron configuration management
│   └── MetadataStore/
│       └── MetadataHandler.php    # Custom metadata storage (UNUSED)
├── templates/                 # Twig templates (if needed)
├── composer.json              # Dependencies
├── enable                     # Module auto-enable marker
├── README.md                  # Quick start guide
├── TECHNICAL_DOCUMENTATION.md # Detailed technical docs
└── MIGRATION.md              # v1 → v2 migration guide
```

### Layered Architecture

```
┌─────────────────────────────────────────────────────────┐
│  PRESENTATION LAYER (public/)                           │
│  ┌─────────┐  ┌──────────┐  ┌──────────────┐          │
│  │ Router  │→ │ Handlers │→ │ Middlewares  │           │
│  └─────────┘  └──────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│  BUSINESS LOGIC LAYER (src/)                            │
│  ┌──────────────┐  ┌──────────┐  ┌──────────┐         │
│  │ OpenIDClient │  │  Logger  │  │   Cron   │          │
│  └──────────────┘  └──────────┘  └──────────┘          │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│  INTEGRATION LAYER (hooks/)                             │
│  SimpleSAMLphp Hook System - Event-Driven Integration  │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│  DATA LAYER                                             │
│  MySQL (Metadata) + Redis (Cache) + External Services  │
└─────────────────────────────────────────────────────────┘
```

---

## Core Components Explained

### 1. OpenIDClient (src/OpenIDClient.php)

**Purpose**: Handles all OIDC authentication for API access and external service calls.

**Key Responsibilities:**
- Obtain application tokens via client credentials flow
- Validate incoming JWT bearer tokens
- Encrypt/decrypt tokens for secure caching
- Verify token signatures and expiration

**Important Methods:**

```php
// Get a bearer token for calling external services
public static function getAppToken(): ?string

// Validate incoming API requests
public static function validateToken(?string $accessToken, array $scopes = []): bool
```

**Security Features:**
- **AES-256-CTR encryption** for cached tokens
- **JWT signature verification** using OIDC provider's keys
- **Scope-based authorization** (requires `simplesaml/read_write`)
- **Token expiration validation** with 30-second clock skew tolerance
- **Automatic cache invalidation** when tokens expire

**Flow Diagram:**

```
API Request → Extract Bearer Token → Validate JWT Signature
                                            ↓
                                    Check Expiration
                                            ↓
                                    Verify Scopes
                                            ↓
                                  Allow/Deny Request
```

### 2. Logger & LoggerTrait (src/Logger.php, src/LoggerTrait.php)

**Purpose**: Centralized logging with automatic context detection.

**Logger Class:**
```php
Logger::log("Operation completed", false);        // INFO
Logger::log("Error occurred", true);              // ERROR
Logger::log("Custom context", false, "MyFunc");   // Custom context
```

**LoggerTrait Usage:**
```php
class MyClass {
    use LoggerTrait;

    public function myMethod() {
        self::log("Starting operation");  // Auto-detects: MyClass:myMethod
    }
}
```

**Log Format:**
```
[RSSLogger][ClassName:methodName]: Message content
```

**Integration:**
- Logs go to SimpleSAMLphp's native logging system
- Uses `\SimpleSAML\Logger::info()` and `\SimpleSAML\Logger::error()`
- Logs appear in `/var/www/html/simplesamlphp/log/simplesamlphp.log`

### 3. Cron (src/Cron.php)

**Purpose**: Fetch cron configurations from the client-config service.

**How It Works:**

```php
$cronConfig = Cron::getConfig();
// Returns: [
//     'set_name' => [
//         'sources' => [...],
//         'outputDir' => '...',
//         'outputFormat' => 'pdo',
//         ...
//     ]
// ]
```

**Integration Point:**
- Called by `config/module_metarefresh.php`
- Merges external cron configs with local configs
- Powers SimpleSAMLphp's metarefresh module

### 4. MetadataHandler (src/MetadataStore/MetadataHandler.php)

**Status**: ⚠️ **CURRENTLY UNUSED** (kept for reference)

**Original Purpose**: Custom metadata storage handler for fetching metadata from client-config service.

**Note**: The current implementation uses SimpleSAMLphp's built-in `MetaDataStorageHandlerPdo` for database storage instead.

### 5. API Handlers (public/handlers.php)

**Three Main Handlers:**

#### cronRefresh()
```php
POST /module.php/rss/cron-refresh
Body: { "set": "idp_metadata", "entityId": "https://example.com/idp" }
```
- Triggers metadata refresh for a specific cron set
- Calls `hook_cron_refresh` which uses SimpleSAMLphp's metarefresh system
- Deletes old metadata before inserting new (for clean updates)

#### metadataParse()
```php
POST /module.php/rss/metadata-parse
Body: { "xmlMetadata": "<EntityDescriptor>...</EntityDescriptor>" }
```
- Parses SAML XML metadata into JSON format
- Validates XML against SAML metadata schema
- Returns array of parsed IdP metadata entities
- Used by client applications to preview metadata

#### metadataRefresh()
```php
POST /module.php/rss/metadata-refresh
Body: { "entityData": "{\"entityid\":\"...\", ...}" }
```
- Manually upserts metadata into database
- Bypasses cron system for immediate updates
- Validates entity data before storage

### 6. Middleware System (public/middlewares.php)

**Two Middlewares:**

#### auth() - Authentication Middleware
```php
function auth(Request $request, RequestHandler $handler)
```
- Runs **first** in the pipeline
- Extracts `Authorization: Bearer <token>` header
- Validates token using `OpenIDClient::validateToken()`
- Requires scope: `simplesaml/read_write`
- Throws `HttpUnauthorizedException` if invalid

#### json() - Response Middleware
```php
function json(Request $request, RequestHandler $handler)
```
- Runs **last** in the pipeline
- Sets `Content-Type: application/json` header
- Ensures all responses are JSON-formatted

**Pipeline:**
```
Request → auth() → handler() → json() → Response
```

---

## How It Works: Request Flows

### Flow 1: API Request with OIDC Authentication

```
┌──────────────┐
│ External API │
│   Client     │
└──────┬───────┘
       │ 1. POST /cron-refresh
       │    Authorization: Bearer <jwt_token>
       ▼
┌──────────────────┐
│  auth()          │
│  middleware      │◄── Validates JWT signature
└────────┬─────────┘    Checks expiration
         │              Verifies scopes
         │ ✓ Valid
         ▼
┌──────────────────┐
│  cronRefresh()   │
│  handler         │
└────────┬─────────┘
         │ 2. Calls SimpleSAML hook system
         ▼
┌──────────────────┐
│  hook_cron_      │
│  refresh         │◄── Loads metarefresh config
└────────┬─────────┘    Fetches metadata from sources
         │              Writes to database
         │
         ▼
┌──────────────────┐
│  MySQL Database  │
│  (saml20_idp_    │
│   remote table)  │
└──────────────────┘
```

### Flow 2: Cron-Based Metadata Refresh

```
┌──────────────────┐
│ SimpleSAMLphp    │
│ Cron System      │
└────────┬─────────┘
         │ Runs daily/hourly
         ▼
┌──────────────────┐
│ Cron::getConfig()│◄── Fetches cron config from client-config
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ metarefresh      │
│ module           │◄── SimpleSAMLphp's built-in module
└────────┬─────────┘
         │ For each configured source
         ▼
┌──────────────────┐
│ MetaLoader       │◄── Fetches metadata from URLs
└────────┬─────────┘    Validates XML
         │              Filters by whitelist/blacklist
         ▼
┌──────────────────┐
│ hook_cron_       │
│ refresh          │◄── RSS-specific handling
└────────┬─────────┘    Deletes old entity before insert
         │
         ▼
┌──────────────────┐
│ MySQL Database   │
└──────────────────┘
```

### Flow 3: External Service Integration (client-config)

```
┌──────────────────┐
│ RSS Module       │
│ Needs Data       │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ OpenIDClient::   │
│ getAppToken()    │◄── Check Redis cache
└────────┬─────────┘    If missing/expired:
         │                - Call OIDC provider
         │                - Get new token
         │                - Encrypt & cache
         ▼
┌──────────────────┐
│ OIDC Provider    │
│ (mock service)   │◄── POST /token (client_credentials)
└────────┬─────────┘    Returns: access_token (JWT)
         │
         │ Bearer Token
         ▼
┌──────────────────┐
│ Client-Config    │
│ Service          │◄── GET /metadata, /cron, etc.
└────────┬─────────┘    With Authorization header
         │
         │ JSON Response
         ▼
┌──────────────────┐
│ hook_client_     │
│ config           │◄── Validates response
└────────┬─────────┘    Returns data to caller
         │
         ▼
    [Used by RSS Module]
```

---

## Configuration Guide

### Environment Variables (docker-compose.yml)

```bash
# External Service URLs
RSS_CLIENT_CONFIG_URL=http://mock-client-config:3001   # Client-config API base URL
RSS_OPENID_URL=http://mock-oidc-provider:3002          # OIDC provider URL

# OIDC Credentials
RSS_OPENID_CLIENT_ID=rss-client                        # OIDC client identifier
RSS_OPENID_CLIENT_SECRET=rss-secret                    # OIDC client secret

# Security
RSS_CIPHER_PASSPHRASE=mock-32-character-passphrase-key  # AES-256 encryption key (32+ chars)

# Cache Keys (optional)
# rss.openid.cache.key=rss_oidc_tokens      # Default
# rss.metadata.cache.key=rss_metadata       # Default
```

### Module Configuration (config-templates/module_rss.php)

This file is a **template** that reads environment variables:

```php
'rss.client_config.url' => getenv('RSS_CLIENT_CONFIG_URL') ?: 'https://localhost/api/client-config',
'rss.openid.url' => getenv('RSS_OPENID_URL') ?: 'https://localhost/auth',
'rss.openid.client.id' => getenv('RSS_OPENID_CLIENT_ID') ?: 'rss-client',
'rss.openid.client.secret' => getenv('RSS_OPENID_CLIENT_SECRET') ?: 'rss-secret',
'rss.openid.cipher.passphrase' => getenv('RSS_CIPHER_PASSPHRASE') ?: 'default-passphrase-change-me',
'rss.api.basepath' => '/simplesaml/module.php/rss',
```

**Deployment Note:** In production, copy this to `config/module_rss.php` and customize as needed.

### Metarefresh Integration (config/module_metarefresh.php)

```php
<?php
$sets = [];

$isRSSModuleEnabled = SimpleSAML\Module::isModuleEnabled('rss');
if ($isRSSModuleEnabled) {
    // Dynamically fetch cron configs from client-config service
    $sets = array_merge($sets, SimpleSAML\Module\rss\Cron::getConfig());
}

$config = ['sets' => $sets];
```

**How It Works:**
1. SimpleSAMLphp's cron system loads this file
2. If RSS module is enabled, call `Cron::getConfig()`
3. `Cron::getConfig()` calls `hook_client_config` with endpoint `"cron"`
4. External service returns array of cron sets
5. Sets are merged and processed by metarefresh module

---

## Database & Storage

### MySQL Schema

**Primary Table:** `saml20_idp_remote`

```sql
CREATE TABLE saml20_idp_remote (
    entity_id VARCHAR(255) NOT NULL PRIMARY KEY,
    entity_data TEXT NOT NULL,
    entity_type VARCHAR(50),
    expire TIMESTAMP NULL
);
```

**Key Fields:**
- `entity_id`: SAML entity ID (e.g., `https://idp.example.com`)
- `entity_data`: Serialized PHP array of metadata
- `entity_type`: Always `"saml20-idp-remote"` for RSS
- `expire`: Optional expiration timestamp

**Operations:**

```php
// Upsert (used by metadata_refresh hook)
$pdoHandler = new MetaDataStorageHandlerPdo([]);
$pdoHandler->addEntry($entityId, "saml20-idp-remote", $entityData);

// Delete (used before cron refresh)
$db = Database::getInstance();
$db->write(
    "DELETE FROM saml20_idp_remote WHERE entity_id = :entity_id",
    ['entity_id' => $entityId]
);
```

### Redis Caching

**Two Cache Stores:**

1. **OIDC Tokens** (key: `rss_oidc_tokens`)
```php
// Structure: {cache_key: {type: encrypted_token}}
"rss_oidc_tokens" => [
    "app_token" => "base64_encoded_encrypted_jwt"
]
```

2. **Metadata** (key: `rss_metadata`)
```php
// Structure: {cache_key: {set_name: metadata_array}}
"rss_metadata" => [
    "saml20-idp-remote" => [
        "https://idp1.example.com" => [...],
        "https://idp2.example.com" => [...]
    ]
]
```

**Cache Operations:**

```php
use SimpleSAML\Store\StoreFactory;

$store = StoreFactory::getInstance('redis');

// Set cache
$store->set('rss_oidc_tokens', 'app_token', $encryptedToken);

// Get cache
$cached = $store->get('rss_oidc_tokens', 'app_token');

// Delete cache
$store->delete('rss_oidc_tokens', 'app_token');
```

**Cache TTL:**
- **Tokens**: Automatic (based on JWT `exp` claim)
- **Metadata**: Configurable per metarefresh set (default: no expiration)

---

## Authentication & Security

### OIDC Flow (Client Credentials)

```
┌─────────────┐                                    ┌─────────────────┐
│ RSS Module  │                                    │ OIDC Provider   │
└──────┬──────┘                                    └────────┬────────┘
       │                                                    │
       │ 1. POST /token                                     │
       │    grant_type=client_credentials                   │
       │    client_id=rss-client                            │
       │    client_secret=rss-secret                        │
       ├───────────────────────────────────────────────────►│
       │                                                    │
       │                                    2. Generate JWT │
       │                                       Sign with RS256
       │                                                    │
       │ 3. Response                                        │
       │    { "access_token": "eyJhbG...",                  │
       │      "token_type": "Bearer",                       │
       │      "expires_in": 3600,                           │
       │      "scope": "simplesaml/read_write" }            │
       │◄───────────────────────────────────────────────────┤
       │                                                    │
       │ 4. Encrypt token with AES-256-CTR                  │
       │    Store in Redis cache                            │
       │                                                    │
       ▼                                                    │
```

### Token Validation Process

```php
public static function validateToken(?string $accessToken, array $scopes = []): bool
{
    // 1. Basic validation
    if (!isset($accessToken) || empty(trim($accessToken))) return false;

    // 2. JWT format check (3 parts: header.payload.signature)
    $parts = explode('.', $accessToken);
    if (count($parts) !== 3) return false;

    // 3. Verify JWT signature using OIDC provider's public key
    if (!$client->verifyJWTsignature($accessToken)) return false;

    // 4. Extract claims
    $jwtClaims = $client->getAccessTokenPayload();
    if (!isset($jwtClaims)) return false;

    // 5. Check expiration (with 30-second clock skew)
    if ($jwtClaims->exp <= (time() + 30)) return false;

    // 6. Verify required scopes
    if (count($scopes) > 0) {
        $jwtScopes = explode(" ", $jwtClaims->scope ?? '');
        if (empty(array_intersect($scopes, $jwtScopes))) return false;
    }

    return true;
}
```

### Token Encryption (AES-256-CTR)

**Why Encrypt?**
- Tokens stored in Redis cache
- Additional security layer beyond HTTPS
- Prevents token leakage if cache is compromised

**How It Works:**

```php
// Encryption (before caching)
private static function encryptTokenForCache(string $token, string $passphrase): string
{
    $iv = random_bytes(16);  // Random IV for each encryption
    $encrypted = openssl_encrypt($token, 'AES-256-CTR', $passphrase, 0, $iv);
    return base64_encode($iv . $encrypted);  // IV + encrypted data
}

// Decryption (when reading from cache)
private static function decryptTokenFromCache(string $encryptedData, string $passphrase): string|false
{
    $data = base64_decode($encryptedData);
    $iv = substr($data, 0, 16);              // Extract IV
    $encrypted = substr($data, 16);          // Extract encrypted data
    return openssl_decrypt($encrypted, 'AES-256-CTR', $passphrase, 0, $iv);
}
```

**Security Notes:**
- Uses **random IV** for each encryption (stored with encrypted data)
- Passphrase should be **32+ characters** for AES-256
- IV is **not secret** (it's okay to store it with encrypted data)

### API Security Best Practices

1. **Always use HTTPS in production**
2. **Rotate client secrets regularly** (every 90 days recommended)
3. **Use strong cipher passphrases** (32+ random characters)
4. **Monitor failed authentication attempts**
5. **Invalidate cached tokens on security events**

---

## Testing Framework

### Test Structure

```
testing/
├── unit/                    # Unit tests (63 tests, 283 assertions)
│   ├── LoggerTest.php
│   ├── OpenIDClientTest.php
│   ├── CronTest.php
│   └── ...
├── integration/             # Integration tests (67 tests, 662 assertions)
│   ├── APIEndpointsTest.php
│   ├── HooksTest.php
│   ├── DatabaseTest.php
│   └── ...
├── fixtures/                # Test data
│   ├── sample_metadata.xml
│   ├── mock_responses.json
│   └── ...
├── bootstrap.php            # Test environment setup
├── TestUtils.php            # Test helper functions
└── phpunit.xml             # PHPUnit configuration
```

### Running Tests

```bash
# All tests (130 tests, 945 assertions)
./run-tests.sh

# Unit tests only
./run-tests.sh unit

# Integration tests only
./run-tests.sh integration

# Using Make
make test
make test-unit
make test-integration
```

### Test Coverage

**Unit Tests Cover:**
- OpenIDClient token generation and validation
- Logger context detection
- Cron configuration fetching
- Individual hook functions

**Integration Tests Cover:**
- Full API request/response cycles
- Authentication middleware
- Database operations
- Hook system integration
- External service mocking (MSW)

### Mock Services

**Two Mock Services for Testing:**

1. **mock-oidc-provider** (Port 3002)
   - Simulates OIDC authentication server
   - Issues JWT tokens for testing
   - Validates tokens using shared secret

2. **mock-client-config** (Port 3001)
   - Simulates external client-config service
   - Returns mock metadata and cron configs
   - Requires valid bearer token

**Starting Mock Services:**
```bash
cd mock-services
docker-compose up -d
```

---

## Common Maintenance Tasks

### 1. Adding a New API Endpoint

**Steps:**

1. **Add route** in `public/index.php`:
```php
$app->post('/new-endpoint', 'newEndpointHandler');
```

2. **Create handler** in `public/handlers.php`:
```php
function newEndpointHandler(Request $request, Response $response): Response
{
    $body = json_decode((string)$request->getBody(), true);

    if (json_last_error() !== JSON_ERROR_NONE) {
        throw new HttpBadRequestException($request, "Invalid JSON");
    }

    // Your logic here

    $response->getBody()->write(json_encode(["message" => "Success"]));
    return $response;
}
```

3. **Add tests** in `testing/integration/APIEndpointsTest.php`

4. **Update documentation** in this file and `TECHNICAL_DOCUMENTATION.md`

### 2. Modifying OIDC Configuration

**Scenario:** Changing OIDC provider or credentials

1. Update environment variables in `docker-compose.yml`:
```yaml
environment:
  - RSS_OPENID_URL=https://new-oidc-provider.com
  - RSS_OPENID_CLIENT_ID=new-client-id
  - RSS_OPENID_CLIENT_SECRET=new-secret
```

2. Clear cached tokens:
```bash
./simplesaml exec simplesamlphp php -r "
  \$store = SimpleSAML\Store\StoreFactory::getInstance('redis');
  \$store->delete('rss_oidc_tokens', 'app_token');
"
```

3. Restart services:
```bash
./simplesaml restart
```

4. Verify authentication:
```bash
./simplesaml test
```

### 3. Adding a New Hook

**Steps:**

1. **Create hook file** in `hooks/hook_my_feature.php`:
```php
<?php
declare(strict_types=1);

function rss_hook_my_feature(array &$hookInfo): void
{
    assert(array_key_exists('input', $hookInfo));
    assert(array_key_exists('result', $hookInfo));

    // Your hook logic

    $hookInfo['result'] = $processedData;
}
```

2. **Register hook** in `module.php`:
```php
'hooks' => [
    'my_feature' => 'rss_hook_my_feature',
    // ... existing hooks
],
```

3. **Call hook** from your code:
```php
use SimpleSAML\Module;

$hookInfo = ['input' => $data, 'result' => []];
Module::callHooks('my_feature', $hookInfo);
$result = $hookInfo['result'];
```

4. **Add tests** in `testing/integration/HooksTest.php`

### 4. Debugging Authentication Issues

**Common Issues & Solutions:**

**Issue:** 401 Unauthorized on API requests

```bash
# 1. Check token validity
curl -X POST http://localhost:8080/sp/module.php/rss/metadata-parse \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"xmlMetadata": "<EntityDescriptor>...</EntityDescriptor>"}'

# 2. Check logs
./simplesaml logs simplesamlphp | grep -i "rsslogger"

# 3. Verify OIDC provider is accessible
docker exec simplesamlphp curl http://mock-oidc-provider:3002/health

# 4. Clear token cache and retry
docker exec simplesamlphp php -r "
  \$store = SimpleSAML\Store\StoreFactory::getInstance('redis');
  \$store->delete('rss_oidc_tokens', 'app_token');
"
```

**Issue:** Token validation fails

```php
// Add debug logging in OpenIDClient::validateToken()
self::log("Token parts: " . print_r($parts, true));
self::log("JWT claims: " . print_r($jwtClaims, true));
self::log("Token expiration: " . $jwtClaims->exp . " vs now: " . time());
```

### 5. Updating Dependencies

```bash
# Inside container
./simplesaml exec simplesamlphp bash

# Update composer dependencies
cd modules/rss
composer update

# Run tests to verify compatibility
cd /var/www/html/testing
./run-tests.sh
```

**Key Dependencies:**
- `slim/slim: ^4.0` - REST API framework
- `jumbojett/openid-connect-php: ^0.9` - OIDC client
- `predis/predis: ^2.4.0` - Redis client

### 6. Monitoring & Logging

**View RSS Module Logs:**

```bash
# Real-time logs
./simplesaml logs simplesamlphp | grep RSSLogger

# Filter by component
./simplesaml logs simplesamlphp | grep "OpenIDClient"
./simplesaml logs simplesamlphp | grep "hook_cron_refresh"

# Error logs only
./simplesaml logs simplesamlphp | grep -i "error.*rsslogger"
```

**Key Metrics to Monitor:**
- Token refresh frequency (should match expiration)
- Metadata refresh success rate
- API response times
- Failed authentication attempts

---

## Troubleshooting Guide

### Problem: Module Not Loading

**Symptoms:**
- API endpoints return 404
- Hooks not executing

**Diagnosis:**
```bash
# Check if module is enabled
docker exec simplesamlphp ls modules/rss/enable

# Verify autoloader
docker exec simplesamlphp php -r "
  require '/var/www/html/simplesamlphp/vendor/autoload.php';
  var_dump(class_exists('SimpleSAML\Module\rss\Logger'));
"

# Check SimpleSAMLphp recognizes module
./simplesaml exec simplesamlphp php -r "
  require 'src/_autoload.php';
  var_dump(SimpleSAML\Module::isModuleEnabled('rss'));
"
```

**Solutions:**
1. Ensure `enable` file exists in `modules/rss/`
2. Run `composer install` in `modules/rss/`
3. Restart services: `./simplesaml restart`

### Problem: Metadata Not Refreshing

**Symptoms:**
- Stale metadata in database
- Cron jobs not running

**Diagnosis:**
```bash
# Check cron configuration
docker exec simplesamlphp php -r "
  require 'src/_autoload.php';
  \$config = SimpleSAML\Module\rss\Cron::getConfig();
  print_r(\$config);
"

# Manually trigger cron refresh
curl -X POST http://localhost:8080/sp/module.php/rss/cron-refresh \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"set": "your_set_name", "entityId": "https://idp.example.com"}'

# Check database
docker exec mysql mysql -u simplesaml -psimplesamlpass simplesaml \
  -e "SELECT entity_id, LEFT(entity_data, 100) FROM saml20_idp_remote;"
```

**Solutions:**
1. Verify `config/module_metarefresh.php` calls `Cron::getConfig()`
2. Check client-config service is returning cron configs
3. Verify OIDC token is valid
4. Check SimpleSAMLphp cron is configured

### Problem: Redis Connection Issues

**Symptoms:**
- Tokens not caching
- Repeated OIDC calls

**Diagnosis:**
```bash
# Check Redis connectivity
docker exec simplesamlphp redis-cli -h redis ping

# Check cache contents
docker exec redis redis-cli KEYS "*rss*"
docker exec redis redis-cli GET "saml_rss_oidc_tokens:app_token"

# Verify Redis config in SimpleSAMLphp
docker exec simplesamlphp grep -A5 "store.type.*redis" config/config.php
```

**Solutions:**
1. Ensure Redis service is running: `docker ps | grep redis`
2. Verify Redis environment variables in docker-compose.yml
3. Check SimpleSAMLphp store configuration uses Redis
4. Clear Redis cache: `docker exec redis redis-cli FLUSHDB`

### Problem: External Service Timeouts

**Symptoms:**
- Empty results from hooks
- Slow API responses

**Diagnosis:**
```bash
# Test client-config service directly
curl http://localhost:3001/health
curl -H "Authorization: Bearer $(docker exec simplesamlphp php -r 'require \"src/_autoload.php\"; echo SimpleSAML\Module\rss\OpenIDClient::getAppToken();')" \
  http://localhost:3001/metadata

# Check network connectivity from container
docker exec simplesamlphp curl http://mock-client-config:3001/health
docker exec simplesamlphp curl http://mock-oidc-provider:3002/health

# Monitor logs during request
./simplesaml logs simplesamlphp | grep -i "client_config"
```

**Solutions:**
1. Verify mock services are running: `docker-compose ps`
2. Check service health endpoints
3. Increase timeout in SimpleSAMLphp HTTP utils
4. Verify network configuration in docker-compose.yml

---

## Development Workflow

### Local Development Setup

1. **Start all services:**
```bash
./simplesaml start
```

2. **Verify installation:**
```bash
./simplesaml test
```

3. **Make code changes** in `modules/rss/`

4. **Run tests:**
```bash
./run-tests.sh
```

5. **Check logs:**
```bash
./simplesaml logs simplesamlphp
```

### Making Changes

**Typical Workflow:**

```bash
# 1. Create feature branch
git checkout -b feature/my-feature

# 2. Make changes to RSS module
vim modules/rss/src/MyNewClass.php

# 3. Add tests
vim testing/unit/MyNewClassTest.php

# 4. Run tests
./run-tests.sh

# 5. Check code style (if linter available)
# composer run lint

# 6. Commit changes
git add .
git commit -m "feat: add MyNewClass for X functionality"

# 7. Restart services to test in browser
./simplesaml restart

# 8. Test API endpoints manually
curl -X POST http://localhost:8080/sp/module.php/rss/my-endpoint \
  -H "Authorization: Bearer ..." \
  -H "Content-Type: application/json" \
  -d '{"test": "data"}'
```

### Code Style Guidelines

**PHP Standards:**
- PHP 8.1+ strict typing: `declare(strict_types=1);`
- PSR-4 autoloading
- Type hints for all parameters and return types
- Use `LoggerTrait` for logging

**Example:**

```php
<?php

declare(strict_types=1);

namespace SimpleSAML\Module\rss;

/**
 * Example class demonstrating code style
 */
class ExampleClass
{
    use LoggerTrait;

    /**
     * Example method with proper typing
     *
     * @param string $input Input parameter
     * @return array Processed result
     * @throws \Exception When validation fails
     */
    public function processData(string $input): array
    {
        self::log("Processing data");

        if (empty($input)) {
            self::log("Input is empty", true);
            throw new \Exception("Input cannot be empty");
        }

        // Implementation
        $result = ['processed' => $input];

        self::log("Data processed successfully");
        return $result;
    }
}
```

### Debugging Tips

1. **Enable debug mode** in SimpleSAMLphp:
```php
// config/config.php
'logging.level' => SimpleSAML\Logger::DEBUG,
'showerrors' => true,
```

2. **Add debug logging:**
```php
self::log("Debug: variable value = " . print_r($variable, true));
```

3. **Use PHP interactive shell:**
```bash
./simplesaml exec simplesamlphp php -a
php > require 'src/_autoload.php';
php > $token = SimpleSAML\Module\rss\OpenIDClient::getAppToken();
php > var_dump($token);
```

4. **Check API responses with verbose curl:**
```bash
curl -v -X POST http://localhost:8080/sp/module.php/rss/metadata-parse \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d @testing/fixtures/sample_metadata.xml
```

---

## Additional Resources

### Documentation Files

- **README.md**: Quick start guide
- **TECHNICAL_DOCUMENTATION.md**: Detailed technical reference
- **MIGRATION.md**: v1 to v2 migration guide
- **COMPLETION_SUMMARY.md**: Implementation history

### SimpleSAMLphp Documentation

- [SimpleSAMLphp v2 Docs](https://simplesamlphp.org/docs/stable/)
- [Module Development Guide](https://simplesamlphp.org/docs/stable/simplesamlphp-modules)
- [Hook System](https://simplesamlphp.org/docs/stable/simplesamlphp-hookinterface)

### External Dependencies

- [Slim Framework Docs](https://www.slimframework.com/docs/v4/)
- [OpenID Connect PHP Client](https://github.com/jumbojett/OpenID-Connect-PHP)
- [PHPUnit Documentation](https://phpunit.de/documentation.html)

### Getting Help

- **Project CLAUDE.md**: Project-specific guidance
- **Logs**: `./simplesaml logs simplesamlphp`
- **Tests**: `./run-tests.sh` - comprehensive test suite
- **Health Check**: `http://localhost:8080/healthz.php`

---

## Conclusion

The RSS module is a **well-architected bridge** between SimpleSAMLphp and external services, providing:

✅ **Secure API access** via OIDC authentication
✅ **Flexible metadata management** with multiple endpoints
✅ **Automated synchronization** via cron jobs
✅ **Comprehensive testing** (130 tests, 945 assertions)
✅ **Production-ready** Docker deployment

**Key Takeaways for Maintenance:**

1. **Authentication is central** - All API calls require valid JWT tokens
2. **Hooks power integration** - SimpleSAMLphp's hook system enables deep integration
3. **Caching is critical** - Redis caching reduces external service calls
4. **Testing is comprehensive** - Always run tests after changes
5. **Logging is detailed** - Use logs to debug issues

**Next Steps:**

1. Run the test suite: `./run-tests.sh`
2. Explore the API endpoints with curl/Postman
3. Review the mock services to understand external dependencies
4. Make a small change and see the full workflow

Good luck with maintaining the RSS module! 🚀
