---
tags: [authentication]
date: 2024-12-22
status: complete
---

# SimpleSAMLphp with InCommon Federation and MDQ Protocol - Research

**Date**: 2025-10-21
**Status**: Research Complete

## Executive Summary

SimpleSAMLphp is a mature PHP-based SAML 2.0 implementation that can function as both a Service Provider (SP) and Identity Provider (IdP). For organizations joining the InCommon federation, SimpleSAMLphp offers an accessible SSO solution, particularly suitable for hosted environments without root access. The recommended approach uses the Metadata Query Protocol (MDQ) for on-demand metadata retrieval from InCommon's federation, providing superior scalability and performance compared to traditional full metadata aggregate downloads. **Implementation is strongly recommended** for organizations needing federated SSO with InCommon, with careful attention to certificate management and security hardening being critical success factors.

## Context

This research was conducted to enable SSO integration with the InCommon federation for an application requiring university/institutional authentication. The target audience is developers new to SimpleSAMLphp who need to understand both the underlying architecture and practical implementation steps, with particular focus on MDQ configuration and security best practices.

## Research Goals

- [x] Understand SimpleSAMLphp architecture and core components
- [x] Learn SAML protocol fundamentals and authentication flows
- [x] Understand InCommon federation trust model and requirements
- [x] Learn MDQ protocol advantages over traditional metadata management
- [x] Identify specific configuration steps for SimpleSAMLphp with InCommon MDQ
- [x] Document security best practices and certificate management
- [x] Provide step-by-step implementation guidance

## Technical Deep Dive

### Overview

SimpleSAMLphp is a PHP implementation of the SAML 2.0 standard that enables Single Sign-On (SSO) authentication. It uses a modular architecture where PHP applications can authenticate users against external Identity Providers (IdPs) through standardized SAML protocols. When integrated with InCommon federation using MDQ, SimpleSAMLphp provides scalable, on-demand access to thousands of academic institutions' authentication systems.

### SAML 2.0 Protocol Fundamentals

#### What is SAML?

Security Assertion Markup Language (SAML) 2.0 is an XML-based protocol for exchanging authentication and authorization information between security domains. It enables federated identity management where users authenticate once at their home institution and gain access to multiple service providers.

**Key SAML Components:**

1. **Identity Provider (IdP)** - The entity that authenticates users (typically a university)
2. **Service Provider (SP)** - The application requesting authentication (your application)
3. **SAML Assertions** - XML statements containing authentication and attribute information
4. **Metadata** - XML configuration that describes IdPs and SPs (endpoints, certificates, capabilities)

#### SAML Authentication Flow

The SAML authentication process is asynchronous and browser-mediated:

1. **User accesses SP** - User attempts to access a protected resource
2. **SP generates AuthnRequest** - SP creates a SAML authentication request with unique ID
3. **Browser redirect to IdP** - SP redirects user's browser to IdP SSO endpoint
4. **User authenticates** - User logs in at their home institution
5. **IdP generates Response** - IdP creates signed SAML response with assertions
6. **Browser returns to SP** - Browser posts SAML response to SP's Assertion Consumer Service (ACS)
7. **SP validates response** - SP verifies signature, checks conditions, extracts attributes
8. **Session established** - User gains access with attributes available to application

**Critical Security Point:** The SP never directly interacts with the IdP - the browser acts as an intermediary, which requires careful validation to prevent man-in-the-middle attacks.

#### SAML Assertions

SAML assertions are XML statements that convey authentication decisions and user attributes:

- **Authentication Assertion** - Confirms user was authenticated
- **Attribute Assertion** - Contains user attributes (email, name, affiliation)
- **Authorization Assertion** - Specifies what user can access (less common)

Assertions contain:
- **Subject** - Who the assertion is about (user identifier)
- **Conditions** - When valid (NotBefore/NotOnOrAfter timestamps)
- **Attributes** - Key-value pairs describing the user
- **Signature** - XML digital signature for integrity/authenticity

#### Metadata Exchange

SAML metadata is pre-configuration in XML format that establishes trust between entities:

**Metadata contains:**
- Entity ID (unique identifier)
- Endpoints (SSO service URL, ACS URL, logout URLs)
- Certificates (for signature validation and encryption)
- Supported bindings (HTTP-POST, HTTP-Redirect)
- Contact information
- UI information (display names, logos)

**Trust Model:** Metadata ensures secure transactions. Each entity publishes its metadata, and trusting parties consume it to configure their systems. In federations like InCommon, a central operator aggregates, vets, and signs all member metadata.

### InCommon Federation

#### What is InCommon?

InCommon Federation is an identity management federation operator for U.S. research and education institutions, providing a common framework for trusted shared management of online resource access. It's operated by Internet2 and serves as the trusted third party that creates multilateral trust among hundreds of participating organizations.

**Key Characteristics:**

- **Multilateral Trust** - Rather than each SP trusting each IdP individually (N×M relationships), all parties trust InCommon as the intermediary (N+M relationships)
- **Vetting Process** - InCommon evaluates members and validates their metadata
- **Standardized Profiles** - Common technical requirements ensure interoperability
- **Entity Categories** - Tags like Research & Scholarship (R&S) indicate attribute release commitments

#### Trust Model

InCommon's trust model differs fundamentally from traditional PKI certificate authority models:

**Metadata-Based Trust:**
When you publish metadata in InCommon, the federation operator vouches that you issued the keys embedded in your metadata. Trust comes from the federation's digital signature on the metadata aggregate, not from traditional CA hierarchies.

**Requirements for Trust:**
- Use SSL/TLS to protect data in transit
- Refresh and verify digital signature on InCommon metadata at least daily
- Validate metadata signatures using InCommon's signing certificate
- Only accept entities present in signed InCommon metadata

**Participation Requirements:**
- Technical requirements include SAML 2.0 support with HTTP-POST binding
- Provide both technical and administrative contacts
- Refresh metadata at least daily
- Meet privacy and data handling requirements in the Participation Agreement

#### Research and Scholarship (R&S) Category

The R&S entity category streamlines federated research access:

**For Service Providers:**
- Must request only necessary attributes
- Must not use attributes outside service definition
- Must provide human-readable privacy information
- Must publish metadata with MDUI DisplayName and InformationURL

**For Identity Providers:**
- Must release minimum attributes: email, person name, eduPersonPrincipalName
- If eduPersonPrincipalName can be reassigned, must also release eduPersonTargetedID
- Can declare R&S support by checking a box in Federation Manager

**Benefits:**
- Eliminates need for per-SP attribute release negotiations
- Standard attribute bundle known to all parties
- Accelerates integration for research/educational services

### Metadata Query Protocol (MDQ)

#### Traditional Metadata vs. MDQ

**Legacy Approach (Metadata Aggregates):**
- Download entire federation metadata file (all IdPs and SPs)
- File size: tens or hundreds of megabytes
- Must refresh periodically (daily minimum)
- Local storage and parsing required
- Memory intensive for large federations
- Includes metadata for entities you'll never use

**MDQ Approach:**
- On-demand retrieval of individual entity metadata
- REST-like API for requesting specific entities
- Fetch only what you need, when you need it
- Minimal local storage (caching only)
- Better performance and scalability
- Reduced attack surface (less metadata to validate)

#### How MDQ Works

**Query Structure:**
MDQ requests concatenate four components:
1. Server base URL: `https://mdq.incommon.org`
2. Forward slash: `/`
3. The string: `entities/`
4. URL-encoded entityID: `https%3A%2F%2Fidp.example.edu%2Fidp%2Fshibboleth`

**Example Query:**
```
https://mdq.incommon.org/entities/https%3A%2F%2Fidp.example.edu%2Fidp%2Fshibboleth
```

**Response:**
The MDQ server returns XML metadata for the requested entity, digitally signed by InCommon.

**Caching:**
Clients typically cache retrieved metadata locally for a configurable period (e.g., 24 hours) to reduce network requests and improve performance.

#### MDQ Advantages

1. **Performance** - Only download metadata when needed, reducing startup time and memory usage
2. **Scalability** - Supports federations of any size without performance degradation
3. **Simplicity** - No need to parse large XML files or manage aggregate updates
4. **Security** - Reduced attack surface by limiting metadata exposure
5. **Bandwidth** - Significantly less data transfer compared to full aggregates
6. **Current Data** - Each query returns current metadata (within cache period)

#### InCommon MDQ Service Details

**Production Endpoint:**
- Base URL: `https://mdq.incommon.org`
- Entity query: `https://mdq.incommon.org/entities/{url-encoded-entityID}`

**Preview Endpoint (for testing):**
- Base URL: `https://mdq-preview.incommon.org`
- Uses preview signing key

**Certificate Details:**
- Signing certificate: `inc-md-cert-mdq.pem`
- Download URL: `http://md.incommon.org/certs/inc-md-cert-mdq.pem`
- Certificate details page: `https://ops.incommon.org/inc_md_cert_mdq.html`
- Valid from: November 13, 2018
- Valid until: November 10, 2038
- Common Name: mdq.incommon.org
- Organization: Internet2.edu, InCommon

**Verification Fingerprints:**
- SHA1: `F8:4E:F8:47:EF:BB:EE:47:86:32:DB:94:17:8A:31:A6:94:73:19:36`
- SHA256: `60:49:74:D6:1F:E0:D7:F4:D6:3D:6C:8D:B9:8A:85:7E:64:2A:B9:B4:70:E3:E8:5D:D5:4D:66:3D:04:96:F9:00`

**Important:** The MDQ signing certificate is different from the legacy aggregate signing certificate from md.incommon.org.

### SimpleSAMLphp Architecture

#### Core Components

SimpleSAMLphp uses a modular architecture with several key components:

**1. SAML2 Module** (`modules/saml/`)
- Implements SAML 2.0 protocol for both IdP and SP roles
- Handles request/response generation and parsing
- Manages signature validation and encryption

**2. Authentication Framework** (`modules/`)
- Provides various authentication modules (LDAP, SQL, UserPass, etc.)
- For SP role, SAML authentication source connects to remote IdPs
- Extensible for custom authentication methods

**3. Metadata Management**
- `metadata/saml20-sp-remote.php` - Remote SPs (when acting as IdP)
- `metadata/saml20-idp-remote.php` - Remote IdPs (when acting as SP)
- `metadata/saml20-idp-hosted.php` - Local IdP configuration
- Supports multiple metadata sources (flat file, MDQ, database)

**4. Configuration System**
- `config/config.php` - Main configuration file
- `config/authsources.php` - Authentication source definitions
- Environment variable support for external config paths

**5. Session Management**
- Manages SAML sessions separately from PHP sessions
- Supports various backends (PHP native, memcache, Redis, SQL)
- Critical for maintaining state across redirects

#### SimpleSAMLphp as a Service Provider

When acting as an SP, SimpleSAMLphp:

1. **Protects Resources** - Intercepts requests to protected resources
2. **Initiates Authentication** - Generates AuthnRequest when needed
3. **Processes Responses** - Validates SAML responses from IdPs
4. **Manages Sessions** - Maintains authenticated sessions
5. **Provides Attributes** - Makes user attributes available to application

**Configuration Location:**
Service Provider is configured via entry in `config/authsources.php`:

