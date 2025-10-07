# X12 Identifier Configuration Guide

**Date**: October 7, 2025  
**Status**: Implementation Complete  
**Purpose**: Guide for configuring X12 ISA/GS identifiers in partner configurations

---

## Overview

X12 identifiers are critical components of EDI transactions that uniquely identify trading partners at both the Interchange (ISA) and Functional Group (GS) levels. These identifiers are required for generating valid X12 envelopes for outbound transactions and properly routing inbound transactions.

### What are X12 Identifiers?

X12 EDI transactions use a hierarchical envelope structure:
- **ISA (Interchange Control Header)**: Outermost envelope identifying sender and receiver at the interchange level
- **GS (Functional Group Header)**: Inner envelope identifying application sender and receiver
- **ST (Transaction Set Header)**: Individual transaction identifier

### Why are they Required?

Without properly configured X12 identifiers, the platform cannot:
- Generate valid outbound X12 files
- Route inbound transactions to the correct trading partner
- Validate sender/receiver identity for security purposes
- Generate acknowledgments (TA1, 999) with proper sender/receiver identifiers

---

## X12 Identifier Configuration Model

### Model Structure

The `X12IdentifierConfig` model contains the following properties:

```csharp
public class X12IdentifierConfig
{
    public string IsaSenderId { get; set; }
    public string IsaReceiverId { get; set; }
    public string IsaSenderQualifier { get; set; } = "ZZ";
    public string IsaReceiverQualifier { get; set; } = "ZZ";
    public string GsApplicationSenderId { get; set; }
    public string GsApplicationReceiverId { get; set; }
}
```

### JSON Schema

Add this to your partner configuration JSON:

```json
{
  "x12Identifiers": {
    "isaSenderId": "string (max 15 chars)",
    "isaReceiverId": "string (max 15 chars)",
    "isaSenderQualifier": "string (2 chars, default: ZZ)",
    "isaReceiverQualifier": "string (2 chars, default: ZZ)",
    "gsApplicationSenderId": "string (2-15 chars)",
    "gsApplicationReceiverId": "string (2-15 chars)"
  }
}
```

---

## Field Descriptions

### ISA Sender ID (`isaSenderId`)

**X12 Segment**: ISA06  
**Format**: Up to 15 characters, left-justified and space-padded  
**Description**: Unique identifier for the sender at the interchange level

**Examples**:
- `"MYCOMPANYID"` - Company identifier
- `"123456789"` - Tax ID or DUNS number
- `"HEALTHPLAN01"` - Health plan identifier

### ISA Receiver ID (`isaReceiverId`)

**X12 Segment**: ISA08  
**Format**: Up to 15 characters, left-justified and space-padded  
**Description**: Unique identifier for the receiver at the interchange level

**Examples**:
- `"PARTNERA"` - Partner identifier
- `"987654321"` - Partner's Tax ID or DUNS number
- `"CLEARINGHOUSE"` - Clearinghouse identifier

### ISA Sender Qualifier (`isaSenderQualifier`)

**X12 Segment**: ISA05  
**Format**: Exactly 2 characters  
**Default**: `"ZZ"` (Mutually Defined)  
**Description**: Indicates the type of identification used in ISA06

**Common Values**:
| Code | Description |
|------|-------------|
| `01` | Duns (Dun & Bradstreet) |
| `14` | Duns Plus Suffix |
| `20` | Health Industry Number (HIN) |
| `27` | Carrier Identification Number |
| `28` | Fiscal Intermediary Identification Number |
| `29` | Medicare Provider Number |
| `30` | U.S. Federal Tax ID Number |
| `33` | NCPDP Provider ID |
| `ZZ` | Mutually Defined (most common) |

### ISA Receiver Qualifier (`isaReceiverQualifier`)

**X12 Segment**: ISA07  
**Format**: Exactly 2 characters  
**Default**: `"ZZ"` (Mutually Defined)  
**Description**: Indicates the type of identification used in ISA08

*Same values as ISA Sender Qualifier*

### GS Application Sender Code (`gsApplicationSenderId`)

**X12 Segment**: GS02  
**Format**: 2 to 15 characters  
**Description**: Sender's identification code at the functional group level (application-specific)

**Examples**:
- `"MYCOMPANYAPP"` - Company application identifier
- `"CLAIMSYS"` - Claims system identifier
- `"ELIGSVC"` - Eligibility service identifier

### GS Application Receiver Code (`gsApplicationReceiverId`)

**X12 Segment**: GS03  
**Format**: 2 to 15 characters  
**Description**: Receiver's identification code at the functional group level (application-specific)

**Examples**:
- `"PARTNERAAPP"` - Partner application identifier
- `"CLEARINGAPP"` - Clearinghouse application identifier
- `"PAYERSYS"` - Payer system identifier

---

## Configuration Examples

### Example 1: External Payer with Tax ID

```json
{
  "partnerCode": "ACME-HEALTH",
  "name": "ACME Health Insurance",
  "x12Identifiers": {
    "isaSenderId": "987654321",
    "isaReceiverId": "123456789",
    "isaSenderQualifier": "30",
    "isaReceiverQualifier": "30",
    "gsApplicationSenderId": "ACMEHEALTH",
    "gsApplicationReceiverId": "MYCOMPANYAPP"
  }
}
```

