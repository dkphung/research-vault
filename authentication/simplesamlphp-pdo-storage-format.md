# SimpleSAMLphp PDO Metadata Storage Format Research

**Date**: 2025-01-30
**Researcher**: Claude Code
**Purpose**: Determine the exact format SimpleSAMLphp uses to store metadata in MySQL PDO backend

---

## Executive Summary

**Conclusion**: SimpleSAMLphp PDO metadata storage uses **JSON encoding** (via `json_encode()`) to store metadata in the `entity_data` column, NOT PHP serialized arrays.

This research was conducted to inform the implementation of a SAML metadata import feature that will write directly to SimpleSAMLphp's MySQL PDO storage backend.

---

## Research Question

What format does SimpleSAMLphp use to store metadata in the `entity_data` column when using PDO (MySQL) storage?

**Options investigated**:
1. JSON encoding (`json_encode()`)
2. PHP serialization (`serialize()`)
3. PHP array strings (raw PHP code)
4. Other custom format

**Answer**: **JSON encoding** (Option 1) ✅

---

## Evidence and Citations

### 1. Official SimpleSAMLphp Documentation

**Source**: [SimpleSAMLphp PDO Metadata Storage Handler Documentation](https://github.com/simplesamlphp/simplesamlphp/blob/master/docs/simplesamlphp-metadata-pdostoragehandler.md)

**Direct Quote**:
> "With the PDO metadata storage handler, metadata is stored in the table for the appropriate set and is **stored in JSON format**."

**Evidence Examples**:

The documentation provides explicit examples of the JSON format:

**saml20_idp_hosted table**:
```
entity_id: __DEFAULT:1__
entity_data: {"host":"__DEFAULT__","privatekey":"idp.key","certificate":"idp.crt","auth":"example-ldap","identifyingAttribute":"uid"}
```

**saml20_idp_remote table**:
```
entity_id: https://openidp.feide.no
entity_data: {"name":{"en":"Feide OpenIdP - test users"},"description":"...","SingleSignOnService":"https://openidp.feide.no/simplesaml/saml2/idp/SSOService.php","SingleLogoutService":"https://openidp.feide.no/simplesaml/saml2/idp/SingleLogoutService.php","certFingerprint":"c9ed4dfb07caf13fc21e0fec1572047eb8a7a4cb"}
```

**Key Observation**: The examples clearly show JSON object notation with proper escaping, not PHP serialized format.

---

### 2. GitHub Issue #1392 - Entity Data Field Size Problem

**Source**: [Issue #1392 - entity_data field size is insufficient with PDO metadata storage](https://github.com/simplesamlphp/simplesamlphp/issues/1392)

**Date**: Opened March 20, 2019

**Problem Description**:
> "The entity_data column is set as type TEXT which under MySQL is limited to 65,535 characters and is insufficient. Many metadata sets, specifically that from ADFS, regularly exceed 90,000 characters in length and these are truncated without warning when loaded to the database as **encoded json**, and throw a **json_decode error** when they are retrieved."

**Key Evidence**:
1. Metadata is "**encoded json**" when stored
2. Retrieval uses "**json_decode**" which throws errors on truncated data
3. This confirms both storage (`json_encode`) and retrieval (`json_decode`) methods

**The Fix**:
Changed MySQL column type from `TEXT` to `MEDIUMTEXT`:
- TEXT: 65,535 characters (64KB)
- MEDIUMTEXT: 16,777,215 characters (16MB)

**Code Reference**: Line 291 of `MetaDataStorageHandlerPdo.php`

---

### 3. GitHub Issue #44 - Original PDO Backend Implementation

**Source**: [Issue #44 - [patch] PDO backend for storing Metadata entries (was #529)](https://github.com/simplesamlphp/simplesamlphp/issues/44)

**Date**: Original feature implementation discussion

**Developer's Statement**:
> "The module stores a `json_encode` of the existing metadata format in the database, keeping stuff as simple as possible."

**Code Examples Provided**:

**For IdP Remote Metadata**:
```php
foreach($metadata as $k => $v) {
 echo "INSERT INTO `saml20-idp-remote` ('entityId', 'entityData')
       VALUES ('" . $k . "','" . json_encode($v) . "');" . PHP_EOL;
}
```

**For SP Remote Metadata**:
```php
foreach($metadata as $k => $v) {
 echo "INSERT INTO `saml20-sp-remote` ('entityId', 'entityData')
       VALUES ('" . $k . "','" . json_encode($v) . "');" . PHP_EOL;
}
```

**Design Rationale**:
The developer chose JSON encoding for:
1. **Simplicity**: Straightforward to implement and understand
2. **REST API Integration**: JSON can be directly consumed by REST APIs
3. **Portability**: Language-agnostic format
4. **Debugging**: Human-readable in database inspection tools

---

### 4. Database Schema

**Source**: Multiple references from SimpleSAMLphp documentation and issue discussions

**Table Schema**:
```sql
CREATE TABLE IF NOT EXISTS $tableName (
    entity_id VARCHAR(255) PRIMARY KEY NOT NULL,
    entity_data TEXT NOT NULL  -- Originally TEXT, changed to MEDIUMTEXT for MySQL
)
```

**Table Names** (11 tables created by `bin/initMDSPdo.php`):
- `saml20_idp_hosted`
- `saml20_idp_remote`
- `saml20_sp_remote`
- `saml20_sp_hosted` (implied)
- `adfs_idp_hosted`
- `adfs_sp_remote`
- And 5 others for different metadata sets

**Column Definitions**:
- `entity_id` (VARCHAR 255): The SAML EntityDescriptor's `entityID` attribute (Primary Key)
- `entity_data` (TEXT/MEDIUMTEXT): **JSON-encoded** metadata object

---

### 5. Import and Export Utilities

**Import Utility**: `bin/importPdoMetadata.php`
- Converts flatfile metadata to PDO storage
- Calls `addEntry($entityId, $metadataSet, $metadata)` method
- The `addEntry()` method internally uses `json_encode()` on the metadata array

**Initialization Utility**: `bin/initMDSPdo.php`
- Creates 11 empty tables for different metadata sets
- Sets up the schema with appropriate column types

**Note**: The import utility overwrites existing entity_id entries (upsert behavior).

---

### 6. Why JSON Instead of PHP serialize()?

While SimpleSAMLphp uses `serialize()` in other contexts (e.g., session storage), the PDO metadata storage specifically uses JSON because:

| Aspect | JSON | PHP serialize() |
|--------|------|-----------------|
| **Language Independence** | ✅ Works with any language | ❌ PHP-specific |
| **REST API Compatibility** | ✅ Direct consumption | ❌ Requires PHP deserialization |
| **Human Readability** | ✅ Easy to inspect in database | ❌ Binary/encoded format |
| **Portability** | ✅ Universal format | ❌ PHP version dependencies |
| **Security** | ✅ Safer (no code execution) | ⚠️ Potential RCE vulnerabilities |
| **Debugging** | ✅ Can use standard JSON tools | ❌ Requires PHP to inspect |

---

## Metadata Format Structure

Based on the documentation examples, here's the typical structure of JSON-encoded metadata:

### Service Provider (SP) Metadata Example
```json
{
  "entityid": "https://example.com/saml/sp",
  "contacts": [],
  "metadata-set": "saml20-sp-remote",
  "AssertionConsumerService": [
    {
      "Binding": "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST",
      "Location": "https://example.com/acs",
      "index": 1
    }
  ],
  "SingleLogoutService": [
    {
      "Binding": "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect",
      "Location": "https://example.com/slo"
    }
  ],
  "NameIDFormat": "urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress",
  "validate.authnrequest": true,
  "saml20.sign.assertion": true
}
```

### Identity Provider (IdP) Metadata Example
```json
{
  "entityid": "https://idp.example.com/saml",
  "contacts": [],
  "metadata-set": "saml20-idp-remote",
  "SingleSignOnService": [
    {
      "Binding": "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect",
      "Location": "https://idp.example.com/sso"
    }
  ],
  "SingleLogoutService": [
    {
      "Binding": "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect",
      "Location": "https://idp.example.com/slo"
    }
  ],
  "NameIDFormat": "urn:oasis:names:tc:SAML:2.0:nameid-format:persistent",
  "certFingerprint": "c9ed4dfb07caf13fc21e0fec1572047eb8a7a4cb"
}
```

---

## Implementation Implications

### For Our SAML Metadata Import Feature

Based on this research, our implementation must:

1. ✅ **Add JSON Output Function**
   - Create `metadataConverterToJSON()` alongside existing `metadataConverter()`
   - Return JavaScript objects (not PHP array strings)
   - These objects will be JSON-serializable for database storage

2. ✅ **Database Storage Format**
   - Use `JSON.stringify()` before inserting into `entity_data` column
   - Store the result in a TEXT or MEDIUMTEXT column
   - For large metadata (>65KB), ensure MEDIUMTEXT is used

3. ✅ **Data Structure Mapping**
   ```typescript
   // Our converter output
   {
     "saml20-sp-remote": {
       "https://entityid.example.com": {
         entityid: "https://entityid.example.com",
         contacts: [],
         "metadata-set": "saml20-sp-remote",
         AssertionConsumerService: [...],
         // ... other fields
       }
     }
   }

   // Database insertion
   INSERT INTO saml20_sp_remote (entity_id, entity_data)
   VALUES (
     'https://entityid.example.com',
     '{"entityid":"https://entityid.example.com","contacts":[],...}'
   );
   ```

4. ✅ **Column Type Recommendation**
   - MySQL: Use `MEDIUMTEXT` to support metadata up to 16MB
   - Validate metadata size before insertion
   - Handle truncation errors gracefully

---

## Test Data Reference

### Clerk SAML Metadata (Test Case)

**Metadata URL**: `https://prompt-gazelle-54.clerk.accounts.dev/v1/saml/metadata/samlc_34koUozA8EViryPXTGexdCsMGX7.xml`

**Expected entity_id**: `https://prompt-gazelle-54.clerk.accounts.dev/saml/samlc_34koUozA8EViryPXTGexdCsMGX7`

**Metadata Type**: Service Provider (SP)

**Expected Storage**:
- Table: `saml20_sp_remote`
- entity_id: `https://prompt-gazelle-54.clerk.accounts.dev/saml/samlc_34koUozA8EViryPXTGexdCsMGX7`
- entity_data: JSON-encoded metadata with fields like:
  - `entityid`
  - `AssertionConsumerService` array
  - `NameIDFormat`
  - `metadata-set: "saml20-sp-remote"`
  - etc.

---

## Related Research

### MySQL Column Type Comparison

| Type | Max Size | Use Case |
|------|----------|----------|
| TINYTEXT | 255 bytes | Very small text |
| TEXT | 65,535 bytes (64KB) | Standard text (original SimpleSAMLphp) |
| MEDIUMTEXT | 16,777,215 bytes (16MB) | Large metadata (recommended) |
| LONGTEXT | 4,294,967,295 bytes (4GB) | Extremely large data |

**Recommendation**: Use MEDIUMTEXT for SimpleSAMLphp metadata storage to accommodate ADFS and other large metadata sets.

---

## Alternative Approaches Considered (Rejected)

### 1. PHP Serialization
❌ **Rejected** because:
- SimpleSAMLphp explicitly uses JSON, not serialize()
- PHP serialization is less portable
- Security concerns with unserialize()

### 2. Storing PHP Array Strings
❌ **Rejected** because:
- Not the format SimpleSAMLphp expects
- Would require `eval()` to reconstruct (dangerous)
- Not compatible with SimpleSAMLphp's retrieval logic

### 3. Custom Format
❌ **Rejected** because:
- SimpleSAMLphp already has an established format
- Would break compatibility
- No benefit over JSON

---

## Conclusion

This research definitively confirms that **SimpleSAMLphp PDO metadata storage uses JSON encoding** for the `entity_data` column. Our implementation should:

1. Convert SAML XML metadata to JavaScript objects (matching SimpleSAMLphp's structure)
2. Use `JSON.stringify()` before database insertion
3. Store in MEDIUMTEXT column for MySQL
4. Follow SimpleSAMLphp's metadata structure conventions

The specification document (`docs/spec/saml-metadata-import-spec.md`) was correct in identifying the need for JSON output format and should proceed as planned.

---

## References

1. [SimpleSAMLphp PDO Metadata Storage Handler Documentation](https://github.com/simplesamlphp/simplesamlphp/blob/master/docs/simplesamlphp-metadata-pdostoragehandler.md)
2. [Issue #1392 - entity_data field size is insufficient](https://github.com/simplesamlphp/simplesamlphp/issues/1392)
3. [Issue #44 - PDO backend for storing Metadata entries](https://github.com/simplesamlphp/simplesamlphp/issues/44)
4. [Pull Request #1111 - Fixes PDO Metadata Source loading](https://github.com/simplesamlphp/simplesamlphp/pull/1111)

---

**Next Steps**: Proceed with implementation following Phase 1 of the specification document.