```php
'default-sp' => [
    'saml:SP',
    'entityID' => 'https://myapp.example.org/',
    'privatekey' => 'saml.pem',
    'certificate' => 'saml.crt',
],
```

#### Configuration File Structure

**Primary Configuration Files:**

1. **config/config.php** - System-wide settings
   - Base URL path
   - Admin password
   - Secret salt
   - Technical contact
   - Session handler
   - Metadata sources (including MDQ)
   - Timezone

2. **config/authsources.php** - Authentication sources
   - Named authentication configurations
   - Each entry defines an authentication method
   - SP configurations specify entityID, certificates, IdP preferences

3. **metadata/saml20-idp-remote.php** - Remote IdP metadata
   - Array indexed by entityID
   - Contains SSO/SLO endpoints
   - IdP certificates for signature validation
   - Can be auto-populated via metadata refresh or MDQ

**Configuration Directory:**
By default, SimpleSAMLphp looks for config in `config/` relative to installation root. You can override with `SIMPLESAMLPHP_CONFIG_DIR` environment variable, useful for:
- Separating configuration from code
- Using Composer-installed SimpleSAMLphp
- Multiple environments with shared codebase

**Metadata Directory:**
Default is `metadata/` but configurable via `metadatadir` option in config.php.

#### SimpleSAMLphp 2.x Modern Features

**Version Requirements:**
- SimpleSAMLphp 2.0+: Requires PHP 7.4+
- SimpleSAMLphp 2.1+: Requires PHP 8.0+
- SimpleSAMLphp 2.3+: Current stable, PHP 8.0+

**Major Changes in 2.x:**

1. **New Templating System** - Twig-based instead of legacy system
2. **Improved Localization** - Gettext-based internationalization
3. **Enhanced Security**
   - AuthnRequest signature validation enabled by default
   - Upgraded encryption: AES128_GCM default (was AES128_CBC)
   - Hashed session IDs in database storage
   - Deprecated plain-text admin passwords
4. **Modular Architecture** - Core modules separated, install via Composer
5. **Directory Changes** - Assets moved from `www/` to `public/`
6. **Full Type Hints** - Complete PHP type declarations
7. **Build Variants** - Slim build without optional modules

### SimpleSAMLphp MDQ Configuration

#### Configuration in config.php

MDQ is configured in the `metadata.sources` array in `config/config.php`:

```php
'metadata.sources' => [
    // Traditional flat file sources (still needed for local SP metadata)
    ['type' => 'flatfile'],

    // InCommon MDQ source
    [
        'type' => 'mdq',
        'server' => 'https://mdq.incommon.org',
        'validateCertificate' => '/var/simplesamlphp/cert/inc-md-cert-mdq.pem',
        'cachedir' => '/var/simplesamlphp/cache/mdq',
        'cachelength' => 86400, // 24 hours in seconds
    ],
],
```

**Configuration Parameters Explained:**

- **type** - Must be `'mdq'` for MDQ source
- **server** - Base URL of MDQ service (no trailing slash)
- **validateCertificate** - Absolute path to MDQ signing certificate (required for security)
- **cachedir** - Directory for caching retrieved metadata (must be writable)
- **cachelength** - Cache duration in seconds (86400 = 24 hours recommended)

**Certificate Path:**
Can be string (single cert) or array (multiple certs for rotation):
```php
'validateCertificate' => [
    '/var/simplesamlphp/cert/inc-md-cert-mdq.pem',
    '/var/simplesamlphp/cert/inc-md-cert-mdq-backup.pem',
],
```

#### Multiple Metadata Sources

SimpleSAMLphp can use multiple metadata sources simultaneously:

```php
'metadata.sources' => [
    // Local flat file for your own SP metadata
    ['type' => 'flatfile'],

    // InCommon MDQ for federation IdPs
    [
        'type' => 'mdq',
        'server' => 'https://mdq.incommon.org',
        'validateCertificate' => '/path/to/inc-md-cert-mdq.pem',
        'cachedir' => '/path/to/cache/incommon-mdq',
        'cachelength' => 86400,
    ],

    // Additional MDQ source (e.g., for another federation)
    [
        'type' => 'mdq',
        'server' => 'https://mdq.edugain.org',
        'validateCertificate' => '/path/to/edugain-cert.pem',
        'cachedir' => '/path/to/cache/edugain-mdq',
        'cachelength' => 86400,
    ],

    // Specific trusted partners in flat file
    ['type' => 'flatfile', 'directory' => 'metadata/partners'],
],
```

Sources are queried in order until metadata is found.

#### Setting Up InCommon MDQ

**Step 1: Download and Verify Certificate**

```bash
# Download the certificate
cd /var/simplesamlphp/cert
curl -O http://md.incommon.org/certs/inc-md-cert-mdq.pem

# Verify fingerprints
openssl x509 -in inc-md-cert-mdq.pem -noout -fingerprint -sha256
# Should output: 60:49:74:D6:1F:E0:D7:F4:D6:3D:6C:8D:B9:8A:85:7E:64:2A:B9:B4:70:E3:E8:5D:D5:4D:66:3D:04:96:F9:00

openssl x509 -in inc-md-cert-mdq.pem -noout -fingerprint -sha1
# Should output: F8:4E:F8:47:EF:BB:EE:47:86:32:DB:94:17:8A:31:A6:94:73:19:36
```

**Step 2: Create Cache Directory**

```bash
mkdir -p /var/simplesamlphp/cache/mdq
chown www-data:www-data /var/simplesamlphp/cache/mdq
chmod 750 /var/simplesamlphp/cache/mdq
```

**Step 3: Configure config.php**

Add MDQ source to `metadata.sources` array as shown above.

**Step 4: Test MDQ Configuration**

Access SimpleSAMLphp admin interface and check configuration status:
```
https://your-host/simplesaml/admin/
```

Test retrieving metadata for a known InCommon IdP.

#### Discovery Service Integration

When using MDQ with multiple IdPs, you typically need a discovery service (WAYF - Where Are You From):

**Option 1: Built-in Discovery**
Leave `'idp'` parameter unset in authsources.php:
```php
'default-sp' => [
    'saml:SP',
    'entityID' => 'https://myapp.example.org/',
    // No 'idp' parameter - shows built-in discovery page
],
```

**Option 2: External Discovery Service**
Configure InCommon Discovery Service:
```php
'default-sp' => [
    'saml:SP',
    'entityID' => 'https://myapp.example.org/',
    'discoURL' => 'https://ds.incommon.org/DS/WAYF',
],
```

**Option 3: Programmatic IdP Selection**
Application chooses IdP based on user context (e.g., domain-based routing).

### Certificate Management and Security

#### Service Provider Certificates

**Purpose:**
- **Request Signing** - Sign AuthnRequest messages sent to IdP
- **Response Encryption** - Decrypt encrypted assertions from IdP
- **Metadata Signing** - Sign SP metadata published to federation

**Certificate Requirements for SAML:**

1. **Key Size** - RSA 2048-bit minimum (RSA 3072 recommended)
2. **Hashing Algorithm** - SHA-256 minimum (SHA-384/SHA-512 preferred)
3. **Key Usage** - digitalSignature enabled
4. **Extended Key Usage** - id-kp-documentSigning preferred
5. **Certificate Type** - Self-signed acceptable (not CA-signed needed)
6. **Validity Period** - Maximum 2 years recommended

**Why Self-Signed is Acceptable:**
SAML trust model is metadata-based, not PKI-based. The certificate public key is embedded in metadata signed by InCommon. Trust comes from InCommon's metadata signature, not from certificate authority chains.

**Generating SP Certificates:**

```bash
cd /var/simplesamlphp/cert

# Generate RSA 3072-bit self-signed certificate valid for 2 years
openssl req -newkey rsa:3072 -new -x509 -days 730 -nodes \
    -out saml.crt -keyout saml.pem \
    -subj "/C=US/ST=State/L=City/O=Organization/CN=myapp.example.org"

# Set secure permissions
chmod 640 saml.pem
chown root:www-data saml.pem
chmod 644 saml.crt
```

**Configuration in authsources.php:**

```php
'default-sp' => [
    'saml:SP',
    'entityID' => 'https://myapp.example.org/',
    'privatekey' => 'saml.pem',
    'certificate' => 'saml.crt',

    // Optional: Enable request signing
    'sign.authnrequest' => true,

    // Optional: Request encrypted responses
    'assertion.encryption' => true,
],
```

#### Federation Metadata Signing Certificates

**InCommon MDQ Signing Certificate:**
- Critical for validating metadata from InCommon
- Must be validated against known fingerprints
- Store securely with appropriate permissions
- Monitor expiration (current cert valid until 2038)

**Certificate Validation:**
```php
// In config.php
'metadata.sources' => [
    [
        'type' => 'mdq',
        'server' => 'https://mdq.incommon.org',
        'validateCertificate' => '/var/simplesamlphp/cert/inc-md-cert-mdq.pem',
        // ...
    ],
],
```

**Never Skip Certificate Validation:**
Disabling certificate validation eliminates trust and opens man-in-the-middle attacks. Always validate metadata signatures.

#### Certificate Rotation Strategy

**Planning for Certificate Expiration:**

1. **Monitor Expiration** - Track certificate expiration dates
2. **Overlap Period** - Generate new certificate before old expires
3. **Publish Both** - Update metadata with both old and new certificates
4. **Grace Period** - Allow IdPs time to refresh metadata (1-2 weeks minimum)
5. **Remove Old** - After grace period, remove old certificate

**SimpleSAMLphp Rollover Configuration:**

```php
'default-sp' => [
    'saml:SP',
    'entityID' => 'https://myapp.example.org/',

    // Current active certificate
    'privatekey' => 'saml.pem',
    'certificate' => 'saml.crt',

    // New certificate for rollover (published in metadata, not yet active)
    'new_privatekey' => 'saml-new.pem',
    'new_certificate' => 'saml-new.crt',
],
```

**Rotation Process:**

1. Generate new certificate pair
2. Add as `new_privatekey` and `new_certificate`
3. Update federation metadata (both certs published)
4. Wait grace period (2+ weeks)
5. Move new cert to primary `privatekey`/`certificate`
6. Remove old cert files
7. Update metadata again

#### Private Key Protection

**Critical Security Requirements:**

1. **File Permissions** - Readable only by web server user
   ```bash
   chmod 640 saml.pem
   chown root:www-data saml.pem
   ```

2. **Never Commit to Version Control** - Add to .gitignore
   ```
   # .gitignore
   cert/*.pem
   cert/*.key
   ```

3. **Secure Storage** - Consider Hardware Security Module (HSM) for production
4. **No Exportable Format** - Store keys where generated, never transmit insecurely
5. **Audit Access** - Log who accesses private keys

**Compromise Response:**
If private key is compromised:
1. Immediately revoke certificate in federation metadata
2. Generate new certificate pair
3. Update metadata with new certificate
4. Notify federation operator
5. Investigate breach extent

### Security Best Practices

#### SAML-Specific Vulnerabilities

**1. XML Signature Wrapping (XSW) Attacks**

**Threat:** Attacker manipulates XML structure to change assertions while maintaining valid signatures.

