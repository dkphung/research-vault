# InCommon Federation Integration with Clerk SAML SSO

**Date:** October 19, 2025
**Purpose:** Research how to query InCommon MDQ service to retrieve IdP metadata and establish SAML SSO connections in Clerk

## Context

InCommon Federation is a trusted identity federation for higher education and research institutions in the United States. It provides a metadata aggregation service that allows Service Providers (SPs) to discover and trust Identity Providers (IdPs) from member institutions.

This research explores:
1. How to query the InCommon Metadata Query (MDQ) service to retrieve IdP metadata
2. How to use that metadata to create SAML SSO connections in Clerk
3. Whether and how to register Clerk as a Service Provider in InCommon Federation

## InCommon Metadata Query (MDQ) Service

### Overview

InCommon provides a Metadata Query (MDQ) service that allows on-demand retrieval of individual entity metadata instead of downloading large aggregate metadata files. This is the recommended approach as of 2024.

### Production Endpoint

```
https://mdq.incommon.org/entities/{entityID}
```

Where `{entityID}` is the URL-encoded entity ID of the IdP you want to query.

### Example Query

To query metadata for an IdP with entity ID `https://idp.university.edu/shibboleth`:

```
https://mdq.incommon.org/entities/https://idp.university.edu/shibboleth
```

The entity ID must be URL-encoded if it contains special characters.

### Preview Environment

For testing upcoming changes:

```
https://mdq-preview.incommon.org/entities/{entityID}
```

### Response Format

The MDQ service returns SAML metadata XML containing:
- `<md:EntityDescriptor>` - The root element with the entity ID
- `<md:IDPSSODescriptor>` - IdP configuration including:
  - `<md:KeyDescriptor>` - X.509 certificates for signing/encryption
  - `<md:NameIDFormat>` - Supported name identifier formats
  - `<md:SingleSignOnService>` - SSO endpoint URLs and bindings
- Additional metadata about the organization and contact information

### Metadata Signing and Verification

**Critical Security Requirement:** All metadata from InCommon is digitally signed and MUST be verified before use.

#### Signing Certificate

Production metadata signing certificate:
- **Location:** https://spaces.at.internet2.edu/display/MDQ/production+metadata+signing+key
- **SHA256 Fingerprint:** `2F:9D:9A:A1:FE:D1:92:F0:64:A8:C6:31:5D:39:FA:CF:1E:08:84:0D:27:21:F3:31:B1:70:A5:2B:88:81:9F:5B`
- **SHA1 Fingerprint:** `7D:B4:BB:28:D3:D5:C8:52:E0:80:B3:62:43:2A:AF:34:B2:A6:0E:DD`
- **Validity Period:** December 16, 2013 to December 18, 2037

#### Verification Steps

1. Download the certificate from the official InCommon source
2. Compute SHA-1 and SHA-256 fingerprints of the downloaded certificate
3. Compare computed fingerprints to the published fingerprints above
4. If they match, use the certificate to verify the XML signature on metadata

#### Best Practices

- Refresh and verify metadata at least daily
- Optimal configuration: refresh metadata every hour
- Always verify the XML signature before trusting metadata

**Note:** There's a separate certificate for the newer Metadata Distribution Service 2.0, which is different from the legacy aggregate certificate.

## SAML Metadata Structure

### Required Elements in IdP Metadata

A typical SAML IdP metadata contains:

```xml
<md:EntityDescriptor entityID="https://idp.example.edu/shibboleth">
  <md:IDPSSODescriptor protocolSupportEnumeration="urn:oasis:names:tc:SAML:2.0:protocol">

    <!-- Signing Certificate -->
    <md:KeyDescriptor use="signing">
      <ds:KeyInfo>
        <ds:X509Data>
          <ds:X509Certificate>MIIDuDCCAqC...</ds:X509Certificate>
        </ds:X509Data>
      </ds:KeyInfo>
    </md:KeyDescriptor>

    <!-- Encryption Certificate (optional, may be same as signing) -->
    <md:KeyDescriptor use="encryption">
      <ds:KeyInfo>
        <ds:X509Data>
          <ds:X509Certificate>MIIDuDCCAqC...</ds:X509Certificate>
        </ds:X509Data>
      </ds:KeyInfo>
    </md:KeyDescriptor>

    <!-- Name ID Format -->
    <md:NameIDFormat>urn:oasis:names:tc:SAML:2.0:nameid-format:transient</md:NameIDFormat>

    <!-- SSO Service Endpoint -->
    <md:SingleSignOnService
      Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
      Location="https://idp.example.edu/idp/profile/SAML2/POST/SSO"/>

    <md:SingleSignOnService
      Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
      Location="https://idp.example.edu/idp/profile/SAML2/Redirect/SSO"/>

  </md:IDPSSODescriptor>

  <md:Organization>
    <md:OrganizationName xml:lang="en">Example University</md:OrganizationName>
    <md:OrganizationDisplayName xml:lang="en">Example U</md:OrganizationDisplayName>
    <md:OrganizationURL xml:lang="en">https://www.example.edu</md:OrganizationURL>
  </md:Organization>

</md:EntityDescriptor>
```

