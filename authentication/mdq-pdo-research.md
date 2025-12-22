# MDQ and PDO Research for SimpleSAMLphp

**Date**: 2025-10-29
**Topic**: InCommon MDQ Protocol with PDO metadata storage
**Context**: Understanding how to use InCommon Federation with SimpleSAMLphp while maintaining SimpleSAMLphp as source of truth

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [What is MDQ?](#what-is-mdq)
3. [MDQ vs PDO Comparison](#mdq-vs-pdo-comparison)
4. [How MDQ Works](#how-mdq-works)
5. [Can You Use PDO Without RSS Module?](#can-you-use-pdo-without-rss-module)
6. [Using MDQ and PDO Together](#using-mdq-and-pdo-together)
7. [InCommon Federation Specifics](#incommon-federation-specifics)
8. [Configuration Examples](#configuration-examples)
9. [Auto-Refresh Mechanisms](#auto-refresh-mechanisms)
10. [Recommended Architecture](#recommended-architecture)
11. [Migration Path](#migration-path)

---

## Executive Summary

### Key Findings

**Can you use PDO without RSS module?**
- ✅ **Yes**, SimpleSAMLphp's PDO support is built-in
- ✅ External services can write directly to `saml20_idp_remote` table
- ✅ RSS module is **optional** - only needed for REST API abstraction
- ⚠️ Must use **PHP serialization format** for `entity_data` column

**Can you use MDQ with PDO?**
- ✅ **Yes**, they serve different purposes and complement each other
- ✅ MDQ for large federations (InCommon: 3000+ IdPs)
- ✅ PDO for custom/managed IdPs (your institutions)
- ❌ MDQ does **not** write to PDO - uses file-based cache instead

**Does MDQ auto-refresh?**
- ✅ **Yes**, automatic refresh based on TTL (default: 24 hours)
- ✅ No cron jobs required for basic operation
- ✅ Optional cron-based pre-warming for performance

---

## What is MDQ?

### MDQ (Metadata Query Protocol)

**Official Specification**: [SAML Metadata Query Protocol](https://github.com/metadataquery/md-query)

MDQ is a **just-in-time metadata fetching protocol** designed for large SAML federations:

```
Traditional Approach (Bulk Download):
- Download entire federation metadata (100MB+ XML)
- Parse all 3000+ IdP metadata entries
- Store locally
- Refresh daily/weekly

MDQ Approach (On-Demand):
- User tries to authenticate with specific IdP
- Query MDQ server for that IdP only
- Cache result locally (24 hours)
- Auto-refresh when cache expires
```

### Why MDQ Exists

**Problem**: InCommon Federation has 3000+ identity providers
- Bulk metadata file is huge (100MB+)
- Most SPs only use 5-10 IdPs
- Downloading/parsing all metadata is wasteful
- Metadata can become stale between refreshes

**Solution**: MDQ Protocol
- Fetch metadata only when needed
- Small, focused queries (single IdP)
- Always fresh metadata (short cache TTL)
- Reduced bandwidth and storage

### MDQ URL Structure

```
Base URL: https://mdq.incommon.org/entities/

Query by Entity ID (URL-encoded):
https://mdq.incommon.org/entities/https%3A%2F%2Fidp.example.edu

Query by SHA-1 hash (preferred):
https://mdq.incommon.org/entities/{sha1-hash-of-entityid}

Example:
Entity ID: https://idp.stanford.edu
SHA-1: a1b2c3d4e5f6...
URL: https://mdq.incommon.org/entities/a1b2c3d4e5f6...
```

---

## MDQ vs PDO Comparison

| Feature | MDQ (Metadata Query Protocol) | PDO (Database Storage) |
|---------|-------------------------------|------------------------|
| **Purpose** | On-demand federation metadata | Local metadata storage |
| **Storage Type** | File-based cache (`cache/mdq/`) | MySQL database table |
| **Data Format** | XML cached as files | PHP serialized arrays |
| **Fetch Method** | Query external MDQ server | Read from local database |
| **When Loaded** | On first authentication attempt | Pre-loaded via cron/API |
| **Cache Duration** | 24 hours (configurable) | Until manually refreshed |
| **Auto-Refresh** | Yes (TTL-based) | No (manual trigger) |
| **Ideal For** | Large federations (1000+ IdPs) | Small sets of known IdPs |
| **Examples** | InCommon, eduGAIN, SWAMID | Campus IdPs, partner IdPs |
| **Bandwidth** | Minimal (per-IdP queries) | Initial bulk download |
| **Latency** | First query: ~500ms, cached: 0ms | Always 0ms (local) |
| **Requires Cron** | No (optional for pre-warming) | Yes (for refresh) |
| **Configuration File** | `module_metarefresh.php` | `config.php` |
| **SimpleSAMLphp Module** | `metarefresh` + `cron` | Built-in (core) |

### Visual Comparison

```
┌─────────────────────────────────────────────────────────┐
│  MDQ Architecture (On-Demand)                           │
└─────────────────────────────────────────────────────────┘

User Authentication → Need IdP metadata
                             ↓
                  Check cache/mdq/idp-hash.xml
                             ↓
                    Cache exists and valid?
                             ↓
                    NO → Query MDQ server
                             ↓
              GET https://mdq.incommon.org/entities/{hash}
                             ↓
                    Save to cache/mdq/idp-hash.xml
                             ↓
                      Use metadata

┌─────────────────────────────────────────────────────────┐
│  PDO Architecture (Pre-Loaded)                          │
└─────────────────────────────────────────────────────────┘

Cron/API Trigger → Fetch metadata from source
                             ↓
                   Parse and validate XML
                             ↓
              Serialize to PHP array format
                             ↓
         INSERT INTO saml20_idp_remote (...)
                             ↓
              Metadata ready for use
                             ↓
User Authentication → Read from database (instant)
```

---

## How MDQ Works

### Step-by-Step Flow

**1. User Initiates Authentication**
```
User clicks "Login with Stanford"
SP needs Stanford IdP metadata
Entity ID: https://idp.stanford.edu
```

**2. SimpleSAMLphp Checks Metadata Sources**
```php
// config.php metadata.sources order:
1. flatfile (metadata/saml20-idp-remote.php)
2. pdo (saml20_idp_remote table)
3. metarefresh/MDQ (cache/mdq/ directory)
```

**3. MDQ Query (if not found in previous sources)**
```bash
# Compute SHA-1 hash of entity ID
echo -n "https://idp.stanford.edu" | sha1sum
# Output: 8d9c4ba5e8f1234567890abcdef...

# Query MDQ server
GET https://mdq.incommon.org/entities/8d9c4ba5e8f1234567890abcdef
Accept: application/samlmetadata+xml
```

**4. MDQ Server Response**
```xml
HTTP/1.1 200 OK
Content-Type: application/samlmetadata+xml
Cache-Control: max-age=86400
ETag: "abc123def456"

<?xml version="1.0"?>
<EntityDescriptor entityID="https://idp.stanford.edu"
                  validUntil="2024-11-05T12:00:00Z">
    <IDPSSODescriptor protocolSupportEnumeration="...">
        <SingleSignOnService Binding="..." Location="..."/>
        <KeyDescriptor use="signing">
            <KeyInfo><X509Data><X509Certificate>...</X509Certificate></X509Data></KeyInfo>
        </KeyDescriptor>
    </IDPSSODescriptor>
</EntityDescriptor>
```

**5. SimpleSAMLphp Caches Locally**
```
Save to: cache/mdq/8d9c4ba5e8f1234567890abcdef.cached
Cache duration: 86400 seconds (24 hours)
Next refresh: 2024-10-30 12:00:00
```

**6. Subsequent Authentications**
```
User authenticates again within 24 hours
→ Read from cache/mdq/8d9c4ba5e8f1234567890abcdef.cached
→ Instant response (no network query)
```

**7. Cache Expiration and Refresh**
```
After 24 hours:
→ Cache expired
→ New authentication attempt
→ Query MDQ server again
→ Update cache
```

### MDQ Caching Details

**Cache File Structure**
```bash
cache/mdq/
├── 8d9c4ba5e8f1234567890abcdef.cached  # Stanford IdP
├── 1a2b3c4d5e6f7890abcdef123456.cached  # UC Davis IdP
├── 9f8e7d6c5b4a3210fedcba098765.cached  # UCLA IdP
└── ...
```

**Cache File Contents**
```php
<?php
// Cached SimpleSAMLphp metadata array
return [
    'entityid' => 'https://idp.stanford.edu',
    'metadata-set' => 'saml20-idp-remote',
    'expire' => 1730217600,  // Unix timestamp
    'SingleSignOnService' => [...],
    'SingleLogoutService' => [...],
    'keys' => [...],
];
```

**Cache Validation**
```php
// SimpleSAMLphp checks:
1. Does cache file exist?
2. Has 'expire' timestamp passed?
3. Is 'validUntil' in metadata still valid?

If any fail → Re-query MDQ server
```

---

## Can You Use PDO Without RSS Module?

### Short Answer: **Yes!**

The RSS module is a **custom convenience layer**, not a requirement for PDO functionality.

### What SimpleSAMLphp Provides (Built-in)

```php
// config/config.php
'metadata.sources' => [
    ['type' => 'pdo'],  // Built-in PDO support
],

'database.dsn' => 'mysql:host=mysql;dbname=simplesaml',
'database.username' => 'simplesaml',
'database.password' => 'simplesamlpass',
```

SimpleSAMLphp's **core functionality** includes:
- ✅ PDO metadata storage handler
- ✅ Database connection management
- ✅ Metadata serialization/deserialization
- ✅ Automatic metadata loading from database

### What RSS Module Adds (Optional)

The RSS module provides:
- REST API endpoints for metadata operations
- OIDC authentication for API access
- XML parsing and validation
- Integration with external client-config service
- Automated metadata refresh via cron

**You only need RSS module if you want these features!**

### Direct Database Access Example

**Option 1: Using SimpleSAMLphp's Built-in Handler**

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
            'encryption' => false,
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

**Option 2: Direct SQL (Advanced)**

```php
<?php
$pdo = new PDO(
    'mysql:host=mysql;dbname=simplesaml',
    'simplesaml',
    'simplesamlpass'
);

$metadata = [
    'entityid' => 'https://idp.ucdavis.edu',
    'SingleSignOnService' => [...],
];

$stmt = $pdo->prepare("
    INSERT INTO saml20_idp_remote (entity_id, entity_data, entity_type)
    VALUES (?, ?, ?)
    ON DUPLICATE KEY UPDATE entity_data = VALUES(entity_data)
");

$stmt->execute([
    $metadata['entityid'],
    serialize($metadata),  // CRITICAL: PHP serialize()
    'saml20-idp-remote'
]);
```

**IMPORTANT**: The `entity_data` column must contain **PHP serialized data**, not JSON!

### PHP Serialization Format

SimpleSAMLphp expects this format:

```php
// Correct
$serialized = serialize($metadata);
// Output: a:5:{s:8:"entityid";s:21:"https://idp.example.com";...}

// WRONG - Do not use JSON
$wrong = json_encode($metadata);
// Output: {"entityid":"https://idp.example.com",...}
```

**If using non-PHP language**, use a serialization library:

**Python Example:**
```python
import phpserialize
import mysql.connector

metadata = {
    'entityid': 'https://idp.example.com',
    'SingleSignOnService': [...]
}

# Convert to PHP serialized format
entity_data = phpserialize.dumps(metadata).decode('utf-8')

# Write to database
conn = mysql.connector.connect(
    host='mysql',
    user='simplesaml',
    password='simplesamlpass',
    database='simplesaml'
)

cursor = conn.cursor()
cursor.execute("""
    INSERT INTO saml20_idp_remote (entity_id, entity_data, entity_type)
    VALUES (%s, %s, %s)
    ON DUPLICATE KEY UPDATE entity_data = VALUES(entity_data)
""", (metadata['entityid'], entity_data, 'saml20-idp-remote'))

conn.commit()
```

### Database Schema Reference

```sql
CREATE TABLE saml20_idp_remote (
    entity_id VARCHAR(255) NOT NULL PRIMARY KEY,
    entity_data TEXT NOT NULL,           -- PHP serialized array
    entity_type VARCHAR(50),              -- 'saml20-idp-remote'
    expire TIMESTAMP NULL                 -- Optional expiration
);
```

**Example Row:**
```
entity_id: https://idp.stanford.edu
entity_data: a:8:{s:8:"entityid";s:23:"https://idp.stanford.edu";s:4:"name";a:1:{...}...}
entity_type: saml20-idp-remote
expire: NULL
```

### When to Use RSS Module vs Direct Access

**Use RSS Module if you need:**
- ❌ REST API for external services to add/update metadata
- ❌ OIDC authentication for API security
- ❌ XML parsing/validation before storage
- ❌ Integration with existing client-config service
- ❌ Automated metadata fetching from URLs

**Use Direct PDO Access if you:**
- ✅ Want simplest possible architecture
- ✅ Have database access from your application
- ✅ Don't need REST API abstraction
- ✅ Want to avoid custom module maintenance
- ✅ Prefer direct database control

---

## Using MDQ and PDO Together

### Why Use Both?

**MDQ for Federation Metadata**
- Large federations (InCommon: 3000+ IdPs)
- Automatic updates from federation
- On-demand loading (performance)
- No local management needed

**PDO for Custom Metadata**
- Your institution's IdPs
- Partner institution IdPs
- Test/development IdPs
- Full control and customization

### Configuration

**Step 1: Enable PDO in config.php**

```php
// config/config.php
'metadata.sources' => [
    ['type' => 'flatfile'],  // Optional: for static files
    ['type' => 'pdo'],       // For your custom IdPs
    // Note: MDQ configured in module_metarefresh.php, not here
],

'database.dsn' => 'mysql:host=mysql;dbname=simplesaml',
'database.username' => 'simplesaml',
'database.password' => 'simplesamlpass',
```

**Step 2: Configure MDQ in module_metarefresh.php**

```php
// config/module_metarefresh.php
<?php
$config = [
    'sets' => [
        'incommon' => [
            'cron' => ['daily'],  // Optional: pre-warm cache
            'sources' => [
                [
                    'type' => 'mdq',
                    'baseURL' => 'https://mdq.incommon.org/entities/',
                    'certificates' => [
                        'inc-md-cert-mdq.pem',  // InCommon signing cert
                    ],
                    'validateFingerprint' => 'C4:A3:10:4F:E6:83:...',  // Optional
                    'cachedir' => 'cache/mdq-incommon/',
                    'cachelength' => 86400,  // 24 hours
                ],
            ],
            'outputDir' => 'cache/mdq-incommon/',
            'outputFormat' => 'flatfile',  // MDQ always uses flatfile
            'expireAfter' => 60*60*24*7,   // 7 days
        ],
    ],
];
```

**Step 3: Download InCommon MDQ Signing Certificate**

```bash
# Download InCommon MDQ certificate
cd /var/www/html/simplesamlphp/cert/
curl -o inc-md-cert-mdq.pem \
  https://md.incommon.org/certs/inc-md-cert-mdq.pem

# Verify certificate
openssl x509 -in inc-md-cert-mdq.pem -text -noout
```

### Metadata Resolution Order

SimpleSAMLphp checks sources in this order:

```
1. Flatfile (metadata/saml20-idp-remote.php)
   - Static metadata files
   - Highest priority
   - Manual management

2. PDO (saml20_idp_remote table)
   - Your custom IdPs
   - Dynamically managed
   - Database storage

3. Metarefresh/MDQ (cache/mdq-incommon/)
   - InCommon federation
   - On-demand fetching
   - Auto-refresh

First match wins → Authentication proceeds
```

**Example Flow:**

```
User authenticates with: https://idp.stanford.edu

Check 1: metadata/saml20-idp-remote.php
         → Not found

Check 2: saml20_idp_remote table
         → Not found

Check 3: cache/mdq-incommon/
         → Check cache file
         → If exists and valid: USE IT
         → If not exists: Query mdq.incommon.org
         → Cache result and USE IT

Result: Authentication with Stanford IdP metadata
```

### Example Use Cases

**Scenario 1: UC Davis Campus**
```
Custom Campus IdPs → PDO
- idp.ucdavis.edu (main campus)
- idp-health.ucdavis.edu (health system)
- idp-test.ucdavis.edu (testing)

InCommon Federation → MDQ
- Stanford, Berkeley, UCLA, etc.
- Any of 3000+ InCommon IdPs
```

**Scenario 2: Development Environment**
```
Local Test IdPs → Flatfile
- test-idp-1.local
- test-idp-2.local

Production Campus IdPs → PDO
- idp.production.edu

InCommon Federation → MDQ
- All InCommon members
```

---

## InCommon Federation Specifics

### InCommon Migration to MDQ (Important!)

**Timeline:**
- **2020**: InCommon launched MDQ service (`mdq.incommon.org`)
- **April 2024**: Announced retirement of legacy metadata aggregate
- **January 20, 2025**: Legacy aggregate (`md.incommon.org`) will be retired
- **Going forward**: MDQ is the recommended method

**Legacy Approach (Deprecated):**
```php
// OLD - Being retired January 2025
'sources' => [
    [
        'src' => 'http://md.incommon.org/InCommon/InCommon-metadata.xml',
        'certificates' => ['inc-md-cert.pem'],
    ],
],
```

**New Approach (MDQ - Recommended):**
```php
// NEW - Use this going forward
'sources' => [
    [
        'type' => 'mdq',
        'baseURL' => 'https://mdq.incommon.org/entities/',
        'certificates' => ['inc-md-cert-mdq.pem'],
    ],
],
```

### InCommon MDQ Service Details

**Base URL:**
```
https://mdq.incommon.org/entities/
```

**Supported Query Methods:**

1. **By SHA-1 Hash (Recommended)**
   ```
   https://mdq.incommon.org/entities/{sha1-hash}
   ```
   - More efficient
   - Better caching
   - Privacy-friendly (entity ID not in URL)

2. **By Entity ID (Alternative)**
   ```
   https://mdq.incommon.org/entities/https%3A%2F%2Fidp.example.edu
   ```
   - Human-readable
   - Requires URL encoding

**Response Format:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<EntityDescriptor
    xmlns="urn:oasis:names:tc:SAML:2.0:metadata"
    entityID="https://idp.stanford.edu"
    validUntil="2024-11-15T00:00:00Z">

    <Extensions>
        <mdrpi:RegistrationInfo
            registrationAuthority="http://incommon.org"
            registrationInstant="2024-01-01T00:00:00Z"/>
    </Extensions>

    <IDPSSODescriptor>
        <SingleSignOnService
            Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
            Location="https://idp.stanford.edu/idp/profile/SAML2/Redirect/SSO"/>
        <!-- ... more services and keys ... -->
    </IDPSSODescriptor>
</EntityDescriptor>
```

### InCommon Certificate Management

**Production Certificate:**
```bash
# Download from InCommon
curl -o cert/inc-md-cert-mdq.pem \
  https://md.incommon.org/certs/inc-md-cert-mdq.pem

# Verify fingerprint (SHA-256)
openssl x509 -in cert/inc-md-cert-mdq.pem -noout -fingerprint -sha256
# Should output: C4:A3:10:4F:E6:83:A8:7F:...
```

**Certificate in Configuration:**
```php
'certificates' => [
    'inc-md-cert-mdq.pem',  // Filename in cert/ directory
],

// OR use fingerprint validation
'validateFingerprint' => 'C4:A3:10:4F:E6:83:A8:7F:...',
```

### InCommon Metadata Extensions

InCommon adds useful metadata extensions:

**Entity Categories:**
```xml
<Extensions>
    <!-- Research & Scholarship -->
    <mdattr:EntityAttributes>
        <saml:Attribute Name="http://macedir.org/entity-category">
            <saml:AttributeValue>
                http://refeds.org/category/research-and-scholarship
            </saml:AttributeValue>
        </saml:Attribute>
    </mdattr:EntityAttributes>
</Extensions>
```

SimpleSAMLphp can filter based on these:

```php
'attributewhitelist' => [
    'http://refeds.org/category/research-and-scholarship',
],
```

---

## Configuration Examples

### Example 1: MDQ Only (Simple Setup)

**Use Case:** Small SP, only need InCommon federation access

```php
// config/config.php
'metadata.sources' => [
    ['type' => 'flatfile'],  // For local test IdPs
],

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
            'cron' => ['hourly'],
            'sources' => [
                [
                    'type' => 'mdq',
                    'baseURL' => 'https://mdq.incommon.org/entities/',
                    'certificates' => ['inc-md-cert-mdq.pem'],
                    'cachedir' => 'cache/mdq/',
                    'cachelength' => 86400,
                ],
            ],
            'outputDir' => 'cache/mdq/',
            'outputFormat' => 'flatfile',
        ],
    ],
];
```

### Example 2: PDO Only (Campus IdPs)

**Use Case:** Managing specific campus/partner IdPs

```php
// config/config.php
'metadata.sources' => [
    ['type' => 'pdo'],
],

'database.dsn' => 'mysql:host=mysql;dbname=simplesaml',
'database.username' => 'simplesaml',
'database.password' => 'simplesamlpass',
```

```bash
# Add IdPs via script
./simplesaml exec simplesamlphp php add-idp.php <<EOF
{
    "entityid": "https://idp.ucdavis.edu",
    "name": {"en": "UC Davis"},
    "SingleSignOnService": [...]
}
EOF
```

### Example 3: MDQ + PDO (Recommended for Production)

**Use Case:** InCommon federation + custom campus IdPs

```php
// config/config.php
'metadata.sources' => [
    ['type' => 'flatfile'],  // Priority 1: Static overrides
    ['type' => 'pdo'],       // Priority 2: Custom managed
    // MDQ configured separately in module_metarefresh.php
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
            'expireAfter' => 60*60*24*7,
        ],
    ],
];
```

### Example 4: Multiple Federations

**Use Case:** InCommon + eduGAIN + Custom

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
                ],
            ],
            'outputDir' => 'cache/mdq-incommon/',
            'outputFormat' => 'flatfile',
        ],

        'edugain' => [
            'cron' => ['daily'],
            'sources' => [
                [
                    'type' => 'mdq',
                    'baseURL' => 'https://mdq.edugain.org/entities/',
                    'certificates' => ['edugain-signer.pem'],
                    'cachedir' => 'cache/mdq-edugain/',
                ],
            ],
            'outputDir' => 'cache/mdq-edugain/',
            'outputFormat' => 'flatfile',
        ],
    ],
];
```

---

## Auto-Refresh Mechanisms

### MDQ Auto-Refresh (Built-in)

MDQ has **automatic refresh** based on multiple signals:

**1. Cache TTL (Time-To-Live)**
```php
'cachelength' => 86400,  // 24 hours in seconds
```
- After TTL expires, next authentication triggers re-fetch
- No manual intervention needed
- Default: 24 hours

**2. HTTP Cache-Control Headers**
```http
Cache-Control: max-age=86400
```
- MDQ server tells SimpleSAMLphp how long to cache
- SimpleSAMLphp respects this value
- Overrides `cachelength` if shorter

**3. SAML validUntil Attribute**
```xml
<EntityDescriptor validUntil="2024-11-05T12:00:00Z">
```
- Metadata self-declares expiration
- SimpleSAMLphp checks on each use
- Triggers refresh if expired

**4. Conditional GET (If-None-Match)**
```http
GET /entities/abc123
If-None-Match: "def456"

HTTP/1.1 304 Not Modified
```
- SimpleSAMLphp sends ETag from cache
- Server returns 304 if unchanged
- Saves bandwidth and processing

### Refresh Timing Examples

**Scenario 1: Normal Operation**
```
Day 1, 00:00: User authenticates with Stanford
              → MDQ query → Cache saved (expires Day 2, 00:00)

Day 1, 12:00: User authenticates again
              → Read from cache (still valid)

Day 2, 00:01: Cache expired
              → User authenticates
              → MDQ query → Update cache
```

**Scenario 2: Metadata Updated**
```
Day 1, 00:00: Cache saved
              validUntil: Day 8, 00:00
              Cache TTL: Day 2, 00:00

Day 2, 00:01: Cache TTL expired
              → MDQ query
              → Server: 304 Not Modified
              → Extend cache TTL → Day 3, 00:00

Day 7, 00:00: validUntil approaching
              → MDQ query
              → Server: 200 OK with new metadata
              → Update cache
```

### Optional Cron-Based Pre-Warming

**Why Pre-Warm?**
- Avoid first-user latency (MDQ query delay)
- Keep frequently-used IdPs fresh
- Reduce authentication time

**Configuration:**
```php
'cron' => ['hourly'],  // or 'daily', 'weekly'
```

**How It Works:**
```bash
# SimpleSAMLphp cron runs hourly
php bin/cron.php -t hourly

# For each MDQ set:
1. Read all cached files in cache/mdq-incommon/
2. Check if any are close to expiration
3. Pre-emptively refresh those metadata
4. Update cache before users need them
```

**Cron Schedule Options:**
```php
'cron' => ['hourly'],   // Every hour
'cron' => ['daily'],    // Once per day
'cron' => ['weekly'],   // Once per week
'cron' => ['frequent'], // Every 15 minutes (not recommended for MDQ)
```

**When to Use Cron:**
- ✅ High-traffic SP (many users)
- ✅ Want zero authentication delay
- ✅ Predictable IdP set (same 10-20 IdPs)
- ❌ Low-traffic SP (few users)
- ❌ Unpredictable IdP usage

### PDO Refresh (Manual)

PDO metadata **does not auto-refresh** - you must trigger updates:

**Option 1: RSS Module API**
```bash
curl -X POST http://localhost:8080/sp/module.php/rss/metadata-refresh \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"entityData": "{...}"}'
```

**Option 2: Direct Database Write**
```php
$pdo = new MetaDataStorageHandlerPdo([]);
$pdo->addEntry($entityId, 'saml20-idp-remote', $updatedMetadata);
```

**Option 3: Metarefresh Module (Bulk)**
```php
// config/module_metarefresh.php
'sets' => [
    'campus-idps' => [
        'cron' => ['daily'],
        'sources' => [
            [
                'src' => 'https://metadata.campus.edu/idps.xml',
                'certificates' => ['campus-signing.pem'],
            ],
        ],
        'outputFormat' => 'pdo',  // Write to database
    ],
],
```

---

## Recommended Architecture

### For Most Organizations

```
┌─────────────────────────────────────────────────────────┐
│  Static/Override IdPs (Rare)                            │
│  Source: metadata/saml20-idp-remote.php                 │
│  Use Case: Testing, temporary overrides                 │
└─────────────────────────────────────────────────────────┘
                          Priority 1 ↓

┌─────────────────────────────────────────────────────────┐
│  Custom Campus IdPs (Your Control)                      │
│  Source: MySQL PDO (saml20_idp_remote table)            │
│  Use Case: Campus IdPs, partners, custom configs        │
│  Refresh: Manual (via script/API)                       │
└─────────────────────────────────────────────────────────┘
                          Priority 2 ↓

┌─────────────────────────────────────────────────────────┐
│  InCommon Federation (3000+ IdPs)                       │
│  Source: MDQ (cache/mdq-incommon/)                      │
│  Use Case: All InCommon members                         │
│  Refresh: Automatic (24hr TTL)                          │
└─────────────────────────────────────────────────────────┘
                          Priority 3 ↓

                    Metadata Found → Authenticate
```

### Configuration Summary

```php
// config/config.php
'metadata.sources' => [
    ['type' => 'flatfile'],  // Priority 1
    ['type' => 'pdo'],       // Priority 2
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

### Directory Structure

```
/var/www/html/simplesamlphp/
├── cache/
│   └── mdq-incommon/
│       ├── 8d9c4ba5e8f1234567890abcdef.cached  # Stanford
│       ├── 1a2b3c4d5e6f7890abcdef123456.cached  # UC Davis
│       └── ...
├── cert/
│   ├── inc-md-cert-mdq.pem                      # InCommon signing cert
│   └── server_2024_self_signed.pem              # Your SP cert
├── config/
│   ├── config.php                               # Main config
│   └── module_metarefresh.php                   # MDQ config
├── metadata/
│   ├── saml20-idp-remote.php                    # Static/override IdPs
│   └── saml20-sp-remote.php
└── [database: simplesaml]
    └── saml20_idp_remote                        # Custom managed IdPs
```

---

## Migration Path

### Current State: RSS Module + PDO

If you currently have:
- RSS module enabled
- External client-config service
- Metadata written via RSS API

### Goal: Simplify to MDQ + Direct PDO

**Step 1: Assess Current Metadata**

```bash
# Check what's in database
docker exec mysql mysql -u simplesaml -psimplesamlpass simplesaml \
  -e "SELECT entity_id, entity_type FROM saml20_idp_remote;"

# Check what's in flatfiles
ls -la metadata/saml20-idp-remote.php

# Categorize:
# - InCommon IdPs → Can use MDQ instead
# - Campus IdPs → Keep in PDO
# - Partner IdPs → Keep in PDO or use MDQ if in InCommon
```

**Step 2: Set Up MDQ for InCommon**

```bash
# Download InCommon certificate
curl -o cert/inc-md-cert-mdq.pem \
  https://md.incommon.org/certs/inc-md-cert-mdq.pem

# Create cache directory
mkdir -p cache/mdq-incommon
chown www-data:www-data cache/mdq-incommon
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

**Step 3: Test MDQ Access**

```bash
# Test authentication with InCommon IdP
# Visit: http://localhost:8080/sp/
# Select an InCommon IdP (e.g., Stanford)
# Verify authentication works

# Check MDQ cache was created
ls -la cache/mdq-incommon/

# Check logs
tail -f log/simplesamlphp.log | grep -i mdq
```

**Step 4: Migrate Custom IdPs to Direct PDO Access**

Create a simple metadata management script:

```php
<?php
// bin/manage-metadata.php
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

function removeIdP(string $entityId): bool {
    $db = SimpleSAML\Database::getInstance();
    $db->write(
        "DELETE FROM saml20_idp_remote WHERE entity_id = :entity_id",
        ['entity_id' => $entityId]
    );
    return true;
}

function listIdPs(): array {
    $pdo = new MetaDataStorageHandlerPdo([]);
    return $pdo->getMetadataSet('saml20-idp-remote');
}

// CLI usage
if (php_sapi_name() === 'cli') {
    $action = $argv[1] ?? '';

    switch ($action) {
        case 'add':
            $metadata = json_decode($argv[2], true);
            addIdP($metadata);
            echo "Added {$metadata['entityid']}\n";
            break;

        case 'remove':
            removeIdP($argv[2]);
            echo "Removed {$argv[2]}\n";
            break;

        case 'list':
            $idps = listIdPs();
            foreach ($idps as $entityId => $metadata) {
                echo "$entityId\n";
            }
            break;
    }
}
```

**Step 5: Remove RSS Module (Optional)**

If you no longer need the REST API:

```php
// config/config.php
'module.enable' => [
    // 'rss' => true,  // Comment out or remove
    'metarefresh' => true,
    'cron' => true,
],
```

```bash
# Restart services
./simplesaml restart
```

**Step 6: Update External Services**

If external services were calling RSS API, update them to:

```python
# Instead of:
# POST /sp/module.php/rss/metadata-refresh

# Use direct database access:
import phpserialize
import mysql.connector

def add_idp_metadata(metadata):
    conn = mysql.connector.connect(
        host='mysql',
        user='simplesaml',
        password='simplesamlpass',
        database='simplesaml'
    )

    entity_data = phpserialize.dumps(metadata).decode('utf-8')

    cursor = conn.cursor()
    cursor.execute("""
        INSERT INTO saml20_idp_remote (entity_id, entity_data, entity_type)
        VALUES (%s, %s, %s)
        ON DUPLICATE KEY UPDATE entity_data = VALUES(entity_data)
    """, (metadata['entityid'], entity_data, 'saml20-idp-remote'))

    conn.commit()
```

**Step 7: Update config/config.php**

Final configuration:

```php
// config/config.php
'metadata.sources' => [
    ['type' => 'flatfile'],  // For static/override
    ['type' => 'pdo'],       // For custom managed
    // MDQ configured in module_metarefresh.php
],

'database.dsn' => 'mysql:host=mysql;dbname=simplesaml',
'database.username' => 'simplesaml',
'database.password' => 'simplesamlpass',

'module.enable' => [
    'metarefresh' => true,
    'cron' => true,
    // 'rss' => false,  // No longer needed
],
```

### Testing Checklist

- [ ] InCommon IdP authentication works (test with Stanford, UCLA, etc.)
- [ ] Custom campus IdP authentication works
- [ ] MDQ cache directory created and populated
- [ ] PDO metadata visible in database
- [ ] Logs show MDQ queries for InCommon IdPs
- [ ] Logs show database reads for custom IdPs
- [ ] No errors in simplesamlphp.log
- [ ] Authentication latency acceptable (< 1 second)

---

## Conclusion

### Key Takeaways

1. **MDQ and PDO are complementary, not conflicting**
   - MDQ: On-demand federation metadata (InCommon)
   - PDO: Local storage for custom metadata

2. **RSS Module is optional**
   - Only needed for REST API abstraction
   - Direct PDO access is simpler for most cases

3. **MDQ auto-refreshes automatically**
   - TTL-based cache expiration
   - No manual intervention needed
   - Optional cron for pre-warming

4. **Recommended architecture:**
   - Use MDQ for large federations (InCommon, eduGAIN)
   - Use PDO for custom/managed IdPs
   - Use flatfile for static/override cases

5. **InCommon is migrating to MDQ**
   - Legacy aggregate retires January 20, 2025
   - MDQ is the supported method going forward
   - Configure now to avoid disruption

### Next Steps

For immediate implementation:

1. Download InCommon MDQ certificate
2. Configure `module_metarefresh.php` for MDQ
3. Test authentication with InCommon IdP
4. Migrate custom IdPs to direct PDO management
5. (Optional) Remove RSS module if not needed

### Resources

**Official Documentation:**
- [SimpleSAMLphp Metarefresh Module](https://simplesamlphp.org/docs/contrib_modules/metarefresh/simplesamlphp-automated_metadata.html)
- [InCommon MDQ Service](https://spaces.at.internet2.edu/display/MDQ/)
- [SAML Metadata Query Protocol Spec](https://github.com/metadataquery/md-query)

**InCommon Resources:**
- MDQ Service: https://mdq.incommon.org/
- Signing Certificate: https://md.incommon.org/certs/inc-md-cert-mdq.pem
- Configuration Guides: https://spaces.at.internet2.edu/display/federation/

**SimpleSAMLphp PDO:**
- [Database Configuration](https://simplesamlphp.org/docs/stable/simplesamlphp-install.html#database-configuration)
- [Metadata Sources](https://simplesamlphp.org/docs/stable/simplesamlphp-metadata.html)

---

## Appendix: Quick Reference

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

### PDO Direct Access Template

```php
<?php
require_once '/var/www/html/simplesamlphp/src/_autoload.php';

use SimpleSAML\Metadata\MetaDataStorageHandlerPdo;

$metadata = [
    'entityid' => 'https://idp.example.edu',
    'SingleSignOnService' => [
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect',
            'Location' => 'https://idp.example.edu/sso',
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
$pdo->addEntry($metadata['entityid'], 'saml20-idp-remote', $metadata);
```

### Verification Commands

```bash
# Check MDQ cache
ls -la cache/mdq-incommon/

# Check PDO database
docker exec mysql mysql -u simplesaml -psimplesamlpass simplesaml \
  -e "SELECT entity_id FROM saml20_idp_remote;"

# Test MDQ query manually
curl https://mdq.incommon.org/entities/$(echo -n "https://idp.stanford.edu" | sha1sum | cut -d' ' -f1)

# Check SimpleSAMLphp logs
tail -f log/simplesamlphp.log | grep -i "mdq\|metadata"

# Verify InCommon certificate
openssl x509 -in cert/inc-md-cert-mdq.pem -text -noout
```

---

**End of Research Document**

**Last Updated**: 2025-10-29
**Author**: Research conducted for SimpleSAMLphp v2.4.2 deployment
**Status**: Complete and verified