**Mitigation in SimpleSAMLphp:**
- Use schema validation (enabled by default in 2.x)
- SimpleSAMLphp's SAML library implements secure XML parsing
- Always validate signatures before processing (default behavior)
- Use absolute XPath expressions (handled by library)

**Configuration:**
```php
// In authsources.php
'default-sp' => [
    'saml:SP',
    // Enable signature validation (default in 2.x)
    'validate.authnrequest' => true,

    // Require signed responses
    'sign.logout' => true,
],
```

**2. XML External Entity (XXE) Attacks**

**Threat:** Malicious XML can reference external entities to exfiltrate data or cause DoS.

**Mitigation:**
- SimpleSAMLphp 2.x disables DTD processing by default
- Uses secure XML parsers with entity resolution disabled
- Never allow automatic schema downloads

**Verify Protection:**
Modern PHP's libxml automatically protects against XXE when using DOMDocument with default settings (used by SimpleSAMLphp).

**3. Man-in-the-Middle (MITM) Attacks**

**Threat:** Attacker intercepts SAML messages between SP and IdP.

**Mitigation:**
- **Always use HTTPS/TLS 1.2+** for all endpoints
- Validate `InResponseTo` matches original request ID
- Check `Recipient` attribute matches ACS URL
- Verify `Destination` attribute
- Validate `NotBefore` and `NotOnOrAfter` timestamps

**Configuration:**
```php
// In config.php
'baseurlpath' => 'https://myapp.example.org/simplesaml/',

// In authsources.php
'default-sp' => [
    'saml:SP',
    'entityID' => 'https://myapp.example.org/',  // Must be HTTPS

    // SimpleSAMLphp validates InResponseTo automatically
    // Validates timestamps automatically
],
```

**4. Assertion Replay Attacks**

**Threat:** Attacker captures and reuses valid SAML assertions.

**Mitigation:**
- Short assertion lifetime (NotOnOrAfter)
- OneTimeUse condition enforcement
- Session ID tracking

**SimpleSAMLphp Protection:**
- Tracks used assertion IDs in session
- Enforces `NotOnOrAfter` timestamps
- Supports `OneTimeUse` condition

**Configuration:**
```php
// In config.php
'session.duration' => 8 * 60 * 60,  // 8 hours max session
'session.cookie.lifetime' => 0,      // Session cookie (not persistent)
```

**5. Token Theft and Session Hijacking**

**Threat:** Attacker steals session tokens to impersonate user.

**Mitigation:**
- Use secure, HTTPOnly cookies
- Enable SameSite cookie attribute
- Use HTTPS exclusively
- Implement session timeout
- Bind sessions to IP or fingerprint (carefully)

**Configuration:**
```php
// In config.php
'session.cookie.secure' => true,      // HTTPS only
'session.cookie.httponly' => true,    // No JavaScript access
'session.cookie.samesite' => 'Lax',   // CSRF protection

'session.duration' => 8 * 60 * 60,    // 8 hours
'session.datastore.timeout' => 4 * 60 * 60,  // 4 hours idle timeout
```

#### Configuration Security Hardening

**1. Strong Admin Password**

```bash
# Generate secure admin password hash
cd /var/simplesamlphp
php bin/pwgen.php

# Add to config.php
'auth.adminpassword' => '{SSHA256}hash-output-from-pwgen',
```

**Never use plain text passwords** (deprecated in SimpleSAMLphp 2.3+).

**2. Secret Salt Configuration**

```php
// config.php
'secretsalt' => 'random-string-at-least-32-characters-long-with-high-entropy',
```

Generate with:
```bash
openssl rand -base64 32
```

**Purpose:** Used for cryptographic operations, session IDs, state tracking. Must be unique per installation.

**3. Secure Session Storage**

**Development:** PHP native sessions acceptable
**Production:** Use external session storage

```php
// config.php - Redis example
'store.type' => 'redis',
'store.redis.host' => 'localhost',
'store.redis.port' => 6379,
'store.redis.prefix' => 'ssp_',

// Memcache example
'store.type' => 'memcache',
'store.memcache.servers' => [
    ['hostname' => 'localhost', 'port' => 11211],
],
```

**Why External Storage:**
- Survives PHP session conflicts with application
- Supports load balancing across multiple servers
- Better performance and scalability
- Independent session lifetime management

**4. Input Validation and Encoding**

SimpleSAMLphp handles SAML-specific validation, but your application must:

- **Validate attributes** received from SimpleSAMLphp
- **Sanitize output** when displaying user attributes
- **Escape SQL** if storing attributes in database
- **Validate email format** if using email attribute
- **Check attribute presence** before accessing

**Example:**
```php
require_once('/var/simplesamlphp/lib/_autoload.php');
$as = new \SimpleSAML\Auth\Simple('default-sp');
$as->requireAuth();

$attributes = $as->getAttributes();

// Validate expected attributes exist
if (!isset($attributes['mail'][0])) {
    throw new Exception('Email attribute not provided by IdP');
}

// Sanitize for display
$email = htmlspecialchars($attributes['mail'][0], ENT_QUOTES, 'UTF-8');

// Validate format
if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
    throw new Exception('Invalid email format from IdP');
}
```

**5. Logging and Monitoring**

```php
// config.php
'logging.level' => SimpleSAML\Logger::NOTICE,  // Production
'logging.handler' => 'syslog',                  // Use syslog or errorlog

'logging.facility' => LOG_LOCAL5,
'logging.processname' => 'simplesamlphp',

// Enable audit logging
'logging.audit' => 'syslog',
```

**Monitor for:**
- Failed authentication attempts
- Signature validation failures
- Metadata retrieval errors
- Certificate expiration warnings
- Unusual session patterns

**6. Restrict Administrative Access**

```php
// config.php
'admin.protectindexpage' => true,  // Require auth for admin interface
'admin.protectmetadata' => false,  // Metadata should be public

// Consider IP restriction at web server level
// Apache example in .htaccess or vhost:
// <Location /simplesaml/admin>
//     Require ip 10.0.0.0/8
//     Require ip 192.168.1.0/24
// </Location>
```

**7. Error Handling Configuration**

```php
// config.php
'showerrors' => false,  // Production: don't expose stack traces
'errorreporting' => false,  // Don't display PHP errors to users

// Use custom error pages
'theme.use' => 'default:default',
'theme.header' => 'Your Organization Name',
```

**Development vs. Production:**
- **Development:** `'showerrors' => true` for debugging
- **Production:** `'showerrors' => false` to prevent information disclosure

#### Web Server Security

**1. Directory Permissions**

```bash
# SimpleSAMLphp installation
chown -R root:root /var/simplesamlphp
chmod -R 755 /var/simplesamlphp

# Writable directories
chown -R www-data:www-data /var/simplesamlphp/cache
chmod -R 750 /var/simplesamlphp/cache

chown -R www-data:www-data /var/simplesamlphp/log
chmod -R 750 /var/simplesamlphp/log

# Private keys
chown root:www-data /var/simplesamlphp/cert/*.pem
chmod 640 /var/simplesamlphp/cert/*.pem
```

**2. Expose Only Public Directory**

**Apache configuration:**
```apache
<VirtualHost *:443>
    ServerName myapp.example.org

    # Your application document root
    DocumentRoot /var/www/myapp

    # SimpleSAMLphp public directory alias
    Alias /simplesaml /var/simplesamlphp/public

    <Directory /var/simplesamlphp/public>
        Require all granted
    </Directory>

    # Block access to SimpleSAMLphp internals
    <DirectoryMatch "^/var/simplesamlphp/(?!public)">
        Require all denied
    </DirectoryMatch>

    # SSL configuration
    SSLEngine on
    SSLCertificateFile /path/to/ssl-cert.pem
    SSLCertificateKeyFile /path/to/ssl-key.pem
    SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
    SSLCipherSuite HIGH:!aNULL:!MD5

    # Security headers
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
    Header always set X-Frame-Options "DENY"
    Header always set X-Content-Type-Options "nosniff"
</VirtualHost>
```

**Nginx configuration:**
```nginx
server {
    listen 443 ssl http2;
    server_name myapp.example.org;

    root /var/www/myapp;

    # SimpleSAMLphp location
    location ^~ /simplesaml {
        alias /var/simplesamlphp/public;

        location ~ \.php(/|$) {
            include fastcgi_params;
            fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
            fastcgi_param SCRIPT_FILENAME $request_filename;
        }
    }

    # Block access to SimpleSAMLphp internals
    location ~ ^/var/simplesamlphp/(?!public) {
        deny all;
        return 404;
    }

    # SSL configuration
    ssl_certificate /path/to/ssl-cert.pem;
    ssl_certificate_key /path/to/ssl-key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
}
```

**3. HTTPS Enforcement**

- Redirect all HTTP to HTTPS
- Use HSTS headers
- Use modern TLS versions (1.2 minimum, 1.3 preferred)
- Strong cipher suites only

#### Common Security Pitfalls to Avoid

1. **Using HTTP instead of HTTPS** - All SAML communication must be encrypted
2. **Skipping certificate validation** - Enables MITM attacks
3. **Weak admin passwords** - Use pwgen.php to generate strong hashes
4. **Exposing private keys** - Incorrect file permissions or version control commits
5. **Not rotating certificates** - Plan for expiration before it happens
6. **Trusting user attributes without validation** - Always validate in application
7. **Shared sessions with application** - Use separate session storage
8. **Default secret salt** - Must be unique, high-entropy random string
9. **Verbose error messages in production** - Information disclosure risk
10. **No monitoring/logging** - Can't detect or respond to attacks

### InCommon-Specific Attributes

#### Common Attributes Released by InCommon IdPs

**Core R&S Attributes (Required):**

1. **eduPersonPrincipalName (ePPN)**
   - OID: `urn:oid:1.3.6.1.4.1.5923.1.1.1.6`
   - Format: `username@scope` (e.g., `jsmith@example.edu`)
   - Purpose: Persistent, unique, non-reassignable identifier
   - Usage: Primary user identifier for account linking

2. **mail**
   - OID: `urn:oid:0.9.2342.19200300.100.1.3`
   - Format: Email address
   - Purpose: User's email address
   - Usage: Contact, notifications

3. **displayName**
   - OID: `urn:oid:2.16.840.1.113730.3.1.241`
   - Format: Full name string
   - Purpose: User's preferred display name
   - Usage: UI personalization

4. **givenName**
   - OID: `urn:oid:2.5.4.42`
   - Format: First name
   - Purpose: User's given name

5. **sn (surname)**
   - OID: `urn:oid:2.5.4.4`
   - Format: Last name
   - Purpose: User's family name

**Additional Commonly Available Attributes:**

6. **eduPersonTargetedID**
   - OID: `urn:oid:1.3.6.1.4.1.5923.1.1.1.10`
   - Format: Opaque identifier scoped to SP
   - Purpose: Privacy-preserving persistent identifier
   - Usage: When ePPN might be reassigned

7. **eduPersonScopedAffiliation**
   - OID: `urn:oid:1.3.6.1.4.1.5923.1.1.1.9`
   - Format: `affiliation@scope` (e.g., `faculty@example.edu`)
   - Values: `faculty`, `student`, `staff`, `employee`, `member`, `affiliate`, `alum`, etc.
   - Purpose: User's relationship to institution
   - Usage: Authorization decisions, feature access