### Key Fields to Extract for Clerk

From the IdP metadata, you need:

1. **Entity ID** - The unique identifier for the IdP (from `entityID` attribute)
2. **SSO URL** - The endpoint where SAML requests should be sent (from `<md:SingleSignOnService>`)
3. **Certificate** - The X.509 certificate for validating SAML responses (from `<md:KeyDescriptor use="signing">`)

## Clerk SAML Connection Setup

### API Endpoint

```
POST /saml_connections
```

### Method 1: Using IdP Metadata URL (Recommended)

If the IdP provides a metadata URL (which InCommon MDQ does), this is the simplest approach:

```typescript
const response = await clerkClient.samlConnections.createSamlConnection({
  name: 'University of Example',
  provider: 'saml_custom', // Use custom for InCommon institutions
  domain: 'example.edu',
  idpMetadataUrl: 'https://mdq.incommon.org/entities/https://idp.example.edu/shibboleth',
  attributeMapping: {
    emailAddress: 'urn:oid:0.9.2342.19200300.100.1.3', // mail attribute
    firstName: 'urn:oid:2.5.4.42',                      // givenName
    lastName: 'urn:oid:2.5.4.4',                        // sn (surname)
  },
})
```

**Important Notes:**
- Clerk will automatically fetch and parse the metadata from the URL
- The MDQ URL includes the entity ID, so it's entity-specific
- You may need to handle URL encoding of the entity ID properly

### Method 2: Using Raw Metadata XML

If you fetch the metadata yourself and want to provide it directly:

```typescript
const metadataXml = `<?xml version="1.0"?>
<md:EntityDescriptor ...>
  <!-- Full metadata XML here -->
</md:EntityDescriptor>`

const response = await clerkClient.samlConnections.createSamlConnection({
  name: 'University of Example',
  provider: 'saml_custom',
  domain: 'example.edu',
  idpMetadata: metadataXml,
  attributeMapping: {
    emailAddress: 'urn:oid:0.9.2342.19200300.100.1.3',
    firstName: 'urn:oid:2.5.4.42',
    lastName: 'urn:oid:2.5.4.4',
  },
})
```

### Method 3: Using Individual Parameters

If you parse the metadata yourself and extract the required fields:

```typescript
const response = await clerkClient.samlConnections.createSamlConnection({
  name: 'University of Example',
  provider: 'saml_custom',
  domain: 'example.edu',
  idpEntityId: 'https://idp.example.edu/shibboleth',
  idpSsoUrl: 'https://idp.example.edu/idp/profile/SAML2/POST/SSO',
  idpCertificate: '-----BEGIN CERTIFICATE-----\nMIIDuDCCAqC...\n-----END CERTIFICATE-----',
  attributeMapping: {
    emailAddress: 'urn:oid:0.9.2342.19200300.100.1.3',
    firstName: 'urn:oid:2.5.4.42',
    lastName: 'urn:oid:2.5.4.4',
  },
})
```

### Priority Order

When multiple parameters are provided, Clerk uses this priority:
1. `idpMetadata` (raw XML) - highest priority
2. `idpMetadataUrl` - second priority
3. Individual parameters (`idpEntityId`, `idpSsoUrl`, `idpCertificate`) - used if neither above is provided

### Common Attribute Mappings for InCommon

InCommon institutions typically use eduPerson schema attributes. Common mappings:

| Clerk Field | Common InCommon Attribute | OID Format |
|-------------|---------------------------|------------|
| emailAddress | mail | urn:oid:0.9.2342.19200300.100.1.3 |
| firstName | givenName | urn:oid:2.5.4.42 |
| lastName | sn (surname) | urn:oid:2.5.4.4 |
| userId | eduPersonPrincipalName | urn:oid:1.3.6.1.4.1.5923.1.1.1.6 |

**Note:** Attribute naming conventions vary by institution. You may need to check the specific IdP's attribute release policy.

### Error Handling

Clerk may return these SAML-specific errors:

- `saml_failed_to_fetch_idp_metadata` (400) - Unable to retrieve metadata from URL
- `saml_failed_to_parse_idp_metadata` (422) - Metadata XML is invalid or malformed
- `saml_email_address_domain_mismatch` (400) - User's email domain doesn't match configured domain

If metadata fetching fails, fall back to providing explicit configuration (Method 3).

## Registering Clerk as a Service Provider in InCommon

### Process Overview

Registering Clerk as an SP in InCommon Federation is a **manual process** that requires:
1. InCommon membership (institutional or sponsored)
2. Access to the Federation Manager as a Site Administrator
3. Clerk SP metadata

### Prerequisites

- Your institution must be an InCommon member
- You need Site Administrator access to Federation Manager
- You should have a clear understanding of your organization's entity ID strategy

### Steps to Register

1. **Access Federation Manager**
   - Log in as a Site Administrator
   - Navigate to the SA Dashboard
   - Click "Add New Service Provider"

2. **Entity Configuration**
   - Enter the Entity ID for your Clerk SP
     - Format: `https://clerk.{your-domain}/saml/metadata/{connection-id}` (example)
     - **Critical:** Entity IDs cannot be easily changed after publication
   - Define the scope of your service provider

3. **SP Metadata Submission**
   - Provide Clerk's SP metadata, which includes:
     - **Entity ID** - Unique identifier for Clerk SP
     - **ACS URL** - Assertion Consumer Service URL where IdP sends responses
     - **Certificate** - Public key for encrypting assertions (optional)

   **Finding Clerk SP Metadata:**
   - In Clerk Dashboard, create a SAML connection
   - Navigate to the connection's configuration page
   - Find the "Service Provider configuration" section
   - Copy the Metadata URL, ACS URL, and Entity ID

4. **Metadata URL vs. Individual Values**
   - **Recommended:** Provide the Metadata URL (quickest and most reliable)
   - **Alternative:** Provide ACS URL and Entity ID individually

5. **Review and Submit**
   - Navigate to "Review and Submit" section
   - Submit your entity for publication
   - **Important:** Changes aren't published until you complete this step

### Publication Timeline

- InCommon reviews metadata submissions Monday-Friday at ~2:30 PM Eastern
- Updated metadata published at ~3:00 PM Eastern (times may vary)
- **Publication window:** 24-72 business hours depending on submission time
- **Additional propagation:** 24-48 hours for eduGAIN metadata distribution

### Important Considerations

1. **Entity ID Selection**
   - Choose carefully - very difficult to change later
   - Should be stable and long-lived
   - Typically uses HTTPS URL format
   - Should be unique across all InCommon entities

2. **Organizational Scope**
   - Determine if this is a single-tenant or multi-tenant deployment
   - For multi-tenant (multiple universities using your Clerk instance), consider whether each needs separate entity registration

3. **Attribute Release**
   - IdPs control what attributes they release
   - You may need to work with each university's IdP administrator to ensure proper attribute release
   - Standard InCommon attribute release policies may not include all needed attributes

4. **Research & Scholarship (R&S) Category**
   - Consider whether your SP qualifies for R&S entity category
   - R&S entities get standard attribute release from participating IdPs
   - Requirements: https://refeds.org/category/research-and-scholarship

## Recommended Implementation Workflow

### Phase 1: Query and Test (No InCommon Registration)

1. **Identify Target IdP**
   - Obtain the entity ID of the university's IdP from the institution
   - Example: `https://idp.university.edu/shibboleth`

2. **Query InCommon MDQ**
   ```bash
   curl -v "https://mdq.incommon.org/entities/https://idp.university.edu/shibboleth"
   ```

3. **Verify Metadata Signature**
   - Download InCommon signing certificate
   - Verify certificate fingerprints match published values
   - Use XML signature verification tool to validate metadata

4. **Create Clerk SAML Connection**
   ```typescript
   // Option A: Use MDQ URL directly (simplest)
   const connection = await clerkClient.samlConnections.createSamlConnection({
     name: 'University Name',
     provider: 'saml_custom',
     domain: 'university.edu',
     idpMetadataUrl: 'https://mdq.incommon.org/entities/https://idp.university.edu/shibboleth',
     attributeMapping: {
       emailAddress: 'urn:oid:0.9.2342.19200300.100.1.3',
       firstName: 'urn:oid:2.5.4.42',
       lastName: 'urn:oid:2.5.4.4',
     },
   })

   // Option B: Fetch metadata yourself and parse
   const metadataXml = await fetch('https://mdq.incommon.org/entities/...')
   // Parse XML to extract idpEntityId, idpSsoUrl, idpCertificate
   const connection = await clerkClient.samlConnections.createSamlConnection({
     name: 'University Name',
     provider: 'saml_custom',
     domain: 'university.edu',
     idpEntityId: parsedEntityId,
     idpSsoUrl: parsedSsoUrl,
     idpCertificate: parsedCertificate,
     attributeMapping: { /* ... */ },
   })
   ```