**Use Case**: Trading partner agreement specifies Tax ID numbers (qualifier `30`) for both parties.

### Example 2: Internal Partner with Mutually Defined IDs

```json
{
  "partnerCode": "INTERNAL-ELIGIBILITY",
  "name": "Internal Eligibility Service",
  "x12Identifiers": {
    "isaSenderId": "MYCOMPANYID",
    "isaReceiverId": "ELIGSERVICE",
    "isaSenderQualifier": "ZZ",
    "isaReceiverQualifier": "ZZ",
    "gsApplicationSenderId": "EDIPLATFORM",
    "gsApplicationReceiverId": "ELIGAPP"
  }
}
```

**Use Case**: Internal system using mutually defined identifiers (qualifier `ZZ`).

### Example 3: Clearinghouse with NCPDP Provider ID

```json
{
  "partnerCode": "PHARMACY-CLEARINGHOUSE",
  "name": "Pharmacy Claims Clearinghouse",
  "x12Identifiers": {
    "isaSenderId": "1234567",
    "isaReceiverId": "7654321",
    "isaSenderQualifier": "33",
    "isaReceiverQualifier": "33",
    "gsApplicationSenderId": "PHARM001",
    "gsApplicationReceiverId": "CLEAR001"
  }
}
```

**Use Case**: Pharmacy clearinghouse using NCPDP Provider IDs (qualifier `33`).

### Example 4: Bidirectional Partner

```json
{
  "partnerCode": "PARTNERA",
  "name": "Partner A Health",
  "x12Identifiers": {
    "isaSenderId": "MYCOMPANYID",
    "isaReceiverId": "PARTNERA",
    "isaSenderQualifier": "ZZ",
    "isaReceiverQualifier": "ZZ",
    "gsApplicationSenderId": "MYCOMPANYAPP",
    "gsApplicationReceiverId": "PARTNERAAPP"
  }
}
```

**Use Case**: Bidirectional trading partner where identifiers switch based on transaction direction (inbound vs outbound).

---

## How Identifiers are Used

### Outbound Transactions

When generating outbound X12 files, the platform uses X12 identifiers to create proper envelopes:

**Generated ISA Segment**:
```
ISA*00*          *00*          *ZZ*MYCOMPANYID    *ZZ*PARTNERA       *250107*1530*U*00401*000000001*0*P*:~
```

**Generated GS Segment**:
```
GS*HC*MYCOMPANYAPP*PARTNERAAPP*20250107*1530*1*X*004010X098A1~
```

### Inbound Transactions

When receiving inbound X12 files, the platform:
1. Parses ISA and GS segments
2. Extracts sender/receiver identifiers
3. Looks up partner configuration by matching `isaReceiverId` (we are the receiver) and `isaSenderId` (partner is the sender)
4. Routes transaction to appropriate Service Bus subscription
5. Validates sender identity for security

### Acknowledgments (999, TA1)

When generating acknowledgments, identifiers are **reversed**:
- Original ISA sender becomes acknowledgment ISA receiver
- Original ISA receiver becomes acknowledgment ISA sender
- Same logic applies to GS identifiers

**Example**:
Original 270 inquiry:
```
ISA...ZZ*PARTNERA       *ZZ*MYCOMPANYID    ...
GS*...PARTNERAAPP*MYCOMPANYAPP...
```

Generated 271 response:
```
ISA...ZZ*MYCOMPANYID    *ZZ*PARTNERA       ...
GS*...MYCOMPANYAPP*PARTNERAAPP...
```

---

## Validation Rules

The platform validates X12 identifiers with the following rules:

### Required Fields

All six identifier fields must be populated:
- `isaSenderId` (required, 1-15 characters)
- `isaReceiverId` (required, 1-15 characters)
- `isaSenderQualifier` (required, exactly 2 characters, default: `ZZ`)
- `isaReceiverQualifier` (required, exactly 2 characters, default: `ZZ`)
- `gsApplicationSenderId` (required, 2-15 characters)
- `gsApplicationReceiverId` (required, 2-15 characters)

### Format Rules

- **ISA IDs**: Maximum 15 characters, automatically space-padded by platform
- **ISA Qualifiers**: Exactly 2 characters, must be valid X12 qualifier code
- **GS IDs**: 2-15 characters, no automatic padding

### Validation Methods

**C# Validation**:
```csharp
var validator = new ConfigurationValidator(logger);
var errors = validator.ValidateX12Identifiers(partner.X12Identifiers, partner.PartnerCode);

if (errors.Any())
{
    // Handle validation errors
}
```

**Validation Output Example**:
```
ISA Sender ID is required
GS Application Sender ID must be at least 2 characters
ISA Sender Qualifier must be exactly 2 characters
```

---

## Partner Onboarding Workflow

### Step 1: Obtain Identifiers from Trading Partner