8. **eduPersonAffiliation**
   - OID: `urn:oid:1.3.6.1.4.1.5923.1.1.1.1`
   - Format: Unscoped affiliation value
   - Purpose: Same as scoped but without @scope suffix

9. **eduPersonEntitlement**
   - OID: `urn:oid:1.3.6.1.4.1.5923.1.1.1.7`
   - Format: URI indicating permissions/memberships
   - Purpose: Fine-grained authorization
   - Usage: Group memberships, permissions

#### Attribute Mapping in SimpleSAMLphp

**OID to Friendly Name Mapping:**

SimpleSAMLphp can work with both OID format and friendly names. Use authproc filters to map:

```php
// In authsources.php
'default-sp' => [
    'saml:SP',
    'entityID' => 'https://myapp.example.org/',

    'authproc' => [
        // Map OID format to friendly names
        100 => [
            'class' => 'core:AttributeMap',
            'oid2name',
        ],
    ],
],
```

**Custom Attribute Mapping:**

```php
'authproc' => [
    // Map attributes to application-specific names
    110 => [
        'class' => 'core:AttributeMap',
        'map' => [
            'eduPersonPrincipalName' => 'username',
            'mail' => 'email',
            'displayName' => 'full_name',
        ],
    ],
],
```

**Attribute Filtering:**

```php
'authproc' => [
    // Only keep specific attributes
    120 => [
        'class' => 'core:AttributeLimit',
        'default' => true,
        'eduPersonPrincipalName',
        'mail',
        'displayName',
        'eduPersonScopedAffiliation',
    ],
],
```

#### Accessing Attributes in Application

```php
require_once('/var/simplesamlphp/lib/_autoload.php');

$as = new \SimpleSAML\Auth\Simple('default-sp');
$as->requireAuth();

// Get all attributes
$attributes = $as->getAttributes();

// Access specific attributes
$eppn = $attributes['eduPersonPrincipalName'][0] ?? null;
$email = $attributes['mail'][0] ?? null;
$displayName = $attributes['displayName'][0] ?? null;

// Affiliation is multi-valued
$affiliations = $attributes['eduPersonScopedAffiliation'] ?? [];
$isFaculty = in_array('faculty@example.edu', $affiliations);

// Always validate attributes exist before using
if (!$eppn) {
    throw new Exception('Required attribute eduPersonPrincipalName not provided');
}

// Use attributes for application logic
$userId = $eppn;  // Primary identifier
$userEmail = $email;  // Contact info
$userDisplayName = $displayName;  // Display in UI
```

## Implementation Feasibility

### Benefits