5. **Coordinate with University IdP Administrator**
   - Provide Clerk's SP metadata (ACS URL, Entity ID) to the university
   - **Important:** University must add your SP to their IdP configuration
   - They may add you as a local SP or request you join InCommon

6. **Test Authentication Flow**
   - Attempt SSO login with a test account
   - Verify attribute release
   - Adjust attribute mappings as needed

### Phase 2: InCommon Registration (If Required)

**When to Register:**
- Multiple universities request it
- Universities won't configure local SP entries
- You want automatic trust without per-university configuration
- You want to leverage R&S attribute release

**Registration Process:**
1. Confirm InCommon membership or get sponsored
2. Access Federation Manager as Site Administrator
3. Retrieve Clerk SP metadata from Clerk Dashboard
4. Submit SP registration with proper entity ID
5. Wait 24-72 business hours for publication
6. Monitor metadata propagation
7. Test with participating universities

**Alternative: Per-University Configuration**
- Each university manually adds your SP to their IdP
- No InCommon registration required
- More manual coordination but faster initial setup
- Good for pilot or small number of institutions

## Technical Challenges and Solutions

### Challenge 1: MDQ URL Accessibility

**Issue:** Clerk may not be able to fetch from InCommon MDQ if:
- URL encoding issues with entity ID
- Network/firewall restrictions
- Certificate validation issues

**Solutions:**
1. Use raw metadata XML (`idpMetadata`) instead of URL
2. Fetch metadata yourself and extract individual parameters
3. Implement a metadata proxy/cache in your infrastructure

### Challenge 2: Metadata Signing Verification

**Issue:** InCommon metadata is signed, but Clerk may not verify signatures when fetching.

**Solutions:**
1. Implement verification in your application layer before passing to Clerk
2. Use a SAML metadata proxy that verifies signatures
3. Trust InCommon's TLS/HTTPS for transport security (less secure)

**Recommended:** Implement signature verification in your code:
```typescript
import { SignedXml } from 'xml-crypto'
import * as fs from 'fs'

function verifyInCommonMetadata(metadataXml: string, certPath: string): boolean {
  const cert = fs.readFileSync(certPath).toString()
  const sig = new SignedXml()
  sig.keyInfoProvider = {
    getKey: () => cert,
    getKeyInfo: () => '<X509Data></X509Data>'
  }
  sig.loadSignature(metadataXml)
  return sig.checkSignature(metadataXml)
}
```

### Challenge 3: Attribute Mapping Variability

**Issue:** Different universities release attributes with different names/formats.

**Solutions:**
1. Maintain a mapping configuration per institution
2. Use OID format (more standardized) instead of friendly names
3. Request universities follow InCommon attribute release policies
4. Implement fallback attribute mappings

```typescript
const attributeMappings = {
  'university1.edu': {
    emailAddress: 'mail',
    firstName: 'givenName',
    lastName: 'sn',
  },
  'university2.edu': {
    emailAddress: 'urn:oid:0.9.2342.19200300.100.1.3',
    firstName: 'urn:oid:2.5.4.42',
    lastName: 'urn:oid:2.5.4.4',
  },
}
```

### Challenge 4: Entity Discovery

**Issue:** How do users find the right IdP to authenticate with?

**Solutions:**
1. **Domain-based discovery:** Map email domain to IdP entity ID
2. **Search interface:** Allow users to search for their institution
3. **Bookmark/remember:** Store user's IdP choice for future logins
4. **SAML Discovery Service:** Use InCommon's or build your own

Example discovery mapping:
```typescript
const idpDiscovery = {
  'university.edu': 'https://idp.university.edu/shibboleth',
  'college.edu': 'https://sso.college.edu/idp',
}

function discoverIdP(email: string): string | null {
  const domain = email.split('@')[1]
  return idpDiscovery[domain] || null
}
```

## Security Considerations

1. **Metadata Verification**
   - Always verify InCommon metadata signatures
   - Refresh metadata regularly (hourly recommended)
   - Monitor for certificate expiration

2. **Certificate Management**
   - Store InCommon signing certificate securely
   - Implement certificate rotation procedures
   - Monitor InCommon announcements for certificate changes

3. **Entity ID Validation**
   - Validate that SAML responses come from expected entity IDs
   - Don't trust entity IDs in unsigned assertions
   - Maintain allowlist of trusted IdP entity IDs

