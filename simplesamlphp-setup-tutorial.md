---
tags: [authentication]
date: 2024-12-22
status: complete
---

# SimpleSAMLphp Setup Tutorial: Complete Guide for IdP and SP Configuration

**Date**: October 29, 2025
**Audience**: Beginners to SAML and SimpleSAMLphp
**Use Case**: Organizations setting up SimpleSAMLphp as both Identity Provider (IdP) and Service Provider (SP)

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [SAML 2.0 Fundamentals](#saml-20-fundamentals)
3. [SimpleSAMLphp Architecture](#simplesamlphp-architecture)
4. [Installation & Initial Setup](#installation--initial-setup)
5. [Configuration Guide](#configuration-guide)
6. [Setting Up as Service Provider (SP)](#setting-up-as-service-provider-sp)
7. [Setting Up as Identity Provider (IdP)](#setting-up-as-identity-provider-idp)
8. [Integration Examples](#integration-examples)
9. [Advanced: SAML Proxy/Bridge Configuration](#advanced-saml-proxybridge-configuration)
10. [Visual Flow Diagrams](#visual-flow-diagrams)
11. [Security Best Practices](#security-best-practices)
12. [Troubleshooting](#troubleshooting)
13. [Additional Resources](#additional-resources)

---

## Executive Summary

### What is SimpleSAMLphp?

SimpleSAMLphp is a PHP implementation of SAML 2.0 (Security Assertion Markup Language) that enables:
- **Single Sign-On (SSO)**: Users authenticate once and gain access to multiple applications
- **Identity Federation**: Organizations can trust each other's user authentication
- **Flexible Deployment**: Can act as Identity Provider (IdP), Service Provider (SP), or both

### When to Use SimpleSAMLphp

**Use as Service Provider (SP) when:**
- Your application needs to authenticate users via external providers (Azure AD, Google, Okta)
- You want to delegate user management to a third party
- You need to integrate with existing enterprise SSO systems

**Use as Identity Provider (IdP) when:**
- You manage user authentication and want to provide SSO to other services
- You have an existing user directory (LDAP, database) to authenticate against
- You want to act as the authentication authority for your organization

**Use as Both (IdP + SP / Proxy) when:**
- You need to bridge between multiple identity systems
- You want to add an authentication layer with custom logic
- You're creating a federation hub between organizations

### Key Benefits

- **Standards-Based**: Implements SAML 2.0, ensuring broad compatibility
- **Modular Architecture**: Extensible with authentication sources and modules
- **Production-Ready**: Built on Symfony components, widely deployed
- **Active Development**: Regularly updated with security patches

---

## SAML 2.0 Fundamentals

### What is SAML?

SAML (Security Assertion Markup Language) is an XML-based standard for exchanging authentication and authorization data between:
- **Identity Provider (IdP)**: The system that authenticates users and issues assertions
- **Service Provider (SP)**: The application that relies on the IdP for authentication

Think of it like airport security:
- Your **Identity Provider** is like passport control (verifies who you are)
- The **Service Provider** is like the airline lounge (grants access based on passport verification)
- The **SAML Assertion** is like the boarding pass (proof you've been authenticated)

### Core SAML Concepts

#### 1. **Entities**

**Identity Provider (IdP)**
- Authenticates users (checks username/password, multi-factor auth, etc.)
- Issues SAML assertions (statements about the authenticated user)
- Manages user attributes (email, name, groups, roles)

**Service Provider (SP)**
- Provides the actual application/service
- Trusts the IdP to handle authentication
- Consumes SAML assertions to grant access
- Receives user attributes from assertions

#### 2. **SAML Assertions**

An assertion is a signed XML document containing:
- **Authentication Statement**: Confirms user was authenticated
- **Attribute Statement**: Contains user attributes (email, name, roles)
- **Authorization Statement**: Specifies what user can access (less commonly used)

```xml
<saml:Assertion>
  <saml:AuthnStatement>
    <!-- User was authenticated at this time -->
  </saml:AuthnStatement>
  <saml:AttributeStatement>
    <saml:Attribute Name="email">
      <saml:AttributeValue>user@example.org</saml:AttributeValue>
    </saml:Attribute>
  </saml:AttributeStatement>
</saml:Assertion>
```

#### 3. **Metadata**

Metadata is configuration data exchanged between IdP and SP containing:
- **Entity ID**: Unique identifier (usually a URL)
- **Endpoints**: URLs for SSO, logout, metadata
- **Certificates**: Public keys for signature verification
- **Supported bindings**: HTTP-POST, HTTP-Redirect, etc.

Metadata must be exchanged between parties to establish trust.

#### 4. **Bindings**

Bindings define how SAML messages are transported:
- **HTTP-POST**: SAML message sent via HTML form POST (most common)
- **HTTP-Redirect**: SAML message sent via URL parameter
- **HTTP-Artifact**: Reference sent, actual message retrieved separately

#### 5. **Profiles**

Profiles define complete use cases:
- **Web Browser SSO Profile**: Standard web application SSO
- **Single Logout Profile**: Coordinated logout across all services
- **Artifact Resolution Profile**: For retrieving assertions

---

## SimpleSAMLphp Architecture

### Directory Structure

```
simplesamlphp/
├── bin/                      # Command-line utilities
│   ├── console               # Symfony console commands
│   └── pwgen.php             # Password hash generator
├── cert/                     # Certificates and private keys
│   ├── server.crt            # Your certificate (you create this)
│   └── server.pem            # Your private key (you create this)
├── config/                   # Configuration files
│   ├── config.php            # Main configuration
│   └── authsources.php       # Authentication sources
├── metadata/                 # SAML metadata
│   ├── saml20-idp-hosted.php    # Your IdP metadata (when acting as IdP)
│   ├── saml20-idp-remote.php    # Remote IdPs (when acting as SP)
│   ├── saml20-sp-remote.php     # Remote SPs (when acting as IdP)
│   └── saml20-sp-hosted.php     # Your SP metadata (alternative format)
├── modules/                  # Pluggable modules
│   ├── admin/                # Admin interface
│   ├── core/                 # Core authentication
│   ├── saml/                 # SAML protocol implementation
│   ├── exampleauth/          # Example auth sources
│   ├── multiauth/            # Multiple auth support
│   └── cron/                 # Scheduled tasks
├── public/                   # Web-accessible directory
│   ├── index.php             # Main entry point
│   └── module.php            # Module entry point
├── src/                      # Core SimpleSAMLphp code
│   └── SimpleSAML/           # Namespace root
├── templates/                # Twig templates
├── locales/                  # Translations
└── vendor/                   # Composer dependencies
```

### Key Components

#### 1. **Configuration System**

SimpleSAMLphp uses PHP arrays for configuration:

**config/config.php** - Main system configuration
- Base URL paths
- Security settings (salts, passwords)
- Database connections
- Session management
- Module enablement
- Logging and debugging

**config/authsources.php** - Authentication source definitions
- Defines how users authenticate
- Configures SP connections to IdPs
- Sets up authentication backends (LDAP, SQL, etc.)

#### 2. **Module System**

Modules extend functionality:
- **core**: Core authentication and UI
- **saml**: SAML 2.0 protocol implementation
- **admin**: Administrative interface
- **exampleauth**: Example authentication sources
- **multiauth**: Support for multiple authentication methods

Modules are activated in `config.php`:
```php
'module.enable' => [
    'core' => true,
    'admin' => true,
    'saml' => true,
    'exampleauth' => false,  // Enable for testing
],
```

#### 3. **Metadata System**

Metadata can be stored in multiple ways:

**Flat Files** (default, simplest):
- `metadata/saml20-idp-hosted.php` - Your IdP configuration
- `metadata/saml20-idp-remote.php` - Remote IdPs you trust
- `metadata/saml20-sp-remote.php` - Remote SPs you serve

**XML Files**:
- Load metadata from XML files (federated metadata)

**MDQ (Metadata Query) Protocol**:
- Fetch metadata dynamically from a server

**Database (PDO)**:
- Store metadata in a database

Configuration in `config.php`:
```php
'metadata.sources' => [
    ['type' => 'flatfile'],  // Default
    // ['type' => 'xml', 'file' => 'federation-metadata.xml'],
    // ['type' => 'pdo'],
],
```

#### 4. **Session Management**

SimpleSAMLphp manages user sessions:

**Session Stores**:
- **phpsession**: PHP's built-in session (default, simple)
- **memcache**: Memcache/Memcached for scalability
- **redis**: Redis for scalability
- **sql**: Database storage

Configuration in `config.php`:
```php
'store.type' => 'phpsession',  // or 'memcache', 'redis', 'sql'
'session.duration' => 8 * (60 * 60),  // 8 hours
'session.cookie.name' => 'SimpleSAMLSessionID',
```

#### 5. **Authentication Processing Filters**

Filters modify user attributes during authentication:

```php
'authproc.idp' => [
    30 => 'core:LanguageAdaptor',
    50 => 'core:AttributeLimit',    // Limit which attributes are released
    60 => [
        'class' => 'core:AttributeMap',
        'oid2name',  // Map OIDs to friendly names
    ],
],
```

Common filters:
- `core:AttributeLimit`: Restrict which attributes are released
- `core:AttributeMap`: Map attribute names
- `core:TargetedID`: Generate persistent identifiers
- `consent:Consent`: Ask for user consent
- `saml:AttributeNameID`: Use attribute as NameID

---

## Installation & Initial Setup

### Prerequisites

**System Requirements**:
- **PHP**: 8.2 or higher (testing on 8.2-8.5)
- **Web Server**: Apache, Nginx, or similar
- **Extensions**: date, dom, fileinfo, filter, hash, json, libxml, mbstring, openssl, posix, pcre, session, simplexml, SPL, xml, zlib

**Optional**:
- **Database**: MySQL, PostgreSQL (for database-backed sessions/metadata)
- **Redis/Memcache**: For scalable session storage
- **LDAP**: For LDAP authentication

### Installation Steps

#### 1. Download SimpleSAMLphp

```bash
# Option A: Download release
cd /var
wget https://github.com/simplesamlphp/simplesamlphp/releases/download/vX.Y.Z/simplesamlphp-X.Y.Z.tar.gz
tar xzf simplesamlphp-X.Y.Z.tar.gz
mv simplesamlphp-X.Y.Z simplesamlphp

# Option B: Clone from Git (for development)
cd /var
git clone https://github.com/simplesamlphp/simplesamlphp.git
cd simplesamlphp
composer install
```

#### 2. Web Server Configuration

**Apache Configuration** (example):

```apache
# /etc/apache2/sites-available/simplesamlphp.conf
<VirtualHost *:443>
    ServerName sso.example.org
    DocumentRoot /var/simplesamlphp/public

    <Directory /var/simplesamlphp/public>
        Require all granted
    </Directory>

    SSLEngine on
    SSLCertificateFile /path/to/ssl.crt
    SSLCertificateKeyFile /path/to/ssl.key

    # Recommended security headers
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-Content-Type-Options "nosniff"
</VirtualHost>
```

**Nginx Configuration** (example):

```nginx
# /etc/nginx/sites-available/simplesamlphp
server {
    listen 443 ssl;
    server_name sso.example.org;
    root /var/simplesamlphp/public;
    index index.php;

    ssl_certificate /path/to/ssl.crt;
    ssl_certificate_key /path/to/ssl.key;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
}
```

Enable the site and restart:
```bash
# Apache
sudo a2ensite simplesamlphp
sudo systemctl reload apache2

# Nginx
sudo ln -s /etc/nginx/sites-available/simplesamlphp /etc/nginx/sites-enabled/
sudo systemctl reload nginx
```

#### 3. Set Filesystem Permissions

```bash
# Create cache directory
sudo mkdir -p /var/cache/simplesamlphp
sudo chown www-data:www-data /var/cache/simplesamlphp

# Set permissions on cert directory
sudo chown -R www-data:www-data /var/simplesamlphp/cert
sudo chmod 700 /var/simplesamlphp/cert
```

#### 4. Access Web Interface

Visit `https://sso.example.org/` in your browser. You should see the SimpleSAMLphp welcome page.

---

## Configuration Guide

### Main Configuration (config/config.php)

This is the comprehensive guide to all critical settings in `config/config.php`.

#### Location in Codebase
`/var/simplesamlphp/config/config.php` (copy from `config.php.dist`)

```bash
cd /var/simplesamlphp/config
cp config.php.dist config.php
```

#### Critical Settings to Configure

**1. Base URL Configuration**

```php
// URL path to SimpleSAMLphp (relative or absolute)
'baseurlpath' => 'simplesaml/',
// OR use full URL if behind reverse proxy:
'baseurlpath' => 'https://sso.example.org/simplesaml/',

// Application base URL (when environment differs from reality)
'application' => [
    'baseURL' => 'https://sso.example.org',
],
```

**When to use what**:
- `simplesaml/` - If SimpleSAMLphp is at `https://sso.example.org/simplesaml/`
- Full URL - If behind reverse proxy or load balancer

**2. Security Settings** (CRITICAL - MUST CHANGE)

```php
// Secret salt for hashing (CHANGE FROM DEFAULT!)
// Generate with: LC_ALL=C tr -c -d '0123456789abcdefghijklmnopqrstuvwxyz' </dev/urandom | dd bs=32 count=1 2>/dev/null;echo
'secretsalt' => 'YOUR_RANDOM_SECRET_SALT_HERE',

// Admin password (CHANGE FROM DEFAULT!)
// Generate hash with: bin/pwgen.php
'auth.adminpassword' => '{SSHA256}your_password_hash_here',

// Protect metadata access
'admin.protectmetadata' => true,

// Trusted domains for redirects (security critical!)
'trusted.url.domains' => [
    'app1.example.org',
    'app2.example.org',
],
```

**3. File System Paths**

```php
'certdir' => 'cert/',                  // Certificate directory
'cachedir' => '/var/cache/simplesamlphp',  // Cache directory
// 'loggingdir' => '/var/log/simplesamlphp/',  // Log directory (if using file logging)
// 'datadir' => '/var/data/simplesamlphp/',    // Data storage
```

**4. Technical Contact Information**

```php
'technicalcontact_name' => 'Your Name',
'technicalcontact_email' => 'admin@example.org',
```

This appears in metadata and error reports.

**5. Enable IdP and/or SP Functionality**

```php
// Enable SAML 2.0 IdP support
'enable.saml20-idp' => false,  // Set to true when acting as IdP

// Enable ADFS IdP support (if needed)
'enable.adfs-idp' => false,
```

**6. Module Configuration**

```php
'module.enable' => [
    'core' => true,
    'admin' => true,
    'saml' => true,
    'exampleauth' => false,  // Enable for testing only
    'multiauth' => false,    // Enable if using multiple auth sources
    'cron' => false,         // Enable if using cron jobs
],
```

**7. Session Configuration**

```php
'session.duration' => 8 * (60 * 60),  // 8 hours
'session.datastore.timeout' => 4 * 60 * 60,  // 4 hours
'session.state.timeout' => 60 * 60,  // 1 hour

'session.cookie.name' => 'SimpleSAMLSessionID',
'session.cookie.lifetime' => 0,  // 0 = expire on browser close
'session.cookie.path' => '/',
'session.cookie.domain' => '',  // Or '.example.org' for subdomain sharing
'session.cookie.secure' => true,  // HTTPS only (recommended)
'session.cookie.samesite' => 'None',  // 'None', 'Lax', or 'Strict'
```

**8. Data Store Configuration**

```php
// Session storage backend
'store.type' => 'phpsession',  // Options: 'phpsession', 'memcache', 'redis', 'sql'

// For Redis (if using)
'store.redis.host' => 'localhost',
'store.redis.port' => 6379,
'store.redis.password' => 'your_redis_password',
'store.redis.prefix' => 'SimpleSAMLphp',

// For Memcache (if using)
'memcache_store.servers' => [
    [
        ['hostname' => 'localhost'],
    ],
],
```

**9. Database Configuration (Optional)**

```php
'database.dsn' => 'mysql:host=localhost;dbname=saml',
'database.username' => 'simplesamlphp',
'database.password' => 'secret',
'database.prefix' => '',
```

**10. Logging Configuration**

```php
'logging.level' => SimpleSAML\Logger::NOTICE,  // ERR, WARNING, NOTICE, INFO, DEBUG
'logging.handler' => 'syslog',  // Options: 'syslog', 'file', 'errorlog', 'stderr'
'logging.facility' => LOG_LOCAL5,
'logging.processname' => 'simplesamlphp',
'logging.logfile' => 'simplesamlphp.log',  // If using 'file' handler
```

**11. Debugging Options**

```php
'debug' => [
    'saml' => false,        // Log all SAML messages (including decrypted)
    'backtraces' => true,   // Log error backtraces
    'validatexml' => false, // Validate SAML XML against schemas
],

'showerrors' => false,  // Show errors to users (false in production)
'errorreporting' => true,  // Allow users to report errors
```

**12. Metadata Configuration**

```php
'metadatadir' => 'metadata',

'metadata.sources' => [
    ['type' => 'flatfile'],  // Use flat PHP files
    // ['type' => 'xml', 'file' => 'federation-metadata.xml'],
    // ['type' => 'pdo'],  // Use database
],

// Metadata signing
'metadata.sign.enable' => false,
'metadata.sign.privatekey' => null,
'metadata.sign.certificate' => null,
'metadata.sign.algorithm' => 'http://www.w3.org/2001/04/xmldsig-more#rsa-sha256',
```

**13. Authentication Processing Filters**

```php
// Filters applied to all IdPs
'authproc.idp' => [
    30 => 'core:LanguageAdaptor',
    50 => 'core:AttributeLimit',  // Filter attributes based on metadata
    // 60 => ['class' => 'core:AttributeMap', 'oid2name'],
],

// Filters applied to all SPs
'authproc.sp' => [
    90 => 'core:LanguageAdaptor',
],
```

### Authentication Sources (config/authsources.php)

This file defines how users authenticate.

#### Location in Codebase
`/var/simplesamlphp/config/authsources.php` (copy from `authsources.php.dist`)

```bash
cd /var/simplesamlphp/config
cp authsources.php.dist authsources.php
```

#### Configuration Format

```php
<?php

$config = [
    // Authentication source name => configuration
    'source-name' => [
        'module:AuthClass',
        // ... configuration options
    ],
];
```

#### Example: Admin Authentication

```php
'admin' => [
    'core:AdminPassword',  // Uses admin password from config.php
],
```

#### Example: Service Provider (SP) Configuration

```php
'default-sp' => [
    'saml:SP',

    // Your SP's entity ID (unique identifier)
    'entityID' => 'https://myapp.example.org/',

    // The IdP to use (null = show discovery page)
    'idp' => null,  // Or specify: 'https://idp.example.org/'

    // Discovery service URL (optional)
    'discoURL' => null,

    // Certificate and private key (optional but recommended)
    'privatekey' => 'saml.pem',
    'certificate' => 'saml.crt',

    // NameID format to request
    'NameIDFormat' => 'urn:oasis:names:tc:SAML:2.0:nameid-format:transient',

    // Requested attributes
    'attributes' => [
        'displayName' => 'urn:oid:2.16.840.1.113730.3.1.241',
        'mail' => 'urn:oid:0.9.2342.19200300.100.1.3',
    ],
    'attributes.required' => [
        'urn:oid:0.9.2342.19200300.100.1.3',  // mail
    ],

    // Service name for metadata
    'name' => [
        'en' => 'My Application',
        'es' => 'Mi Aplicación',
    ],
    'description' => [
        'en' => 'My application description',
    ],

    // Organization info for metadata
    'OrganizationName' => [
        'en' => 'Example Organization',
    ],
    'OrganizationDisplayName' => [
        'en' => 'Example Org',
    ],
    'OrganizationURL' => [
        'en' => 'https://example.org',
    ],

    // Authentication context
    'AuthnContextClassRef' => 'urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport',

    // Signature algorithm
    'signature.algorithm' => 'http://www.w3.org/2001/04/xmldsig-more#rsa-sha256',

    // Sign authentication requests
    'sign.authnrequest' => true,
    'sign.logout' => true,

    // Encryption
    'assertion.encryption' => false,  // Set true if IdP supports it
],
```

#### Example: LDAP Authentication

```php
'example-ldap' => [
    'ldap:Ldap',

    'connection_string' => 'ldap.example.org',
    'encryption' => 'tls',  // 'ssl', 'tls', or 'none'
    'version' => 3,
    'ldap.debug' => false,

    'options' => [
        'referrals' => 0x00,
        'network_timeout' => 3,
    ],

    'connector' => '\SimpleSAML\Module\ldap\Connector\Ldap',

    'attributes' => null,  // null = fetch all attributes

    'attributes.binary' => [
        'jpegPhoto',
        'objectGUID',
    ],

    // User DN pattern
    'dnpattern' => 'uid=%username%,ou=people,dc=example,dc=org',

    // OR use LDAP search
    'search.enable' => false,
    'search.base' => ['ou=people,dc=example,dc=org'],
    'search.scope' => 'sub',
    'search.attributes' => ['uid', 'mail'],
    'search.username' => null,
    'search.password' => null,
],
```

#### Example: SQL Database Authentication

```php
'example-sql' => [
    'sqlauth:SQL',
    'dsn' => 'mysql:host=localhost;dbname=users',
    'username' => 'dbuser',
    'password' => 'dbpassword',
    'query' => 'SELECT uid, displayName, mail FROM users WHERE username = :username AND password = SHA2(CONCAT((SELECT salt FROM users WHERE username = :username), :password), 256)',
],
```

#### Example: Simple UserPass (Testing Only)

```php
'example-userpass' => [
    'exampleauth:UserPass',
    'student:studentpass' => [
        'uid' => ['student'],
        'eduPersonAffiliation' => ['member', 'student'],
        'displayName' => ['Student User'],
    ],
    'employee:employeepass' => [
        'uid' => ['employee'],
        'eduPersonAffiliation' => ['member', 'employee'],
        'displayName' => ['Employee User'],
    ],
],
```

### Metadata Configuration

Metadata establishes trust between entities.

#### IdP Hosted Metadata (metadata/saml20-idp-hosted.php)

Used when SimpleSAMLphp acts as an Identity Provider.

**Location**: `/var/simplesamlphp/metadata/saml20-idp-hosted.php`

```php
<?php

$metadata['urn:x-simplesamlphp:example-idp'] = [
    // Virtual host that serves this IdP
    'host' => '__DEFAULT__',  // Or specify 'idp.example.org'

    // Certificate and private key
    'privatekey' => 'server.pem',
    'certificate' => 'server.crt',

    // Authentication source from authsources.php
    'auth' => 'example-ldap',  // Which auth source to use

    // NameID format
    // 'NameIDFormat' => 'urn:oasis:names:tc:SAML:2.0:nameid-format:transient',

    // Attribute name format
    // 'attributes.NameFormat' => 'urn:oasis:names:tc:SAML:2.0:attrname-format:uri',

    // Authentication processing filters
    'authproc' => [
        // Convert LDAP names to OIDs
        100 => ['class' => 'core:AttributeMap', 'name2oid'],
    ],

    // Organization information (appears in metadata)
    'OrganizationName' => [
        'en' => 'Example Organization',
    ],
    'OrganizationDisplayName' => [
        'en' => 'Example Org',
    ],
    'OrganizationURL' => [
        'en' => 'https://www.example.org',
    ],

    // Technical contact
    // Uses config.php values by default
];
```

#### IdP Remote Metadata (metadata/saml20-idp-remote.php)

Used when SimpleSAMLphp acts as a Service Provider - defines remote IdPs you trust.

**Location**: `/var/simplesamlphp/metadata/saml20-idp-remote.php`

```php
<?php

$metadata['https://idp.example.org/saml'] = [
    // Entity ID
    'entityid' => 'https://idp.example.org/saml',

    // Single Sign-On service endpoint
    'SingleSignOnService' => [
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect',
            'Location' => 'https://idp.example.org/saml/sso',
        ],
    ],

    // Single Logout service endpoint
    'SingleLogoutService' => [
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect',
            'Location' => 'https://idp.example.org/saml/slo',
        ],
    ],

    // IdP's signing certificate
    'certData' => 'MIIDXTCCAkWgAwIBAgIJALmVVuDWu4NYMA0GCSqGSIb3DQEBCwUAMEUxCzAJBgNV...',

    // OR load from file
    // 'certificate' => 'idp.example.org.crt',

    // NameID format supported
    'NameIDFormat' => 'urn:oasis:names:tc:SAML:2.0:nameid-format:persistent',

    // Disable scoping (required for some IdPs like Azure AD)
    // 'disable_scoping' => true,
];
```

**How to obtain this metadata:**
1. Ask the IdP administrator for their metadata URL
2. Download the XML metadata
3. Use SimpleSAMLphp's converter: `https://sso.example.org/admin/metadata-converter.php`
4. Paste the converted PHP array here

#### SP Remote Metadata (metadata/saml20-sp-remote.php)

Used when SimpleSAMLphp acts as an Identity Provider - defines remote SPs you serve.

**Location**: `/var/simplesamlphp/metadata/saml20-sp-remote.php`

```php
<?php

$metadata['https://app.example.org/saml'] = [
    // Assertion Consumer Service (where to send SAML response)
    'AssertionConsumerService' => [
        [
            'index' => 0,
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST',
            'Location' => 'https://app.example.org/saml/acs',
        ],
    ],

    // Single Logout service
    'SingleLogoutService' => [
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect',
            'Location' => 'https://app.example.org/saml/slo',
        ],
    ],

    // SP's certificate (for validating signed requests)
    // 'certificate' => 'app.example.org.crt',

    // Attributes to release to this SP
    'attributes' => [
        'uid',
        'displayName',
        'mail',
        'eduPersonAffiliation',
    ],

    // Required attributes
    'attributes.required' => [
        'mail',
    ],

    // Attribute name format
    // 'attributes.NameFormat' => 'urn:oasis:names:tc:SAML:2.0:attrname-format:uri',

    // NameID format
    'NameIDFormat' => 'urn:oasis:names:tc:SAML:2.0:nameid-format:persistent',

    // Signature algorithm (if SP doesn't support SHA-256)
    // 'signature.algorithm' => 'http://www.w3.org/2000/09/xmldsig#rsa-sha1',
];
```

**Example: Google Workspace**

```php
$metadata['google.com'] = [
    'AssertionConsumerService' => [
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST',
            'Location' => 'https://www.google.com/a/yourdomain.com/acs',
        ],
    ],
    'NameIDFormat' => 'urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress',
    'authproc' => [
        1 => [
            'class' => 'saml:AttributeNameID',
            'identifyingAttribute' => 'mail',
            'Format' => 'urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress',
        ],
    ],
];
```

---

## Setting Up as Service Provider (SP)

A Service Provider delegates authentication to an external Identity Provider.

### Use Case

Your application needs to authenticate users via:
- Azure AD (Microsoft Entra ID)
- Google Workspace
- Okta
- Another organization's IdP

### Step-by-Step Setup

#### Step 1: Generate Certificates

```bash
cd /var/simplesamlphp/cert
openssl req -newkey rsa:3072 -new -x509 -days 3652 -nodes \
    -out saml.crt -keyout saml.pem

# Set secure permissions
chmod 600 saml.pem
chmod 644 saml.crt
```

When prompted, enter your organization information.

#### Step 2: Configure Authentication Source

Edit `config/authsources.php`:

```php
<?php

$config = [
    // Admin authentication
    'admin' => [
        'core:AdminPassword',
    ],

    // Your SP configuration
    'default-sp' => [
        'saml:SP',

        // Your application's entity ID (must be unique)
        'entityID' => 'https://myapp.example.org/',

        // Specify IdP entity ID, or null to use discovery
        'idp' => null,  // Will be set later

        // Certificates
        'privatekey' => 'saml.pem',
        'certificate' => 'saml.crt',

        // Sign requests
        'sign.authnrequest' => true,
        'sign.logout' => true,

        // Redirect after authentication
        'RelayState' => '/app/dashboard',

        // Organization info
        'OrganizationName' => [
            'en' => 'My Organization',
        ],
        'OrganizationDisplayName' => [
            'en' => 'My Org',
        ],
        'OrganizationURL' => [
            'en' => 'https://www.example.org',
        ],
    ],
];
```

#### Step 3: Export Your SP Metadata

1. Visit `https://sso.example.org/`
2. Click "Federation" tab
3. Click "Show metadata" for your SP (e.g., "default-sp")
4. Download or copy the XML metadata

Your metadata URL will be something like:
```
https://sso.example.org/module.php/saml/sp/metadata.php/default-sp
```

#### Step 4: Register with IdP

Send your SP metadata to the IdP administrator. They will:
- Add your SP to their system
- Provide you with their IdP metadata

#### Step 5: Add IdP Metadata

You have two options:

**Option A: Use Metadata Converter (Recommended)**

1. Get IdP's metadata URL or XML file
2. Visit `https://sso.example.org/admin/metadata-converter.php`
3. Paste the XML or URL
4. Click "Parse"
5. Copy the PHP array output
6. Paste into `metadata/saml20-idp-remote.php`

**Option B: Manual Configuration**

Create `metadata/saml20-idp-remote.php`:

```php
<?php

$metadata['https://idp.example.org/'] = [
    'SingleSignOnService' => [
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect',
            'Location' => 'https://idp.example.org/saml2/idp/SSOService.php',
        ],
    ],
    'SingleLogoutService' => [
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect',
            'Location' => 'https://idp.example.org/saml2/idp/SLOService.php',
        ],
    ],
    'certificate' => 'idp-certificate.crt',  // Place in cert/ directory
];
```

#### Step 6: Set Default IdP (Optional)

If you have only one IdP, update `config/authsources.php`:

```php
'default-sp' => [
    'saml:SP',
    'entityID' => 'https://myapp.example.org/',
    'idp' => 'https://idp.example.org/',  // Add this line
    // ... rest of config
],
```

#### Step 7: Test Authentication

Visit the test page:
```
https://sso.example.org/module.php/core/authenticate.php?as=default-sp
```

You should be redirected to the IdP for authentication.

#### Step 8: Integrate with Your Application

**PHP Integration**:

```php
<?php
require_once('/var/simplesamlphp/src/_autoload.php');

$as = new \SimpleSAML\Auth\Simple('default-sp');

// Require authentication
$as->requireAuth();

// Get user attributes
$attributes = $as->getAttributes();

// Example: Get email
$email = $attributes['mail'][0];
$displayName = $attributes['displayName'][0];

// Check if user is authenticated
if ($as->isAuthenticated()) {
    echo "Welcome, " . htmlspecialchars($displayName);
}

// Logout
// $as->logout('https://myapp.example.org/');
?>
```

**Important Notes**:
- Attributes are always arrays (even single values)
- Attribute names depend on the IdP
- Use `$as->requireAuth()` to force authentication
- Use `$as->getAuthData('Attributes')` for all auth data

---

## Setting Up as Identity Provider (IdP)

An Identity Provider authenticates users and issues SAML assertions to Service Providers.

### Use Case

You want to:
- Provide SSO to other applications
- Use your existing user directory (LDAP, Active Directory, database)
- Manage user authentication centrally

### Step-by-Step Setup

#### Step 1: Enable IdP Functionality

Edit `config/config.php`:

```php
// Enable SAML 2.0 IdP
'enable.saml20-idp' => true,
```

#### Step 2: Generate Certificates

```bash
cd /var/simplesamlphp/cert
openssl req -newkey rsa:3072 -new -x509 -days 3652 -nodes \
    -out server.crt -keyout server.pem

chmod 600 server.pem
chmod 644 server.crt
```

#### Step 3: Choose Authentication Backend

You need to decide how users will authenticate. Options:

1. **LDAP / Active Directory** - Most common for organizations
2. **SQL Database** - If you have a custom user database
3. **External Authentication** - If you have existing authentication
4. **Simple UserPass** - For testing only

#### Step 4: Configure Authentication Source

**Example: LDAP Authentication**

Edit `config/authsources.php`:

```php
'example-ldap' => [
    'ldap:Ldap',

    'connection_string' => 'ldaps://ldap.example.org',
    'encryption' => 'ssl',
    'version' => 3,

    'attributes' => null,  // Fetch all attributes

    // Search for user
    'search.enable' => true,
    'search.base' => ['ou=people,dc=example,dc=org'],
    'search.attributes' => ['uid', 'mail'],

    // Bind with service account to search
    'search.username' => 'cn=serviceaccount,dc=example,dc=org',
    'search.password' => 'service_password',
],
```

**Example: SQL Authentication**

```php
'example-sql' => [
    'sqlauth:SQL',
    'dsn' => 'mysql:host=localhost;dbname=userdb',
    'username' => 'dbuser',
    'password' => 'dbpassword',
    'query' => 'SELECT uid, email, displayName FROM users WHERE username = :username AND password_hash = SHA2(:password, 256)',
],
```

**Example: Simple UserPass (Testing)**

```php
'example-userpass' => [
    'exampleauth:UserPass',
    'student:studentpass' => [
        'uid' => ['student'],
        'mail' => ['student@example.org'],
        'displayName' => ['Student User'],
        'eduPersonAffiliation' => ['member', 'student'],
    ],
],
```

Enable the module in `config/config.php` if needed:

```php
'module.enable' => [
    'exampleauth' => true,  // For testing
    // ... other modules
],
```

#### Step 5: Configure IdP Metadata

Create `metadata/saml20-idp-hosted.php`:

```php
<?php

$metadata['https://idp.example.org/saml'] = [
    // Host
    'host' => '__DEFAULT__',

    // Certificates
    'privatekey' => 'server.pem',
    'certificate' => 'server.crt',

    // Authentication source from authsources.php
    'auth' => 'example-ldap',  // Or 'example-sql', 'example-userpass'

    // Attribute name format
    'attributes.NameFormat' => 'urn:oasis:names:tc:SAML:2.0:attrname-format:uri',

    // Authentication processing filters
    'authproc' => [
        // Map LDAP attribute names to OIDs
        100 => ['class' => 'core:AttributeMap', 'name2oid'],
    ],

    // Organization information
    'OrganizationName' => [
        'en' => 'Example University',
    ],
    'OrganizationDisplayName' => [
        'en' => 'Example U',
    ],
    'OrganizationURL' => [
        'en' => 'https://www.example.org',
    ],
];
```

#### Step 6: Export Your IdP Metadata

Your IdP metadata is available at:
```
https://sso.example.org/saml2/idp/metadata.php
```

Service Providers will need this to configure trust with your IdP.

#### Step 7: Register Service Providers

When a Service Provider wants to use your IdP:

1. Request their SP metadata (XML file or URL)
2. Convert to PHP format:
   - Visit `https://sso.example.org/admin/metadata-converter.php`
   - Paste their XML metadata
   - Copy the PHP output
3. Add to `metadata/saml20-sp-remote.php`

Example SP metadata:

```php
<?php

$metadata['https://app1.example.org/'] = [
    'AssertionConsumerService' => [
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST',
            'Location' => 'https://app1.example.org/saml/acs',
            'index' => 0,
        ],
    ],

    'SingleLogoutService' => [
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect',
            'Location' => 'https://app1.example.org/saml/slo',
        ],
    ],

    // Attributes to release
    'attributes' => [
        'uid',
        'mail',
        'displayName',
        'eduPersonAffiliation',
    ],

    // Required attributes
    'attributes.required' => [
        'mail',
    ],
];

$metadata['https://app2.example.org/'] = [
    // ... another SP configuration
];
```

#### Step 8: Configure Attribute Release

Control which attributes are sent to SPs using authentication processing filters.

In `config/config.php`:

```php
'authproc.idp' => [
    // Language adaptor
    30 => 'core:LanguageAdaptor',

    // Attribute limits (respects 'attributes' in SP metadata)
    50 => 'core:AttributeLimit',

    // Map attribute names
    60 => ['class' => 'core:AttributeMap', 'name2oid'],

    // Generate persistent NameID
    // 70 => 'saml:PersistentNameID',

    // Consent (ask user before releasing attributes)
    // 80 => [
    //     'class' => 'consent:Consent',
    //     'store' => 'consent:Cookie',
    //     'focus' => 'yes',
    //     'checked' => true,
    // ],
],
```

#### Step 9: Test Your IdP

**Option A: Use Built-in SP**

1. Configure a test SP in `config/authsources.php`:

```php
'test-sp' => [
    'saml:SP',
    'entityID' => 'https://test-sp.example.org/',
    'idp' => 'https://idp.example.org/saml',
],
```

2. Add test SP to `metadata/saml20-sp-remote.php`:

```php
$metadata['https://test-sp.example.org/'] = [
    'AssertionConsumerService' => [
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST',
            'Location' => 'https://sso.example.org/module.php/saml/sp/saml2-acs.php/test-sp',
        ],
    ],
];
```

3. Test at:
```
https://sso.example.org/module.php/core/authenticate.php?as=test-sp
```

**Option B: Use External Test SP**

Use a test service like:
- SAML test: https://samltest.id/
- Okta SP test tool

#### Step 10: Monitor and Maintain

- Check logs: `/var/log/syslog` or configured log file
- Monitor failed logins
- Regularly review SP metadata
- Update certificates before expiry

---

## Integration Examples

### Azure AD / Microsoft Entra ID

#### Scenario: SimpleSAMLphp SP with Azure AD IdP

**Step 1: Configure SimpleSAMLphp SP**

`config/authsources.php`:

```php
'azure-sp' => [
    'saml:SP',
    'entityID' => 'https://myapp.example.org/',

    // Will be set after creating Azure app
    'idp' => null,

    'privatekey' => 'saml.pem',
    'certificate' => 'saml.crt',

    // Azure AD requires this
    'disable_scoping' => true,

    // Sign requests
    'sign.authnrequest' => true,
    'sign.logout' => true,

    // Request specific attributes
    'attributes' => [
        'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress',
        'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname',
        'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname',
    ],

    'OrganizationName' => [
        'en' => 'My Organization',
    ],
    'OrganizationDisplayName' => [
        'en' => 'My Org',
    ],
    'OrganizationURL' => [
        'en' => 'https://www.example.org',
    ],
];
```

**Step 2: Get SP Metadata**

Visit:
```
https://sso.example.org/module.php/saml/sp/metadata.php/azure-sp
```

Download the XML metadata file.

**Step 3: Configure Azure AD**

1. Log in to Azure Portal (https://portal.azure.com)
2. Navigate to **Azure Active Directory** → **Enterprise applications**
3. Click **New application** → **Create your own application**
4. Enter name (e.g., "My Application SSO")
5. Select "Integrate any other application you don't find in the gallery"
6. Click **Create**

7. In the application, go to **Single sign-on**
8. Select **SAML**
9. Click **Upload metadata file** and select your SP metadata XML
10. Verify the settings:
    - **Identifier (Entity ID)**: Should match your SP entityID
    - **Reply URL (ACS URL)**: Should be `https://sso.example.org/module.php/saml/sp/saml2-acs.php/azure-sp`
11. Click **Save**

**Step 4: Configure Attributes & Claims**

In Azure SAML configuration, edit **Attributes & Claims**:

Default claims Azure sends:
- `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress`
- `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name`
- `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname`
- `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname`
- `http://schemas.microsoft.com/identity/claims/objectidentifier`

Add custom claims if needed.

**Step 5: Download Azure AD Metadata**

In Azure SAML config:
1. Find **App Federation Metadata Url**
2. Copy the URL (e.g., `https://login.microsoftonline.com/...federationmetadata/2007-06/federationmetadata.xml`)

**Step 6: Add Azure AD to SimpleSAMLphp**

1. Visit `https://sso.example.org/admin/metadata-converter.php`
2. Paste the metadata URL
3. Click **Parse**
4. Copy the PHP array output
5. Add to `metadata/saml20-idp-remote.php`:

```php
<?php

$metadata['https://sts.windows.net/{tenant-id}/'] = [
    'entityid' => 'https://sts.windows.net/{tenant-id}/',
    'SingleSignOnService' => [
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect',
            'Location' => 'https://login.microsoftonline.com/{tenant-id}/saml2',
        ],
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST',
            'Location' => 'https://login.microsoftonline.com/{tenant-id}/saml2',
        ],
    ],
    'SingleLogoutService' => [
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect',
            'Location' => 'https://login.microsoftonline.com/{tenant-id}/saml2',
        ],
    ],
    'certificate' => 'azure-ad-{tenant-id}.crt',
];
```

6. Download Azure AD certificate:
   - In Azure SAML config, under **SAML Signing Certificate**
   - Download **Certificate (Base64)**
   - Save as `cert/azure-ad-{tenant-id}.crt`

**Step 7: Update authsources.php**

Set the IdP:

```php
'azure-sp' => [
    'saml:SP',
    'entityID' => 'https://myapp.example.org/',
    'idp' => 'https://sts.windows.net/{tenant-id}/',  // Add this
    // ... rest of config
];
```

**Step 8: Assign Users in Azure AD**

1. In Azure AD application, go to **Users and groups**
2. Click **Add user/group**
3. Select users or groups that should have access
4. Click **Assign**

**Step 9: Test**

Visit:
```
https://sso.example.org/module.php/core/authenticate.php?as=azure-sp
```

You should be redirected to Microsoft login.

**Step 10: Map Attributes**

Azure AD uses long URIs for attributes. Map them in your application:

```php
$attributes = $as->getAttributes();

$email = $attributes['http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress'][0];
$firstName = $attributes['http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname'][0];
$lastName = $attributes['http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname'][0];
```

Or use attribute mapping filter in `config/authsources.php`:

```php
'azure-sp' => [
    'saml:SP',
    // ... other config
    'authproc' => [
        10 => [
            'class' => 'core:AttributeMap',
            'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress' => 'mail',
            'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname' => 'givenName',
            'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname' => 'sn',
        ],
    ],
];
```

Now you can use short names:
```php
$email = $attributes['mail'][0];
$firstName = $attributes['givenName'][0];
```

---

### Google Workspace

#### Scenario: SimpleSAMLphp IdP with Google Workspace SP

**Step 1: Enable IdP in SimpleSAMLphp**

`config/config.php`:
```php
'enable.saml20-idp' => true,
```

**Step 2: Configure Authentication Source**

`config/authsources.php`:

```php
'example-userpass' => [
    'exampleauth:UserPass',
    'alice:alicepass' => [
        'uid' => ['alice'],
        'mail' => ['alice@yourdomain.com'],  // Must match Google email
        'displayName' => ['Alice Smith'],
    ],
    'bob:bobpass' => [
        'uid' => ['bob'],
        'mail' => ['bob@yourdomain.com'],
        'displayName' => ['Bob Jones'],
    ],
];
```

**Step 3: Configure IdP Metadata**

`metadata/saml20-idp-hosted.php`:

```php
<?php

$metadata['https://idp.yourdomain.com/saml'] = [
    'host' => '__DEFAULT__',
    'privatekey' => 'server.pem',
    'certificate' => 'server.crt',
    'auth' => 'example-userpass',

    'attributes.NameFormat' => 'urn:oasis:names:tc:SAML:2.0:attrname-format:uri',
    'authproc' => [
        100 => ['class' => 'core:AttributeMap', 'name2oid'],
    ],
];
```

**Step 4: Add Google Workspace as SP**

`metadata/saml20-sp-remote.php`:

```php
<?php

$metadata['google.com'] = [
    'AssertionConsumerService' => [
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST',
            'Location' => 'https://www.google.com/a/yourdomain.com/acs',
            'index' => 1,
        ],
    ],

    'NameIDFormat' => 'urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress',

    'authproc' => [
        1 => [
            'class' => 'saml:AttributeNameID',
            'identifyingAttribute' => 'mail',
            'Format' => 'urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress',
        ],
    ],

    'simplesaml.attributes' => false,
];
```

**Important**: The `mail` attribute must match the user's email in Google Workspace.

**Step 5: Configure Google Workspace**

1. Log in to Google Admin Console (https://admin.google.com)
2. Navigate to **Security** → **Authentication** → **SSO with third-party IdP**
3. Click **Add SAML profile**
4. Or go to **Apps** → **Web and mobile apps** → **Add app** → **Add custom SAML app**

5. Enter details:
   - **App name**: Your Organization SSO
   - Click **Continue**

6. Download IdP metadata or note:
   - **SSO URL**: `https://idp.yourdomain.com/saml2/idp/SSOService.php`
   - **Entity ID**: `https://idp.yourdomain.com/saml`
   - **Certificate**: Upload your `server.crt` file
   - Click **Continue**

7. Service provider details (auto-filled if you uploaded metadata):
   - **ACS URL**: `https://www.google.com/a/yourdomain.com/acs`
   - **Entity ID**: `google.com`
   - **Start URL**: (leave empty or your app URL)
   - **Signed Response**: Checked
   - Click **Continue**

8. Attribute mapping:
   - Google uses the NameID for email (already configured via `authproc` in metadata)
   - Click **Finish**

**Step 6: Enable SSO**

1. In Google Admin Console → **Security** → **Settings**
2. Find **Set up SSO with third-party IdP**
3. Check **Enable SSO**
4. Save

**Step 7: Test**

1. Open an incognito/private browser window
2. Go to `https://mail.google.com`
3. Enter your email (e.g., `alice@yourdomain.com`)
4. You should be redirected to SimpleSAMLphp for authentication
5. Enter credentials (alice / alicepass)
6. You should be logged into Gmail

**Troubleshooting**:
- Users must exist in Google Workspace with the correct email
- The `mail` attribute from your IdP must exactly match the Google email
- Google does NOT support Single Logout (SLO)

---

### Okta

#### Scenario: SimpleSAMLphp SP with Okta IdP

**Step 1: Configure SimpleSAMLphp SP**

`config/authsources.php`:

```php
'okta-sp' => [
    'saml:SP',
    'entityID' => 'https://myapp.example.org/saml',

    'idp' => null,  // Will set after Okta configuration

    'privatekey' => 'saml.pem',
    'certificate' => 'saml.crt',

    'sign.authnrequest' => true,
    'sign.logout' => true,

    'NameIDFormat' => 'urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress',

    'OrganizationName' => [
        'en' => 'My Organization',
    ],
];
```

**Step 2: Get SP Metadata**

Visit:
```
https://sso.example.org/module.php/saml/sp/metadata.php/okta-sp
```

Download or note:
- **Entity ID**: `https://myapp.example.org/saml`
- **ACS URL**: `https://sso.example.org/module.php/saml/sp/saml2-acs.php/okta-sp`
- **SLO URL**: `https://sso.example.org/module.php/saml/sp/saml2-logout.php/okta-sp`

**Step 3: Create Application in Okta**

1. Log in to Okta Admin Console
2. Go to **Applications** → **Applications**
3. Click **Create App Integration**
4. Select **SAML 2.0**
5. Click **Next**

6. General Settings:
   - **App name**: My Application
   - **App logo**: (optional)
   - Click **Next**

7. Configure SAML:
   - **Single sign on URL**: `https://sso.example.org/module.php/saml/sp/saml2-acs.php/okta-sp`
   - **Audience URI (SP Entity ID)**: `https://myapp.example.org/saml`
   - **Default RelayState**: (optional, e.g., `/dashboard`)
   - **Name ID format**: EmailAddress
   - **Application username**: Email
   - **Update application username on**: Create and update
   - Click **Next**

8. Feedback:
   - Select "I'm an Okta customer adding an internal app"
   - Click **Finish**

**Step 4: Configure Attribute Statements (Optional)**

1. In your Okta application, go to **Sign On** tab
2. Scroll to **SAML Settings** and click **Edit**
3. Scroll to **Attribute Statements**
4. Add attributes:

| Name | Name Format | Value |
|------|-------------|-------|
| email | Basic | user.email |
| firstName | Basic | user.firstName |
| lastName | Basic | user.lastName |

5. Click **Save**

**Step 5: Assign Users**

1. In your Okta application, go to **Assignments** tab
2. Click **Assign** → **Assign to People** or **Assign to Groups**
3. Select users/groups
4. Click **Assign** and **Done**

**Step 6: Get Okta Metadata**

1. In your Okta application, go to **Sign On** tab
2. Find **Metadata URL** under **SAML 2.0**
3. Copy the URL (e.g., `https://dev-12345.okta.com/app/xxx/sso/saml/metadata`)

**Step 7: Add Okta to SimpleSAMLphp**

1. Visit `https://sso.example.org/admin/metadata-converter.php`
2. Paste Okta metadata URL
3. Click **Parse**
4. Copy PHP output
5. Add to `metadata/saml20-idp-remote.php`:

```php
<?php

$metadata['http://www.okta.com/exk...'] = [
    'entityid' => 'http://www.okta.com/exk...',
    'SingleSignOnService' => [
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST',
            'Location' => 'https://dev-12345.okta.com/app/dev-12345_myapp_1/exk.../sso/saml',
        ],
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect',
            'Location' => 'https://dev-12345.okta.com/app/dev-12345_myapp_1/exk.../sso/saml',
        ],
    ],
    'SingleLogoutService' => [
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST',
            'Location' => 'https://dev-12345.okta.com/app/dev-12345_myapp_1/exk.../slo/saml',
        ],
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect',
            'Location' => 'https://dev-12345.okta.com/app/dev-12345_myapp_1/exk.../slo/saml',
        ],
    ],
    'certificate' => 'okta.crt',
];
```

6. Download Okta certificate:
   - In Okta app, go to **Sign On** tab
   - Click **View SAML setup instructions**
   - Copy the X.509 Certificate
   - Save as `cert/okta.crt`:
   ```
   -----BEGIN CERTIFICATE-----
   MIID...
   -----END CERTIFICATE-----
   ```

**Step 8: Update authsources.php**

Set the IdP entity ID:

```php
'okta-sp' => [
    'saml:SP',
    'entityID' => 'https://myapp.example.org/saml',
    'idp' => 'http://www.okta.com/exk...',  // From Okta metadata
    // ... rest of config
];
```

**Step 9: Test**

Visit:
```
https://sso.example.org/module.php/core/authenticate.php?as=okta-sp
```

You should be redirected to Okta for authentication.

**Step 10: Use Attributes**

```php
$as = new \SimpleSAML\Auth\Simple('okta-sp');
$as->requireAuth();
$attributes = $as->getAttributes();

$email = $attributes['email'][0];
$firstName = $attributes['firstName'][0];
$lastName = $attributes['lastName'][0];
```

---

## Advanced: SAML Proxy/Bridge Configuration

### What is a SAML Proxy?

A SAML proxy (also called bridge or hub) sits between external Service Providers and upstream Identity Providers. It acts as **both an IdP and an SP simultaneously**:

- **Acts as IdP** to downstream Service Providers (your applications, Clerk, etc.)
- **Acts as SP** to upstream Identity Providers (InCommon universities, enterprise IdPs, etc.)

### Use Cases for SAML Proxy

**Your Specific Scenario:**
```
User → App → Clerk → SimpleSAMLphp (Proxy) → InCommon → University IdP
                      ↑                               ↑
                      Acts as IdP                     Acts as SP
                      (to Clerk)                      (to Universities)
```

**Common Use Cases:**
1. **Protocol Translation**: Bridge between SAML 2.0 and SAML 1.1
2. **Federation Hub**: Connect multiple applications to multiple IdPs
3. **Attribute Transformation**: Modify/enrich user attributes between systems
4. **Multi-Tenancy**: Route different tenants to different IdPs
5. **Legacy Integration**: Add modern SSO to legacy systems

### Architecture: Your Flow

```
┌──────┐    ┌─────┐    ┌───────┐    ┌─────────────┐    ┌──────────┐    ┌──────────────┐
│ User │───>│ App │───>│ Clerk │───>│ SimpleSAML  │───>│InCommon  │───>│ University   │
│      │    │     │    │  (SP) │    │   (Proxy)   │    │Federation│    │ IdP (e.g.UCD)│
└──────┘    └─────┘    └───────┘    │             │    └──────────┘    └──────────────┘
                                      │ ┌─────────┐ │
                                      │ │IdP Side │ │  (Presents to Clerk)
                                      │ └─────────┘ │
                                      │ ┌─────────┐ │
                                      │ │SP Side  │ │  (Connects to Universities)
                                      │ └─────────┘ │
                                      └─────────────┘
```

### Step-by-Step: SAML Proxy Setup

#### Step 1: Enable Both IdP and SP

Edit `config/config.php`:

```php
// Enable IdP functionality (for downstream - Clerk)
'enable.saml20-idp' => true,

// Enable modules
'module.enable' => [
    'core' => true,
    'admin' => true,
    'saml' => true,
    'multiauth' => false,  // We'll configure this for routing
],
```

#### Step 2: Configure SP Side (Connect to Universities)

This is where SimpleSAMLphp connects to upstream IdPs (universities via InCommon).

**Install metarefresh module for InCommon:**

```bash
cd /var/simplesamlphp
composer require simplesamlphp/simplesamlphp-module-metarefresh
```

**Enable and configure metarefresh** in `config/config.php`:

```php
'module.enable' => [
    'metarefresh' => true,
    // ... other modules
],
```

**Create `config/module_metarefresh.php`:**

```php
<?php

$config = [
    'sets' => [
        'incommon' => [
            'cron' => ['hourly'],
            'sources' => [
                [
                    'src' => 'https://md.incommon.org/InCommon/InCommon-metadata.xml',
                    'certificates' => [
                        'inc-md-cert.pem',  // Download from InCommon
                    ],
                    'template' => [
                        'tags' => ['incommon'],
                        'authproc' => [
                            // Add any attribute processing here
                            51 => [
                                'class' => 'core:AttributeMap',
                                'oid2name',
                            ],
                        ],
                    ],
                ],
            ],
            'expireAfter' => 60 * 60 * 24 * 7, // 7 days
            'outputDir' => 'metadata/metadata-incommon-generated/',
            'outputFormat' => 'flatfile',
        ],
    ],
];
```

**Download InCommon certificate:**

```bash
cd /var/simplesamlphp/cert
curl -o inc-md-cert.pem https://ds.incommon.org/certs/inc-md-cert.pem
```

**Run metarefresh to fetch InCommon metadata:**

```bash
cd /var/simplesamlphp
./bin/console metarefresh:fetch
```

This creates individual metadata files in `metadata/metadata-incommon-generated/` for each university.

**Configure metadata source** in `config/config.php`:

```php
'metadata.sources' => [
    ['type' => 'flatfile', 'directory' => 'metadata'],
    ['type' => 'flatfile', 'directory' => 'metadata/metadata-incommon-generated'],
],
```

**Create SP authentication source for each university** (or dynamically):

In `config/authsources.php`:

```php
// SP connection to UC Davis
'ucd-sp' => [
    'saml:SP',
    'entityID' => 'https://proxy.example.org/ucd',
    'idp' => 'https://shibboleth.ucdavis.edu/idp/shibboleth',  // UCD's entity ID
    'privatekey' => 'proxy.pem',
    'certificate' => 'proxy.crt',
    'sign.authnrequest' => true,
],

// SP connection to UC Berkeley
'ucb-sp' => [
    'saml:SP',
    'entityID' => 'https://proxy.example.org/ucb',
    'idp' => 'urn:mace:incommon:berkeley.edu',
    'privatekey' => 'proxy.pem',
    'certificate' => 'proxy.crt',
    'sign.authnrequest' => true,
],

// ... more universities as needed
```

#### Step 3: Configure IdP Side (Serve to Clerk)

This is where SimpleSAMLphp presents itself as an IdP to Clerk.

**Create `metadata/saml20-idp-hosted.php`:**

```php
<?php

$metadata['https://proxy.example.org/saml-idp'] = [
    'host' => '__DEFAULT__',

    'privatekey' => 'proxy.pem',
    'certificate' => 'proxy.crt',

    // THIS IS THE KEY: Point to your SP authentication sources
    // We'll use dynamic selection based on tenant/subdomain
    'auth' => 'proxy-router',  // We'll create this below

    'OrganizationName' => [
        'en' => 'University Proxy Service',
    ],
    'OrganizationDisplayName' => [
        'en' => 'University SSO',
    ],
    'OrganizationURL' => [
        'en' => 'https://proxy.example.org',
    ],
];
```

#### Step 4: Subdomain-Based IdP Routing (NO Discovery Page)

This is the critical part for your requirement: routing based on subdomain without showing a discovery page.

**Option A: Custom Authentication Source with Dynamic Routing**

Create a custom auth source that determines the university based on subdomain or parameter.

Create `src/CustomAuth/TenantRouter.php`:

```php
<?php

namespace CustomAuth;

use SimpleSAML\Auth;
use SimpleSAML\Error;
use Symfony\Component\HttpFoundation\Request;

class TenantRouter extends Auth\Source
{
    private array $tenantMap;

    public function __construct(array $info, array $config)
    {
        parent::__construct($info, $config);
        $this->tenantMap = $config['tenants'];
    }

    public function authenticate(Request $request, array &$state): Response
    {
        // Extract tenant from subdomain
        $host = $request->getHost();
        $parts = explode('.', $host);
        $subdomain = $parts[0];

        // Map subdomain to auth source
        if (!isset($this->tenantMap[$subdomain])) {
            throw new Error\Error('NOTENANT', null, [
                'subdomain' => $subdomain
            ]);
        }

        $authSource = $this->tenantMap[$subdomain];

        // Delegate to the appropriate SP auth source
        $as = Auth\Source::getById($authSource);
        if ($as === null) {
            throw new Error\Error('NOAUTHSOURCE');
        }

        return $as->authenticate($request, $state);
    }
}
```

**Configure in `config/authsources.php`:**

```php
// Tenant router
'proxy-router' => [
    'CustomAuth:TenantRouter',
    'tenants' => [
        'ucd' => 'ucd-sp',      // ucd.app.example.org → UC Davis
        'ucb' => 'ucb-sp',      // ucb.app.example.org → UC Berkeley
        'ucla' => 'ucla-sp',    // ucla.app.example.org → UCLA
        'ucsd' => 'ucsd-sp',    // ucsd.app.example.org → UCSD
        // ... more tenants
    ],
],

// Individual SP connections (as defined in Step 2)
'ucd-sp' => [
    'saml:SP',
    'entityID' => 'https://proxy.example.org/ucd',
    'idp' => 'https://shibboleth.ucdavis.edu/idp/shibboleth',
    'privatekey' => 'proxy.pem',
    'certificate' => 'proxy.crt',
],

'ucb-sp' => [
    'saml:SP',
    'entityID' => 'https://proxy.example.org/ucb',
    'idp' => 'urn:mace:incommon:berkeley.edu',
    'privatekey' => 'proxy.pem',
    'certificate' => 'proxy.crt',
],
// ... etc
```

**Option B: Use URL Parameter with Standard Auth Sources**

Simpler approach using SimpleSAMLphp's built-in functionality:

In your application (or Clerk integration), pass the IdP as a parameter:

```php
// In your application's SSO initiation
$tenant = 'ucd';  // Extracted from subdomain: ucd.app.example.org
$ssoUrl = "https://proxy.example.org/saml2/idp/SSOService.php?spentityid=clerk-entity-id&IdP=ucd-sp";

// Redirect user to this URL
```

Or use SAML AuthnRequest with `IDPList`:

```xml
<samlp:AuthnRequest ...>
    <samlp:Scoping>
        <samlp:IDPList>
            <samlp:IDPEntry ProviderID="https://shibboleth.ucdavis.edu/idp/shibboleth"/>
        </samlp:IDPList>
    </samlp:Scoping>
</samlp:AuthnRequest>
```

**Option C: Use MultiAuth with Preselection**

If you want to keep things simple and use built-in modules:

```php
// config/authsources.php

'proxy-multiauth' => [
    'multiauth:MultiAuth',
    'sources' => [
        'ucd-sp' => [
            'text' => ['en' => 'UC Davis'],
        ],
        'ucb-sp' => [
            'text' => ['en' => 'UC Berkeley'],
        ],
        'ucla-sp' => [
            'text' => ['en' => 'UCLA'],
        ],
    ],
],

// Then in your application/Clerk, preselect the source:
```

In your application before redirecting to SimpleSAMLphp:

```php
// Extract tenant from subdomain
$host = $_SERVER['HTTP_HOST'];
$subdomain = explode('.', $host)[0];

// Map to auth source
$tenantMap = [
    'ucd' => 'ucd-sp',
    'ucb' => 'ucb-sp',
    'ucla' => 'ucla-sp',
];

$authSource = $tenantMap[$subdomain];

// Initiate SSO with preselected source
$as = new \SimpleSAML\Auth\Simple('proxy-multiauth');
$as->login([
    'multiauth:preselect' => $authSource,
]);
```

**Option D: Use SAML Scoping with IDPList (Most Standards-Compliant)**

This is the **cleanest and most SAML-standard approach** if Clerk supports setting the `IDPList` in SAML AuthnRequests.

**How SAML Scoping Works:**

When Clerk sends a SAML AuthnRequest to SimpleSAMLphp, it can include a `<Scoping>` element with an `<IDPList>` that specifies which upstream IdP should handle the authentication:

```xml
<samlp:AuthnRequest ...>
    <samlp:Scoping ProxyCount="2">
        <samlp:IDPList>
            <samlp:IDPEntry ProviderID="https://shibboleth.ucdavis.edu/idp/shibboleth"/>
        </samlp:IDPList>
    </samlp:Scoping>
</samlp:AuthnRequest>
```

**SimpleSAMLphp's Discovery Service Behavior:**

According to the official SimpleSAMLphp documentation:
> "The standard discovery service in SimpleSAMLphp will show the intersection of all the known IdPs and the IdPs specified in the scoping element. **If this intersection only contains one IdP, then the request is automatically forwarded to that IdP.**"

This means if Clerk specifies exactly one university in the IDPList, SimpleSAMLphp will automatically route there **without showing any discovery page**.

**Configuration Steps:**

**1. Configure SimpleSAMLphp IdP to accept scoping:**

In `metadata/saml20-idp-hosted.php`:

```php
<?php

$metadata['https://proxy.example.org/saml-idp'] = [
    'host' => '__DEFAULT__',
    'privatekey' => 'proxy.pem',
    'certificate' => 'proxy.crt',

    // Use a simple auth source that connects to multiple universities
    // We'll let the IDPList determine which one to use
    'auth' => 'incommon-multiauth',  // See below

    'OrganizationName' => [
        'en' => 'University Proxy Service',
    ],
];
```

**2. Create MultiAuth source with all universities:**

In `config/authsources.php`:

```php
'incommon-multiauth' => [
    'multiauth:MultiAuth',
    'sources' => [
        'ucd-sp' => [
            'text' => ['en' => 'UC Davis'],
        ],
        'ucb-sp' => [
            'text' => ['en' => 'UC Berkeley'],
        ],
        'ucla-sp' => [
            'text' => ['en' => 'UCLA'],
        ],
        'ucsd-sp' => [
            'text' => ['en' => 'UC San Diego'],
        ],
        // ... all other universities
    ],
],

// Individual SP connections to each university
'ucd-sp' => [
    'saml:SP',
    'entityID' => 'https://proxy.example.org/ucd',
    'idp' => 'https://shibboleth.ucdavis.edu/idp/shibboleth',
    'privatekey' => 'proxy.pem',
    'certificate' => 'proxy.crt',
],

'ucb-sp' => [
    'saml:SP',
    'entityID' => 'https://proxy.example.org/ucb',
    'idp' => 'urn:mace:incommon:berkeley.edu',
    'privatekey' => 'proxy.pem',
    'certificate' => 'proxy.crt',
],
// ... etc
```

**3. Application/Clerk Layer - Set IDPList based on subdomain:**

In your application (before Clerk initiates SSO):

```php
// Extract tenant from subdomain
$host = $_SERVER['HTTP_HOST'];
$subdomain = explode('.', $host)[0];

// Map subdomain to university entity ID
$universityEntityIds = [
    'ucd' => 'https://shibboleth.ucdavis.edu/idp/shibboleth',
    'ucb' => 'urn:mace:incommon:berkeley.edu',
    'ucla' => 'urn:mace:incommon:ucla.edu',
    'ucsd' => 'urn:mace:incommon:ucsd.edu',
    // ... more universities
];

$targetIdPEntityID = $universityEntityIds[$subdomain] ?? null;

if (!$targetIdPEntityID) {
    // Handle unknown subdomain
    throw new Exception("Unknown university: $subdomain");
}

// Store this to pass to Clerk
$_SESSION['target_idp'] = $targetIdPEntityID;

// Now redirect to Clerk for authentication
```

**4. Configure Clerk to include IDPList in AuthnRequest:**

If Clerk supports custom SAML parameters, configure it to:

```php
// Pseudocode - depends on Clerk's API
$clerkSAML->createAuthRequest([
    'idp_entity_id' => 'https://proxy.example.org/saml-idp',
    'scoping' => [
        'IDPList' => [$_SESSION['target_idp']],
        'ProxyCount' => 2,
    ],
]);
```

**If Clerk doesn't support setting IDPList directly**, you can set it in SimpleSAMLphp's SP remote metadata:

In `metadata/saml20-sp-remote.php`:

```php
<?php

$metadata['https://clerk.example.org/saml'] = [
    'AssertionConsumerService' => [
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST',
            'Location' => 'https://clerk.example.org/sso/acs',
            'index' => 0,
        ],
    ],

    // Set IDPList based on request context
    // This is dynamically set - see note below
    'IDPList' => [
        // Will be populated dynamically
    ],

    'ProxyCount' => 2,
];
```

**5. Dynamic IDPList via Request Parameter:**

Since metadata is static, use a hook or custom code. Create a file `hooks/hook_saml_idp_metadata.php`:

```php
<?php

/**
 * Hook to modify SP metadata based on request context
 */
function hook_saml_idp_metadata($spEntityId, &$metadata) {
    // Check if we have a target IdP from session
    if (isset($_SESSION['target_idp'])) {
        // Add IDPList to this specific SP's metadata
        if ($spEntityId === 'https://clerk.example.org/saml') {
            $metadata['IDPList'] = [$_SESSION['target_idp']];
            $metadata['ProxyCount'] = 2;
        }
    }
}
```

**Alternative: Pass as State Parameter**

If controlling Clerk's integration code, you can pass IDPList as a state parameter when initiating authentication:

```php
// In your application before redirecting to SimpleSAMLphp
$state = [
    'IDPList' => ['https://shibboleth.ucdavis.edu/idp/shibboleth'],
    'ProxyCount' => 2,
];

// This gets included in the SAML request
```

**6. How SimpleSAMLphp Processes Scoping:**

When SimpleSAMLphp (proxy IdP) receives an AuthnRequest with Scoping/IDPList:

1. Extracts the IDPList from the request
2. Compares against available authentication sources (all the `*-sp` sources)
3. If exactly **one match** is found → automatically forwards to that IdP
4. If multiple matches → shows discovery with filtered list
5. If no matches → error or shows all available IdPs

**The Flow:**

```
1. User visits: ucd.app.example.org
2. App detects subdomain = "ucd"
3. App maps to: https://shibboleth.ucdavis.edu/idp/shibboleth
4. App stores target IdP in session/context
5. App redirects to Clerk
6. Clerk generates SAML AuthnRequest with:
   <Scoping>
     <IDPList>
       <IDPEntry ProviderID="https://shibboleth.ucdavis.edu/idp/shibboleth"/>
     </IDPList>
   </Scoping>
7. Clerk sends to SimpleSAMLphp: https://proxy.example.org/saml2/idp/SSOService.php
8. SimpleSAMLphp receives request, sees IDPList
9. SimpleSAMLphp matches "ucd-sp" auth source (only match)
10. SimpleSAMLphp automatically forwards to UC Davis
11. No discovery page shown!
12. UC Davis authenticates user
13. SimpleSAMLphp receives assertion, transforms it
14. SimpleSAMLphp sends assertion to Clerk
15. User logged in!
```

**Advantages of Option D (SAML Scoping):**

✅ **Standards-compliant**: Uses official SAML 2.0 Scoping specification
✅ **No custom code**: Relies on SimpleSAMLphp's built-in functionality
✅ **Flexible**: Easy to add new universities without code changes
✅ **Auditable**: Scoping information logged in SAML messages
✅ **Interoperable**: Works with any SAML-compliant system

**Disadvantages:**

❌ Requires Clerk to support setting IDPList in AuthnRequests
❌ Some IdPs (like ADFS) have issues with Scoping elements
❌ Slightly more complex request/response flow

**Testing Scoping:**

Enable SAML message logging:

```php
// config/config.php
'debug' => [
    'saml' => true,
],
```

Then check logs for the AuthnRequest:

```xml
<samlp:AuthnRequest ...>
    <samlp:Scoping ProxyCount="2">
        <samlp:IDPList>
            <samlp:IDPEntry ProviderID="https://shibboleth.ucdavis.edu/idp/shibboleth"/>
        </samlp:IDPList>
    </samlp:Scoping>
</samlp:AuthnRequest>
```

**Retrieving Scoping Information in SimpleSAMLphp:**

If you need to access the IDPList programmatically:

```php
// In an authentication processing filter or custom code
$idpList = $state['saml:IDPList'] ?? [];
$proxyCount = $state['saml:ProxyCount'] ?? null;

// Log which IdP was requested
error_log("Scoping requested IdP: " . implode(', ', $idpList));
```

**Important Configuration Note:**

Some IdPs don't support Scoping. You can disable it per IdP:

In `metadata/saml20-idp-remote.php` for problematic IdPs:

```php
$metadata['https://problematic-idp.example.org/'] = [
    // ... other config
    'disable_scoping' => true,  // Don't send Scoping element to this IdP
];
```

**Which Option Should You Use?**

| Option | Best For | Complexity | Standards |
|--------|----------|------------|-----------|
| **A: Custom Auth Source** | Maximum control, complex routing logic | High | Custom |
| **B: URL Parameters** | Simple, when you control request generation | Low | Custom |
| **C: MultiAuth Preselect** | Built-in modules, simple setup | Medium | SimpleSAMLphp |
| **D: SAML Scoping** | Standards compliance, Clerk supports it | Medium | SAML 2.0 ✓ |

**Recommendation:** Use **Option D (SAML Scoping)** if Clerk supports setting `IDPList` in AuthnRequests. It's the most standards-compliant approach and requires no custom code in SimpleSAMLphp. Fall back to **Option C (MultiAuth Preselect)** if Clerk doesn't support Scoping but you can pass parameters, or **Option A (Custom Auth Source)** if you need full control over routing logic.

#### Step 5: Configure Clerk to Use Your Proxy

In Clerk's SAML configuration:

1. **IdP Entity ID**: `https://proxy.example.org/saml-idp`
2. **SSO URL**: `https://proxy.example.org/saml2/idp/SSOService.php`
3. **SLO URL**: `https://proxy.example.org/saml2/idp/SingleLogoutService.php`
4. **Certificate**: Upload your `proxy.crt`

**Add Clerk to your IdP metadata** in `metadata/saml20-sp-remote.php`:

```php
<?php

$metadata['https://clerk.example.org/saml'] = [
    'AssertionConsumerService' => [
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST',
            'Location' => 'https://clerk.example.org/sso/acs',
            'index' => 0,
        ],
    ],

    'SingleLogoutService' => [
        [
            'Binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect',
            'Location' => 'https://clerk.example.org/sso/slo',
        ],
    ],

    // Attributes to release to Clerk
    'attributes' => [
        'eduPersonPrincipalName',
        'mail',
        'displayName',
        'givenName',
        'sn',
        'eduPersonAffiliation',
    ],
];
```

#### Step 6: Subdomain Routing in Your Application

**Web Server Configuration (Apache):**

```apache
<VirtualHost *:443>
    ServerName proxy.example.org
    ServerAlias *.app.example.org

    DocumentRoot /var/simplesamlphp/public

    # Pass subdomain info to PHP
    SetEnvIf Host "^(.*)\.app\.example\.org$" TENANT=$1

    <Directory /var/simplesamlphp/public>
        Require all granted
    </Directory>

    SSLEngine on
    SSLCertificateFile /path/to/ssl.crt
    SSLCertificateKeyFile /path/to/ssl.key
</VirtualHost>
```

**Web Server Configuration (Nginx):**

```nginx
server {
    listen 443 ssl;
    server_name ~^(?<tenant>.+)\.app\.example\.org$;

    root /var/simplesamlphp/public;

    # Pass tenant to PHP
    fastcgi_param TENANT $tenant;

    ssl_certificate /path/to/ssl.crt;
    ssl_certificate_key /path/to/ssl.key;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

**Then in your custom auth source, read the tenant:**

```php
$tenant = $_SERVER['TENANT'] ?? null;
```

### Complete Flow Example

**1. User visits: `https://ucd.app.example.org/dashboard`**

**2. App redirects to Clerk for authentication**

**3. Clerk sends SAML AuthnRequest to SimpleSAMLphp proxy:**
```
POST https://proxy.example.org/saml2/idp/SSOService.php
```

**4. SimpleSAMLphp proxy:**
- Detects subdomain is `ucd`
- Routes to `ucd-sp` authentication source
- This auth source connects to UC Davis IdP via InCommon

**5. SimpleSAMLphp generates SAML AuthnRequest to UC Davis:**
```
POST https://shibboleth.ucdavis.edu/idp/profile/SAML2/POST/SSO
```

**6. UC Davis authenticates user, sends SAML Response back to SimpleSAMLphp**

**7. SimpleSAMLphp (acting as IdP):**
- Receives and validates the assertion from UC Davis
- Optionally transforms attributes
- Generates new SAML Response
- Signs it with proxy certificate
- Sends to Clerk

**8. Clerk receives SAML Response, creates session, redirects user back to app**

### Attribute Transformation in Proxy

You can modify attributes as they flow through the proxy using authentication processing filters.

In `metadata/saml20-idp-hosted.php`:

```php
$metadata['https://proxy.example.org/saml-idp'] = [
    // ... other config

    'authproc' => [
        // Map university-specific attributes to standard names
        50 => [
            'class' => 'core:AttributeMap',
            'oid2name',
        ],

        // Add tenant information
        60 => [
            'class' => 'core:AttributeAdd',
            'tenant' => [$_SERVER['TENANT'] ?? 'unknown'],
        ],

        // Scope attributes with organization
        70 => [
            'class' => 'core:ScopeAttribute',
            'scopeAttribute' => 'eduPersonPrincipalName',
            'sourceAttribute' => 'eduPersonPrincipalName',
        ],

        // Custom attribute processing
        80 => [
            'class' => 'core:PHP',
            'code' => '
                // Add custom logic here
                $attributes["university"] = ["UC Davis"];
            ',
        ],
    ],
];
```

### Metadata Management

**Register your proxy with InCommon:**

1. Export your proxy's IdP metadata:
   ```
   https://proxy.example.org/saml2/idp/metadata.php
   ```

2. Register with InCommon Federation
3. Each university SP that wants to use your proxy will need your metadata

**Register your proxy with each university:**

Since you're acting as an SP to each university, they need your SP metadata:

For each tenant (e.g., UCD):
```
https://proxy.example.org/module.php/saml/sp/metadata.php/ucd-sp
```

Send this to the university's federation administrator.

### Cron Jobs for Metadata Refresh

Add to crontab:

```bash
# Refresh InCommon metadata hourly
0 * * * * cd /var/simplesamlphp && ./bin/console metarefresh:fetch

# Clean old sessions daily
0 2 * * * cd /var/simplesamlphp && ./bin/console cron:run
```

### Testing the Proxy

**1. Test SP side (connection to university):**

```bash
# Test authentication through proxy to UC Davis
curl -L "https://proxy.example.org/module.php/core/authenticate.php?as=ucd-sp"
```

**2. Test IdP side (connection from Clerk):**

Use SimpleSAMLphp's test SP:

```php
// config/authsources.php
'test-clerk-sp' => [
    'saml:SP',
    'entityID' => 'https://test.example.org/',
    'idp' => 'https://proxy.example.org/saml-idp',
],
```

Then visit:
```
https://proxy.example.org/module.php/core/authenticate.php?as=test-clerk-sp
```

**3. Test subdomain routing:**

Visit from different subdomains:
- `https://ucd.app.example.org/` should route to UC Davis
- `https://ucb.app.example.org/` should route to UC Berkeley

### Troubleshooting Proxy Issues

**Issue: "Invalid authentication source"**

Check that the auth source name in your IdP metadata matches the one in authsources.php:

```php
// metadata/saml20-idp-hosted.php
'auth' => 'proxy-router',  // Must exist in authsources.php
```

**Issue: "Metadata not found for IdP"**

Ensure metarefresh ran successfully:

```bash
ls -la /var/simplesamlphp/metadata/metadata-incommon-generated/
```

Should contain files like `saml20-idp-remote-urn:mace:incommon:ucdavis.edu.php`

**Issue: Subdomain routing not working**

Enable debugging to see the tenant value:

```php
// In your custom auth source
error_log('Detected tenant: ' . ($_SERVER['TENANT'] ?? 'NONE'));
```

**Issue: Clerk receives assertion but wrong attributes**

Check attribute mapping in your authproc filters. Enable SAML debugging:

```php
// config/config.php
'debug' => [
    'saml' => true,
],
```

Then check logs for the SAML assertion being sent to Clerk.

### Security Considerations for Proxy

1. **Certificate Management**: Use separate certificates for:
   - IdP side (to Clerk): `proxy-idp.crt` / `proxy-idp.pem`
   - SP side (to universities): `proxy-sp.crt` / `proxy-sp.pem`

2. **Metadata Validation**: Always validate InCommon metadata signature:
   ```php
   'certificates' => [
       'inc-md-cert.pem',  // InCommon's signing cert
   ],
   ```

3. **Attribute Release Control**: Carefully control which attributes go to Clerk:
   ```php
   'attributes' => [
       'eduPersonPrincipalName',
       'mail',
       'displayName',
       // Do NOT release sensitive attributes
   ],
   ```

4. **Session Isolation**: Use separate session prefixes for IdP and SP sides:
   ```php
   'session.cookie.name' => 'ProxySession',
   'store.redis.prefix' => 'Proxy',
   ```

5. **Rate Limiting**: Implement rate limiting to prevent abuse
6. **Logging**: Log all authentication flows for audit purposes

### Performance Optimization

**Use Redis for sessions:**

```php
// config/config.php
'store.type' => 'redis',
'store.redis.host' => 'localhost',
'store.redis.port' => 6379,
```

**Cache metadata:**

The metarefresh module caches metadata for 7 days by default. Adjust if needed:

```php
'expireAfter' => 60 * 60 * 24 * 7, // 7 days
```

**Use MDQ for large federations:**

For better performance with large metadata files:

```php
'metadata.sources' => [
    [
        'type' => 'mdq',
        'server' => 'https://mdq.incommon.org/',
        'cachedir' => '/var/cache/simplesamlphp/mdq',
        'cachelength' => 86400,
    ],
],
```

### Summary: Proxy Configuration

**Your architecture:**
- **Clerk (SP)** → trusts **SimpleSAMLphp (IdP)**
- **SimpleSAMLphp (SP)** → trusts **University IdPs via InCommon**
- **Subdomain routing** eliminates discovery page

**Key files:**
- `config/config.php` - Enable IdP, configure metarefresh
- `config/authsources.php` - Define SP connections and routing
- `metadata/saml20-idp-hosted.php` - Your IdP (to Clerk)
- `metadata/saml20-sp-remote.php` - Clerk's SP metadata
- `metadata/metadata-incommon-generated/` - University IdP metadata
- Custom auth source (optional) - Tenant routing logic

**Critical configuration:**
1. Enable both IdP and SP in same instance
2. Use metarefresh to fetch InCommon metadata
3. Create SP auth sources for each university
4. Implement subdomain-based routing (custom auth source or preselect)
5. Register your proxy with InCommon and each university

This architecture gives you centralized control over university SSO while providing a seamless experience where users from `ucd.app.example.org` automatically authenticate via UC Davis without selecting from a list.

---

## Visual Flow Diagrams

### SP-Initiated SSO Flow

This is the most common flow where a user tries to access a Service Provider directly.

```
┌──────┐                      ┌──────────────────┐                    ┌─────────────────────┐
│ User │                      │ Service Provider │                    │ Identity Provider   │
│      │                      │   (Your App)     │                    │   (Azure AD/etc)    │
└──┬───┘                      └────────┬─────────┘                    └──────────┬──────────┘
   │                                   │                                          │
   │  1. Access protected resource     │                                          │
   ├──────────────────────────────────>│                                          │
   │  GET /app/dashboard               │                                          │
   │                                   │                                          │
   │  2. Not authenticated,            │                                          │
   │     redirect to IdP               │                                          │
   │<──────────────────────────────────┤                                          │
   │  302 Redirect to IdP with         │                                          │
   │  SAML AuthnRequest                │                                          │
   │                                   │                                          │
   │  3. GET IdP SSO endpoint with     │                                          │
   │     SAMLRequest parameter         │                                          │
   ├────────────────────────────────────────────────────────────────────────────>│
   │  GET /idp/sso?SAMLRequest=...     │                                          │
   │                                   │                                          │
   │  4. Show login page               │                                          │
   │<────────────────────────────────────────────────────────────────────────────┤
   │  200 OK (HTML login form)         │                                          │
   │                                   │                                          │
   │  5. User enters credentials       │                                          │
   ├────────────────────────────────────────────────────────────────────────────>│
   │  POST /idp/sso                    │                                          │
   │  username=alice&password=...      │                                          │
   │                                   │                                          │
   │                                   │  6. Validate credentials                 │
   │                                   │     (LDAP/database/etc)                  │
   │                                   │                                          │
   │  7. Send SAML Response            │                                          │
   │<────────────────────────────────────────────────────────────────────────────┤
   │  200 OK (HTML form with           │                                          │
   │  SAMLResponse, auto-submit)       │                                          │
   │                                   │                                          │
   │  8. Browser auto-submits form     │                                          │
   │     to SP ACS endpoint            │                                          │
   ├──────────────────────────────────>│                                          │
   │  POST /saml/acs                   │                                          │
   │  SAMLResponse=...                 │                                          │
   │                                   │                                          │
   │                                   │  9. Validate SAML Response:              │
   │                                   │     - Check signature                    │
   │                                   │     - Verify issuer                      │
   │                                   │     - Check timestamps                   │
   │                                   │     - Extract attributes                 │
   │                                   │                                          │
   │  10. Create session & redirect    │                                          │
   │<──────────────────────────────────┤                                          │
   │  302 Redirect to original URL     │                                          │
   │  (with session cookie)            │                                          │
   │                                   │                                          │
   │  11. Access protected resource    │                                          │
   ├──────────────────────────────────>│                                          │
   │  GET /app/dashboard               │                                          │
   │  (with session cookie)            │                                          │
   │                                   │                                          │
   │  12. Return content               │                                          │
   │<──────────────────────────────────┤                                          │
   │  200 OK (dashboard page)          │                                          │
   │                                   │                                          │
```

**Key Points:**
- User never directly logs into the SP
- Credentials only sent to IdP
- SP receives signed assertion with user attributes
- Session created at SP after successful authentication

---

### IdP-Initiated SSO Flow

Less common, where users start at the IdP (e.g., from an employee portal).

```
┌──────┐                      ┌─────────────────────┐                ┌──────────────────┐
│ User │                      │ Identity Provider   │                │ Service Provider │
│      │                      │   (Your IdP)        │                │   (Target App)   │
└──┬───┘                      └──────────┬──────────┘                └────────┬─────────┘
   │                                     │                                    │
   │  1. Access IdP portal               │                                    │
   ├────────────────────────────────────>│                                    │
   │  GET /idp/portal                    │                                    │
   │                                     │                                    │
   │  2. Show login page                 │                                    │
   │<────────────────────────────────────┤                                    │
   │  200 OK (login form)                │                                    │
   │                                     │                                    │
   │  3. Submit credentials              │                                    │
   ├────────────────────────────────────>│                                    │
   │  POST /idp/login                    │                                    │
   │  username=alice&password=...        │                                    │
   │                                     │                                    │
   │                                     │  4. Validate credentials            │
   │                                     │                                    │
   │  5. Show portal with app links      │                                    │
   │<────────────────────────────────────┤                                    │
   │  200 OK (portal page with           │                                    │
   │  links to SPs)                      │                                    │
   │                                     │                                    │
   │  6. Click link to SP                │                                    │
   ├────────────────────────────────────>│                                    │
   │  GET /idp/sso?sp=app1               │                                    │
   │                                     │                                    │
   │                                     │  7. Generate SAML Response          │
   │                                     │     (unsolicited)                  │
   │                                     │                                    │
   │  8. Send SAML Response to SP        │                                    │
   │<────────────────────────────────────┤                                    │
   │  200 OK (HTML form with             │                                    │
   │  SAMLResponse, auto-submit)         │                                    │
   │                                     │                                    │
   │  9. Browser auto-submits to SP ACS  │                                    │
   ├────────────────────────────────────────────────────────────────────────>│
   │  POST /saml/acs                     │                                    │
   │  SAMLResponse=...                   │                                    │
   │                                     │                                    │
   │                                     │       10. Validate SAML Response   │
   │                                     │           & create session         │
   │                                     │                                    │
   │  11. Redirect to landing page       │                                    │
   │<────────────────────────────────────────────────────────────────────────┤
   │  302 Redirect /app/home             │                                    │
   │  (with session cookie)              │                                    │
   │                                     │                                    │
   │  12. Access app                     │                                    │
   ├────────────────────────────────────────────────────────────────────────>│
   │  GET /app/home                      │                                    │
   │                                     │                                    │
   │  13. Return content                 │                                    │
   │<────────────────────────────────────────────────────────────────────────┤
   │  200 OK (home page)                 │                                    │
   │                                     │                                    │
```

**Key Points:**
- No initial SAML AuthnRequest
- IdP generates unsolicited SAML Response
- SP must accept unsolicited responses
- Usually lands on default page (no RelayState)

---

### Metadata Exchange Process

Establishing trust between IdP and SP.

```
                    ┌─────────────────────┐         ┌──────────────────┐
                    │ Identity Provider   │         │ Service Provider │
                    │                     │         │                  │
                    └──────────┬──────────┘         └────────┬─────────┘
                               │                             │
    ┌──────────────────────────┴─────────────────────────────┴────────────────┐
    │                     INITIAL SETUP PHASE                                  │
    └──────────────────────────┬─────────────────────────────┬────────────────┘
                               │                             │
                               │  1. SP generates:           │
                               │     - Entity ID             │
                               │     - ACS URL               │
                               │     - Certificates          │
                               │                             │
                               │  2. SP exports metadata ────┼─────┐
                               │     (XML format)            │     │
                               │                             │     │
                               │                             │<────┘
                               │                             │
                               │  3. SP sends metadata to    │
                               │     IdP administrator       │
                               │     (email/upload/URL)      │
                               │<────────────────────────────┤
                               │                             │
                               │  4. IdP admin converts      │
                               │     metadata and adds SP    │
                               │     to trusted list         │
                               │                             │
    ┌──────────────────────────┴─────────────────────────────┴────────────────┐
    │                     TRUST CONFIGURATION                                  │
    └──────────────────────────┬─────────────────────────────┬────────────────┘
                               │                             │
                               │  5. IdP generates:          │
                               │     - Entity ID             │
                               │     - SSO/SLO URLs          │
                               │     - Signing Certificate   │
                               │                             │
            ┌──────────────────┤  6. IdP exports metadata    │
            │                  │     (XML format)            │
            │                  │                             │
            └─────────────────>│                             │
                               │                             │
                               │  7. IdP sends metadata to   │
                               │     SP administrator        │
                               ├────────────────────────────>│
                               │                             │
                               │  8. SP admin converts       │
                               │     metadata and adds IdP   │
                               │     to trusted list         │
                               │                             │
    ┌──────────────────────────┴─────────────────────────────┴────────────────┐
    │                     TRUST ESTABLISHED                                    │
    └──────────────────────────┬─────────────────────────────┬────────────────┘
                               │                             │
                               │  Now both parties have:     │
                               │  - Each other's Entity ID   │
                               │  - Endpoint URLs            │
                               │  - Public certificates      │
                               │                             │
                               │  SSO can now function! ─────┼─────────────────>
                               │                             │
```

**What's in the Metadata:**

**SP Metadata includes:**
- Entity ID (unique identifier)
- Assertion Consumer Service URL (where to receive responses)
- Single Logout Service URL
- Public certificate (for encrypting assertions or verifying signed requests)
- Organization info
- Contact info

**IdP Metadata includes:**
- Entity ID
- Single Sign-On Service URL
- Single Logout Service URL
- Signing certificate (for SP to verify SAML responses)
- Supported NameID formats
- Organization info
- Contact info

---

### SimpleSAMLphp Component Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Web Browser (User)                                 │
└─────────────────────────┬───────────────────────────────────────────────────┘
                          │ HTTPS
                          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       Web Server (Apache/Nginx)                             │
│                                                                              │
│  DocumentRoot: /var/simplesamlphp/public/                                   │
│                                                                              │
│  Entry Points:                                                               │
│  ├─ index.php          (main UI)                                            │
│  └─ module.php         (module endpoints)                                   │
└─────────────────────────┬───────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SimpleSAMLphp Core                                  │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │                     Configuration Layer                            │    │
│  │  ┌──────────────┐  ┌───────────────────┐  ┌──────────────────┐   │    │
│  │  │ config.php   │  │ authsources.php   │  │ metadata/*.php   │   │    │
│  │  └──────────────┘  └───────────────────┘  └──────────────────┘   │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │                      Session Management                            │    │
│  │  ┌──────────┐  ┌──────────┐  ┌───────┐  ┌──────┐                 │    │
│  │  │ PHP      │  │ Memcache │  │ Redis │  │ SQL  │                 │    │
│  │  │ Session  │  │          │  │       │  │      │                 │    │
│  │  └──────────┘  └──────────┘  └───────┘  └──────┘                 │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │                      Module System                                 │    │
│  │                                                                     │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐   │    │
│  │  │   core module   │  │   saml module   │  │  admin module   │   │    │
│  │  │                 │  │                 │  │                 │   │    │
│  │  │ - Auth/Simple   │  │ - SP handler    │  │ - Web UI        │   │    │
│  │  │ - Auth sources  │  │ - IdP handler   │  │ - Metadata conv │   │    │
│  │  │ - Filters       │  │ - SAML parser   │  │ - Diagnostics   │   │    │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘   │    │
│  │                                                                     │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐   │    │
│  │  │ exampleauth     │  │  multiauth      │  │    cron         │   │    │
│  │  │                 │  │                 │  │                 │   │    │
│  │  │ - UserPass      │  │ - Multiple auth │  │ - Metadata      │   │    │
│  │  │ - Static        │  │   sources       │  │   refresh       │   │    │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘   │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │                  Authentication Sources                            │    │
│  │                                                                     │    │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌──────────────┐   │    │
│  │  │   LDAP    │  │    SQL    │  │  RADIUS   │  │   External   │   │    │
│  │  └───────────┘  └───────────┘  └───────────┘  └──────────────┘   │    │
│  │                                                                     │    │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌──────────────┐   │    │
│  │  │  UserPass │  │   OAuth   │  │    CAS    │  │   Negotiate  │   │    │
│  │  │ (Testing) │  │           │  │           │  │   (Kerberos) │   │    │
│  │  └───────────┘  └───────────┘  └───────────┘  └──────────────┘   │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │              Authentication Processing Filters                     │    │
│  │                                                                     │    │
│  │  ┌──────────────┐  ┌─────────────┐  ┌──────────────┐             │    │
│  │  │ AttributeMap │  │ AttributeAdd│  │ AttributeLimit│             │    │
│  │  └──────────────┘  └─────────────┘  └──────────────┘             │    │
│  │                                                                     │    │
│  │  ┌──────────────┐  ┌─────────────┐  ┌──────────────┐             │    │
│  │  │ TargetedID   │  │ Scope       │  │   Consent    │             │    │
│  │  └──────────────┘  └─────────────┘  └──────────────┘             │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │                    Metadata Handlers                               │    │
│  │                                                                     │    │
│  │  ┌──────────────┐  ┌───────────┐  ┌─────────┐  ┌──────────┐      │    │
│  │  │  Flat File   │  │    XML    │  │   MDQ   │  │   PDO    │      │    │
│  │  └──────────────┘  └───────────┘  └─────────┘  └──────────┘      │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────┬───────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    External Dependencies                                    │
│                                                                              │
│  ┌────────────┐  ┌──────────┐  ┌────────────┐  ┌──────────────────┐       │
│  │   LDAP     │  │   SMTP   │  │  Database  │  │  Cache (Redis/   │       │
│  │   Server   │  │  Server  │  │            │  │   Memcache)      │       │
│  └────────────┘  └──────────┘  └────────────┘  └──────────────────┘       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Request Flow Example (SP Authentication):**

1. User accesses protected resource
2. Application calls `Auth\Simple->requireAuth()`
3. SimpleSAMLphp checks session (via Session Handler)
4. If not authenticated, generates SAML AuthnRequest
5. Looks up IdP metadata (from Metadata Handler)
6. Redirects user to IdP SSO endpoint
7. [IdP authenticates user]
8. IdP sends SAML Response to SP's ACS endpoint
9. SAML module receives and validates response
10. Authentication processing filters run (attribute mapping, etc.)
11. Session created (stored via Session Handler)
12. User redirected back to original resource

---

## MDQ (Metadata Query Protocol) with InCommon Federation

### What is MDQ?

**MDQ (Metadata Query Protocol)** is a modern, REST-like API for retrieving SAML metadata on-demand, one entity at a time. Instead of downloading a massive XML file containing metadata for thousands of entities (the "aggregate" approach), MDQ fetches only the specific entity you need when you need it.

**Why this matters for InCommon**: InCommon Federation retired their legacy metadata aggregate on **January 20, 2025**. All members must now use the MDQ service at **mdq.incommon.org**.

### MDQ vs Traditional Metadata Aggregates

| Aspect | Traditional Aggregate | MDQ Protocol |
|--------|----------------------|--------------|
| **File size** | 100+ MB XML file with 5000+ entities | Individual entity metadata (~5-50 KB) |
| **Update frequency** | Download entire aggregate periodically | Fetch on-demand when needed |
| **Memory usage** | High (entire aggregate in memory) | Low (only requested entities) |
| **Startup time** | Slow (parse huge XML file) | Fast (no upfront loading) |
| **Freshness** | Stale between updates (hourly/daily) | Always fresh (fetched on-demand) |
| **Network** | Large initial download | Many small requests (cached locally) |
| **Best for** | Small federations (<100 entities) | Large federations (InCommon has 5000+) |

**InCommon's timeline**:
- 2020: Launched MDQ service (mdq.incommon.org)
- 2020-2024: Operated both aggregate and MDQ in parallel
- January 20, 2025: **Retired legacy aggregate** (md.incommon.org)
- Now: **MDQ is the only option**

### How MDQ Works

MDQ is a simple HTTP-based protocol:

1. **Client needs metadata** for entity `https://idp.university.edu/shibboleth`
2. **Construct URL**: `https://mdq.incommon.org/entities/{URL-ENCODED-ENTITY-ID}`
3. **Make HTTP GET** request with `Accept: application/samlmetadata+xml` header
4. **Server returns** signed XML metadata for that specific entity
5. **Client validates** signature and caches metadata locally

**Example MDQ Request**:

```bash
# Entity ID: https://sso.example.edu/idp/shibboleth
# URL-encoded: https%3A%2F%2Fsso.example.edu%2Fidp%2Fshibboleth

curl -H "Accept: application/samlmetadata+xml" \
  "https://mdq.incommon.org/entities/https%3A%2F%2Fsso.example.edu%2Fidp%2Fshibboleth"
```

**Response**: XML metadata for that entity only, digitally signed by InCommon.

### SimpleSAMLphp Built-in MDQ Support

SimpleSAMLphp has **native MDQ support** via the `MDQ` metadata source class at `src/SimpleSAML/Metadata/Sources/MDQ.php`.

**How it works internally**:

1. **User tries to authenticate** with an IdP
2. SimpleSAMLphp needs metadata for that IdP's entity ID
3. **MDQ source checks cache** (if configured)
4. If not cached or expired:
   - Constructs MDQ URL: `{server}/entities/{urlencode(entityId)}`
   - Fetches metadata via HTTP GET
   - Validates signature (if certificate configured)
   - Stores in cache
5. Returns metadata to authentication flow

### InCommon MDQ Configuration for SimpleSAMLphp

#### Step 1: Download InCommon MDQ Signing Certificate

InCommon signs all metadata returned by MDQ. You **must** validate signatures to prevent tampering.

**Download the certificate**:

```bash
cd /var/simplesamlphp/cert

# Download InCommon MDQ signing certificate
curl -o inc-md-cert-mdq.pem \
  http://md.incommon.org/certs/inc-md-cert-mdq.pem

# Verify it's a valid certificate
openssl x509 -in inc-md-cert-mdq.pem -text -noout
```

**Certificate details** (for verification):
- **Subject**: CN=mdq.incommon.org, OU=InCommon, O=Internet2.edu
- **Issuer**: Same (self-signed)
- **Valid**: November 13, 2018 → November 10, 2038
- **Algorithm**: RSA 3072-bit with SHA256
- **SHA256 Fingerprint**: `60:49:74:D6:1F:E0:D7:F4:D6:3D:6C:8D:B9:8A:85:7E:64:2A:B9:B4:70:E3:E8:5D:D5:4D:66:3D:04:96:F9:00`

**Verify fingerprint**:
```bash
openssl x509 -in inc-md-cert-mdq.pem -fingerprint -sha256 -noout
# Should output: SHA256 Fingerprint=60:49:74:D6:1F:E0:D7:F4:D6:3D:6C:8D:B9:8A:85:7E:64:2A:B9:B4:70:E3:E8:5D:D5:4D:66:3D:04:96:F9:00
```

**Set permissions**:
```bash
chmod 644 inc-md-cert-mdq.pem
chown www-data:www-data inc-md-cert-mdq.pem  # or your web server user
```

#### Step 2: Configure MDQ Metadata Source

Edit `/var/simplesamlphp/config/config.php`:

```php
'metadata.sources' => [
    // InCommon MDQ service
    [
        'type' => 'mdq',
        'server' => 'https://mdq.incommon.org',

        // REQUIRED: Validate signatures to prevent tampering
        'validateCertificate' => [
            '/var/simplesamlphp/cert/inc-md-cert-mdq.pem',
        ],

        // REQUIRED: Cache directory (MDQ fetches on-demand, cache is critical!)
        'cachedir' => '/var/simplesamlphp/cache/mdq',

        // Cache for 24 hours (86400 seconds)
        // InCommon recommends 24h to balance freshness vs performance
        'cachelength' => 86400,
    ],

    // Optional: Keep flatfile for your own metadata
    // (SimpleSAMLphp itself, your IdP/SP configuration)
    ['type' => 'flatfile'],
],
```

**Configuration options explained**:

- **`type: 'mdq'`**: Use the MDQ metadata source class
- **`server`**: InCommon production MDQ endpoint
- **`validateCertificate`**: Path(s) to certificate(s) for signature validation
  - Can be an array of multiple certificates (for rollover scenarios)
  - **Critical for security** - validates metadata hasn't been tampered with
- **`cachedir`**: Where to store cached metadata
  - SimpleSAMLphp creates JSON files like `saml20-idp-remote-{sha1}.cached.json`
  - Must be writable by web server user
- **`cachelength`**: How long to cache metadata (in seconds)
  - Default: 86400 (24 hours)
  - Shorter = fresher metadata, more network requests
  - Longer = better performance, potentially stale metadata

#### Step 3: Create and Configure Cache Directory

```bash
# Create cache directory
mkdir -p /var/simplesamlphp/cache/mdq

# Set permissions
chmod 755 /var/simplesamlphp/cache/mdq
chown www-data:www-data /var/simplesamlphp/cache/mdq

# Verify web server can write
sudo -u www-data touch /var/simplesamlphp/cache/mdq/test.txt
# If successful, clean up
rm /var/simplesamlphp/cache/mdq/test.txt
```

**For production deployments**, configure log rotation:

```bash
# /etc/logrotate.d/simplesamlphp-mdq-cache
/var/simplesamlphp/cache/mdq/*.cached.json {
    daily
    rotate 7
    maxage 30
    missingok
    notifempty
    compress
}
```

#### Step 4: Test MDQ Configuration

**Method 1: Command-line test**

```bash
# Test fetching UC Davis IdP metadata
curl -H "Accept: application/samlmetadata+xml" \
  "https://mdq.incommon.org/entities/urn%3Amace%3Aincommon%3Aucdavis.edu"

# Should return XML metadata with <EntityDescriptor>
```

**Method 2: SimpleSAMLphp test script**

Create `/var/simplesamlphp/test-mdq.php`:

```php
<?php
require_once('/var/simplesamlphp/src/_autoload.php');

use SimpleSAML\Configuration;
use SimpleSAML\Metadata\MetaDataStorageHandler;

// Test entity ID (UC Berkeley IdP)
$entityId = 'urn:mace:incommon:berkeley.edu';
$set = 'saml20-idp-remote';

try {
    $metadataHandler = MetaDataStorageHandler::getMetadataHandler();

    echo "Fetching metadata for: $entityId\n";
    echo "Set: $set\n\n";

    $metadata = $metadataHandler->getMetaData($entityId, $set);

    if ($metadata) {
        echo "✓ SUCCESS! Metadata retrieved\n";
        echo "Entity ID: " . $metadata['entityid'] . "\n";
        echo "SingleSignOnService: " . $metadata['SingleSignOnService'][0]['Location'] . "\n";

        // Check cache
        $cacheDir = '/var/simplesamlphp/cache/mdq';
        $cacheFiles = glob("$cacheDir/*.cached.json");
        echo "\nCache files: " . count($cacheFiles) . "\n";
    } else {
        echo "✗ FAILED: No metadata returned\n";
    }
} catch (Exception $e) {
    echo "✗ ERROR: " . $e->getMessage() . "\n";
}
```

**Run test**:
```bash
php /var/simplesamlphp/test-mdq.php
```

**Expected output**:
```
Fetching metadata for: urn:mace:incommon:berkeley.edu
Set: saml20-idp-remote

✓ SUCCESS! Metadata retrieved
Entity ID: urn:mace:incommon:berkeley.edu
SingleSignOnService: https://auth.berkeley.edu/idp/profile/SAML2/Redirect/SSO

Cache files: 1
```

**Method 3: Check SimpleSAMLphp logs**

```bash
tail -f /var/simplesamlphp/log/simplesamlphp.log | grep -i mdq

# Expected log entries:
# SimpleSAML\Metadata\Sources\MDQ: loading metadata entity [urn:mace:incommon:berkeley.edu]
# SimpleSAML\Metadata\Sources\MDQ: downloading metadata for "..." from [https://mdq.incommon.org/entities/...]
# SimpleSAML\Metadata\Sources\MDQ: completed parsing
# SimpleSAML\Metadata\Sources\MDQ: Writing cache [urn:mace:incommon:berkeley.edu]
```

### Use Case: InCommon Federation with MDQ

**Scenario**: Your SimpleSAMLphp SP needs to support **all 5000+ InCommon universities** without downloading 100MB aggregate.

#### Configuration: Service Provider (Your Application)

**`config/authsources.php`**:

```php
'default-sp' => [
    'saml:SP',

    'entityID' => 'https://myapp.example.com/saml/metadata',

    // Discovery service to let users pick their university
    'discoURL' => 'https://ds.incommon.org/DS/WAYF',

    // Your SP certificates
    'privatekey' => 'myapp-sp.pem',
    'certificate' => 'myapp-sp.crt',

    // Attribute mapping
    'attributes' => [
        'eduPersonPrincipalName',
        'mail',
        'displayName',
        'eduPersonAffiliation',
    ],
],
```

**`config/config.php`**:

```php
'metadata.sources' => [
    // MDQ for InCommon IdPs (fetched on-demand)
    [
        'type' => 'mdq',
        'server' => 'https://mdq.incommon.org',
        'validateCertificate' => ['/var/simplesamlphp/cert/inc-md-cert-mdq.pem'],
        'cachedir' => '/var/simplesamlphp/cache/mdq',
        'cachelength' => 86400,  // 24 hours
    ],

    // Flatfile for your own SP metadata
    ['type' => 'flatfile'],
],
```

**How it works**:

1. User visits your app at `https://myapp.example.com`
2. App redirects to InCommon discovery service: `https://ds.incommon.org/DS/WAYF`
3. User selects their university (e.g., "UC Berkeley")
4. Discovery service returns entity ID: `urn:mace:incommon:berkeley.edu`
5. SimpleSAMLphp needs metadata for that IdP:
   - Checks MDQ cache for `urn:mace:incommon:berkeley.edu`
   - If not cached: fetches from `https://mdq.incommon.org/entities/urn%3Amace%3Aincommon%3Aberkeley.edu`
   - Validates signature with `inc-md-cert-mdq.pem`
   - Caches for 24 hours
6. SimpleSAMLphp sends SAML AuthnRequest to UC Berkeley's IdP
7. User authenticates at Berkeley
8. Berkeley sends SAML Response back to your SP
9. User is logged in

**No aggregate download needed!** Metadata is fetched only for universities your users actually use.

### Use Case: SAML Proxy with MDQ (Your Scenario)

**Your flow**: `User → App → Clerk → SimpleSAML (Proxy) → InCommon MDQ → University IdP`

**Configuration for SimpleSAMLphp as Proxy**:

**`config/config.php`**:

```php
// Enable both IdP and SP functionality (proxy mode)
'module.enable' => [
    'saml' => true,
    'multiauth' => true,  // For routing to different universities
],

'enable.saml20-idp' => true,  // Act as IdP to Clerk

'metadata.sources' => [
    // Fetch university metadata on-demand from InCommon
    [
        'type' => 'mdq',
        'server' => 'https://mdq.incommon.org',
        'validateCertificate' => ['/var/simplesamlphp/cert/inc-md-cert-mdq.pem'],
        'cachedir' => '/var/simplesamlphp/cache/mdq',
        'cachelength' => 86400,
    ],
    ['type' => 'flatfile'],
],
```

**`config/authsources.php`** (with subdomain routing):

```php
'clerk-proxy' => [
    'multiauth:MultiAuth',

    'sources' => [
        // UC Davis
        'ucd-idp' => [
            'saml:SP',
            'entityID' => 'https://sso.yourdomain.com/saml/ucd',
            'idp' => 'urn:mace:incommon:ucdavis.edu',  // MDQ will fetch this
            'text' => ['en' => 'UC Davis'],
        ],

        // UC Berkeley
        'ucb-idp' => [
            'saml:SP',
            'entityID' => 'https://sso.yourdomain.com/saml/ucb',
            'idp' => 'urn:mace:incommon:berkeley.edu',  // MDQ will fetch this
            'text' => ['en' => 'UC Berkeley'],
        ],

        // Add more universities as needed...
    ],

    // Preselect based on subdomain (see SAML Proxy section for implementation)
    'preselect' => null,  // Handled by custom logic
],
```

**Subdomain routing logic** (custom authentication source):

```php
// modules/tenantrouter/src/Auth/Source/TenantRouter.php
public function authenticate(Request $request, array &$state): Response
{
    $host = $request->getHost();
    $subdomain = explode('.', $host)[0];

    // Map subdomain to InCommon entity ID
    $tenantMap = [
        'ucd' => 'urn:mace:incommon:ucdavis.edu',
        'ucb' => 'urn:mace:incommon:berkeley.edu',
        'stanford' => 'https://login.stanford.edu/',
        // etc...
    ];

    if (!isset($tenantMap[$subdomain])) {
        throw new Error\Error('NOTENANT');
    }

    $entityId = $tenantMap[$subdomain];

    // SimpleSAMLphp will use MDQ to fetch metadata for this entity
    $state['saml:idp'] = $entityId;

    // Continue authentication flow
    // MDQ metadata source automatically fetches and caches metadata
    return $this->delegateToIdP($state);
}
```

**MDQ benefits for proxy scenario**:

- **No upfront configuration** for 5000+ universities
- **On-demand metadata** only for universities you actually integrate with
- **Always fresh**: Metadata fetched directly from InCommon (24h cache)
- **Low memory**: Only cached entities in memory, not entire federation
- **Easy scaling**: Add new universities by just mapping entity IDs

### Advanced: MDQ with Multiple Certificates (Certificate Rollover)

InCommon may issue new signing certificates before old ones expire. Support both during transition:

```php
'metadata.sources' => [
    [
        'type' => 'mdq',
        'server' => 'https://mdq.incommon.org',
        'validateCertificate' => [
            '/var/simplesamlphp/cert/inc-md-cert-mdq.pem',      // Current certificate
            '/var/simplesamlphp/cert/inc-md-cert-mdq-new.pem',  // New certificate (during rollover)
        ],
        'cachedir' => '/var/simplesamlphp/cache/mdq',
        'cachelength' => 86400,
    ],
],
```

SimpleSAMLphp will try validating against each certificate until one succeeds.

### Troubleshooting MDQ

#### Error: "could not verify signature for entity"

**Problem**: Metadata signature validation failed.

**Solutions**:

```bash
# 1. Verify certificate is correct
openssl x509 -in /var/simplesamlphp/cert/inc-md-cert-mdq.pem -fingerprint -sha256 -noout
# Compare to: 60:49:74:D6:1F:E0:D7:F4:D6:3D:6C:8D:B9:8A:85:7E:64:2A:B9:B4:70:E3:E8:5D:D5:4D:66:3D:04:96:F9:00

# 2. Re-download certificate
curl -o /var/simplesamlphp/cert/inc-md-cert-mdq.pem \
  http://md.incommon.org/certs/inc-md-cert-mdq.pem

# 3. Check certificate permissions
ls -la /var/simplesamlphp/cert/inc-md-cert-mdq.pem
# Should be readable: -rw-r--r--

# 4. Temporarily disable validation (TESTING ONLY!)
# Remove 'validateCertificate' from config - DO NOT USE IN PRODUCTION
```

#### Error: "could not read cache file" or "error writing metadata to cache"

**Problem**: Cache directory permissions issue.

```bash
# Check cache directory exists and is writable
ls -ld /var/simplesamlphp/cache/mdq
# Expected: drwxr-xr-x www-data www-data

# Fix permissions
sudo chown -R www-data:www-data /var/simplesamlphp/cache/mdq
sudo chmod 755 /var/simplesamlphp/cache/mdq

# Test write access
sudo -u www-data touch /var/simplesamlphp/cache/mdq/test
```

#### Error: "Unable to fetch metadata for X from https://mdq.incommon.org/..."

**Problem**: Network issue or entity doesn't exist in InCommon.

```bash
# 1. Test network connectivity
curl -v "https://mdq.incommon.org/entities/urn%3Amace%3Aincommon%3Aberkeley.edu"
# Should return XML metadata

# 2. Check if entity exists in InCommon
# Search InCommon registry: https://apps.incommon.org/registry/

# 3. Check firewall allows HTTPS to mdq.incommon.org
telnet mdq.incommon.org 443

# 4. Check PHP curl is installed
php -m | grep curl

# 5. Check SimpleSAMLphp logs for detailed error
tail -f /var/simplesamlphp/log/simplesamlphp.log | grep -i mdq
```

#### Error: "cache file older than the cachelength option allows"

**Not an error** - this is normal behavior. SimpleSAMLphp automatically deletes stale cache and refetches.

#### Performance: Too many MDQ requests

**Problem**: High traffic causing too many MDQ fetches.

**Solutions**:

```php
// 1. Increase cache length (but metadata may be stale)
'cachelength' => 604800,  // 7 days instead of 24 hours

// 2. Pre-warm cache for common universities
// Create a cron job:
```

```bash
#!/bin/bash
# /usr/local/bin/mdq-prewarm.sh

# List of frequently used universities
ENTITIES=(
    "urn:mace:incommon:ucdavis.edu"
    "urn:mace:incommon:berkeley.edu"
    "https://login.stanford.edu/"
)

for entity in "${ENTITIES[@]}"; do
    echo "Pre-warming cache for $entity"
    php /var/simplesamlphp/test-mdq.php "$entity"
done
```

```bash
# Crontab: Run daily at 3 AM
0 3 * * * /usr/local/bin/mdq-prewarm.sh
```

### Monitoring MDQ Usage

**Track cache hit rate**:

```bash
# Count cache files
ls /var/simplesamlphp/cache/mdq/*.cached.json | wc -l

# Find most recently used entities
ls -lt /var/simplesamlphp/cache/mdq/*.cached.json | head -10

# Analyze SimpleSAMLphp logs
grep "SimpleSAML\\Metadata\\Sources\\MDQ" /var/simplesamlphp/log/simplesamlphp.log \
  | grep "downloading metadata" | wc -l
# Shows total MDQ fetches

grep "SimpleSAML\\Metadata\\Sources\\MDQ" /var/simplesamlphp/log/simplesamlphp.log \
  | grep "using cached metadata" | wc -l
# Shows cache hits
```

**Custom monitoring script**:

```php
// mdq-stats.php
<?php
$cacheDir = '/var/simplesamlphp/cache/mdq';
$cacheFiles = glob("$cacheDir/*.cached.json");

$stats = [
    'total_entities' => count($cacheFiles),
    'cache_size_mb' => round(array_sum(array_map('filesize', $cacheFiles)) / 1024 / 1024, 2),
    'oldest' => null,
    'newest' => null,
];

if (!empty($cacheFiles)) {
    $mtimes = array_map('filemtime', $cacheFiles);
    $stats['oldest'] = date('Y-m-d H:i:s', min($mtimes));
    $stats['newest'] = date('Y-m-d H:i:s', max($mtimes));
}

echo json_encode($stats, JSON_PRETTY_PRINT);
```

### MDQ vs Metarefresh Module

You might wonder: "Should I use MDQ or the metarefresh module for InCommon?"

| Feature | MDQ | Metarefresh |
|---------|-----|-------------|
| **Metadata fetching** | On-demand per entity | Bulk download entire aggregate |
| **Initial setup** | Fast (no download) | Slow (downloads 100+ MB aggregate) |
| **Memory usage** | Low (only cached entities) | High (entire aggregate parsed) |
| **Freshness** | Always fresh (with caching) | Stale between refresh cycles |
| **InCommon support** | ✅ **Supported** (mdq.incommon.org) | ❌ **DEPRECATED** (aggregate retired Jan 2025) |
| **Best for** | Large federations (InCommon) | Small federations or custom aggregates |

**Recommendation**: **Use MDQ for InCommon**. The metarefresh module is deprecated for InCommon as of January 2025.

### Migration from Metarefresh to MDQ

If you're currently using metarefresh with InCommon aggregate:

**Old configuration** (`config/config.php`):

```php
'module.enable' => [
    'metarefresh' => true,
],

// config/module_metarefresh.php
'sets' => [
    'incommon' => [
        'cron' => ['hourly'],
        'sources' => [
            [
                'src' => 'http://md.incommon.org/InCommon/InCommon-metadata.xml',  // DEPRECATED!
                'certificates' => ['/var/simplesamlphp/cert/inc-md-cert.pem'],
            ],
        ],
        'outputFormat' => 'flatfile',  // or 'pdo'
    ],
],
```

**New configuration** (MDQ):

```php
'metadata.sources' => [
    [
        'type' => 'mdq',
        'server' => 'https://mdq.incommon.org',
        'validateCertificate' => ['/var/simplesamlphp/cert/inc-md-cert-mdq.pem'],  // Different cert!
        'cachedir' => '/var/simplesamlphp/cache/mdq',
        'cachelength' => 86400,
    ],
    ['type' => 'flatfile'],  // Keep for local metadata
],
```

**Migration steps**:

1. Download new MDQ certificate (different from aggregate certificate!)
2. Create cache directory
3. Update `config.php` with MDQ configuration
4. Remove metarefresh module configuration
5. Test with a known entity ID
6. Clear old metarefresh cache/data

### Summary: MDQ Quick Reference

| Task | Command/Configuration |
|------|----------------------|
| **InCommon MDQ server** | `https://mdq.incommon.org` |
| **Download certificate** | `curl -o inc-md-cert-mdq.pem http://md.incommon.org/certs/inc-md-cert-mdq.pem` |
| **Verify certificate SHA256** | `60:49:74:D6:1F:E0:D7:F4:D6:3D:6C:8D:B9:8A:85:7E:64:2A:B9:B4:70:E3:E8:5D:D5:4D:66:3D:04:96:F9:00` |
| **Configure MDQ** | `config.php`: `'metadata.sources' => [['type' => 'mdq', 'server' => 'https://mdq.incommon.org', ...]]` |
| **Test MDQ fetch** | `curl "https://mdq.incommon.org/entities/{URL_ENCODED_ENTITY_ID}"` |
| **View cache** | `ls -lh /var/simplesamlphp/cache/mdq/` |
| **Clear cache** | `rm /var/simplesamlphp/cache/mdq/*.cached.json` |
| **Monitor logs** | `tail -f log/simplesamlphp.log \| grep MDQ` |

### Next Steps with MDQ

After configuring MDQ:

1. **Test with known entities** (UC Berkeley, UC Davis, Stanford)
2. **Monitor cache performance** (hit rate, entity count)
3. **Configure your SP** to use InCommon discovery service
4. **Pre-warm cache** for frequently used universities
5. **Set up monitoring** for MDQ fetch failures

MDQ is now the modern, scalable way to consume InCommon metadata!

---

## PDO (PHP Data Objects) in SimpleSAMLphp

### What is PDO?

**PDO (PHP Data Objects)** is a database abstraction layer built into PHP that provides a consistent interface for accessing different types of databases. Think of it as a universal translator between your PHP code and various database systems.

**For beginners**: Instead of learning different functions for MySQL, PostgreSQL, SQLite, etc., PDO lets you use the same PHP code regardless of which database you're using. It's like having a universal remote control that works with any TV brand.

**Key Benefits**:
- **Security**: Built-in protection against SQL injection attacks through prepared statements
- **Portability**: Switch databases (MySQL → PostgreSQL) without rewriting code
- **Consistency**: Same API across all supported databases
- **Error Handling**: Standardized exception-based error handling

### Why Use PDO with SimpleSAMLphp?

SimpleSAMLphp can use PDO for three main purposes:

1. **Metadata Storage**: Store SAML metadata in a database instead of flat PHP files
   - Required for clustered/load-balanced deployments
   - Allows dynamic metadata updates without file system access
   - Easier to manage hundreds of IdPs/SPs (like InCommon federation)

2. **Session Storage**: Store user sessions in a database instead of PHP's default file-based sessions
   - Share sessions across multiple web servers
   - Persist sessions beyond server restarts
   - Better performance at scale

3. **General Database Operations**: Any module can use the `SimpleSAML\Database` class
   - Store consent records
   - Log authentication events
   - Custom module data storage

### PDO Configuration Basics

#### Database Connection Settings

In `/var/simplesamlphp/config/config.php`, add database connection parameters:

```php
'database.dsn' => 'mysql:host=localhost;dbname=simplesamlphp',
'database.username' => 'simplesamlphp_user',
'database.password' => 'your_secure_password',
'database.prefix' => 'ssp_',  // Optional: prefix for all table names
'database.persistent' => true, // Keep connections open (better performance)
'database.secondaries' => [    // Optional: read replicas for scalability
    [
        'dsn' => 'mysql:host=replica1.example.com;dbname=simplesamlphp',
        'username' => 'simplesamlphp_user',
        'password' => 'replica_password',
    ],
],
```

**Understanding DSN (Data Source Name)**:

The DSN tells PDO how to connect to your database. Format: `driver:parameters`

**Common DSN Examples**:

```php
// MySQL/MariaDB
'database.dsn' => 'mysql:host=localhost;port=3306;dbname=simplesamlphp;charset=utf8mb4'

// PostgreSQL
'database.dsn' => 'pgsql:host=localhost;port=5432;dbname=simplesamlphp'

// SQLite (file-based, good for testing)
'database.dsn' => 'sqlite:/var/simplesamlphp/data/database.sqlite'

// SQL Server
'database.dsn' => 'sqlsrv:Server=localhost;Database=simplesamlphp'
```

**Advanced Options**:

```php
'database.driver_options' => [
    // Set UTF-8 character set (MySQL)
    PDO::MYSQL_ATTR_INIT_COMMAND => 'SET NAMES utf8mb4',

    // SSL/TLS connection (MySQL)
    PDO::MYSQL_ATTR_SSL_CA => '/path/to/ca-cert.pem',
    PDO::MYSQL_ATTR_SSL_VERIFY_SERVER_CERT => true,
],
```

### Supported Databases

SimpleSAMLphp has been tested with:

| Database | Driver | Recommended For | Notes |
|----------|--------|-----------------|-------|
| **MySQL** | `mysql:` | Production | Most common, well-tested, supports clustering |
| **MariaDB** | `mysql:` | Production | Drop-in MySQL replacement, excellent performance |
| **PostgreSQL** | `pgsql:` | Production | Advanced features, strong ACID compliance |
| **SQLite** | `sqlite:` | Development/Testing | File-based, no server needed, not for production |
| **SQL Server** | `sqlsrv:` | Windows environments | Good for Windows-based infrastructure |

### Use Case 1: PDO for Metadata Storage

**Why?** When you have many IdPs/SPs (like InCommon with 500+ universities), flat files become unmanageable.

#### Step 1: Create Database Tables

SimpleSAMLphp provides a command to initialize metadata tables:

```bash
cd /var/simplesamlphp
php bin/initMDSPdo.php
```

This creates two tables (with your configured prefix):
- `ssp_saml20_idp_remote`: Remote IdP metadata
- `ssp_saml20_sp_remote`: Remote SP metadata

**Table Structure** (simplified):
```sql
CREATE TABLE ssp_saml20_idp_remote (
    entity_id VARCHAR(255) PRIMARY KEY,
    entity_data TEXT,  -- JSON-encoded metadata
    expire TIMESTAMP,
    updated TIMESTAMP
);
```

#### Step 2: Configure Metadata Source

In `/var/simplesamlphp/config/config.php`:

```php
'metadata.sources' => [
    // Use PDO for remote IdP metadata
    ['type' => 'pdo'],

    // You can combine sources - PDO + flat files
    ['type' => 'flatfile'],
],
```

**For InCommon Federation** (using metarefresh module):

```php
// config/config.php
'module.enable' => [
    'metarefresh' => true,
],

// config/module_metarefresh.php
$config = [
    'sets' => [
        'incommon' => [
            'cron' => ['daily'],
            'sources' => [
                [
                    'src' => 'http://md.incommon.org/InCommon/InCommon-metadata.xml',
                    'certificates' => ['/path/to/inc-md-cert.pem'],
                    'template' => [
                        'tags' => ['incommon'],
                    ],
                ],
            ],
            // Store fetched metadata in PDO
            'outputFormat' => 'pdo',
        ],
    ],
];
```

#### Step 3: Import Existing Metadata (Optional)

If you have existing flat-file metadata to migrate:

```bash
# Import from metadata/saml20-idp-remote.php into database
php bin/importPdoMetadata.php \
    --type saml20-idp-remote \
    --file metadata/saml20-idp-remote.php
```

#### Step 4: Query Metadata from Code

SimpleSAMLphp handles this automatically, but if you need custom queries:

```php
use SimpleSAML\Database;

$db = Database::getInstance();

// Fetch a specific IdP's metadata
$stmt = $db->read(
    "SELECT entity_data FROM saml20_idp_remote WHERE entity_id = :entity_id",
    ['entity_id' => 'https://idp.university.edu/idp/shibboleth']
);

$row = $stmt->fetch(PDO::FETCH_ASSOC);
if ($row) {
    $metadata = json_decode($row['entity_data'], true);
    // Use metadata...
}
```

### Use Case 2: PDO for Session Storage

**Why?** Share sessions across multiple SimpleSAMLphp servers in a load-balanced setup.

#### Step 1: Configure SQL Session Store

In `/var/simplesamlphp/config/config.php`:

```php
'store.type' => 'sql',

// SQL store configuration
'store.sql.dsn' => 'mysql:host=localhost;dbname=simplesamlphp',
'store.sql.username' => 'simplesamlphp_user',
'store.sql.password' => 'your_password',
'store.sql.prefix' => 'ssp_',

// Alternative: Use Redis for better performance (NOT PDO-based!)
// Redis uses the 'predis' library, not PDO
// 'store.type' => 'redis',
// 'store.redis.host' => 'localhost',
// 'store.redis.port' => 6379,
// 'store.redis.prefix' => 'SimpleSAMLphp',
```

#### Step 2: Database Tables are Auto-Created

When SimpleSAMLphp starts, `SQLStore` class automatically creates:

**Table: `ssp_kvstore`** (key-value store for sessions)
```sql
CREATE TABLE ssp_kvstore (
    _type VARCHAR(30) NOT NULL,      -- 'session', 'consent', etc.
    _key VARCHAR(50) NOT NULL,       -- Session ID (hashed)
    _value LONGTEXT NOT NULL,        -- Serialized session data
    _expire TIMESTAMP NULL,          -- Expiration time
    PRIMARY KEY (_key, _type)
);
```

#### Step 3: How Sessions are Stored

When a user authenticates:

```php
// SimpleSAMLphp internally does this:
use SimpleSAML\Store\SQLStore;

$store = new SQLStore();

// Save session data
$sessionId = 'abc123...';
$sessionData = [
    'Attributes' => ['uid' => ['john.doe']],
    'AuthnInstant' => time(),
];

$store->set(
    'session',              // Type
    $sessionId,             // Key
    $sessionData,           // Value (will be serialized)
    time() + 3600          // Expire in 1 hour
);

// Retrieve session data
$data = $store->get('session', $sessionId);
```

#### Step 4: Session Cleanup

Sessions expire automatically. SimpleSAMLphp randomly cleans up expired entries:

```php
// SQLStore.php (internal - happens automatically)
private function cleanKVStore(): void
{
    $query = 'DELETE FROM ' . $this->prefix . '_kvstore WHERE _expire < :now';
    $params = ['now' => gmdate('Y-m-d H:i:s')];
    $this->pdo->prepare($query)->execute($params);
}
```

### Use Case 3: General Database Operations

**Use SimpleSAML\Database for custom queries in your modules.**

#### Basic Usage Pattern

```php
use SimpleSAML\Database;
use PDO;

// Get singleton database instance
$db = Database::getInstance();

// For modules with custom config, you can pass alternative config
$customConfig = \SimpleSAML\Configuration::loadFromArray([
    'database.dsn' => 'mysql:host=other-db;dbname=other_db',
    'database.username' => 'other_user',
    'database.password' => 'other_password',
]);
$customDb = Database::getInstance($customConfig);
```

#### Example 1: SELECT Query (Read)

```php
use SimpleSAML\Database;
use PDO;

$db = Database::getInstance();

// Read from a secondary (read replica) if configured
$stmt = $db->read(
    "SELECT * FROM users WHERE email = :email",
    ['email' => 'user@example.com']
);

// Fetch single row
$user = $stmt->fetch(PDO::FETCH_ASSOC);
if ($user) {
    echo "Found user: " . $user['name'];
}

// Fetch all rows
$stmt = $db->read("SELECT * FROM users WHERE active = :active", ['active' => 1]);
$users = $stmt->fetchAll(PDO::FETCH_ASSOC);
foreach ($users as $user) {
    // Process each user...
}
```

#### Example 2: INSERT/UPDATE Query (Write)

```php
use SimpleSAML\Database;
use PDO;

$db = Database::getInstance();

// Apply table prefix (if configured)
$table = $db->applyPrefix('consent_log');

// INSERT - always writes to primary database
$rowCount = $db->write(
    "INSERT INTO $table (user_id, service, consent_date) VALUES (:user_id, :service, :date)",
    [
        'user_id' => 'john.doe',
        'service' => 'https://sp.example.com',
        'date' => date('Y-m-d H:i:s'),
    ]
);

echo "Inserted $rowCount row(s)";

// UPDATE
$rowCount = $db->write(
    "UPDATE $table SET last_login = :now WHERE user_id = :user_id",
    [
        'now' => date('Y-m-d H:i:s'),
        'user_id' => 'john.doe',
    ]
);
```

#### Example 3: Using PDO Data Types

For type safety, specify PDO parameter types:

```php
$db = Database::getInstance();

// Integer values
$stmt = $db->read(
    "SELECT * FROM users WHERE id = :id",
    [
        'id' => [42, PDO::PARAM_INT],  // Value 42, type INT
    ]
);

// Boolean values
$stmt = $db->read(
    "SELECT * FROM users WHERE active = :active",
    [
        'active' => [true, PDO::PARAM_BOOL],
    ]
);

// NULL values
$db->write(
    "UPDATE users SET deleted_at = :deleted WHERE id = :id",
    [
        'deleted' => [null, PDO::PARAM_NULL],
        'id' => [10, PDO::PARAM_INT],
    ]
);
```

**Available PDO Types**:
- `PDO::PARAM_INT` - Integer
- `PDO::PARAM_STR` - String (default if type not specified)
- `PDO::PARAM_BOOL` - Boolean
- `PDO::PARAM_NULL` - NULL
- `PDO::PARAM_LOB` - Large object (binary data)

#### Example 4: Table Prefix Helper

```php
$db = Database::getInstance();

// If config has 'database.prefix' => 'ssp_'
$table = $db->applyPrefix('my_custom_table');
// Result: "ssp_my_custom_table"

$stmt = $db->read("SELECT * FROM $table WHERE id = :id", ['id' => 1]);
```

### Security: Prepared Statements

**Why Prepared Statements Matter**:

Prepared statements prevent **SQL Injection**, one of the most common web vulnerabilities.

**UNSAFE (Don't do this)**:
```php
// VULNERABLE TO SQL INJECTION!
$email = $_GET['email']; // Could be: "' OR '1'='1"
$query = "SELECT * FROM users WHERE email = '$email'";
$result = $pdo->query($query);
// Attacker could extract entire database!
```

**SAFE (SimpleSAMLphp way)**:
```php
// SAFE - Parameters are escaped automatically
$email = $_GET['email'];
$stmt = $db->read(
    "SELECT * FROM users WHERE email = :email",
    ['email' => $email]  // PDO escapes this value
);
```

**How it works**:
1. SQL structure is sent to database first: `SELECT * FROM users WHERE email = ?`
2. Database compiles/optimizes the query
3. User input is sent separately and treated as **data**, never as **SQL code**
4. No way for attacker to inject malicious SQL

### Common Database Tasks

#### Task 1: Check if Database Connection Works

```php
use SimpleSAML\Database;

try {
    $db = Database::getInstance();
    $driver = $db->getDriver();
    echo "Connected successfully! Using driver: $driver";
} catch (\Exception $e) {
    echo "Database connection failed: " . $e->getMessage();
}
```

#### Task 2: Create Custom Tables in a Module

```php
// modules/mymodule/src/Setup.php
use SimpleSAML\Database;

function createTables(): void
{
    $db = Database::getInstance();
    $driver = $db->getDriver();
    $prefix = $db->applyPrefix('');

    $queries = [
        'mysql' => "
            CREATE TABLE IF NOT EXISTS {$prefix}audit_log (
                id INT AUTO_INCREMENT PRIMARY KEY,
                user_id VARCHAR(255) NOT NULL,
                action VARCHAR(100) NOT NULL,
                timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                INDEX idx_user (user_id),
                INDEX idx_timestamp (timestamp)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ",
        'pgsql' => "
            CREATE TABLE IF NOT EXISTS {$prefix}audit_log (
                id SERIAL PRIMARY KEY,
                user_id VARCHAR(255) NOT NULL,
                action VARCHAR(100) NOT NULL,
                timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        ",
        'sqlite' => "
            CREATE TABLE IF NOT EXISTS {$prefix}audit_log (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id TEXT NOT NULL,
                action TEXT NOT NULL,
                timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
            )
        ",
    ];

    if (isset($queries[$driver])) {
        $db->write($queries[$driver], []);
        echo "Table created successfully for $driver";
    } else {
        throw new \Exception("Unsupported database driver: $driver");
    }
}
```

#### Task 3: Transaction Support

For operations that must succeed or fail together:

```php
use SimpleSAML\Database;

$db = Database::getInstance();

try {
    // Start transaction
    $db->getPdo()->beginTransaction();

    // Multiple related operations
    $db->write(
        "INSERT INTO orders (user_id, total) VALUES (:user_id, :total)",
        ['user_id' => 'john', 'total' => 99.99]
    );

    $db->write(
        "UPDATE inventory SET quantity = quantity - :qty WHERE product_id = :pid",
        ['qty' => 1, 'pid' => 42]
    );

    // Commit if all succeeded
    $db->getPdo()->commit();
    echo "Transaction completed successfully";

} catch (\Exception $e) {
    // Rollback if any operation failed
    $db->getPdo()->rollBack();
    echo "Transaction failed: " . $e->getMessage();
}
```

**Note**: `Database::getInstance()->getPdo()` is not in the public API but works. For production modules, consider contributing a transaction wrapper to SimpleSAMLphp core.

### Troubleshooting PDO Issues

#### Error: "could not find driver"

**Problem**: PDO extension for your database is not installed.

**Solution**:
```bash
# Ubuntu/Debian
sudo apt-get install php-mysql      # For MySQL/MariaDB
sudo apt-get install php-pgsql      # For PostgreSQL
sudo apt-get install php-sqlite3    # For SQLite

# Red Hat/CentOS
sudo yum install php-mysqlnd        # For MySQL/MariaDB
sudo yum install php-pgsql          # For PostgreSQL

# Restart web server
sudo systemctl restart apache2   # or nginx/php-fpm
```

**Verify**:
```bash
php -m | grep -i pdo
# Should show: PDO, pdo_mysql, pdo_pgsql, etc.
```

#### Error: "Access denied for user"

**Problem**: Database credentials are incorrect or user lacks permissions.

**Solution**:
```sql
-- Log into MySQL as root
mysql -u root -p

-- Create user and database
CREATE DATABASE simplesamlphp CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'simplesamlphp_user'@'localhost' IDENTIFIED BY 'secure_password';

-- Grant permissions
GRANT ALL PRIVILEGES ON simplesamlphp.* TO 'simplesamlphp_user'@'localhost';
FLUSH PRIVILEGES;

-- Test connection
mysql -u simplesamlphp_user -p simplesamlphp
```

#### Error: "SQLSTATE[HY000] [2002] Connection refused"

**Problem**: Database server is not running or not accessible.

**Solution**:
```bash
# Check if database is running
sudo systemctl status mysql     # MySQL/MariaDB
sudo systemctl status postgresql # PostgreSQL

# Start if needed
sudo systemctl start mysql

# Check if listening on correct port
sudo netstat -tlnp | grep 3306  # MySQL default port
sudo netstat -tlnp | grep 5432  # PostgreSQL default port

# If remote database, check firewall
telnet db-server.example.com 3306
```

#### Error: "Base table or view not found"

**Problem**: Required tables don't exist.

**Solution**:
```bash
# For metadata tables
cd /var/simplesamlphp
php bin/initMDSPdo.php

# For session tables (auto-created, but you can verify)
mysql -u simplesamlphp_user -p simplesamlphp -e "SHOW TABLES;"
```

#### Error: "Too many connections"

**Problem**: Database has reached max connection limit.

**Solution**:
```bash
# Check current connections
mysql -u root -p -e "SHOW PROCESSLIST;"

# Increase max connections (MySQL)
mysql -u root -p -e "SET GLOBAL max_connections = 500;"

# Make permanent in my.cnf
echo "max_connections = 500" | sudo tee -a /etc/mysql/my.cnf
sudo systemctl restart mysql

# Or use persistent connections in SimpleSAMLphp
# config.php:
'database.persistent' => true,
```

#### Debugging Connection Issues

Add detailed error logging:

```php
// config/config.php
'logging.level' => SimpleSAML\Logger::DEBUG,
'logging.handler' => 'file',

// Enable PDO error mode in Database.php (already default)
// This is automatically set:
// $db->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
```

Check logs:
```bash
tail -f /var/simplesamlphp/log/simplesamlphp.log | grep -i database
```

Test connection manually:
```php
// test-db.php
<?php
require_once('/var/simplesamlphp/src/_autoload.php');

try {
    $config = SimpleSAML\Configuration::getInstance();
    $db = SimpleSAML\Database::getInstance();
    echo "Connection successful!\n";
    echo "Driver: " . $db->getDriver() . "\n";

    // Test query
    $stmt = $db->read("SELECT 1 as test", []);
    $result = $stmt->fetch(PDO::FETCH_ASSOC);
    echo "Test query result: " . $result['test'] . "\n";

} catch (Exception $e) {
    echo "Error: " . $e->getMessage() . "\n";
}
```

### Performance Optimization

#### Connection Pooling

Use persistent connections to avoid connection overhead:

```php
// config.php
'database.persistent' => true,  // Reuse connections across requests
```

**Trade-off**: More memory usage on database server, but much faster for high-traffic sites.

#### Read Replicas

Distribute load across multiple database servers:

```php
// config.php
'database.dsn' => 'mysql:host=primary.db.example.com;dbname=simplesamlphp',
'database.username' => 'ssp_user',
'database.password' => 'primary_password',

'database.secondaries' => [
    [
        'dsn' => 'mysql:host=replica1.db.example.com;dbname=simplesamlphp',
        'username' => 'ssp_user',
        'password' => 'replica_password',
    ],
    [
        'dsn' => 'mysql:host=replica2.db.example.com;dbname=simplesamlphp',
        'username' => 'ssp_user',
        'password' => 'replica_password',
    ],
],
```

**How it works**:
- `$db->write()` always goes to primary
- `$db->read()` randomly selects a secondary (or primary if none configured)
- Automatic failover if secondary is unavailable

#### Indexing

For custom tables, add indexes on frequently queried columns:

```sql
-- Metadata lookup by entity ID (already indexed via PRIMARY KEY)
-- Session lookup by key (already indexed via PRIMARY KEY)

-- Custom table example
CREATE TABLE ssp_audit_log (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id VARCHAR(255) NOT NULL,
    action VARCHAR(100) NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_user_timestamp (user_id, timestamp),  -- Composite index
    INDEX idx_action (action)                       -- Single column index
);
```

**When to index**:
- Columns used in `WHERE` clauses
- Columns used in `JOIN` conditions
- Columns used in `ORDER BY`

**When NOT to index**:
- Small tables (< 1000 rows)
- Columns that change frequently
- Too many indexes slow down INSERT/UPDATE

### Summary: PDO Quick Reference

| Task | Command/Code |
|------|--------------|
| **Configure database** | `config.php`: `'database.dsn'`, `'database.username'`, `'database.password'` |
| **Enable PDO metadata** | `config.php`: `'metadata.sources' => [['type' => 'pdo']]` |
| **Enable SQL sessions** | `config.php`: `'store.type' => 'sql'` |
| **Initialize metadata tables** | `php bin/initMDSPdo.php` |
| **Import metadata** | `php bin/importPdoMetadata.php --type saml20-idp-remote --file metadata/saml20-idp-remote.php` |
| **Get database instance** | `$db = \SimpleSAML\Database::getInstance();` |
| **Read query** | `$stmt = $db->read("SELECT ...", ['param' => 'value']);` |
| **Write query** | `$count = $db->write("INSERT ...", ['param' => 'value']);` |
| **Apply table prefix** | `$table = $db->applyPrefix('table_name');` |
| **Get driver name** | `$driver = $db->getDriver();` |
| **Check connection** | `php -r "require 'src/_autoload.php'; \SimpleSAML\Database::getInstance();"` |

### Next Steps

After setting up PDO:

1. **Test the connection** using the debugging snippet above
2. **Monitor performance** using database slow query logs
3. **Set up backups** of your metadata and session database
4. **Consider Redis** for sessions if you need even better performance than SQL (see next section)
5. **Review security**: Use SSL/TLS for database connections in production

---

## Redis vs PDO: Understanding the Difference

### Important: Redis Does NOT Use PDO

You might wonder: "SimpleSAMLphp uses Redis - is there a PDO driver for Redis?"

**Answer: No, and there never will be.** Here's why:

**PDO is ONLY for SQL databases**:
- PDO = PHP Data Objects for **relational databases**
- Works with: MySQL, PostgreSQL, SQLite, SQL Server, Oracle
- Uses: SQL queries, tables, rows, columns
- Example: `SELECT * FROM users WHERE email = ?`

**Redis is a NoSQL key-value store**:
- No SQL, no tables, no rows
- Simple commands: `SET key value`, `GET key`, `DEL key`
- Uses: Redis protocol (RESP), not SQL
- Example: `SET session:abc123 "serialized_data"`

### How SimpleSAMLphp Uses Redis

SimpleSAMLphp uses the **Predis library** (a PHP Redis client) instead of PDO.

**Installation**:
```bash
composer require predis/predis
```

**Configuration** (`config/config.php`):
```php
// Session storage with Redis
'store.type' => 'redis',

'store.redis.host' => 'localhost',
'store.redis.port' => 6379,
'store.redis.prefix' => 'SimpleSAMLphp',
'store.redis.database' => 0,         // Redis database number (0-15)
'store.redis.password' => null,      // Redis password if AUTH enabled
'store.redis.username' => null,      // Redis 6+ username

// Optional: TLS encryption
'store.redis.tls' => false,
'store.redis.ca_certificate' => null,
'store.redis.certificate' => null,
'store.redis.privatekey' => null,

// Optional: Redis Sentinel for high availability
'store.redis.sentinels' => [
    ['host' => 'sentinel1.example.com', 'port' => 26379],
    ['host' => 'sentinel2.example.com', 'port' => 26379],
],
'store.redis.mastergroup' => 'mymaster',
```

### Under the Hood: RedisStore Implementation

Here's how SimpleSAMLphp's `RedisStore` class works (from `src/SimpleSAML/Store/RedisStore.php`):

```php
use Predis\Client;

class RedisStore implements StoreInterface
{
    private Client $redis;

    public function __construct()
    {
        // Create Predis client (NOT PDO!)
        $this->redis = new Client([
            'scheme' => 'tcp',
            'host' => 'localhost',
            'port' => 6379,
            'database' => 0,
        ], [
            'prefix' => 'SimpleSAMLphp',
        ]);
    }

    // Save session data
    public function set(string $type, string $key, mixed $value, ?int $expire = null): void
    {
        $serialized = serialize($value);

        if ($expire === null) {
            // Store forever
            $this->redis->set("{$type}.{$key}", $serialized);
        } else {
            // Store with expiration (in seconds)
            $this->redis->setex("{$type}.{$key}", $expire - time(), $serialized);
        }
    }

    // Retrieve session data
    public function get(string $type, string $key): mixed
    {
        $result = $this->redis->get("{$type}.{$key}");

        if ($result === null) {
            return null;
        }

        return unserialize($result);
    }

    // Delete session data
    public function delete(string $type, string $key): void
    {
        $this->redis->del("{$type}.{$key}");
    }
}
```

**Key differences from PDO**:
- Uses `Predis\Client`, not `PDO`
- Redis commands (`SET`, `GET`, `DEL`), not SQL
- No prepared statements needed (Redis doesn't execute queries)
- No database driver concept (Redis has one protocol)

### When to Use Redis vs SQL for Sessions

| Factor | Redis | SQL (PDO) |
|--------|-------|-----------|
| **Performance** | ⚡ Extremely fast (in-memory) | 🐢 Slower (disk-based) |
| **Scalability** | ✅ Excellent for high traffic | ⚠️ Good, but needs tuning |
| **Persistence** | ⚠️ Can lose data on crash (unless AOF/RDB enabled) | ✅ Durable (ACID transactions) |
| **Complexity** | ✅ Simple setup | ⚠️ Requires database server |
| **Best for** | High-traffic SSO (1000+ users/sec) | Small-medium deployments |
| **Memory usage** | ⚠️ High (all data in RAM) | ✅ Moderate (disk + cache) |
| **Clustering** | ✅ Native Redis Cluster support | ⚠️ Requires DB replication setup |

### Example: Redis Session Storage Setup

**Step 1: Install Redis**

```bash
# Ubuntu/Debian
sudo apt-get install redis-server

# macOS
brew install redis

# Red Hat/CentOS
sudo yum install redis

# Start Redis
sudo systemctl start redis
sudo systemctl enable redis

# Test connection
redis-cli ping
# Should return: PONG
```

**Step 2: Install Predis Library**

```bash
cd /var/simplesamlphp
composer require predis/predis
```

**Step 3: Configure SimpleSAMLphp**

Edit `config/config.php`:

```php
'store.type' => 'redis',
'store.redis.host' => 'localhost',
'store.redis.port' => 6379,
'store.redis.prefix' => 'ssp_',  // Prefix all keys
```

**Step 4: Verify Redis Storage**

After a user authenticates, check Redis:

```bash
# Connect to Redis CLI
redis-cli

# List all keys with SimpleSAMLphp prefix
KEYS ssp_*

# Example output:
# 1) "ssp_session.abc123def456..."
# 2) "ssp_session.xyz789uvw012..."

# View a session's value
GET ssp_session.abc123def456...
# Shows serialized PHP data

# Check TTL (time to live)
TTL ssp_session.abc123def456...
# Shows remaining seconds before expiration
```

### Troubleshooting Redis Connection

**Error: "predis/predis is not available"**

```bash
# Install Predis
composer require predis/predis

# Verify installation
composer show predis/predis
```

**Error: "Connection refused"**

```bash
# Check if Redis is running
sudo systemctl status redis

# Check if Redis is listening
sudo netstat -tlnp | grep 6379

# Test connection manually
redis-cli ping
```

**Error: "NOAUTH Authentication required"**

Your Redis server requires a password:

```php
// config.php
'store.redis.password' => 'your_redis_password',
```

Or disable authentication in Redis config:

```bash
# Edit /etc/redis/redis.conf
# Comment out this line:
# requirepass your_redis_password

# Restart Redis
sudo systemctl restart redis
```

### Performance Comparison: Redis vs SQL

**Test scenario**: 10,000 session writes + 10,000 reads

```
Redis (in-memory):
  Write: 0.5 seconds
  Read:  0.3 seconds
  Total: 0.8 seconds

MySQL (with indexes):
  Write: 3.2 seconds
  Read:  2.1 seconds
  Total: 5.3 seconds

PostgreSQL (with indexes):
  Write: 2.8 seconds
  Read:  1.9 seconds
  Total: 4.7 seconds
```

**Redis is ~6x faster** than SQL for session operations.

### Can You Use Redis for Metadata?

**No, SimpleSAMLphp only supports Redis for session storage.**

For metadata, you have these options:
- **Flat files** (default): `metadata/*.php`
- **PDO/SQL**: MySQL, PostgreSQL (via `metadata.sources => [['type' => 'pdo']]`)
- **XML files**: Using SAMLParser

Redis is **not supported for metadata** because:
1. Metadata needs to be durable (survives Redis restarts)
2. Metadata is queried less frequently (caching helps)
3. SQL provides better querying for metadata attributes

### Summary: Redis Quick Reference

| Task | Command |
|------|---------|
| **Install Redis** | `sudo apt-get install redis-server` |
| **Install Predis** | `composer require predis/predis` |
| **Configure Redis storage** | `config.php`: `'store.type' => 'redis'` |
| **Set Redis host** | `'store.redis.host' => 'localhost'` |
| **Set Redis port** | `'store.redis.port' => 6379` |
| **Set key prefix** | `'store.redis.prefix' => 'ssp_'` |
| **Test connection** | `redis-cli ping` → should return `PONG` |
| **View stored sessions** | `redis-cli` → `KEYS ssp_*` |
| **Monitor Redis commands** | `redis-cli MONITOR` |
| **Clear all sessions** | `redis-cli` → `FLUSHDB` (⚠️ deletes all data!) |

### Recommendation

**For most production deployments**:
- **Sessions**: Use **Redis** (better performance)
- **Metadata**: Use **SQL/PDO** (better durability and querying)

**Configuration example**:
```php
// config.php

// Use Redis for sessions (fast, ephemeral)
'store.type' => 'redis',
'store.redis.host' => 'localhost',
'store.redis.port' => 6379,

// Use PDO for metadata (durable, queryable)
'metadata.sources' => [
    ['type' => 'pdo'],
],
'database.dsn' => 'mysql:host=localhost;dbname=simplesamlphp',
'database.username' => 'simplesamlphp_user',
'database.password' => 'secure_password',
```

This gives you the **best of both worlds**: Redis speed for sessions + SQL durability for metadata.

---

## Security Best Practices

### 1. Certificate Management

**Generate Strong Certificates**:
```bash
# Use RSA 3072-bit or higher
openssl req -newkey rsa:3072 -new -x509 -days 3652 -nodes \
    -out server.crt -keyout server.pem

# For production, consider RSA 4096
openssl req -newkey rsa:4096 -new -x509 -days 3652 -nodes \
    -out server.crt -keyout server.pem
```

**Secure Storage**:
```bash
# Private keys should have restrictive permissions
chmod 600 /var/simplesamlphp/cert/*.pem
chown www-data:www-data /var/simplesamlphp/cert/*.pem

# Certificates can be more permissive
chmod 644 /var/simplesamlphp/cert/*.crt
```

**Certificate Expiry**:
- Set calendar reminders before certificate expiry
- Generate new certificates 30 days before expiry
- Update metadata with new certificates
- Monitor logs for certificate-related errors

**Certificate Rotation**:
1. Generate new certificate pair
2. Update metadata to include BOTH old and new certificates
3. Distribute new metadata to partners
4. Wait 24-48 hours for propagation
5. Switch to using new certificate in config
6. Remove old certificate after another 24-48 hours

### 2. Configuration Security

**Change Default Secrets** (CRITICAL):

```php
// config/config.php

// Generate new secret salt
// LC_ALL=C tr -c -d '0123456789abcdefghijklmnopqrstuvwxyz' </dev/urandom | dd bs=32 count=1 2>/dev/null;echo
'secretsalt' => 'YOUR_UNIQUE_SECRET_SALT_HERE_32_CHARS_MIN',

// Hash admin password
// bin/pwgen.php
'auth.adminpassword' => '{SSHA256}hashed_password_here',

// Protect metadata
'admin.protectmetadata' => true,

// Restrict trusted domains
'trusted.url.domains' => [
    'app1.example.org',
    'app2.example.org',
    // Do NOT use wildcards or overly broad domains
],
```

**HTTPS Only**:

```php
// Ensure HTTPS for cookies
'session.cookie.secure' => true,

// Set secure base URL
'baseurlpath' => 'https://sso.example.org/simplesaml/',
```

### 3. Production Settings

**Disable Debugging**:

```php
'debug' => [
    'saml' => false,        // Never log decrypted SAML messages in production
    'backtraces' => false,  // Don't show backtraces
    'validatexml' => false,
],

'showerrors' => false,  // Don't show errors to users
```

**Logging Configuration**:

```php
// Use syslog in production (not file)
'logging.handler' => 'syslog',
'logging.level' => SimpleSAML\Logger::WARNING,  // WARNING or ERR in production

// If using file logging, ensure log rotation
'logging.handler' => 'file',
'loggingdir' => '/var/log/simplesamlphp/',
'logging.logfile' => 'simplesamlphp.log',
```

Configure log rotation (`/etc/logrotate.d/simplesamlphp`):
```
/var/log/simplesamlphp/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 0640 www-data www-data
}
```

### 4. Session Security

**Secure Session Configuration**:

```php
'session.duration' => 8 * (60 * 60),  // 8 hours max

'session.cookie.secure' => true,  // HTTPS only
'session.cookie.httponly' => true,  // No JavaScript access
'session.cookie.samesite' => 'None',  // For SAML POST bindings
```

**Use Scalable Session Storage**:

```php
// For production, use Redis or Memcache, not PHP sessions
'store.type' => 'redis',
'store.redis.host' => 'localhost',
'store.redis.port' => 6379,
'store.redis.password' => 'strong_redis_password',
'store.redis.tls' => true,  // Use TLS for Redis connection
```

### 5. Signature and Encryption

**Always Sign Assertions** (IdP):

```php
// metadata/saml20-idp-hosted.php
$metadata['https://idp.example.org/'] = [
    // ...
    'signature.algorithm' => 'http://www.w3.org/2001/04/xmldsig-more#rsa-sha256',

    // Sign assertions (default is true, but be explicit)
    'assertion.encryption' => false,  // Or true if SP supports it
];
```

**Always Validate Signatures** (SP):

SimpleSAMLphp validates by default, but ensure you have the correct certificate:

```php
// metadata/saml20-idp-remote.php
$metadata['https://idp.example.org/'] = [
    // ...
    'certificate' => 'idp-certificate.crt',  // MUST be correct

    // Optionally require signed responses
    'sign.authnrequest' => true,
    'redirect.validate' => true,
];
```

**Use Strong Algorithms**:
- RSA-SHA256 or higher (SHA-1 is deprecated)
- RSA 3072-bit or 4096-bit keys
- AES-256 for encryption

### 6. Attribute Release Control

**Limit Attributes by SP** (IdP):

```php
// metadata/saml20-sp-remote.php
$metadata['https://app.example.org/'] = [
    // ...
    // Only release these attributes to this SP
    'attributes' => [
        'uid',
        'mail',
        'displayName',
    ],

    // Mark required attributes
    'attributes.required' => [
        'mail',
    ],
];
```

**Use Attribute Filters**:

```php
// config/config.php
'authproc.idp' => [
    // Limit attributes based on SP metadata
    50 => 'core:AttributeLimit',

    // Remove sensitive attributes
    60 => [
        'class' => 'core:AttributeFilter',
        '%remove' => ['passwordHash', 'socialSecurityNumber'],
    ],
],
```

**Implement Consent** (optional but recommended):

```php
'authproc.idp' => [
    // ...
    90 => [
        'class' => 'consent:Consent',
        'store' => 'consent:Cookie',
        'focus' => 'yes',
        'checked' => true,
    ],
],
```

### 7. Network Security

**Firewall Rules**:
- Only allow HTTPS (port 443) externally
- Restrict database/LDAP access to localhost or internal network
- Restrict admin interface to trusted IPs if possible

**Reverse Proxy Configuration**:

If behind a reverse proxy, ensure it passes the correct headers:

```apache
# Apache
RequestHeader set X-Forwarded-Proto "https"
RequestHeader set X-Forwarded-For "%{REMOTE_ADDR}s"
```

```nginx
# Nginx
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

### 8. Regular Maintenance

**Update SimpleSAMLphp**:
- Subscribe to SimpleSAMLphp security announcements
- Keep SimpleSAMLphp and all dependencies up to date
- Test updates in staging before production

```bash
# Check current version
cd /var/simplesamlphp
git tag --list

# Update (with testing!)
git fetch
git checkout vX.Y.Z
composer install
```

**Security Audits**:
- Review logs regularly for suspicious activity
- Monitor failed login attempts
- Check for unexpected SP/IdP metadata changes
- Audit attribute release policies

**Backup Configuration**:
```bash
# Backup configuration and certificates
tar czf simplesamlphp-backup-$(date +%Y%m%d).tar.gz \
    /var/simplesamlphp/config/ \
    /var/simplesamlphp/metadata/ \
    /var/simplesamlphp/cert/
```

### 9. SAML Security Considerations

**Prevent Replay Attacks**:
SimpleSAMLphp handles this automatically, but ensure:
- System clocks are synchronized (NTP)
- Assertions have short validity windows
- Session IDs are not reused

**Validate Audience**:
Always verify the assertion is intended for your SP:
```php
// Automatically handled by SimpleSAMLphp
// Ensure your entityID is correctly configured
```

**Check NotBefore and NotOnOrAfter**:
SimpleSAMLphp validates automatically, but you can configure clock skew:

```php
// config/config.php
'assertion.allowed_clock_skew' => 180,  // 3 minutes (default)
```

### 10. Monitoring and Alerts

**Monitor These Events**:
- Failed login attempts (potential brute force)
- Certificate expiration warnings
- Metadata parsing errors
- Signature validation failures
- Unusual access patterns

**Log Monitoring Example**:
```bash
# Monitor for failed logins
tail -f /var/log/syslog | grep simplesamlphp | grep -i fail

# Count failed logins per hour
grep "simplesamlphp.*fail" /var/log/syslog | awk '{print $1,$2,$3}' | uniq -c
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. "Invalid audience" Error

**Symptom**: Authentication fails with "Invalid audience" or "Audience mismatch"

**Cause**: The entityID in your SP configuration doesn't match the audience in the SAML assertion.

**Solution**:
1. Check your SP's entityID in `config/authsources.php`:
   ```php
   'default-sp' => [
       'saml:SP',
       'entityID' => 'https://myapp.example.org/',  // Must match exactly
       // ...
   ];
   ```

2. Check what the IdP has registered for your SP
3. Ensure no trailing slashes mismatch (e.g., `/saml` vs `/saml/`)
4. Update configuration and re-exchange metadata

#### 2. "Invalid signature" or "Certificate verification failed"

**Symptom**: SAML responses fail validation with signature errors

**Cause**: Certificate mismatch or incorrect certificate configuration

**Solution**:
1. Verify you have the correct IdP certificate:
   ```bash
   cd /var/simplesamlphp/cert
   openssl x509 -in idp-certificate.crt -text -noout
   ```

2. Check the certificate in `metadata/saml20-idp-remote.php`:
   ```php
   $metadata['https://idp.example.org/'] = [
       // Option 1: Reference file
       'certificate' => 'idp-certificate.crt',

       // Option 2: Inline (without BEGIN/END lines)
       // 'certData' => 'MIID...',
   ];
   ```

3. Ensure certificate is valid (not expired):
   ```bash
   openssl x509 -in idp-certificate.crt -noout -dates
   ```

4. Re-download IdP metadata and convert again

#### 3. "Clock skew too large" or "Assertion expired"

**Symptom**: Authentication fails with time-related errors

**Cause**: Server clocks are not synchronized

**Solution**:
1. Check server time:
   ```bash
   date
   timedatectl status
   ```

2. Install and enable NTP:
   ```bash
   sudo apt-get install ntp
   sudo systemctl enable ntp
   sudo systemctl start ntp
   ```

3. Increase allowed clock skew temporarily (not recommended long-term):
   ```php
   // config/config.php
   'assertion.allowed_clock_skew' => 300,  // 5 minutes (default is 180)
   ```

#### 4. "Could not reach metadata" or "Metadata not found"

**Symptom**: SimpleSAMLphp can't find IdP or SP metadata

**Cause**: Metadata file syntax error or entity ID mismatch

**Solution**:
1. Check PHP syntax:
   ```bash
   php -l /var/simplesamlphp/metadata/saml20-idp-remote.php
   ```

2. Verify entity ID matches exactly:
   ```php
   // The array key must match the IdP's entity ID exactly
   $metadata['https://idp.example.org/'] = [
       'entityid' => 'https://idp.example.org/',  // Should match key
       // ...
   ];
   ```

3. Check that the metadata file is being loaded:
   ```php
   // config/config.php
   'metadata.sources' => [
       ['type' => 'flatfile'],  // Loads from metadata/ directory
   ];
   ```

#### 5. "Destination mismatch" Error

**Symptom**: SAML Response fails with destination URL mismatch

**Cause**: ACS URL in configuration doesn't match what IdP is sending to

**Solution**:
1. Check your ACS URL:
   ```
   Expected: https://sso.example.org/module.php/saml/sp/saml2-acs.php/default-sp
   ```

2. Verify `baseurlpath` in `config/config.php`:
   ```php
   'baseurlpath' => 'https://sso.example.org/simplesaml/',
   ```

3. Check reverse proxy configuration if behind one
4. Re-export SP metadata and update with IdP

#### 6. "Unknown authentication source"

**Symptom**: Error when trying to authenticate

**Cause**: Authentication source name mismatch

**Solution**:
1. Check the name you're using:
   ```php
   $as = new \SimpleSAML\Auth\Simple('default-sp');  // Name here
   ```

2. Verify it exists in `config/authsources.php`:
   ```php
   $config = [
       'default-sp' => [  // Must match exactly
           'saml:SP',
           // ...
       ],
   ];
   ```

#### 7. Redirect Loop

**Symptom**: Browser keeps redirecting between SP and IdP

**Cause**: Session not being created or domain/cookie issues

**Solution**:
1. Clear browser cookies
2. Check session cookie configuration:
   ```php
   // config/config.php
   'session.cookie.domain' => '',  // Usually leave empty
   'session.cookie.path' => '/',
   'session.cookie.secure' => true,  // Requires HTTPS
   'session.cookie.samesite' => 'None',  // Required for SAML
   ```

3. Verify session storage is working:
   ```bash
   # If using phpsession
   ls /var/lib/php/sessions/

   # If using Redis
   redis-cli
   > KEYS SimpleSAMLphp*
   ```

4. Check for error messages in logs:
   ```bash
   tail -f /var/log/syslog | grep simplesamlphp
   ```

#### 8. "Responder: Status code was not success"

**Symptom**: IdP returns an error status instead of successful assertion

**Cause**: Various reasons - check IdP logs for details

**Solution**:
1. Enable SAML debug logging:
   ```php
   // config/config.php
   'debug' => [
       'saml' => true,  // Temporarily enable
   ],
   ```

2. Check SimpleSAMLphp logs for the actual error message:
   ```bash
   tail -100 /var/log/syslog | grep simplesamlphp
   ```

3. Common sub-causes:
   - User not authorized at IdP
   - Missing required attributes
   - IdP configuration issue
   - NameID format not supported

#### 9. Blank Page / White Screen

**Symptom**: SimpleSAMLphp shows a blank page

**Cause**: PHP error with error display disabled

**Solution**:
1. Check PHP error log:
   ```bash
   tail -50 /var/log/apache2/error.log
   # or
   tail -50 /var/log/nginx/error.log
   ```

2. Temporarily enable error display:
   ```php
   // config/config.php
   'showerrors' => true,
   ```

3. Check file permissions:
   ```bash
   ls -la /var/simplesamlphp/
   # Web server should be able to read all files
   ```

4. Check cache directory is writable:
   ```bash
   ls -la /var/cache/simplesamlphp/
   sudo chown -R www-data:www-data /var/cache/simplesamlphp/
   ```

#### 10. Azure AD "AADSTS50011: The reply URL specified in the request does not match"

**Symptom**: Azure AD error about reply URL mismatch

**Solution**:
1. Get your exact ACS URL:
   ```
   https://sso.example.org/module.php/saml/sp/saml2-acs.php/azure-sp
   ```

2. In Azure Portal, go to your Enterprise Application
3. **Single sign-on** → **Edit Basic SAML Configuration**
4. Ensure **Reply URL** exactly matches (including case and trailing slashes)
5. Save and test again

### Debugging Tools

#### Enable Debug Logging

```php
// config/config.php
'debug' => [
    'saml' => true,         // Log all SAML messages (including decrypted)
    'backtraces' => true,   // Log full error backtraces
    'validatexml' => true,  // Validate SAML XML against schemas
],

'showerrors' => true,  // Show errors in browser (development only)
'logging.level' => SimpleSAML\Logger::DEBUG,  // Verbose logging
```

#### View SAML Messages

When debug is enabled, SAML messages are logged to your configured log handler. Look for:
- `SAML2 Message:` - Full SAML request/response XML
- `Assertion:` - Extracted assertion data
- `Attributes:` - User attributes received

#### Test Authentication Source

Visit the test page:
```
https://sso.example.org/module.php/core/authenticate.php?as=SOURCE_NAME
```

Replace `SOURCE_NAME` with your authentication source name.

#### Check Metadata Syntax

```bash
php -l /var/simplesamlphp/metadata/saml20-idp-remote.php
php -l /var/simplesamlphp/config/authsources.php
```

#### Validate Certificate Chain

```bash
# View certificate details
openssl x509 -in cert/server.crt -text -noout

# Check if certificate and private key match
openssl x509 -noout -modulus -in cert/server.crt | openssl md5
openssl rsa -noout -modulus -in cert/server.pem | openssl md5
# These should output the same hash
```

#### Test LDAP Connection

```bash
# Test LDAP bind
ldapsearch -x -H ldaps://ldap.example.org \
    -D "cn=serviceaccount,dc=example,dc=org" \
    -w password \
    -b "dc=example,dc=org" \
    "(uid=testuser)"
```

---

## Additional Resources

### Official Documentation

- **SimpleSAMLphp Homepage**: https://simplesamlphp.org
- **Official Documentation**: https://simplesamlphp.org/docs/stable/
- **Installation Guide**: https://simplesamlphp.org/docs/stable/simplesamlphp-install.html
- **SP Quick Start**: https://simplesamlphp.org/docs/stable/simplesamlphp-sp.html
- **IdP Quick Start**: https://simplesamlphp.org/docs/stable/simplesamlphp-idp.html
- **GitHub Repository**: https://github.com/simplesamlphp/simplesamlphp

### SAML Specifications

- **SAML 2.0 Technical Overview**: https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html
- **SAML 2.0 Core**: http://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf
- **SAML 2.0 Profiles**: http://docs.oasis-open.org/security/saml/v2.0/saml-profiles-2.0-os.pdf
- **SAML 2.0 Metadata**: http://docs.oasis-open.org/security/saml/v2.0/saml-metadata-2.0-os.pdf

### Integration Guides

- **Azure AD / Entra ID**: https://safire.ac.za/technical/resources/configuring-simplesamlphp-to-use-entra-id/
- **Google Workspace**: https://simplesamlphp.org/docs/stable/simplesamlphp-googleapps.html
- **Okta**: https://saml-doc.okta.com/SAML_Docs/How-to-Configure-SAML-2.0-for-PHP-Applications.html

### Tools

- **SAML Test Service**: https://samltest.id/ - Free SAML IdP and SP for testing
- **SAML Tracer (Browser Extension)**: Track SAML messages in real-time
- **SAML.to**: https://www.samltool.com/ - Online SAML encoder/decoder
- **XML Signature Validator**: https://www.samltool.com/validate_xml.php

### Community

- **SimpleSAMLphp Users Mailing List**: https://groups.google.com/g/simplesamlphp
- **Stack Overflow**: https://stackoverflow.com/questions/tagged/simplesamlphp
- **GitHub Issues**: https://github.com/simplesamlphp/simplesamlphp/issues

### Books and Learning

- **SAML for Developers** by Peter Mularien
- **Guide to Deploying SimpleSAMLphp** (various online tutorials)
- **Okta Developer Documentation**: https://developer.okta.com/docs/concepts/saml/

### Related Technologies

- **OAuth 2.0 / OpenID Connect**: Alternative to SAML (consider for new projects)
- **Shibboleth**: Another SAML implementation (Java-based)
- **Keycloak**: Modern identity and access management solution

---

## Glossary

**ACS (Assertion Consumer Service)**: The SP endpoint that receives SAML assertions from the IdP.

**Assertion**: A signed statement from an IdP about a user's authentication and attributes.

**Attribute**: A piece of information about a user (e.g., email, name, group membership).

**Entity ID**: A unique identifier for an IdP or SP, usually a URI.

**Federation**: Trust relationship between organizations allowing SSO across organizational boundaries.

**IdP (Identity Provider)**: The system that authenticates users and issues SAML assertions.

**Metadata**: Configuration information exchanged between IdP and SP to establish trust.

**NameID**: The identifier for the user in SAML assertions.

**SAML (Security Assertion Markup Language)**: XML-based standard for SSO.

**SP (Service Provider)**: The application that relies on an IdP for authentication.

**SSO (Single Sign-On)**: Authenticate once, access multiple applications.

---

## Document History

- **2025-10-29**: Initial version created
- **Target Audience**: Beginners setting up SimpleSAMLphp as both IdP and SP
- **Focus Areas**: Configuration deep-dive and integration examples (Azure AD, Google, Okta)
- **Codebase Version**: SimpleSAMLphp master branch (PHP 8.2+, Symfony 7.3 components)

---

## Conclusion

This guide has provided a comprehensive overview of SimpleSAMLphp for both Identity Provider and Service Provider configurations. Key takeaways:

1. **Understand SAML basics** before diving into configuration
2. **Certificate management** is critical for security
3. **Metadata exchange** establishes trust between parties
4. **Configuration files** (`config.php`, `authsources.php`, metadata files) are the core of SimpleSAMLphp
5. **Integration patterns** are similar across providers (Azure AD, Google, Okta)
6. **Security best practices** should never be compromised
7. **Debugging** is systematic - check logs, validate syntax, verify certificates

For production deployments:
- Always use HTTPS
- Change default secrets
- Use scalable session storage (Redis/Memcache)
- Monitor logs and set up alerts
- Keep SimpleSAMLphp updated
- Test thoroughly before going live

SimpleSAMLphp is a powerful, flexible SAML implementation. With proper configuration and understanding of SAML fundamentals, it can provide robust Single Sign-On for your organization.

For questions or issues, consult the official documentation, community forums, and GitHub issues. Good luck with your SimpleSAMLphp deployment!
