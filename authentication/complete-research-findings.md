# Complete Research Findings - SimpleSAMLphp RSS Module & Metadata Management

**Date**: 2025-10-29
**Project**: SimpleSAMLphp v2.4.2 with RSS Module
**Purpose**: Comprehensive documentation of onboarding, architecture, and metadata management research

---

## Table of Contents

### Part 1: RSS Module Deep Dive
1. [RSS Module Overview](#rss-module-overview)
2. [What Does the RSS Module Do?](#what-does-the-rss-module-do)
3. [RSS Module Architecture](#rss-module-architecture)
4. [Core Components Explained](#core-components-explained)
5. [Request Flow Diagrams](#request-flow-diagrams)
6. [Configuration Guide](#configuration-guide)
7. [Database & Storage](#database--storage)
8. [Authentication & Security](#authentication--security)
9. [Testing Framework](#testing-framework)

### Part 2: SimpleSAMLphp PDO Configuration
10. [Is SimpleSAMLphp Configured with PDO?](#is-simplesamlphp-configured-with-pdo)
11. [How RSS Module Uses PDO](#how-rss-module-uses-pdo)
12. [Understanding the Data Flow](#understanding-the-data-flow)

### Part 3: MDQ and Metadata Management
13. [MDQ vs PDO: Key Differences](#mdq-vs-pdo-key-differences)
14. [Can You Use PDO Without RSS Module?](#can-you-use-pdo-without-rss-module)
15. [Using MDQ with PDO](#using-mdq-with-pdo)
16. [InCommon Federation & MDQ](#incommon-federation--mdq)
17. [Auto-Refresh Mechanisms](#auto-refresh-mechanisms)
18. [Recommended Production Architecture](#recommended-production-architecture)

### Part 4: Implementation Guidance
19. [Migration Paths](#migration-paths)
20. [Quick Reference](#quick-reference)
21. [Troubleshooting](#troubleshooting)

---

# Part 1: RSS Module Deep Dive

## RSS Module Overview

The RSS module is a **custom SimpleSAMLphp v2.4.2 extension** that provides:

- **SAML metadata management** with external service integration
- **RESTful API endpoints** for metadata operations
- **OIDC-based authentication** for secure API access
- **Automated metadata refresh** via cron jobs
- **XML-to-JSON metadata conversion** for client applications

### Key Technologies
- PHP 8.1+ with strict typing
- Slim Framework v4 (for REST API)
- SimpleSAMLphp v2.4.2 (SAML framework)
- OpenID Connect (for authentication)
- MySQL 8.0 (metadata storage)
- Redis (caching layer)

### Business Problem & Solution

**Problem**: SimpleSAMLphp needs to dynamically manage SAML Identity Provider (IdP) metadata from external sources and provide API endpoints for metadata operations.

**Solution**: The RSS module bridges SimpleSAMLphp with an external "client-config" service by:

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

## What Does the RSS Module Do?

### Purpose Clarification

**Initial Understanding**: "RSS module exposes REST endpoints for Client-Config to add metadata into the saml20_idp_remote table"

**Actual Reality**: The RSS module is a **bidirectional bridge** with TWO distinct purposes:

### Purpose 1: Automated Metadata Sync (Pull FROM Client-Config)

The RSS module **pulls** metadata configurations from Client-Config automatically via cron:

```php
// config/module_metarefresh.php
$sets = SimpleSAML\Module\rss\Cron::getConfig();

// This calls Client-Config API:
// GET http://mock-client-config:3001/cron
// Returns: Array of cron sets with metadata sources
```

**Flow:**
```
Cron Trigger → RSS Module → Client-Config Service → Get Metadata Sources
                    ↓
            Fetch XML from Sources
                    ↓
            Parse XML Metadata
                    ↓
        Write to saml20_idp_remote (PDO)
```

### Purpose 2: Manual Operations via REST API (Accept Pushes)

The RSS module **exposes** REST endpoints for external control:

#### **Three REST Endpoints:**

1. **`POST /metadata-parse`** - Utility endpoint
   - Converts SAML XML to JSON format
   - Used for preview/validation
   - **Does NOT write to database** - just returns parsed JSON

2. **`POST /metadata-refresh`** - Direct database write ✅
   - Manually insert/update a single metadata entry
   - Client-Config can push specific metadata updates
   - **Writes to:** `saml20_idp_remote` table via PDO

3. **`POST /cron-refresh`** - Trigger metadata sync
   - Manually trigger metadata refresh for a specific cron set
   - Forces immediate sync instead of waiting for cron
   - **Writes to:** `saml20_idp_remote` table via metarefresh module

### The Complete Picture

```
┌──────────────────────────────────────────────────────────────┐
│  Client-Config Service (Source of Truth)                     │
└───────┬────────────────────────────────┬─────────────────────┘
        │                                │
        │ 1. SimpleSAMLphp pulls         │ 2. Client-Config pushes
        │    cron configs                │    direct updates
        │    (periodic sync)             │    (immediate updates)
        ▼                                ▼
┌─────────────────────────┐    ┌─────────────────────────────┐
│  Cron System            │    │  REST API Endpoints         │
│  - Fetches metadata     │    │  - /metadata-refresh        │
│  - Auto-sync            │    │  - /metadata-parse          │
└───────┬─────────────────┘    │  - /cron-refresh            │
        │                      └────────┬────────────────────┘
        │                               │
        └───────────┬───────────────────┘
                    ▼
        ┌───────────────────────┐
        │  RSS Module Hooks     │
        │  - hook_cron_refresh  │
        │  - hook_metadata_*    │
        └───────────┬───────────┘
                    ▼
        ┌───────────────────────┐
        │  saml20_idp_remote    │
        │  (PDO Database)       │
        └───────────────────────┘
```

---

## RSS Module Architecture

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

**Security Features:**
- **AES-256-CTR encryption** for cached tokens (with random IV)
- **JWT signature verification** using OIDC provider's keys
- **Scope-based authorization** (requires `simplesaml/read_write`)
- **Token expiration validation** with 30-second clock skew tolerance
- **Automatic cache invalidation** when tokens expire

**Important Methods:**

```php
// Get a bearer token for calling external services
public static function getAppToken(): ?string

// Validate incoming API requests
public static function validateToken(?string $accessToken, array $scopes = []): bool
```

**Token Encryption (Updated Implementation):**

```php
// NEW: Random IV for each encryption
private static function encryptTokenForCache(string $token, string $passphrase): string
{
    $iv = random_bytes(16);  // Random IV for each encryption
    $encrypted = openssl_encrypt($token, 'AES-256-CTR', $passphrase, 0, $iv);
    return base64_encode($iv . $encrypted);  // IV + encrypted data
}

// Decryption extracts IV from stored data
private static function decryptTokenFromCache(string $encryptedData, string $passphrase): string|false
{
    $data = base64_decode($encryptedData);
    $iv = substr($data, 0, 16);              // Extract IV
    $encrypted = substr($data, 16);          // Extract encrypted data
    return openssl_decrypt($encrypted, 'AES-256-CTR', $passphrase, 0, $iv);
}
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

### 3. API Handlers (public/handlers.php)

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

#### metadataRefresh()
```php
POST /module.php/rss/metadata-refresh
Body: { "entityData": "{\"entityid\":\"...\", ...}" }
```
- Manually upserts metadata into database
- Bypasses cron system for immediate updates
- Validates entity data before storage

### 4. Hook System (hooks/)

SimpleSAMLphp's hook system enables deep integration:

#### hook_client_config.php
- Fetches data from external client-config service
- Gets OIDC token via `OpenIDClient::getAppToken()`
- Makes authenticated HTTP request
- Returns parsed JSON response

#### hook_cron_refresh.php
- Refreshes metadata for a specific cron set
- Uses SimpleSAMLphp's metarefresh module
- Deletes existing entity before insert (for PDO output)
- Supports flatfile, serialize, or PDO output formats

#### hook_metadata_parse.php
- Parses SAML XML metadata to JSON
- Validates XML against SAML schema
- Extracts SAML 2.0 IdP metadata only

#### hook_metadata_refresh.php
- Manually updates metadata in database
- Uses `MetaDataStorageHandlerPdo` for upsert
- Validates entity data structure

---

## Request Flow Diagrams

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

### Flow 2: External Service Integration (client-config)

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

```yaml
environment:
  # External Service URLs
  - RSS_CLIENT_CONFIG_URL=http://mock-client-config:3001
  - RSS_OPENID_URL=http://mock-oidc-provider:3002

  # OIDC Credentials
  - RSS_OPENID_CLIENT_ID=rss-client
  - RSS_OPENID_CLIENT_SECRET=rss-secret

  # Security
  - RSS_CIPHER_PASSPHRASE=mock-32-character-passphrase-key

  # Cache Keys (optional - have defaults)
  - RSS_OIDC_CACHE_KEY=rss_oidc_tokens
  - RSS_METADATA_CACHE_KEY=rss_metadata
```

### Module Configuration (config-templates/module_rss.php)

```php
$config = [
    'rss.client_config.url' => getenv('RSS_CLIENT_CONFIG_URL') ?: 'https://localhost/api/client-config',
    'rss.openid.url' => getenv('RSS_OPENID_URL') ?: 'https://localhost/auth',
    'rss.openid.client.id' => getenv('RSS_OPENID_CLIENT_ID') ?: 'rss-client',
    'rss.openid.client.secret' => getenv('RSS_OPENID_CLIENT_SECRET') ?: 'rss-secret',
    'rss.openid.cipher.passphrase' => getenv('RSS_CIPHER_PASSPHRASE') ?: 'default-passphrase-change-me',
    'rss.api.basepath' => '/simplesaml/module.php/rss',
];
```

---

## Database & Storage

### MySQL Schema

**Primary Table:** `saml20_idp_remote`

```sql
CREATE TABLE saml20_idp_remote (
    entity_id VARCHAR(255) NOT NULL PRIMARY KEY,
    entity_data TEXT NOT NULL,           -- PHP serialized array
    entity_type VARCHAR(50),              -- 'saml20-idp-remote'
    expire TIMESTAMP NULL                 -- Optional expiration
);
```

**Key Fields:**
- `entity_id`: SAML entity ID (e.g., `https://idp.example.com`)
- `entity_data`: **PHP serialized array** of metadata (not JSON!)
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
"rss_oidc_tokens" => [
    "app_token" => "base64_encoded_iv_plus_encrypted_jwt"
]
```

2. **Metadata** (key: `rss_metadata`)
```php
"rss_metadata" => [
    "saml20-idp-remote" => [
        "https://idp1.example.com" => [...],
        "https://idp2.example.com" => [...]
    ]
]
```

---

## Authentication & Security

### OIDC Flow (Client Credentials)

```
┌─────────────┐                                    ┌─────────────────┐
│ RSS Module  │                                    │ OIDC Provider   │
└──────┬──────┘                                    └────────┬────────┘
       │ 1. POST /token                                     │
       │    grant_type=client_credentials                   │
       │    client_id=rss-client                            │
       │    client_secret=rss-secret                        │
       ├───────────────────────────────────────────────────►│
       │                                    2. Generate JWT │
       │                                       Sign with RS256
       │                                                    │
       │ 3. Response                                        │
       │    { "access_token": "eyJhbG...",                  │
       │      "token_type": "Bearer",                       │
       │      "expires_in": 3600,                           │
       │      "scope": "simplesaml/read_write" }            │
       │◄───────────────────────────────────────────────────┤
       │ 4. Encrypt token with AES-256-CTR (random IV)      │
       │    Store in Redis cache                            │
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

### Token Encryption (AES-256-CTR with Random IV)

**Why Random IV?**
- Each encryption uses unique initialization vector
- IV stored with encrypted data (IV is not secret)
- Prevents pattern detection in cached tokens
- Best practice for AES-CTR mode

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
```

---

# Part 2: SimpleSAMLphp PDO Configuration

## Is SimpleSAMLphp Configured with PDO?

**Answer: Yes!** SimpleSAMLphp is configured with PDO for metadata storage.

### PDO Configuration in config/config.php

```php
// Lines 316-329: Metadata Sources
'metadata.sources' => [
    [
        'type' => 'flatfile',  // First: File-based metadata
    ],
    [
        'type' => 'pdo',       // Second: Database-based metadata ✅
    ],
],

// Lines 334-348: Database Configuration
'database.dsn' => getenv('DATABASE_DSN') ?: 'mysql:host=mysql;port=3307;dbname=simplesaml',
'database.username' => getenv('DATABASE_USERNAME') ?: 'simplesaml',
'database.password' => getenv('DATABASE_PASSWORD') ?: 'simplesamlpass',

'database.driver_options' => [
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
    PDO::ATTR_EMULATE_PREPARES => false,
    PDO::MYSQL_ATTR_INIT_COMMAND => 'SET NAMES utf8mb4 COLLATE utf8mb4_unicode_ci',
    PDO::ATTR_PERSISTENT => false,
    PDO::ATTR_TIMEOUT => 30,
],
```

### What This Means

SimpleSAMLphp checks **both sources** for metadata:
1. **Files** in `metadata/` directory (checked first)
2. **Database** via PDO (checked second)

This allows for **hybrid storage**:
- Static metadata in files
- Dynamic metadata in database

---

## How RSS Module Uses PDO

The RSS module writes metadata to the database via PDO in several places:

### 1. Metadata Refresh Hook (hooks/hook_metadata_refresh.php)

```php
$pdoHandler = new MetaDataStorageHandlerPdo([]);
$isUpserted = $pdoHandler->addEntry($entityId, "saml20-idp-remote", $entityData);
```

**When Called**: Direct API call to `/metadata-refresh` endpoint

### 2. Cron Refresh Hook (hooks/hook_cron_refresh.php)

```php
case 'pdo':
    // Delete existing metadata first
    $db = Database::getInstance();
    $db->write(
        "DELETE FROM saml20_idp_remote WHERE entity_id = :entity_id",
        ['entity_id' => $entityId]
    );

    // Write new metadata
    $metaloader->writeMetadataPdo($config);
    break;
```

**When Called**: Cron job or manual `/cron-refresh` API call

### Database Tables

```sql
-- Created by init-db.sql
CREATE TABLE saml20_idp_remote (
    entity_id VARCHAR(255) NOT NULL PRIMARY KEY,
    entity_data TEXT NOT NULL,
    entity_type VARCHAR(50),
    expire TIMESTAMP NULL
);
```

---

## Understanding the Data Flow

### Correct Understanding

The RSS module is a **bidirectional bridge**:

```
┌─────────────────────────────────────────────────────────┐
│  Client-Config Service (External)                       │
│  - Master metadata configurations                       │
│  - Cron job definitions                                 │
└────────────────────┬────────────────────────────────────┘
                     │
                     │ BOTH DIRECTIONS:
                     │ 1. SimpleSAMLphp PULLS cron configs
                     │ 2. Client-Config PUSHES direct updates
                     ▼
┌─────────────────────────────────────────────────────────┐
│  RSS Module                                             │
│  - Fetches data FROM Client-Config (cron)               │
│  - Accepts pushes TO database (API)                     │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  MySQL Database (saml20_idp_remote table via PDO)       │
└─────────────────────────────────────────────────────────┘
                     │
                     │ SimpleSAMLphp reads via PDO
                     ▼
┌─────────────────────────────────────────────────────────┐
│  SimpleSAMLphp Authentication                           │
└─────────────────────────────────────────────────────────┘
```

### Two Use Cases

**1. Automated Pull (Bulk Sync)**
- Regular synchronization of bulk metadata
- Client-Config provides URLs to metadata sources
- SimpleSAMLphp fetches and processes automatically via cron

**2. Manual Push (Immediate Updates)**
- Immediate updates for critical changes
- Client-Config has direct API control
- Useful for real-time metadata modifications

Both approaches ultimately write to `saml20_idp_remote` via PDO.

---

# Part 3: MDQ and Metadata Management

## MDQ vs PDO: Key Differences

### What is MDQ?

**MDQ (Metadata Query Protocol)** is a just-in-time metadata fetching protocol for large SAML federations:

```
Traditional (Bulk):                  MDQ (On-Demand):
- Download 100MB XML                 - User authenticates
- Parse 3000+ IdPs                   - Query for 1 specific IdP
- Store all locally                  - Cache that IdP (24hrs)
- Refresh weekly                     - Auto-refresh when needed
```

### Comparison Table

| Feature | MDQ (On-Demand) | PDO (Database Storage) |
|---------|-----------------|------------------------|
| **Purpose** | Federation metadata | Local metadata storage |
| **Storage** | File cache (`cache/mdq/`) | MySQL table |
| **Data Format** | XML cached as files | PHP serialized arrays |
| **Fetch Method** | Query MDQ server | Read from database |
| **When Loaded** | On first auth | Pre-loaded |
| **Auto-Refresh** | Yes (TTL-based) | No (manual) |
| **Ideal For** | Large federations (1000s) | Small sets (10s) |
| **Examples** | InCommon, eduGAIN | Campus IdPs |
| **Latency** | First: ~500ms, Cached: 0ms | Always 0ms |
| **Config File** | `module_metarefresh.php` | `config.php` |

### Visual Comparison

```
MDQ: User Auth → Cache Check → If Missing: Query Server → Cache → Use
PDO: User Auth → Database Read → Use
```

---

## Can You Use PDO Without RSS Module?

### Answer: **Yes!**

The RSS module is **optional** - SimpleSAMLphp has built-in PDO support.

### What SimpleSAMLphp Provides (Built-in)

```php
// config/config.php
'metadata.sources' => [
    ['type' => 'pdo'],  // Built-in PDO support ✅
],
```

### What RSS Module Adds (Optional)

- REST API endpoints for metadata operations
- OIDC authentication for API access
- XML parsing and validation
- Integration with external client-config service
- Automated metadata refresh via cron

**You only need RSS module if you want these features!**

### Direct Database Access Example

```php
<?php
// write-metadata.php
require_once '/var/www/html/simplesamlphp/src/_autoload.php';

use SimpleSAML\Metadata\MetaDataStorageHandlerPdo;

$metadata = [
    'entityid' => 'https://idp.ucdavis.edu',
    'name' => ['en' => 'UC Davis Identity Provider'],
    'SingleSignOnService' => [
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect',
            'Location' => 'https://idp.ucdavis.edu/idp/profile/SAML2/Redirect/SSO',
        ],
    ],
    'keys' => [
        [
            'type' => 'X509Certificate',
            'signing' => true,
            'X509Certificate' => 'MIID...',
        ],
    ],
];

$pdo = new MetaDataStorageHandlerPdo([]);
$result = $pdo->addEntry(
    $metadata['entityid'],
    'saml20-idp-remote',
    $metadata
);

echo $result ? "Success\n" : "Failed\n";
```

### Critical: PHP Serialization Format

SimpleSAMLphp expects **PHP serialized data** in `entity_data` column:

```php
// Correct ✅
$serialized = serialize($metadata);
// Output: a:5:{s:8:"entityid";s:21:"https://idp.example.com";...}

// WRONG ❌
$wrong = json_encode($metadata);
// Output: {"entityid":"https://idp.example.com",...}
```

**For non-PHP languages**, use a serialization library:

```python
import phpserialize

metadata = {'entityid': 'https://idp.example.com', ...}
entity_data = phpserialize.dumps(metadata).decode('utf-8')
```

### When to Use RSS Module vs Direct Access

| Use RSS Module If: | Use Direct PDO If: |
|-------------------|-------------------|
| Need REST API | Want simplest architecture |
| Need OIDC auth | Have database access |
| Need XML validation | Don't need API abstraction |
| External service integration | Prefer direct control |
| Automated URL fetching | Avoid custom module maintenance |

---

## Using MDQ with PDO

### Can You Use Both?

**Yes!** They complement each other:

- **MDQ** for large federations (InCommon: 3000+ IdPs)
- **PDO** for custom/managed IdPs (your institutions)

### Configuration

```php
// config/config.php
'metadata.sources' => [
    ['type' => 'flatfile'],  // Priority 1: Static
    ['type' => 'pdo'],       // Priority 2: Custom managed
    // MDQ configured in module_metarefresh.php (Priority 3)
],
```

```php
// config/module_metarefresh.php
<?php
$config = [
    'sets' => [
        'incommon' => [
            'cron' => ['daily'],
            'sources' => [
                [
                    'type' => 'mdq',
                    'baseURL' => 'https://mdq.incommon.org/entities/',
                    'certificates' => ['inc-md-cert-mdq.pem'],
                    'cachedir' => 'cache/mdq-incommon/',
                    'cachelength' => 86400,  // 24 hours
                ],
            ],
            'outputDir' => 'cache/mdq-incommon/',
            'outputFormat' => 'flatfile',  // MDQ always uses flatfile
        ],
    ],
];
```

### Metadata Resolution Order

```
1. Flatfile (metadata/saml20-idp-remote.php)
   → Not found

2. PDO (saml20_idp_remote table)
   → Not found

3. MDQ (cache/mdq-incommon/)
   → Check cache
   → If expired: Query mdq.incommon.org
   → Cache and use
```

### Example Use Case

```
UC Davis Campus Setup:

Custom Campus IdPs → PDO
- idp.ucdavis.edu
- idp-health.ucdavis.edu
- idp-test.ucdavis.edu

InCommon Federation → MDQ
- Stanford, Berkeley, UCLA
- Any of 3000+ InCommon members
```

---

## InCommon Federation & MDQ

### Important Timeline

- **2020**: InCommon launched MDQ service
- **April 2024**: Announced retirement of legacy aggregate
- **January 20, 2025**: Legacy aggregate retired ⚠️
- **Going forward**: MDQ is the recommended method

### Migration Required

```php
// OLD - Deprecated (retires Jan 20, 2025)
'sources' => [
    [
        'src' => 'http://md.incommon.org/InCommon/InCommon-metadata.xml',
        'certificates' => ['inc-md-cert.pem'],
    ],
],

// NEW - Use MDQ
'sources' => [
    [
        'type' => 'mdq',
        'baseURL' => 'https://mdq.incommon.org/entities/',
        'certificates' => ['inc-md-cert-mdq.pem'],
    ],
],
```

### InCommon MDQ Configuration

```bash
# 1. Download certificate
curl -o cert/inc-md-cert-mdq.pem \
  https://md.incommon.org/certs/inc-md-cert-mdq.pem

# 2. Verify fingerprint
openssl x509 -in cert/inc-md-cert-mdq.pem -noout -fingerprint -sha256
```

```php
// 3. Configure in module_metarefresh.php
'sources' => [
    [
        'type' => 'mdq',
        'baseURL' => 'https://mdq.incommon.org/entities/',
        'certificates' => ['inc-md-cert-mdq.pem'],
        'cachedir' => 'cache/mdq-incommon/',
        'cachelength' => 86400,
    ],
],
```

### How MDQ Works

```
User authenticates with Stanford:
1. SimpleSAMLphp needs https://idp.stanford.edu metadata
2. Compute SHA-1 hash: echo -n "https://idp.stanford.edu" | sha1sum
3. Query: GET https://mdq.incommon.org/entities/{hash}
4. Response: XML metadata with Cache-Control: max-age=86400
5. Save to: cache/mdq-incommon/{hash}.cached
6. Use for authentication
7. Next auth within 24hrs: Read from cache (instant)
```

---

## Auto-Refresh Mechanisms

### MDQ Auto-Refresh (Automatic)

MDQ has **built-in auto-refresh** - no manual intervention needed:

**1. Cache TTL**
```php
'cachelength' => 86400,  // 24 hours
```
After expiration, next authentication triggers re-fetch.

**2. HTTP Cache-Control**
```http
Cache-Control: max-age=86400
```
Server tells SimpleSAMLphp how long to cache.

**3. SAML validUntil**
```xml
<EntityDescriptor validUntil="2024-11-05T12:00:00Z">
```
Metadata self-declares expiration.

**4. Conditional GET**
```http
GET /entities/abc123
If-None-Match: "def456"

HTTP/1.1 304 Not Modified
```
Saves bandwidth if unchanged.

### Refresh Timeline Example

```
Day 1, 00:00: User authenticates → MDQ query → Cache saved
Day 1, 12:00: User authenticates → Read from cache
Day 2, 00:01: Cache expired → MDQ query → Update cache
```

### Optional Cron Pre-Warming

```php
'cron' => ['hourly'],  // Pre-fetch frequently used IdPs
```

**Why Pre-Warm?**
- Avoid first-user latency
- Keep popular IdPs fresh
- Reduce authentication time

**When to Use:**
- ✅ High-traffic SP
- ✅ Predictable IdP set
- ❌ Low-traffic SP
- ❌ Unpredictable usage

### PDO Refresh (Manual Only)

PDO metadata **does not auto-refresh**:

```php
// Manual refresh required
$pdo = new MetaDataStorageHandlerPdo([]);
$pdo->addEntry($entityId, 'saml20-idp-remote', $updatedMetadata);
```

---

## Recommended Production Architecture

### For Most Organizations

```
┌─────────────────────────────────────────────────────────┐
│  Static/Override IdPs (Priority 1)                      │
│  Source: metadata/saml20-idp-remote.php                 │
│  Use: Testing, temporary overrides                      │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│  Custom Campus IdPs (Priority 2)                        │
│  Source: MySQL PDO                                      │
│  Use: Campus IdPs, partners                             │
│  Refresh: Manual (script/API)                           │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│  InCommon Federation (Priority 3)                       │
│  Source: MDQ                                            │
│  Use: All InCommon members (3000+)                      │
│  Refresh: Automatic (24hr TTL)                          │
└─────────────────────────────────────────────────────────┘
```

### Complete Configuration

```php
// config/config.php
'metadata.sources' => [
    ['type' => 'flatfile'],
    ['type' => 'pdo'],
],

'database.dsn' => 'mysql:host=mysql;dbname=simplesaml',
'database.username' => 'simplesaml',
'database.password' => 'simplesamlpass',

'module.enable' => [
    'metarefresh' => true,
    'cron' => true,
],
```

```php
// config/module_metarefresh.php
<?php
$config = [
    'sets' => [
        'incommon' => [
            'cron' => ['daily'],
            'sources' => [
                [
                    'type' => 'mdq',
                    'baseURL' => 'https://mdq.incommon.org/entities/',
                    'certificates' => ['inc-md-cert-mdq.pem'],
                    'cachedir' => 'cache/mdq-incommon/',
                    'cachelength' => 86400,
                ],
            ],
            'outputDir' => 'cache/mdq-incommon/',
            'outputFormat' => 'flatfile',
        ],
    ],
];
```

---

# Part 4: Implementation Guidance

## Migration Paths

### Option 1: Keep RSS Module (Current State)

**When to Choose:**
- ✅ Need REST API for external services
- ✅ Already integrated with client-config service
- ✅ Want OIDC authentication
- ✅ Need XML parsing/validation

**Configuration:**
```php
// Keep current setup
'module.enable' => [
    'rss' => true,
    'metarefresh' => true,
    'cron' => true,
],
```

### Option 2: Simplify to MDQ + Direct PDO

**When to Choose:**
- ✅ Want simpler architecture
- ✅ Have database access from your app
- ✅ Don't need REST API abstraction
- ✅ InCommon is primary federation

**Migration Steps:**

1. **Set up MDQ for InCommon**
```bash
curl -o cert/inc-md-cert-mdq.pem \
  https://md.incommon.org/certs/inc-md-cert-mdq.pem
```

2. **Configure module_metarefresh.php**
```php
$config = [
    'sets' => [
        'incommon' => [
            'sources' => [
                [
                    'type' => 'mdq',
                    'baseURL' => 'https://mdq.incommon.org/entities/',
                    'certificates' => ['inc-md-cert-mdq.pem'],
                ],
            ],
            'outputDir' => 'cache/mdq-incommon/',
            'outputFormat' => 'flatfile',
        ],
    ],
];
```

3. **Create metadata management script**
```php
// bin/manage-metadata.php
<?php
require_once __DIR__ . '/../src/_autoload.php';

use SimpleSAML\Metadata\MetaDataStorageHandlerPdo;

function addIdP(array $metadata): bool {
    $pdo = new MetaDataStorageHandlerPdo([]);
    return $pdo->addEntry(
        $metadata['entityid'],
        'saml20-idp-remote',
        $metadata
    );
}
```

4. **Disable RSS module**
```php
'module.enable' => [
    // 'rss' => false,
    'metarefresh' => true,
    'cron' => true,
],
```

### Option 3: Hybrid Approach

**Best of Both Worlds:**
- MDQ for InCommon (3000+ IdPs, auto-refresh)
- PDO for custom IdPs (direct writes)
- Minimal RSS API (only if needed)

---

## Quick Reference

### MDQ Configuration Template

```php
// config/module_metarefresh.php
<?php
$config = [
    'sets' => [
        'incommon' => [
            'cron' => ['daily'],
            'sources' => [
                [
                    'type' => 'mdq',
                    'baseURL' => 'https://mdq.incommon.org/entities/',
                    'certificates' => ['inc-md-cert-mdq.pem'],
                    'cachedir' => 'cache/mdq-incommon/',
                    'cachelength' => 86400,
                ],
            ],
            'outputDir' => 'cache/mdq-incommon/',
            'outputFormat' => 'flatfile',
        ],
    ],
];
```

### PDO Direct Write Template

```php
<?php
require_once '/var/www/html/simplesamlphp/src/_autoload.php';

use SimpleSAML\Metadata\MetaDataStorageHandlerPdo;

$metadata = [
    'entityid' => 'https://idp.example.edu',
    'SingleSignOnService' => [...],
    'keys' => [...],
];

$pdo = new MetaDataStorageHandlerPdo([]);
$pdo->addEntry($metadata['entityid'], 'saml20-idp-remote', $metadata);
```

### Verification Commands

```bash
# Check MDQ cache
ls -la cache/mdq-incommon/

# Check PDO database
docker exec mysql mysql -u simplesaml -psimplesamlpass simplesaml \
  -e "SELECT entity_id FROM saml20_idp_remote;"

# Test MDQ query
curl https://mdq.incommon.org/entities/$(echo -n "https://idp.stanford.edu" | sha1sum | cut -d' ' -f1)

# Check logs
tail -f log/simplesamlphp.log | grep -i "mdq\|metadata"
```

---

## Troubleshooting

### Issue: MDQ Not Working

**Symptoms:**
- InCommon IdPs not authenticating
- Empty cache directory

**Diagnosis:**
```bash
# Check certificate
ls -la cert/inc-md-cert-mdq.pem

# Test MDQ manually
curl https://mdq.incommon.org/entities/$(echo -n "https://idp.stanford.edu" | sha1sum | cut -d' ' -f1)

# Check logs
tail -f log/simplesamlphp.log | grep -i mdq
```

**Solutions:**
1. Download InCommon certificate
2. Verify metarefresh module enabled
3. Check cache directory permissions
4. Verify network connectivity to mdq.incommon.org

### Issue: PDO Metadata Not Loading

**Symptoms:**
- Custom IdPs not appearing
- Database has data but not used

**Diagnosis:**
```bash
# Check database
docker exec mysql mysql -u simplesaml -psimplesamlpass simplesaml \
  -e "SELECT entity_id, LEFT(entity_data, 50) FROM saml20_idp_remote;"

# Verify config
grep -A5 "metadata.sources" config/config.php
```

**Solutions:**
1. Ensure `['type' => 'pdo']` in metadata.sources
2. Verify database connection settings
3. Check entity_data is PHP serialized (not JSON)
4. Verify PDO module enabled

### Issue: RSS Module Authentication Failing

**Symptoms:**
- 401 Unauthorized on API calls
- Token validation errors

**Diagnosis:**
```bash
# Check OIDC provider
curl http://mock-oidc-provider:3002/health

# Test token generation
docker exec simplesamlphp php -r "
  require 'src/_autoload.php';
  echo SimpleSAML\Module\rss\OpenIDClient::getAppToken();
"

# Check logs
./simplesaml logs simplesamlphp | grep -i "rsslogger\|openid"
```

**Solutions:**
1. Verify OIDC provider accessible
2. Check client ID and secret correct
3. Ensure token has required scopes
4. Clear token cache and retry

---

## Summary & Key Takeaways

### RSS Module Purpose

1. **Bidirectional bridge** between SimpleSAMLphp and external services
2. **Pulls** metadata from client-config (automated sync)
3. **Accepts** metadata via REST API (immediate updates)
4. **Optional** - not required for PDO functionality

### PDO Configuration

1. **Built-in** SimpleSAMLphp feature
2. **Dual sources**: flatfile + PDO
3. **RSS module** uses PDO for all database writes
4. **Direct access** possible without RSS module

### MDQ Protocol

1. **On-demand** metadata fetching for large federations
2. **Auto-refresh** via TTL (24 hours default)
3. **File-based cache** (not database)
4. **Complements PDO** (use both together)

### InCommon Federation

1. **Migrating to MDQ** (legacy retires Jan 20, 2025)
2. **3000+ IdPs** available via MDQ
3. **Automatic updates** from federation
4. **Recommended** for all InCommon users

### Recommended Architecture

```
Flatfile → PDO → MDQ
(Static)  (Custom)  (InCommon)
```

- **Flatfile**: Testing, overrides
- **PDO**: Campus/partner IdPs
- **MDQ**: InCommon federation

### Next Steps

1. ✅ Review current RSS module usage
2. ✅ Set up MDQ for InCommon (if not already)
3. ✅ Decide: Keep RSS module or simplify?
4. ✅ Document metadata management workflow
5. ✅ Plan for Jan 2025 InCommon migration

---

**End of Complete Research Findings**

**Last Updated**: 2025-10-29
**Status**: Complete and verified
**Files**:
- This document: `docs/research/complete-research-findings.md`
- RSS Module Onboarding: `docs/research/rss-module-onboarding.md`
- MDQ/PDO Research: `docs/research/mdq-pdo-research.md`