4. **Attribute Validation**
   - Validate email domain matches expected domain
   - Implement additional authorization checks beyond authentication
   - Don't assume all attributes are always present

5. **Session Management**
   - Implement proper session timeout
   - Support Single Logout (SLO) if possible
   - Monitor for session fixation attacks

## Recommendations

### For Proof of Concept

1. **Start Simple:** Use Method 1 (MDQ URL) for initial testing
2. **Single University:** Partner with one university IT department
3. **Local SP Configuration:** Have university add your SP locally (no InCommon registration)
4. **Test Thoroughly:** Verify authentication, attribute release, and error handling

### For Production Deployment

1. **Register with InCommon:** If supporting multiple universities
2. **Implement Metadata Verification:** Don't trust unverified metadata
3. **Build Discovery Service:** Make it easy for users to find their IdP
4. **Monitor and Log:** Track authentication attempts, failures, and attribute issues
5. **Documentation:** Provide clear instructions for university IT administrators
6. **Support Plan:** Have process for assisting universities with configuration

### For Scale

1. **Metadata Cache:** Cache InCommon metadata locally, refresh hourly
2. **Monitoring:** Alert on metadata fetch failures, certificate expiration
3. **Automation:** Automate SAML connection creation from metadata
4. **Fallback:** Support multiple methods (metadata URL, XML, individual params)
5. **Multi-tenancy:** Design for many universities on one Clerk instance

## Example Implementation Pseudocode

```typescript
// 1. Discover IdP based on user's email domain
const userEmail = 'student@university.edu'
const domain = userEmail.split('@')[1]
const idpEntityId = discoverIdPByDomain(domain)

if (!idpEntityId) {
  throw new Error('University not supported')
}

// 2. Fetch metadata from InCommon MDQ
const mdqUrl = `https://mdq.incommon.org/entities/${encodeURIComponent(idpEntityId)}`
const metadataResponse = await fetch(mdqUrl)
const metadataXml = await metadataResponse.text()

// 3. Verify metadata signature (recommended)
const isValid = verifyInCommonMetadata(
  metadataXml,
  '/path/to/incommon-signing-cert.pem'
)

if (!isValid) {
  throw new Error('Invalid metadata signature')
}

// 4. Check if SAML connection already exists for this domain
let connection = await findExistingSamlConnection(domain)

if (!connection) {
  // 5. Create new SAML connection in Clerk
  connection = await clerkClient.samlConnections.createSamlConnection({
    name: `${domain} SSO`,
    provider: 'saml_custom',
    domain: domain,
    idpMetadata: metadataXml,
    attributeMapping: {
      emailAddress: 'urn:oid:0.9.2342.19200300.100.1.3',
      firstName: 'urn:oid:2.5.4.42',
      lastName: 'urn:oid:2.5.4.4',
    },
    active: true,
  })

  // 6. Store connection mapping
  await storeConnectionMapping(domain, connection.id)
}

// 7. Initiate SAML authentication
return redirectToSamlAuth(connection.id)
```

## Open Questions

1. **Clerk Metadata Refresh:** How often does Clerk refresh metadata from `idpMetadataUrl`?
2. **Signature Verification:** Does Clerk verify InCommon metadata signatures automatically?
3. **Entity ID Format:** What entity ID format does Clerk generate for SP metadata?
4. **Multi-tenancy:** Can one Clerk instance support multiple domains with different SAML connections?
5. **Discovery Service:** Does Clerk have built-in IdP discovery, or must this be custom-built?

## Next Steps

1. **Proof of Concept:**
   - Partner with one university
   - Obtain their IdP entity ID
   - Test MDQ query and metadata retrieval
   - Create test SAML connection in Clerk
   - Coordinate SP setup with university IT
   - Test end-to-end authentication

2. **Metadata Verification:**
   - Implement InCommon signature verification
   - Test with production signing certificate
   - Set up metadata refresh automation

3. **Discovery Service:**
   - Design user interface for institution selection
   - Build domain-to-entity-ID mapping
   - Implement search functionality

4. **InCommon Registration Decision:**
   - Evaluate need for federation membership
   - Cost-benefit analysis of registration vs. per-university setup
   - Timeline for production deployment

## References

- InCommon MDQ Service: https://spaces.at.internet2.edu/display/MDQ
- InCommon Federation: https://incommon.org/federation/
- Clerk SAML Documentation: https://clerk.com/docs/authentication/enterprise-connections/saml
- SAML Metadata Specification: http://docs.oasis-open.org/security/saml/v2.0/
- InCommon Metadata Signing Certificate: https://ops.incommon.org/inc_md_cert_mdq.html