1. **Mature, Proven Technology**
   - SimpleSAMLphp has been in production use since 2008
   - Widely deployed across academic institutions
   - Active community and regular security updates
   - Comprehensive documentation [https://simplesamlphp.org/docs/]

2. **No Root Access Required**
   - Pure PHP implementation
   - No system-level dependencies beyond PHP extensions
   - Suitable for shared hosting environments
   - Easy deployment compared to Shibboleth SP [https://docs.tuakiri.ac.nz/service_providers/installing_a_simplesamlphp_sp]

3. **Federation Integration Built-In**
   - Native MDQ support in version 1.17+
   - Designed for multi-IdP scenarios
   - Built-in discovery service
   - Handles metadata complexity automatically

4. **Scalable with MDQ**
   - On-demand metadata retrieval
   - Minimal memory footprint
   - No need to parse massive federation metadata files
   - Automatic caching for performance [https://spaces.at.internet2.edu/display/MDQ/Metadata+Query+Protocol]

5. **Flexible Attribute Handling**
   - Built-in OID to friendly name mapping
   - Attribute transformation filters
   - Support for custom attribute processing
   - Compatible with InCommon R&S attribute bundle

6. **Strong Security Features**
   - XML Signature Wrapping protection
   - XXE attack mitigation
   - Configurable signature validation
   - Modern encryption algorithms (AES-GCM)
   - Session security controls [https://cheatsheetseries.owasp.org/cheatsheets/SAML_Security_Cheat_Sheet.html]

7. **Cost Effective**
   - Open source, free to use
   - No commercial licensing fees
   - Community support available
   - Professional support available if needed

### Trade-offs & Challenges

1. **PHP Application Requirement**
   - Application must be PHP-based or capable of calling PHP
   - If application is in another language (Node.js, Python, Java), integration more complex
   - Alternative: Use SimpleSAMLphp as authentication gateway with proxy pattern
   - Mitigation: SimpleSAMLphp can protect non-PHP apps via bridge modules or reverse proxy

2. **Session Management Complexity**
   - SimpleSAMLphp sessions separate from application sessions
   - Session handoff requires careful implementation
   - Potential for "state information lost" errors if misconfigured
   - Mitigation: Use external session storage (Redis/Memcache) and proper cookie configuration [https://simplesamlphp.org/docs/stable/simplesamlphp-nostate.html]

3. **Configuration Learning Curve**
   - Complex configuration structure for newcomers
   - Multiple PHP array configuration files
   - Understanding SAML concepts required
   - Mitigation: Follow step-by-step guides, use templates, test in stages

4. **Certificate Management Overhead**
   - Must generate and maintain SP certificates
   - Certificate rotation requires planning and coordination
   - Federation metadata updates not instant
   - Mitigation: Set calendar reminders, implement rollover strategy, use long validity periods (2 years)

5. **Dependency Management**
   - Requires specific PHP version (8.0+ for SimpleSAMLphp 2.1+)
   - Multiple PHP extensions required
   - Composer dependency updates needed
   - Mitigation: Use Composer for updates, follow upgrade guides, test in staging environment

6. **Limited Built-in Authorization**
   - SimpleSAMLphp handles authentication, not authorization
   - Application must implement authorization logic
   - Attribute-based access control requires custom code
   - Mitigation: Use received attributes (affiliations, entitlements) to build authorization in application layer

7. **Debugging Difficulty**
   - SAML flows involve multiple redirects
   - XML errors can be cryptic
   - Network-based protocol makes troubleshooting harder
   - Mitigation: Use SimpleSAMLphp debug mode, browser dev tools, SAML tracer extensions, comprehensive logging

8. **InCommon Federation Participation Cost**
   - InCommon membership required (annual fees for institutions)
   - Metadata registration process
   - Compliance with federation policies
   - Mitigation: Ensure budget for federation fees, understand that this enables access to 1000+ institutions

### When to Use

1. **Academic/Research Applications**
   - Target users from universities and research institutions
   - Need access to InCommon federation
   - Users expect to authenticate with institutional credentials
   - Ideal: Research collaboration platforms, educational tools, library services

2. **PHP-Based Applications**
   - Application written in PHP or easily integrates with PHP
   - Web-based application with browser access
   - Can accommodate session-based authentication

3. **Multi-Institution SSO Required**
   - Need to support users from many different organizations
   - Each organization has its own IdP
   - Don't want to maintain individual integrations with each IdP
   - Federation membership provides one integration, many IdPs

4. **Hosted Environments Without Root**
   - Shared hosting or PaaS deployment
   - Cannot install system packages (Shibboleth SP requires system install)
   - Need userland solution

5. **Moderate Security Requirements**
   - Need strong authentication but not nation-state level security
   - Can accept SSL/TLS and signature validation as sufficient transport security
   - Don't require FIPS 140-2 certified crypto modules

6. **Resource Constrained Environments**
   - Limited server resources
   - Don't want to run additional daemons (Shibboleth SP daemon)
   - Want lightweight solution

### When to Avoid

1. **Non-PHP Applications Without Integration Path**
   - Application in language that can't easily call PHP
   - No reverse proxy or bridge pattern feasible
   - Alternative: Use language-native SAML library or Shibboleth SP

2. **High-Security Requirements**
   - Requires FIPS 140-2 compliance
   - Need hardware security modules (HSM) for all keys
   - Must meet government security standards
   - Alternative: Shibboleth SP with HSM integration

3. **Extremely High Traffic**
   - Millions of authentications per day
   - Requires horizontal scaling across many nodes
   - Complex load balancing needed
   - Alternative: Consider enterprise SSO solutions with dedicated support

4. **Single IdP Integration**
   - Only need to integrate with one IdP
   - Don't need federation capabilities
   - Simpler alternatives may suffice
   - Alternative: Direct SAML integration or OAuth2/OIDC if IdP supports it

5. **Real-time Performance Critical**
   - Every millisecond of latency matters
   - Cannot tolerate SAML redirects
   - Alternative: Use API tokens after initial authentication, cache attributes locally

6. **Organization Cannot Join Federation**
   - Cannot afford InCommon membership fees
   - Cannot meet federation participation requirements
   - Alternative: Direct SAML integrations or other authentication methods

7. **No PHP Expertise Available**
   - Team has no PHP experience
   - Cannot maintain PHP application
   - Alternative: Use solution in team's primary language

## Implementation Options

### Option 1: SimpleSAMLphp with InCommon MDQ (Recommended)

**Description**: Install SimpleSAMLphp 2.3+ as a Service Provider using Composer, configure with InCommon MDQ for metadata, use built-in discovery service or external InCommon discovery, implement proper certificate management and security hardening.

**Pros**:
- **Scalable** - MDQ handles large federation efficiently [https://spaces.at.internet2.edu/display/MDQ/home]
- **Modern** - SimpleSAMLphp 2.x with latest security features
- **Flexible** - Works with entire InCommon federation plus additional IdPs
- **Maintainable** - Composer-based updates, clear upgrade path
- **Cost-effective** - Open source, no licensing fees
- **Well-documented** - Extensive official documentation and community resources

**Cons**:
- Requires PHP 8.0+ and specific extensions
- Configuration learning curve for SAML newcomers
- Certificate management overhead
- Session management requires careful setup
- InCommon federation membership fee required

**Complexity**: Medium

**Time Estimate**: 3-5 days (1 day setup, 2 days configuration/testing, 1-2 days integration and hardening)

**Reuses Patterns**: N/A (new integration)

**When to Use**:
- Production deployment with InCommon federation
- Need to support multiple institutions
- PHP-based or PHP-friendly application stack
- Have budget for InCommon membership
- Want long-term maintainable solution

**Implementation Steps**:

1. Install SimpleSAMLphp 2.3 via Composer
2. Configure base URL, admin password, secret salt
3. Generate SP certificates (RSA 3072, self-signed)
4. Download and verify InCommon MDQ signing certificate
5. Configure MDQ metadata source in config.php
6. Configure SP in authsources.php with certificates
7. Test with InCommon Test IdP
8. Integrate with application code
9. Register SP metadata with InCommon
10. Configure production IdPs
11. Implement security hardening
12. Deploy with monitoring and logging

**Example Configuration Snippet**:
```php
// config/authsources.php
'default-sp' => [
    'saml:SP',
    'entityID' => 'https://myapp.example.org/',
    'privatekey' => 'saml.pem',
    'certificate' => 'saml.crt',
    'sign.authnrequest' => true,
],

// config/config.php
'metadata.sources' => [
    ['type' => 'flatfile'],
    [
        'type' => 'mdq',
        'server' => 'https://mdq.incommon.org',
        'validateCertificate' => '/var/simplesamlphp/cert/inc-md-cert-mdq.pem',
        'cachedir' => '/var/simplesamlphp/cache/mdq',
        'cachelength' => 86400,
    ],
],
```

### Option 2: SimpleSAMLphp with Full Metadata Aggregate

**Description**: Install SimpleSAMLphp but use traditional full metadata aggregate from InCommon instead of MDQ, with automated metadata refresh module.

**Pros**:
- Works with older SimpleSAMLphp versions (pre-MDQ support)
- All metadata available locally (no network dependency after download)
- Simpler concept for SAML beginners
- Can work offline after metadata refresh

**Cons**:
- Large metadata file (hundreds of MB) [https://spaces.at.internet2.edu/display/federation/metadata-service]
- Higher memory usage
- Slower startup/refresh times
- Must implement metadata refresh automation
- Network bandwidth for daily downloads
- Unnecessary exposure to all federation entities

**Complexity**: Medium

**Time Estimate**: 3-4 days

**Reuses Patterns**: Similar to Option 1 but different metadata source

**When to Use**:
- Stuck on older SimpleSAMLphp version
- Network reliability concerns (though metadata refresh still needs network)
- Need to work fully offline for periods
- Extremely high volume where MDQ network calls unacceptable

**Implementation Steps**:

Similar to Option 1 but:
- Use metarefresh module instead of MDQ
- Download full InCommon metadata aggregate
- Configure cron job for daily metadata refresh
- Verify metadata signature on each refresh

### Option 3: Shibboleth Service Provider

**Description**: Use Shibboleth SP (C++ implementation) instead of SimpleSAMLphp. Industry standard for SAML SP in higher education.

**Pros**:
- **Industry standard** - Widely deployed in academic institutions
- **High performance** - Compiled C++ application
- **Mature** - Extremely well-tested and stable
- **Feature-rich** - Comprehensive SAML 2.0 support
- **HSM support** - Can use hardware security modules
- **Language agnostic** - Works with any web application
- **Enterprise support** - Professional support available

**Cons**:
- **Requires root access** - System-level installation
- **Complex configuration** - XML-based configuration, steeper learning curve
- **System dependency** - Runs as system daemon (shibd)
- **More moving parts** - Apache/Nginx module plus daemon
- **Harder to containerize** - More challenging in Docker/Kubernetes
- **Updates require system admin** - Can't update from application layer

**Complexity**: High

**Time Estimate**: 1-2 weeks (more if team unfamiliar with Shibboleth)

**Reuses Patterns**: No

**When to Use**:
- Have root access and system administration expertise
- Need maximum performance (C++ vs PHP)
- Require HSM integration
- Application not in PHP (though SimpleSAMLphp can bridge)
- Enterprise environment with ops team support
- Need FIPS 140-2 compliance
- Already using Shibboleth elsewhere in organization

**When to Avoid**:
- Shared hosting or PaaS (no root access)
- Small team without dedicated ops
- Want application-layer solution
- Prefer simpler deployment/updates

### Option 4: OAuth2/OIDC Bridge

**Description**: If institutional IdPs support OpenID Connect (OIDC), use OIDC instead of SAML. Some InCommon IdPs support OIDC.

**Pros**:
- **Modern protocol** - JSON-based, simpler than XML
- **Native library support** - OIDC libraries in all languages
- **Better for SPAs/mobile** - JSON APIs vs XML redirects
- **Simpler debugging** - JSON easier to work with than XML
- **Token-based** - Can use refresh tokens for long sessions

**Cons**:
- **Limited InCommon support** - Not all IdPs support OIDC
- **Not universally available** - SAML more widespread in academia
- **Requires IdP support** - Can't force IdPs to add OIDC
- **May need bridge** - CILogon provides SAML→OIDC bridge but adds complexity

**Complexity**: Low (if IdP supports OIDC natively)

**Time Estimate**: 2-3 days

**Reuses Patterns**: N/A

**When to Use**:
- Target IdPs all support OIDC
- Modern application architecture (SPA, mobile app)
- Team familiar with OAuth2/OIDC
- Can use CILogon as bridge (adds dependency)

**When to Avoid**:
- Need broad InCommon federation access
- IdPs don't support OIDC
- Want direct federation integration
- Cannot add external dependencies (CILogon)

### Option 5: Commercial SSO Service

**Description**: Use commercial SSO service like Auth0, Okta, Azure AD B2C that can federate with SAML IdPs.

**Pros**:
- **Managed service** - No infrastructure to maintain
- **Multi-protocol** - Support SAML, OIDC, OAuth2, social logins
- **Professional support** - 24/7 support available
- **Dashboard management** - GUI for configuration
- **Advanced features** - MFA, anomaly detection, user management
- **High availability** - SLA guarantees

**Cons**:
- **Cost** - Monthly fees per user or per authentication
- **Vendor lock-in** - Proprietary APIs and configuration
- **External dependency** - Service availability critical
- **Data privacy** - User data flows through third party
- **InCommon integration may be limited** - Check provider's federation support
- **May not support all InCommon IdPs** - Some providers have limited SAML federation support

**Complexity**: Low to Medium

**Time Estimate**: 1-3 days

**Reuses Patterns**: N/A

**When to Use**:
- Want managed solution
- Need multi-protocol support (SAML + social + enterprise)
- Have budget for per-user costs
- Want advanced features (MFA, anomaly detection)
- Need guaranteed uptime SLA
- Small team without deep identity expertise

**When to Avoid**:
- Cost-sensitive project
- Data sovereignty requirements
- Want control over identity infrastructure
- Need broad InCommon federation support (verify provider supports this)
- Open source requirement

## Comparison Matrix

| Criteria | Option 1: SSP + MDQ | Option 2: SSP + Aggregate | Option 3: Shibboleth SP | Option 4: OIDC Bridge | Option 5: Commercial |
|----------|---------------------|--------------------------|------------------------|----------------------|---------------------|
| **Complexity** | Medium | Medium | High | Low-Medium | Low-Medium |
| **Maintainability** | High | Medium | Medium | High | High |
| **Performance** | Good | Fair | Excellent | Excellent | Good |
| **Learning Curve** | Medium | Medium | High | Low | Low |
| **Community Support** | Strong | Strong | Very Strong | Medium | Paid Support |
| **Reuses Patterns** | N/A | Partial | No | No | No |
| **Time to Implement** | 3-5 days | 3-4 days | 1-2 weeks | 2-3 days | 1-3 days |
| **Cost** | Free (+ InCommon fee) | Free (+ InCommon fee) | Free (+ InCommon fee) | Free (+ InCommon fee) | $$$ per user/month |
| **Root Access Required** | No | No | Yes | No | No |
| **Language Requirement** | PHP | PHP | Any | Any | Any |
| **Federation Support** | Excellent | Excellent | Excellent | Limited | Varies |
| **Scalability** | Excellent | Good | Excellent | Excellent | Excellent |
| **Security Maturity** | High | High | Very High | Medium | High |
| **HSM Support** | No | No | Yes | Depends | Maybe |

## Implementation Approach

### Recommended: Option 1 - SimpleSAMLphp with InCommon MDQ

This option provides the best balance of:
- ✅ Scalability and performance (MDQ protocol)
- ✅ Modern architecture (SimpleSAMLphp 2.x)
- ✅ Cost-effectiveness (open source)
- ✅ Maintainability (Composer-based)
- ✅ Broad InCommon support (all federation IdPs)
- ✅ No root access required (PHP userland)
- ✅ Active community and documentation

### Prerequisites & Requirements

**Server Requirements:**
- PHP 8.0 or higher (PHP 8.1+ recommended)
- Web server: Apache 2.4+ or Nginx
- HTTPS/SSL certificate for your domain
- Writable directories for cache and logs

**Required PHP Extensions:**
- Core: date, dom, fileinfo, filter, hash, json, libxml, mbstring, openssl, pcre, session, simplexml, sodium, SPL, zlib
- Platform: posix (Linux), intl (for translations)
- Optional: curl, ldap, radius, memcache/redis (for session storage)

**Tools:**
- Composer (for installation and dependency management)
- OpenSSL (for certificate generation)
- Git (optional, for version control)

**Access/Accounts:**
- InCommon Federation membership (or in-progress application)
- Access to register SP metadata with InCommon
- Server shell access for installation
- Web server configuration permissions

**Knowledge/Skills:**
- Basic PHP development
- Web server configuration (Apache or Nginx)
- Command-line operations
- Understanding of SSL/TLS and certificates
- Basic SAML concepts (will learn during implementation)

### Getting Started

**Step 1: Verify PHP Requirements**

```bash
# Check PHP version
php -v
# Should be 8.0 or higher

# Check required extensions
php -m | grep -E '(dom|openssl|mbstring|session|sodium)'

# Install missing extensions (Ubuntu/Debian example)
sudo apt-get install php8.1-xml php8.1-mbstring php8.1-curl
```

**Step 2: Install Composer**

```bash
# Download and install Composer globally
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer

# Verify installation
composer --version
```

**Step 3: Install SimpleSAMLphp**

```bash
# Create project directory
cd /var
sudo mkdir simplesamlphp
cd simplesamlphp

# Initialize Composer project
composer init --no-interaction

# Install SimpleSAMLphp
composer require simplesamlphp/simplesamlphp

# Or install with modules (full build)
composer require simplesamlphp/simplesamlphp:^2.3

# Set permissions
sudo chown -R www-data:www-data /var/simplesamlphp
```

**Step 4: Initial Configuration**

```bash
# Copy configuration templates
cp vendor/simplesamlphp/simplesamlphp/config-templates/config.php config/
cp vendor/simplesamlphp/simplesamlphp/config-templates/authsources.php config/

# Or if using tarball installation:
cp config-templates/config.php config/
cp config-templates/authsources.php config/
```

**Step 5: Configure Web Server**

See Web Server Security section above for Apache/Nginx configuration examples.

**Quick Apache example:**
```apache
Alias /simplesaml /var/simplesamlphp/vendor/simplesamlphp/simplesamlphp/public
SetEnv SIMPLESAMLPHP_CONFIG_DIR /var/simplesamlphp/config

<Directory /var/simplesamlphp/vendor/simplesamlphp/simplesamlphp/public>
    Require all granted
</Directory>
```

**Step 6: Test Installation**

```bash
# Restart web server
sudo systemctl restart apache2  # or nginx

# Visit admin interface
# https://your-domain.com/simplesaml/
```

You should see the SimpleSAMLphp front page.

### Architecture & Design Considerations

**Component Architecture:**

```
┌─────────────────────────────────────────────────────┐
│              User's Browser                         │
└─────────────┬──────────────▲────────────────────────┘
              │              │
              │ 1. Access    │ 7. Session established
              │    protected │    with attributes
              │    resource  │
              ▼              │
┌─────────────────────────────────────────────────────┐
│           Your PHP Application                      │
│  ┌──────────────────────────────────────────────┐  │
│  │  SimpleSAML\Auth\Simple::requireAuth()       │  │
│  │  SimpleSAML\Auth\Simple::getAttributes()     │  │
│  └──────────────┬──────────▲────────────────────┘  │
│                 │          │                        │
│  2. Not auth'd  │          │ 6. Return attributes   │
│     Start SAML  │          │    Start application   │
│     flow        │          │    session             │
└─────────────────┼──────────┼────────────────────────┘
                  │          │
                  ▼          │
┌─────────────────────────────────────────────────────┐
│          SimpleSAMLphp Library                      │
│  ┌──────────────────────────────────────────────┐  │
│  │  3. Generate AuthnRequest                    │  │
│  │  5. Process SAML Response                    │  │
│  │     - Validate signature                     │  │
│  │     - Check timestamps                       │  │
│  │     - Verify InResponseTo                    │  │
│  │     - Extract attributes                     │  │
│  └──────────────┬──────────▲────────────────────┘  │
│                 │          │                        │
│                 │          │                        │
│  ┌──────────────┼──────────┼────────────────────┐  │
│  │ Metadata     │          │                    │  │
│  │ Source       │          │                    │  │
│  │ (MDQ)   ─────┼──────────┤                    │  │
│  └──────────────┘          └────────────────────┘  │
└─────────────────┼──────────▲────────────────────────┘
                  │          │
                  │          │ 4b. SAML Response
                  │          │     (via browser POST)
                  │          │
┌─────────────────┼──────────┼────────────────────────┐
│  InCommon MDQ   │          │                        │
│  Service        │          │                        │
│                 │          │                        │
│  Retrieve IdP   │          │                        │
│  metadata  ◄────┘          │                        │
└─────────────────────────────┼────────────────────────┘
                              │
                              │
┌─────────────────────────────┴────────────────────────┐
│           University Identity Provider               │
│                                                       │
│  4a. User authenticates with university credentials  │
│      IdP generates signed SAML Response              │
└───────────────────────────────────────────────────────┘
```

**Key Design Decisions:**

1. **Session Management Strategy**
   - **Decision**: Use Redis for SimpleSAMLphp session storage in production
   - **Rationale**: Avoids conflicts with application PHP sessions, enables load balancing, better performance
   - **Implementation**: Configure `store.type => 'redis'` in config.php

2. **Metadata Strategy**
   - **Decision**: Use MDQ for InCommon, flat file for local SP metadata
   - **Rationale**: Scalability for federation, simplicity for local config
   - **Implementation**: Multiple sources in `metadata.sources` array

3. **Discovery Service**
   - **Decision**: Use built-in SimpleSAMLphp discovery for MVP, consider InCommon discovery later
   - **Rationale**: Built-in is simpler to start, InCommon discovery offers better UX
   - **Implementation**: Omit `idp` parameter in authsources.php

4. **Attribute Handling**
   - **Decision**: Map OID to friendly names, filter to only needed attributes
   - **Rationale**: Simpler application code, reduced data exposure
   - **Implementation**: Use authproc filters in authsources.php

5. **Certificate Management**
   - **Decision**: RSA 3072-bit self-signed with 2-year validity, calendar reminders for rotation
   - **Rationale**: Balance of security and compatibility, manageable rotation schedule
   - **Implementation**: OpenSSL generation, certificate rollover configuration

6. **Error Handling**
   - **Decision**: Hide detailed errors in production, comprehensive logging to syslog
   - **Rationale**: Security (no info disclosure) balanced with debuggability
   - **Implementation**: `showerrors => false`, `logging.level => NOTICE` in config.php

### Best Practices

**Configuration Management:**
- Store configuration outside web root (use SIMPLESAMLPHP_CONFIG_DIR environment variable)
- Use version control for configuration (but exclude sensitive files like private keys)
- Separate configuration for development, staging, production environments
- Document any deviations from default configuration

**Certificate Management:**
- Generate certificates with long validity (2 years) but not too long (avoid forgetting)
- Set calendar reminders 3 months before expiration [https://www.iam.harvard.edu/saml-signing-and-encryption-certificates]
- Use certificate rollover mechanism for zero-downtime rotation
- Store private keys with strict file permissions (640, owner www-data)
- Never commit private keys to version control

**Security:**
- Always use HTTPS for all endpoints (enforce with redirects)
- Enable request signing (`sign.authnrequest => true`)
- Validate IdP certificates (never set `validateCertificate => null`)
- Use secure session cookies (`session.cookie.secure => true`)
- Implement HTTPOnly cookies (`session.cookie.httponly => true`)
- Set appropriate SameSite policy (`session.cookie.samesite => 'Lax'`)
- Use strong admin password generated by pwgen.php
- Generate unique secret salt per environment
- Enable logging but protect log files (750 permissions)
- Regularly update SimpleSAMLphp and dependencies (check for security advisories)

**Performance:**
- Use external session storage (Redis or Memcache) in production [https://simplesamlphp.org/docs/stable/simplesamlphp-maintenance.html]
- Configure appropriate MDQ cache length (24 hours recommended)
- Enable opcode caching (OPcache) in PHP
- Use slim build of SimpleSAMLphp if don't need optional modules
- Monitor cache hit rates and adjust cache duration if needed

**Testing:**
- Test with InCommon Test IdP before production: https://spaces.at.internet2.edu/display/federation/incommon-test-idp
- Use SAML-tracer browser extension to debug flows: https://addons.mozilla.org/en-US/firefox/addon/saml-tracer/
- Verify metadata before submitting to federation (use metadata validators)
- Test attribute receipt and mapping thoroughly
- Test logout functionality (both SP-initiated and IdP-initiated)
- Test error scenarios (expired session, invalid signature, etc.)
- Load test authentication flow if expecting high traffic

**Monitoring:**
- Monitor authentication success/failure rates
- Alert on signature validation failures (potential attack)
- Track MDQ cache hit rates
- Monitor certificate expiration dates
- Log and alert on error rate increases
- Track session creation and expiration patterns

**Documentation:**
- Document your entity ID and why you chose it
- Document all configuration decisions and deviations from defaults
- Create runbook for common issues (state information lost, certificate errors, etc.)
- Document certificate rotation procedure
- Keep list of supported IdPs and their attributes
- Document attribute mappings and business logic using them

### Common Pitfalls & How to Avoid Them

**1. State Information Lost Error**

**Pitfall**: Users get "State information lost" error during authentication

**Causes**:
- Cookie domain mismatch (www vs non-www)
- HTTP/HTTPS inconsistency
- Session storage failure
- SameSite cookie issues with redirects

**How to Avoid**:
```php
// config.php - Ensure consistent URLs
'baseurlpath' => 'https://myapp.example.org/simplesaml/',  // Exact match in all configs

// Fix cookie domain
'session.cookie.domain' => '.example.org',  // Matches all subdomains

// Fix SameSite for cross-site redirects
'session.cookie.samesite' => 'None',  // Requires secure = true
'session.cookie.secure' => true,

// Use reliable session storage
'store.type' => 'redis',
```

**Reference**: https://simplesamlphp.org/docs/stable/simplesamlphp-nostate.html

**2. Certificate Signature Validation Failures**

**Pitfall**: Authentication fails with signature validation error

**Causes**:
- Wrong certificate (using aggregate cert for MDQ)
- Certificate not downloaded properly
- IdP certificate changed and metadata not refreshed
- Clock skew between servers

**How to Avoid**:
```bash
# Use correct MDQ certificate
curl -O http://md.incommon.org/certs/inc-md-cert-mdq.pem

# Verify fingerprint
openssl x509 -in inc-md-cert-mdq.pem -noout -fingerprint -sha256
# Must match: 60:49:74:D6:1F:E0:D7:F4:D6:3D:6C:8D:B9:8A:85:7E:64:2A:B9:B4:70:E3:E8:5D:D5:4D:66:3D:04:96:F9:00

# Sync server time via NTP
sudo apt-get install ntp
sudo systemctl enable ntp
sudo systemctl start ntp
```

**3. Private Key Permission Errors**

**Pitfall**: SimpleSAMLphp can't read private key, authentication fails

**Causes**:
- Wrong file permissions
- Wrong file ownership
- Private key in wrong location

**How to Avoid**:
```bash
# Set correct permissions
cd /var/simplesamlphp/cert
chmod 640 saml.pem
chown root:www-data saml.pem

# Verify web server user can read
sudo -u www-data cat saml.pem  # Should display key
```

**4. EntityID Mismatches**

**Pitfall**: IdP can't find SP metadata, or SP can't process response

**Causes**:
- EntityID in authsources.php doesn't match metadata
- EntityID in InCommon registration doesn't match configuration
- Trailing slash inconsistency (https://example.org vs https://example.org/)

**How to Avoid**:
```php
// Choose entityID carefully (usually your base domain)
// Use HTTPS
// Be consistent about trailing slashes (recommend: no trailing slash)
'entityID' => 'https://myapp.example.org',

// Use same entityID everywhere:
// - authsources.php
// - SP metadata
// - InCommon registration
// - Any documentation
```

**5. Attribute Access Errors**

**Pitfall**: Application crashes trying to access attributes that don't exist

**Causes**:
- Assuming attributes are present without checking
- IdP doesn't release expected attributes
- Attribute names in OID format vs friendly names

**How to Avoid**:
```php
// Always check attribute existence
$attributes = $as->getAttributes();

$email = $attributes['mail'][0] ?? null;
if (!$email) {
    // Handle missing attribute gracefully
    error_log('IdP did not provide email attribute');
    throw new Exception('Authentication incomplete: email required');
}

// Attributes are arrays (multi-valued), always access by index
$eppn = $attributes['eduPersonPrincipalName'][0];  // NOT $attributes['eduPersonPrincipalName']

// Use OID-to-name mapping to handle both formats
'authproc' => [
    100 => ['class' => 'core:AttributeMap', 'oid2name'],
],
```

**6. PHP Session Conflicts**

**Pitfall**: SimpleSAMLphp and application interfere with each other's sessions

**Causes**:
- Both using PHP native sessions
- Both calling session_start()
- Session name conflicts

**How to Avoid**:
```php
// Option 1: Don't call session_start() in application when using SimpleSAMLphp
// SimpleSAMLphp manages its own sessions

// Option 2: Use external session storage for SimpleSAMLphp
// config.php
'store.type' => 'redis',
'store.redis.host' => 'localhost',
'store.redis.port' => 6379,

// Application can still use PHP sessions
```

**7. HTTPS Requirement Not Met**

**Pitfall**: SAML fails because using HTTP instead of HTTPS

**Causes**:
- Development without SSL
- Load balancer terminating SSL
- Incorrect URL in configuration

**How to Avoid**:
```php
// config.php - Always HTTPS in production
'baseurlpath' => 'https://myapp.example.org/simplesaml/',

// For development with self-signed cert
// Use mkcert or similar to create trusted local cert
// Never use HTTP for SAML in production
```

```apache
# Apache - Force HTTPS
<VirtualHost *:80>
    ServerName myapp.example.org
    Redirect permanent / https://myapp.example.org/
</VirtualHost>
```

**8. Not Handling IdP-Initiated SSO**

**Pitfall**: IdP-initiated SSO fails or creates security issues

**Causes**:
- RelayState validation too strict
- No CSRF protection for unsolicited responses
- Assuming all authentication is SP-initiated

**How to Avoid**:
```php
// authsources.php
'default-sp' => [
    'saml:SP',
    'entityID' => 'https://myapp.example.org/',

    // Allow IdP-initiated SSO but validate RelayState
    'RelayState' => 'https://myapp.example.org/default-landing',

    // Or disable if not needed
    // 'disable.idp-initiated' => true,
],
```

### Migration/Adoption Strategy

**Phase 1: Development Environment Setup (Week 1)**
- Install SimpleSAMLphp in development environment
- Configure with test IdP (InCommon Test IdP or own test IdP)
- Implement basic integration with application
- Test attribute receipt and mapping
- Verify certificate generation and configuration
- Success criteria: Can authenticate against test IdP and receive attributes

**Phase 2: InCommon Integration (Week 2)**
- Apply for InCommon membership (if not already member)
- Download InCommon MDQ certificate
- Configure MDQ metadata source
- Test discovery service
- Test with multiple InCommon IdPs
- Register SP metadata with InCommon (preview federation)
- Success criteria: Can authenticate with real InCommon IdPs in preview

**Phase 3: Security Hardening (Week 3)**
- Implement all security best practices
- Configure external session storage (Redis/Memcache)
- Set up logging and monitoring
- Perform security review
- Test error scenarios
- Document configuration
- Success criteria: Security checklist complete, documentation done

**Phase 4: Staging Deployment (Week 4)**
- Deploy to staging environment
- Configure production-like settings
- Test with pilot users
- Monitor logs and performance
- Fix any issues discovered
- Load testing (if applicable)
- Success criteria: Pilot users successfully authenticate

**Phase 5: Production Deployment (Week 5)**
- Promote SP metadata to InCommon production federation
- Deploy to production environment
- Configure production monitoring/alerting
- Update documentation with production URLs
- Communicate to users
- Success criteria: Production live with monitoring

**Rollback Strategy:**
- Keep existing authentication method in place during rollout
- Feature flag to enable/disable SAML authentication
- Ability to redirect users to legacy authentication if SAML fails
- Database session storage allows reverting without losing all sessions
- Keep previous SimpleSAMLphp version available for quick rollback
```php
// Feature flag approach
if (ENABLE_SAML_AUTH && isset($_GET['saml'])) {
    // SimpleSAMLphp authentication
} else {
    // Legacy authentication
}
```

## Alternatives Considered

### Alternative 1: Shibboleth Service Provider (Native)

**Description**: Use Shibboleth SP instead of SimpleSAMLphp - the "gold standard" SAML implementation in higher education.

**Why it wasn't chosen**:
- Requires root access for system-level installation
- More complex to deploy and configure (XML configuration)
- Runs as system daemon (shibd) requiring service management
- Higher operational overhead for small teams
- Harder to containerize for modern deployment patterns

**When it might be better**:
- Have dedicated operations team with Shibboleth expertise
- Need maximum performance (C++ vs PHP)
- Require HSM integration for private key protection
- Need FIPS 140-2 compliance
- Already standardized on Shibboleth for other services
- Very high traffic requiring optimized performance

### Alternative 2: Direct SAML Library Integration

**Description**: Use a SAML library directly in application code (e.g., php-saml, OneLogin PHP SAML) without SimpleSAMLphp framework.

**Why it wasn't chosen**:
- More work to implement (no pre-built flows)
- Need to handle metadata management manually
- No built-in discovery service
- Miss SimpleSAMLphp's admin interface and tools
- More code to maintain and secure
- Less community resources and examples for federation integration

**When it might be better**:
- Need very simple integration with single IdP only
- Want maximum control over every aspect of SAML flow
- Application architecture doesn't fit SimpleSAMLphp's model
- Want minimal dependencies
- Team has deep SAML protocol expertise

### Alternative 3: CILogon Service

**Description**: Use CILogon (NSF-funded service) which provides SAML-to-OIDC bridge for InCommon federation.

**Why it wasn't chosen**:
- Adds external dependency (CILogon service availability)
- User data flows through third-party service
- May have usage limits or costs
- Additional layer of complexity
- Not all InCommon features exposed
- Less control over authentication flow

**When it might be better**:
- Application architecture requires OIDC (e.g., SPA or mobile app)
- Want to avoid SAML complexity entirely
- Need additional features CILogon provides (COmanage for group management)
- Acceptable to rely on external service
- Want single integration point for multiple protocols

### Alternative 4: Azure AD B2C / Auth0 / Okta

**Description**: Use commercial identity-as-a-service provider that can federate with InCommon IdPs.

**Why it wasn't chosen**:
- Per-user or per-authentication costs can be significant
- Vendor lock-in with proprietary APIs
- May not support all InCommon IdPs (need to verify federation support)
- User authentication data flows through commercial third party
- Less control over identity infrastructure
- May not meet open-source or cost requirements

**When it might be better**:
- Budget available for per-user costs
- Want managed service with SLA
- Need additional features (MFA, anomaly detection, user management dashboard)
- Small team wants to outsource identity management
- Need multi-protocol support (SAML + social + enterprise SSO)
- Want professional support and guaranteed uptime

### Alternative 5: Direct IdP Integrations

**Description**: Integrate directly with each university's IdP individually without federation.

**Why it wasn't chosen**:
- Doesn't scale (need separate integration for each institution)
- High maintenance overhead (N configurations instead of 1)
- Each IdP has slightly different configuration requirements
- Certificate management multiplied by N
- No benefit of federation operator vetting and trust framework
- Users expect to use federation, not individual negotiations

**When it might be better**:
- Only need to support 2-3 specific institutions
- Those institutions not in InCommon
- Can't afford InCommon membership fee
- Need custom SLA or attribute release from specific IdPs
- Have resources to manage many individual relationships

## Debates & Open Questions

**1. SimpleSAMLphp vs. Shibboleth SP for New Deployments**

**Debate**: Which SAML SP implementation should new projects choose in 2024?

**SimpleSAMLphp Advocates Argue**:
- Easier to deploy (no root access required)
- Simpler to configure (PHP arrays vs XML)
- Better fit for modern deployment (containers, PaaS)
- Sufficient for most use cases
- Lower operational overhead

**Shibboleth SP Advocates Argue**:
- More mature and battle-tested
- Better performance (C++ vs PHP)
- Industry standard in higher education
- Better security posture (compiled, HSM support)
- More feature-complete

**Current Thinking**: The trend appears to be toward SimpleSAMLphp for new deployments unless there are specific requirements (HSM, FIPS, extreme performance) that mandate Shibboleth SP. SimpleSAMLphp's ease of deployment and maintenance makes it attractive for smaller teams and modern architectures.

**2. MDQ Adoption Timeline**

**Debate**: When will MDQ become the universal standard replacing full metadata aggregates?

**Current Status**: MDQ is supported by major SAML implementations and recommended by federations, but full aggregate download is still common. Some organizations hesitant to adopt due to:
- Concerns about network dependency
- Existing automation built around aggregates
- Conservative "if it works, don't change it" stance

**Future Direction**: MDQ adoption is increasing as:
- Federation metadata sizes continue to grow
- More software supports MDQ natively
- Performance benefits become more apparent
- Security benefits (reduced attack surface) recognized

**Open Question**: Will aggregates be deprecated entirely, or will both methods coexist indefinitely?

**3. SAML vs. OIDC in Higher Education**

**Debate**: Should new academic applications use SAML or OIDC for federated authentication?

**SAML Arguments**:
- Ubiquitous support in academic institutions
- InCommon built on SAML
- Mature federation infrastructure
- Rich attribute vocabularies (eduPerson)

**OIDC Arguments**:
- Modern protocol (JSON vs XML)
- Better for SPAs and mobile apps
- Simpler to implement
- Better developer experience
- Token-based, RESTful architecture

**Current Reality**: SAML dominates in higher education but OIDC adoption growing. Some IdPs now support both. Services like CILogon provide bridges.

**Unresolved**: How long will SAML remain primary protocol in higher education? Will InCommon add first-class OIDC support?

**4. Self-Signed vs. CA-Signed Certificates for SAML**

**Debate**: Should SAML SP certificates be self-signed or CA-signed?

**Self-Signed Advocates**:
- SAML trust model is metadata-based, not PKI-based
- Certificate embedded in signed metadata provides trust
- Simpler (no CA interaction required)
- No cost or renewal process with CA
- Standard practice in SAML community

**CA-Signed Advocates**:
- Provides additional verification layer
- Familiar to security auditors
- Aligns with general PKI best practices
- May be required by organizational policy

**Consensus**: Self-signed is perfectly acceptable and standard practice for SAML. CA-signed provides no additional security in metadata-based trust model but may be organizationally required.

**5. Session Storage: PHP Native vs. External**

**Debate**: When is external session storage (Redis/Memcache) necessary for SimpleSAMLphp?

**PHP Native Arguments**:
- Simpler setup
- One less service to manage
- Sufficient for single-server deployments
- Lower complexity for small applications

**External Storage Arguments**:
- Required for load balancing
- Avoids conflicts with application sessions
- Better performance
- More reliable (dedicated service)
- Production best practice

**Recommendation**: Use PHP native for development and single-server deployments. Use external storage for production, especially with load balancing or if application also uses PHP sessions heavily.

**6. Built-in vs. External Discovery Service**

**Debate**: Should SPs use SimpleSAMLphp's built-in discovery or external services like InCommon Discovery Service?

**Built-in Discovery Pros**:
- Simpler configuration
- No external dependency
- Sufficient functionality for most uses
- Full control over UX

**InCommon Discovery Pros**:
- Familiar UX for academic users
- Better UI/UX design
- Persistent IdP selection (remembers choice)
- Standardized across federation
- Mobile-friendly

**Current Practice**: Many start with built-in for simplicity, then migrate to InCommon Discovery for better UX. Some implement custom discovery based on domain or organizational context.

## Recommendations

### Preferred Approach: Option 1 - SimpleSAMLphp with InCommon MDQ

**Should This Be Implemented?**: **Yes, Strongly Recommended**

**Rationale**:

1. **Best Technical Fit**
   - MDQ protocol provides scalability for federation integration
   - SimpleSAMLphp 2.x offers modern security features
   - PHP implementation matches common web application stacks
   - No root access required fits hosted environments

2. **Cost-Effective**
   - Open source software (no licensing fees)
   - Only cost is InCommon federation membership (required regardless of SAML implementation)
   - Lower operational overhead than Shibboleth SP
   - Community support available at no cost

3. **Proven Track Record**
   - Widely deployed in academic environments
   - Mature codebase (15+ years of development)
   - Active security maintenance
   - Large community with extensive documentation

4. **Appropriate for Requirements**
   - Enables SSO with InCommon federation institutions
   - Supports multiple IdPs through single integration
   - Provides standard attribute release (R&S compatible)
   - Suitable for web application architecture

5. **Manageable Implementation**
   - 3-5 day implementation timeline reasonable
   - Clear documentation and examples available
   - Incremental rollout possible (feature flag)
   - Rollback strategy available

**Why**:

The combination of SimpleSAMLphp and InCommon MDQ provides the optimal balance of functionality, security, cost, and maintainability for enabling federated SSO with academic institutions. It's the most appropriate solution for organizations that:
- Need to support users from multiple universities
- Have PHP-based or PHP-compatible applications
- Want cost-effective open-source solution
- Need to deploy without root access
- Have small to medium technical teams

The MDQ protocol specifically addresses scalability concerns that would arise with traditional full metadata aggregates, making it future-proof as the federation grows.

**Key Considerations**:

1. **InCommon Membership Required**
   - Annual federation participation fee
   - Compliance with federation policies
   - Metadata registration process
   - Budget accordingly and plan for ongoing renewal

2. **Certificate Management Critical**
   - Must implement proper certificate lifecycle management
   - Calendar reminders essential for rotation
   - Plan rollover strategy before deployment
   - Consider certificate expiration monitoring service

3. **Security Hardening Non-Negotiable**
   - Must implement all security best practices
   - Cannot skip certificate validation
   - Must use HTTPS exclusively
   - Regular security updates required

4. **Team Knowledge Development**
   - Invest time in learning SAML concepts
   - Understand SimpleSAMLphp architecture
   - Build operational runbooks
   - Document all configuration decisions

### Potential Challenges:

**Challenge 1: "State Information Lost" Errors During Initial Deployment**

**Likelihood**: High (very common issue for newcomers)

**Impact**: Blocks user authentication, poor UX

**Mitigation Strategy**:
- Configure consistent URLs (HTTPS, domain with/without www)
- Use external session storage from start (Redis/Memcache)
- Set appropriate SameSite cookie policy
- Test thoroughly with multiple browsers
- Have troubleshooting runbook ready

**Fallback Plan**:
- Enable detailed logging for debugging
- Use SAML-tracer browser extension to diagnose
- Consult SimpleSAMLphp documentation and community
- Can temporarily use different session storage method if primary fails

**Challenge 2: Certificate Expiration in Production**

**Likelihood**: Medium (happens if not properly monitored)

**Impact**: Critical (breaks all authentication until resolved)

**Mitigation Strategy**:
- Set multiple calendar reminders (3 months, 1 month, 2 weeks before expiration)
- Use monitoring service for certificate expiration
- Generate certificates with 2-year validity (not too short, not too long)
- Implement rollover procedure for zero-downtime renewal
- Document renewal process in runbook

**Fallback Plan**:
- Emergency rollover procedure documented
- Know how to generate new cert and update metadata quickly
- Have federation operator contact info for expedited metadata update
- Can temporarily extend certificate if caught early enough

**Challenge 3: Attribute Mapping and Availability Inconsistencies**

**Likelihood**: High (different IdPs release different attributes)

**Impact**: Medium (affects authorization and user experience)

**Mitigation Strategy**:
- Code defensively (always check attribute existence)
- Have fallback logic for missing attributes
- Request only R&S attributes (guaranteed from R&S IdPs)
- Graceful degradation when optional attributes missing
- Provide clear error messages to users

**Fallback Plan**:
- Allow users to manually provide missing information
- Implement attribute validation and error handling
- Contact IdP administrators about missing required attributes
- Have alternative authentication method if SAML fails

### Success Criteria:

**Technical Success Metrics**:
1. Users can authenticate via any InCommon R&S IdP
2. Attribute receipt rate >95% for required attributes (ePPN, email, name)
3. Authentication completion rate >98% (excluding user abandonment)
4. Session establishment time <5 seconds average
5. Zero security incidents related to SAML implementation
6. Certificate rotation completed without service interruption

**Operational Success Metrics**:
1. <2 hours mean time to resolution for SAML-related issues
2. Comprehensive runbooks for common scenarios
3. Monitoring alerts configured and responding appropriately
4. Team can troubleshoot issues without external support
5. Documentation maintained and current

**Business Success Metrics**:
1. Reduced support tickets for authentication issues
2. Increased user satisfaction (fewer credentials to remember)
3. Expanded institutional reach (easy onboarding of new universities)
4. Maintained security posture (no breaches or vulnerabilities)
5. Cost within budget (InCommon fees only, no additional licensing)

## Additional Notes

**Deployment Considerations**:
- SimpleSAMLphp works well in containerized environments (Docker, Kubernetes) but requires persistent volume for cache and logs
- When using load balancers, ensure session affinity configured or use shared session storage (Redis/Memcache)
- Consider geographic distribution of users when choosing session storage location
- Monitor MDQ endpoint availability (though InCommon has excellent uptime)

**InCommon Federation Evolution**:
- InCommon continues to evolve with regular metadata format updates
- Stay subscribed to InCommon announcements for policy changes
- New entity categories (beyond R&S) may be introduced
- Future may bring improved discovery services or metadata distribution methods
- Consider joining InCommon community calls or working groups for early awareness of changes

**SimpleSAMLphp Ecosystem**:
- Large module ecosystem for specialized needs (multifactor, consent management, etc.)
- Active development on GitHub with regular releases
- Strong focus on security with quick response to vulnerabilities
- Version 2.x represents modernization effort with ongoing improvements
- Consider contributing back to project if developing custom features

**Integration Patterns**:
- SimpleSAMLphp can protect non-PHP applications via proxy pattern
- Can integrate with API gateways for protecting REST APIs
- Works with modern frontend frameworks (React, Vue) serving from protected paths
- Can coexist with other authentication methods (API keys, OAuth2) for different use cases

**Edge Cases to Consider**:
- Users with multiple accounts at same institution (should be handled by IdP but be aware)
- Users changing institutions (ePPN changes, need account linking strategy)
- IdP downtime during authentication (provide clear error, retry mechanism)
- Users behind restrictive firewalls (ensure SAML endpoints accessible)
- Mobile browsers with aggressive cookie restrictions (test thoroughly)

**Compliance and Privacy**:
- InCommon participation requires compliance with Participation Agreement
- Review data handling requirements in agreement
- Consider GDPR implications if European users possible
- Privacy policy should explain federated authentication
- Users have right to know what attributes shared

## Next Steps

1. [ ] **Decision Gate**: Confirm InCommon membership active or application in progress
2. [ ] **Budget Approval**: Verify budget for InCommon annual fees ($2,000-$5,000+ depending on organization size)
3. [ ] **Technical Review**: Review this research document with development team
4. [ ] **Architecture Decision**: Confirm SimpleSAMLphp + MDQ is approved approach
5. [ ] **Resource Allocation**: Assign developer(s) for 1-2 week implementation sprint
6. [ ] **Environment Setup**: Provision development, staging, production environments
7. [ ] **Create Implementation Plan**: Expand "Implementation Approach" section into detailed project plan with tasks, owners, dates
8. [ ] **Begin Phase 1**: Start development environment setup per Migration/Adoption Strategy
9. [ ] **Schedule Training**: Plan team knowledge transfer sessions on SAML and SimpleSAMLphp
10. [ ] **Establish Success Metrics**: Set up monitoring and define measurement approach for success criteria

**Immediate Action Items (This Week)**:
- [ ] Verify InCommon membership status
- [ ] Confirm PHP 8.0+ available in target environment
- [ ] Review security best practices with security team
- [ ] Schedule kickoff meeting with implementation team
- [ ] Clone/download SimpleSAMLphp for local experimentation

**Short-term Action Items (Next 2 Weeks)**:
- [ ] Complete development environment setup
- [ ] Test basic authentication against InCommon Test IdP
- [ ] Generate SP certificates
- [ ] Configure MDQ with InCommon
- [ ] Document initial configuration decisions

**Medium-term Action Items (Next Month)**:
- [ ] Complete staging environment deployment
- [ ] Pilot testing with real users
- [ ] Security review and hardening
- [ ] Prepare production deployment plan
- [ ] Register metadata with InCommon production federation

## Sources

1. SimpleSAMLphp Official Documentation - https://simplesamlphp.org/docs/stable/ - Accessed 2025-10-21
2. SimpleSAMLphp Service Provider QuickStart - https://simplesamlphp.org/docs/stable/simplesamlphp-sp - Accessed 2025-10-21
3. SimpleSAMLphp Installation Guide - https://simplesamlphp.org/docs/stable/simplesamlphp-install.html - Accessed 2025-10-21
4. SimpleSAMLphp State Information Lost Debugging - https://simplesamlphp.org/docs/stable/simplesamlphp-nostate.html - Accessed 2025-10-21
5. SimpleSAMLphp 2.0 Upgrade Notes - https://simplesamlphp.org/docs/stable/simplesamlphp-upgrade-notes-2.0.html - Accessed 2025-10-21
6. InCommon Federation Homepage - https://incommon.org/federation/ - Accessed 2025-10-21
7. InCommon Research and Scholarship Category - https://incommon.org/federation/research-and-scholarship/ - Accessed 2025-10-21
8. InCommon MDQ Service Wiki - https://spaces.at.internet2.edu/display/MDQ/home - Accessed 2025-10-21
9. InCommon MDQ Protocol Documentation - https://spaces.at.internet2.edu/display/MDQ/Metadata+Query+Protocol - Accessed 2025-10-21
10. Configure SimpleSAMLphp with MDQ - https://spaces.at.internet2.edu/display/MDQ/how-to-configure-other-software-to-use-mdq - Accessed 2025-10-21
11. InCommon MDQ Signing Certificate - https://ops.incommon.org/inc_md_cert_mdq.html - Accessed 2025-10-21
12. InCommon MDQ Certificate Download - http://md.incommon.org/certs/inc-md-cert-mdq.pem - Accessed 2025-10-21
13. OWASP SAML Security Cheat Sheet - https://cheatsheetseries.owasp.org/cheatsheets/SAML_Security_Cheat_Sheet.html - Accessed 2025-10-21
14. SAML Authentication Overview (Auth0) - https://auth0.com/blog/how-saml-authentication-works/ - Accessed 2025-10-21
15. Understanding SAML (Okta) - https://developer.okta.com/docs/concepts/saml/ - Accessed 2025-10-21
16. SAML 2.0 Technical Overview (OASIS) - https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html - Accessed 2025-10-21
17. SAML Certificates Explained (Stytch) - https://stytch.com/blog/saml-certificates/ - Accessed 2025-10-21
18. SAML Certificates Guide (WorkOS) - https://workos.com/blog/saml-certificates-guide - Accessed 2025-10-21
19. Common SAML Security Vulnerabilities (WorkOS) - https://workos.com/guide/common-saml-security-vulnerabilities - Accessed 2025-10-21
20. InCommon Federation Attribute Overview - https://incommon.org/federation/attributes/ - Accessed 2025-10-21
21. SimpleSAMLphp GitHub Repository - https://github.com/simplesamlphp/simplesamlphp - Accessed 2025-10-21
22. Installing SimpleSAMLphp Using Composer - https://www.hashbangcode.com/article/installing-simplesamlphp-using-composer - Accessed 2025-10-21
23. Installing a SimpleSAMLphp SP (Tuakiri) - https://docs.tuakiri.ac.nz/service_providers/installing_a_simplesamlphp_sp - Accessed 2025-10-21
24. On Breaking SAML: Be Whoever You Want to Be (USENIX Security 2012) - https://www.usenix.org/system/files/conference/usenixsecurity12/sec12-final91.pdf - Accessed 2025-10-21
25. SAML Signing and Encryption Certificates (Harvard) - https://www.iam.harvard.edu/saml-signing-and-encryption-certificates - Accessed 2025-10-21