During partner onboarding, collect the following information:
- Trading Partner Agreement (TPA) document
- Partner's ISA sender/receiver IDs
- Partner's preferred ISA qualifier codes
- Partner's GS application sender/receiver codes
- Direction of data flow (inbound, outbound, bidirectional)

### Step 2: Configure Identifiers in Partner JSON

Create partner configuration file:
```json
{
  "partnerCode": "PARTNERA",
  "name": "Partner A Health",
  "x12Identifiers": {
    "isaSenderId": "MYCOMPANYID",
    "isaReceiverId": "PARTNERA",
    "isaSenderQualifier": "ZZ",
    "isaReceiverQualifier": "ZZ",
    "gsApplicationSenderId": "MYCOMPANYAPP",
    "gsApplicationReceiverId": "PARTNERAAPP"
  }
}
```

### Step 3: Validate Configuration

Run validation script:
```bash
python scripts/validate_partner_config.py \
  --config partners/partner-PARTNERA.json
```

### Step 4: Deploy Configuration

Deploy via GitHub Actions CI/CD pipeline:
```bash
git add partners/partner-PARTNERA.json
git commit -m "[PARTNER] Add: Partner A Health with X12 identifiers"
git push origin feature/add-partner-a
```

### Step 5: Test with Sample Transactions

Send test 270 inquiry with partner identifiers:
```
ISA*00*          *00*          *ZZ*PARTNERA       *ZZ*MYCOMPANYID    *250107*1530*U*00401*000000001*0*P*:~
GS*HS*PARTNERAAPP*MYCOMPANYAPP*20250107*1530*1*X*005010X279A1~
ST*270*0001*005010X279A1~
...
SE*10*0001~
GE*1*1~
IEA*1*000000001~
```

Verify routing to correct partner subscription and proper acknowledgment generation.

---

## Troubleshooting

### Issue: Partner Configuration Not Found

**Symptom**: Inbound transaction fails with "Partner not found" error

**Diagnosis**:
1. Check if `isaReceiverId` in inbound file matches `isaSenderId` in our configuration
2. Check if `isaSenderId` in inbound file matches `isaReceiverId` in our configuration
3. Verify partner status is `active`

**Resolution**: Update partner configuration with correct identifier mapping.

### Issue: Outbound File Rejected by Partner

**Symptom**: Partner rejects outbound file with "Invalid ISA sender" error

**Diagnosis**:
1. Check partner's expected ISA sender ID
2. Compare with `isaSenderId` in our configuration
3. Verify ISA qualifier matches partner expectation

**Resolution**: Update configuration to match partner's requirements, or request trading partner agreement update.

### Issue: Validation Errors on Configuration Load

**Symptom**: Partner configuration fails validation with identifier errors

**Diagnosis**:
```kusto
PartnerConfigService_CL
| where TimeGenerated > ago(1h)
| where Message contains "validation failed"
| project TimeGenerated, PartnerCode, Message
```

**Resolution**: Fix validation errors in partner JSON:
- Ensure all required fields are populated
- Verify ISA qualifiers are exactly 2 characters
- Check GS IDs are between 2-15 characters

---

## Migration Guide

### Existing Configurations Without X12 Identifiers

For existing partner configurations that don't have X12 identifiers:

**Before**:
```json
{
  "partnerCode": "PARTNERA",
  "name": "Partner A Health"
}
```

**After**:
```json
{
  "partnerCode": "PARTNERA",
  "name": "Partner A Health",
  "x12Identifiers": {
    "isaSenderId": "MYCOMPANYID",
    "isaReceiverId": "PARTNERA",
    "isaSenderQualifier": "ZZ",
    "isaReceiverQualifier": "ZZ",
    "gsApplicationSenderId": "MYCOMPANYAPP",
    "gsApplicationReceiverId": "PARTNERAAPP"
  }
}
```

### Backward Compatibility

The platform maintains backward compatibility:
- Existing configurations without `x12Identifiers` will still load successfully
- `PartnerConfiguration` will have empty string values for identifier fields if not configured
- Functions depending on identifiers should check for empty values and handle gracefully

---

## Reference

### X12 Documentation

- **ANSI X12 Standard**: ISA and GS segment specifications
- **HIPAA 5010 Implementation Guide**: Healthcare-specific identifier requirements
- **Trading Partner Agreements**: Partner-specific identifier mappings

### Related Platform Components

- **InboundRouter.Function**: Uses identifiers for partner lookup and routing
- **ControlNumberGenerator.Function**: Requires identifiers for envelope generation
- **OutboundOrchestrator.Function**: Uses identifiers for acknowledgment generation
- **SftpUploadFunction**: Uses identifiers for file naming and routing

### Code References

- `EDI.Configuration.Models.X12IdentifierConfig`: Model class
- `EDI.Configuration.Models.PartnerConfig`: Parent configuration model
- `EDI.Configuration.Services.PartnerConfigService`: Service for loading and mapping identifiers
- `EDI.Configuration.Validation.ConfigurationValidator`: Validation logic for identifiers

---

**Document Version**: 1.0  
**Last Updated**: October 7, 2025  
**Author**: EDI Platform Team  
**Status**: Complete
