# Glossary and Terminology

**Document Version:** 1.0  
**Last Updated:** October 6, 2025  
**Purpose:** Comprehensive glossary of EDI, X12, healthcare, and Azure platform terminology used throughout the EDI Platform documentation

---

## Table of Contents

1. [EDI Transaction Types](#1-edi-transaction-types)
2. [X12 Segments](#2-x12-segments)
3. [X12 Elements and Qualifiers](#3-x12-elements-and-qualifiers)
4. [Healthcare Terminology](#4-healthcare-terminology)
5. [Azure Services](#5-azure-services)
6. [Platform Components](#6-platform-components)
7. [Process and Workflow Terms](#7-process-and-workflow-terms)
8. [Acronyms](#8-acronyms)

---

## 1. EDI Transaction Types

### 270 - Health Care Eligibility/Benefit Inquiry

**Purpose:** Request eligibility and benefit information from a payer  
**Direction:** Provider/Clearinghouse → Payer  
**Version:** HIPAA 5010 (005010X279A1)  
**Key Segments:** ISA, GS, ST, BHT, HL, NM1, DMG, INS, DTP, EQ  
**Use Cases:**
- Real-time eligibility verification
- Benefit coverage inquiry
- Co-pay and deductible lookup
- Prior authorization status check

**Example Use:** Provider verifies patient's active insurance coverage before appointment

---

### 271 - Health Care Eligibility/Benefit Response

**Purpose:** Return eligibility and benefit information from payer  
**Direction:** Payer → Provider/Clearinghouse  
**Version:** HIPAA 5010 (005010X279A1)  
**Key Segments:** ISA, GS, ST, BHT, HL, NM1, DMG, EB, III, REF  
**Response Codes:**
- EB01 = 1: Active coverage
- EB01 = 6: Inactive coverage
- EB01 = 7: Coverage outside period
- EB01 = 8: Not eligible

**Example Response:** "Member MEMBER123 has active medical coverage effective 2025-01-01 with $35 co-pay for office visits"

---

### 276 - Health Care Claim Status Request

**Purpose:** Inquire about the status of previously submitted claims  
**Direction:** Provider/Clearinghouse → Payer  
**Version:** HIPAA 5010 (005010X212)  
**Key Segments:** ISA, GS, ST, BHT, HL, NM1, REF, DMG, TRN  
**Inquiry Types:**
- Claim status by claim control number
- Claim status by patient and service date
- Batch status inquiry

---

### 277 - Health Care Claim Status Response

**Purpose:** Provide claim processing status information  
**Direction:** Payer → Provider/Clearinghouse  
**Version:** HIPAA 5010 (005010X212)  
**Key Segments:** ISA, GS, ST, BHT, HL, NM1, STC, QTY, AMT  
**Status Codes:**
- STC01 = A1: Processed according to guidelines
- STC01 = A2: Processed as adjusted
- STC01 = A3: Forwarded to another payer
- STC01 = A4: Not yet found/processed
- STC01 = A5: Denied

---

### 278 - Health Care Services Review (Prior Authorization)

**Purpose:** Request or respond to prior authorization/referral requests  
**Direction:** Bidirectional (Request and Response)  
**Version:** HIPAA 5010 (005010X217)  
**Key Segments:** ISA, GS, ST, BHT, HL, NM1, HSD, REF, DTP, HI  
**Certification Types:**
- Initial request
- Reconsideration request
- Extension request
- Cancellation notification

---

### 834 - Benefit Enrollment and Maintenance

**Purpose:** Communicate member enrollment, changes, and terminations  
**Direction:** Sponsor/Employer/TPA → Payer  
**Version:** HIPAA 5010 (005010X220A1)  
**Key Segments:** ISA, GS, ST, BGN, REF, DTP, INS, NM1, HD, DTP, AMT  
**Transaction Types:**
- BGN02 = 00: Original enrollment
- BGN02 = 04: Change/update
- BGN02 = 21: Correction/delete

**Member Actions (INS03):**
- 021: Add member
- 024: Change/update member
- 025: Terminate member
- 030: Audit/compare

**Example:** "Add member MEMBER456 to group GROUPABC effective 2025-02-01 with family coverage"

---

### 835 - Health Care Claim Payment/Advice

**Purpose:** Communicate claim payment details, adjustments, and remittance advice  
**Direction:** Payer → Provider  
**Version:** HIPAA 5010 (005010X221A1)  
**Key Segments:** ISA, GS, ST, BPR, TRN, CLP, CAS, SVC, PLB, AMT  
**Payment Types (BPR01):**
- I: Information only
- C: Payment accompanies remittance
- D: Make payment
- P: Prenotification of payment

**Claim Status (CLP02):**
- 1: Processed as primary
- 2: Processed as secondary
- 3: Processed as tertiary
- 4: Denied
- 19: Processed as primary, forwarded
- 22: Reversal of previous payment

**Example:** "Payment $1,200 for claim CLM123456 with $300 contractual adjustment (CARC 45) and $50 patient responsibility (CARC 3)"

---

### 837 - Health Care Claim

**Purpose:** Submit professional, institutional, or dental claims  
**Direction:** Provider → Payer/Clearinghouse  
**Versions:**
- 837P (005010X222A1): Professional claims
- 837I (005010X223A2): Institutional claims
- 837D (005010X224A2): Dental claims

**Key Segments:** ISA, GS, ST, BHT, HL, NM1, CLM, HI, SBR, SV1/SV2, LX  
**Claim Types (CLM05-1):**
- 11: Office visit
- 12: Home visit
- 21: Inpatient hospital
- 22: Outpatient hospital
- 31: Emergency room

**Example:** "Submit claim for patient DOE, JOHN for office visit on 2025-01-15 with diagnosis codes Z00.00 (general medical exam) and procedure code 99213 (office visit) for $150"

---

### 997 - Functional Acknowledgment (Legacy)

**Purpose:** Acknowledge receipt and syntactic correctness of X12 transmissions  
**Direction:** Receiver → Sender  
**Version:** X12 4010 and earlier  
**Key Segments:** ISA, GS, ST, AK1, AK2, AK3, AK4, AK5, AK9  
**Status:** Deprecated in HIPAA 5010, replaced by 999

---

### 999 - Implementation Acknowledgment

**Purpose:** Acknowledge receipt and report syntactic/semantic errors  
**Direction:** Receiver → Sender  
**Version:** HIPAA 5010 (005010)  
**Key Segments:** ISA, GS, ST, AK1, AK2, IK3, IK4, IK5, CTX, AK9  
**Acceptance Codes (AK5 or IK5):**
- A: Accepted
- E: Accepted but errors noted
- R: Rejected
- P: Partially accepted

**Error Codes (IK403/IK304):**
- 1: Transaction set not supported
- 2: Transaction set trailer missing
- 3: Transaction set control number mismatch
- 4: Number of included segments does not match count
- 5: One or more segments in error

---

### TA1 - Interchange Acknowledgment

**Purpose:** Acknowledge receipt of interchange (ISA/IEA envelope)  
**Direction:** Receiver → Sender  
**Version:** All X12 versions  
**Key Elements:** TA101 (interchange control number), TA102 (date), TA103 (time), TA104 (code), TA105 (note code)  
**Response Codes (TA104):**
- A: Interchange accepted
- E: Interchange accepted with errors
- R: Interchange rejected

---

## 2. X12 Segments

### Envelope Segments

#### ISA - Interchange Control Header

**Purpose:** Marks the beginning of an X12 interchange  
**Length:** Fixed 106 characters  
**Key Elements:**
- ISA01-02: Authorization information qualifier and information
- ISA03-04: Security information qualifier and information
- ISA05-06: Sender ID qualifier and sender ID
- ISA07-08: Receiver ID qualifier and receiver ID
- ISA09: Interchange date (YYMMDD)
- ISA10: Interchange time (HHMM)
- ISA11: Repetition separator (^)
- ISA12: Interchange control version (00501)
- ISA13: Interchange control number (9 digits)
- ISA14: Acknowledgment requested (0=no, 1=yes)
- ISA15: Usage indicator (T=test, P=production)
- ISA16: Component element separator (:)

**Example:**
```
ISA*00*          *00*          *ZZ*SENDER         *ZZ*RECEIVER       *250106*1200*^*00501*000000001*1*P*:~
```

---

#### GS - Functional Group Header

**Purpose:** Marks the beginning of a functional group of transactions  
**Key Elements:**
- GS01: Functional identifier code (BE=834, HC=837, HS=270/271, HP=835)
- GS02: Application sender's code
- GS03: Application receiver's code
- GS04: Date (CCYYMMDD)
- GS05: Time (HHMM or HHMMSS)
- GS06: Group control number
- GS07: Responsible agency code (X=ASC X12)
- GS08: Version/release/industry identifier (005010X220A1)

**Example:**
```
GS*BE*SENDER*RECEIVER*20250106*1200*1*X*005010X220A1~
```

---

#### GE - Functional Group Trailer

**Purpose:** Marks the end of a functional group  
**Key Elements:**
- GE01: Number of transaction sets included
- GE02: Group control number (must match GS06)

**Example:**
```
GE*5*1~
```

---

#### IEA - Interchange Control Trailer

**Purpose:** Marks the end of an X12 interchange  
**Key Elements:**
- IEA01: Number of included functional groups
- IEA02: Interchange control number (must match ISA13)

**Example:**
```
IEA*1*000000001~
```

---

#### ST - Transaction Set Header

**Purpose:** Marks the beginning of a transaction set  
**Key Elements:**
- ST01: Transaction set identifier code (270, 271, 834, 835, 837)
- ST02: Transaction set control number (up to 9 characters)
- ST03: Implementation convention reference (e.g., 005010X220A1)

**Example:**
```
ST*834*0001*005010X220A1~
```

---

#### SE - Transaction Set Trailer

**Purpose:** Marks the end of a transaction set  
**Key Elements:**
- SE01: Number of included segments (including ST and SE)
- SE02: Transaction set control number (must match ST02)

**Example:**
```
SE*45*0001~
```

---

### Transaction-Specific Segments

#### BGN - Beginning Segment (834, 999)

**Purpose:** Identifies transaction purpose and control information  
**Key Elements (834):**
- BGN01: Transaction set purpose code (00=original, 04=change, 21=correction)
- BGN02: Reference identification (transaction control number)
- BGN03: Date (CCYYMMDD)
- BGN04: Time (HHMM or HHMMSS)
- BGN08: Action code (2=add, 4=change)

**Example:**
```
BGN*00*12345*20250106*1200****2~
```

---

#### BHT - Beginning of Hierarchical Transaction (270, 271, 276, 277, 837)

**Purpose:** Identifies transaction hierarchy purpose  
**Key Elements:**
- BHT01: Hierarchical structure code
- BHT02: Transaction set purpose code (00, 11, 13)
- BHT03: Reference identification (trace number)
- BHT04: Date (CCYYMMDD)
- BHT05: Time (HHMM or HHMMSS)
- BHT06: Transaction type code (RQ=request, RP=response)

**Example:**
```
BHT*0022*13*TRN12345*20250106*1200*RP~
```

---

#### BPR - Financial Information (835)

**Purpose:** Payment order and banking information  
**Key Elements:**
- BPR01: Transaction handling code (I, C, D, P)
- BPR02: Monetary amount (total payment)
- BPR03: Credit/debit flag (C, D)
- BPR04: Payment method code (ACH, CHK, BOP, FWT)
- BPR05: Payment format code (CTX, CCD)
- BPR16: Check/EFT effective date (CCYYMMDD)

**Example:**
```
BPR*I*15000.00*C*ACH*CTX***01*123456789**01*987654321**01234**20250115~
```

---

## 3. X12 Elements and Qualifiers

### Claim Filing Indicator Codes

| Code | Description |
|------|-------------|
| AM | Automobile Medical |
| BL | Blue Cross Blue Shield |
| CH | CHAMPUS (TRICARE) |
| CI | Commercial Insurance Co. |
| DS | Disability |
| FI | Federal Employees Program |
| HM | Health Maintenance Organization |
| LM | Liability Medical |
| MA | Medicare Part A |
| MB | Medicare Part B |
| MC | Medicaid |
| OF | Other Federal Program |
| TV | Title V |
| VA | Veterans Affairs Plan |
| WC | Workers' Compensation |
| ZZ | Mutually Defined |

---

### Date/Time Qualifiers

| Qualifier | Description |
|-----------|-------------|
| 003 | Invoice Date / Service Date |
| 007 | Effective Date |
| 036 | Expiration Date |
| 050 | Received Date |
| 090 | Report Start Date |
| 091 | Report End Date |
| 096 | Discharge Date |
| 291 | Date of Birth |
| 307 | Eligibility Begin Date |
| 318 | Admission Date / Statement Start Date |
| 348 | Hire Date |
| 349 | Termination Date |
| 356 | Employment Begin Date |
| 357 | Employment End Date |
| 432 | Accident Date |
| 435 | Admission Date |
| 472 | Service Date |

---

### Entity Identifier Codes (NM1)

| Code | Description |
|------|-------------|
| 1P | Provider |
| 2B | Third-Party Administrator |
| 40 | Receiver |
| 41 | Submitter |
| 70 | Attending Physician |
| 71 | Operating Physician |
| 72 | Other Physician |
| 77 | Service Provider |
| 82 | Rendering Provider |
| 85 | Billing Provider |
| 87 | Pay-to Provider |
| FA | Facility |
| IL | Insured or Subscriber |
| P3 | Primary Care Provider |
| PR | Payer |
| QC | Patient |
| SJ | Service Location |
| TT | Third Party Reviewing |

---

### Place of Service Codes

| Code | Description |
|------|-------------|
| 01 | Pharmacy |
| 02 | Telehealth |
| 11 | Office |
| 12 | Home |
| 21 | Inpatient Hospital |
| 22 | Outpatient Hospital |
| 23 | Emergency Room - Hospital |
| 24 | Ambulatory Surgical Center |
| 31 | Skilled Nursing Facility |
| 32 | Nursing Facility |
| 41 | Ambulance - Land |
| 42 | Ambulance - Air or Water |
| 49 | Independent Clinic |
| 50 | Federally Qualified Health Center |
| 60 | Mass Immunization Center |
| 71 | State or Local Public Health Clinic |
| 72 | Rural Health Clinic |
| 81 | Independent Laboratory |
| 99 | Other Place of Service |

---

### Service Type Codes (EB/EQ)

| Code | Description |
|------|-------------|
| 1 | Medical Care |
| 2 | Surgical |
| 3 | Consultation |
| 4 | Diagnostic X-Ray |
| 5 | Diagnostic Lab |
| 6 | Radiation Therapy |
| 7 | Anesthesia |
| 8 | Surgical Assistance |
| 12 | Durable Medical Equipment Purchase |
| 18 | Durable Medical Equipment Rental |
| 30 | Health Benefit Plan Coverage |
| 33 | Chiropractic |
| 35 | Dental Care |
| 38 | Orthodontics |
| 42 | Home Health Care |
| 47 | Hospital |
| 48 | Hospital - Inpatient |
| 49 | Hospital - Outpatient |
| 50 | Hospital - Emergency Accident |
| 54 | Major Medical |
| 86 | Emergency Services |
| 88 | Pharmacy |
| 92 | Podiatry |
| 98 | Professional (Physician) Visit - Inpatient |
| A4 | Psychiatric - Room and Board |
| A6 | Psychiatric - Inpatient |
| A8 | Rehabilitation |
| AL | Frames |
| AN | Lenses |

---

## 4. Healthcare Terminology

### Adjudication
The process by which a payer reviews and makes a determination on a submitted healthcare claim, resulting in payment, denial, or adjustment.

### Adjustment
A reduction or increase in claim payment amount due to contractual agreements, policy limitations, or patient responsibility.

### CARC (Claim Adjustment Reason Code)
Standardized codes explaining why a claim or service line payment was adjusted. Defined by the Washington Publishing Company (WPC). Common codes:
- 1: Deductible amount
- 2: Coinsurance amount
- 3: Co-payment amount
- 45: Charge exceeds fee schedule
- 50: Not medically necessary
- 96: Non-covered charges

### Clearinghouse
An entity that processes healthcare transactions between providers and payers, performing validation, translation, and routing services.

### Coinsurance
The percentage of covered expenses the patient must pay after the deductible is met (e.g., 20% coinsurance means patient pays 20%, insurer pays 80%).

### Co-payment (Co-pay)
A fixed amount the patient pays for a covered service, typically due at the time of service (e.g., $35 office visit co-pay).

### Deductible
The amount a patient must pay out-of-pocket before insurance coverage begins paying for covered services.

### Denial
A claim determination where the payer refuses to pay any amount, typically due to non-covered services, missing information, or policy violations.

### Dependent
A person covered under another person's insurance policy (spouse, child, or other qualifying family member).

### EFT (Electronic Funds Transfer)
Electronic payment method for transferring funds from payer to provider, typically via ACH.

### ERA (Electronic Remittance Advice)
Electronic version of remittance advice (835 transaction), detailing claim payment information.

### HIPAA (Health Insurance Portability and Accountability Act)
Federal legislation establishing standards for healthcare data privacy, security, and electronic transactions.

### ICD-10-CM (International Classification of Diseases, 10th Revision, Clinical Modification)
Standardized diagnosis coding system used in the United States for reporting diseases and conditions.

### NPI (National Provider Identifier)
A unique 10-digit identification number for healthcare providers required by HIPAA.

### Out-of-Pocket Maximum
The maximum amount a patient pays for covered services in a policy period, after which the insurer pays 100%.

### Payer
An insurance company, health plan, or government program that pays for healthcare services (e.g., Medicare, Blue Cross Blue Shield).

### PHI (Protected Health Information)
Any individually identifiable health information protected under HIPAA privacy regulations.

### Primary Insurance
The insurance policy responsible for first payment on a claim when a patient has multiple coverages.

### Prior Authorization
Pre-approval from a payer required before certain services can be rendered to ensure coverage.

### Provider
A healthcare professional, facility, or organization that delivers medical services (physicians, hospitals, clinics).

### RARC (Remittance Advice Remark Code)
Supplemental codes providing additional explanation for claim adjustments or denials.

### Secondary Insurance
The insurance policy responsible for payment after the primary insurance has processed the claim.

### Subscriber
The primary insured individual who holds the insurance policy; dependents are covered under the subscriber's policy.

### TIN (Tax Identification Number)
A unique identifier assigned by the IRS for tax purposes, used to identify healthcare organizations.

---

## 5. Azure Services

### Azure Blob Storage
Object storage service for storing unstructured data including X12 files, JSON configs, and transaction logs. Organized into containers with hierarchical folder structures.

### Azure Data Factory (ADF)
Cloud-based data integration service for orchestrating data movement and transformation pipelines. Used for validating X12 files, extracting metadata, and triggering downstream processing.

### Azure Event Grid
Event routing service that triggers workflows based on events like blob creation, enabling reactive processing patterns.

### Azure Functions
Serverless compute service for running event-driven code without managing infrastructure. Used for mappers, connectors, routing logic, and processing workflows.

### Azure Key Vault
Secure storage for secrets, encryption keys, and certificates. Used to store SFTP passwords, connection strings, API keys, and partner credentials.

### Azure Monitor
Comprehensive monitoring solution providing metrics, logs, and alerts for application and infrastructure monitoring.

### Azure Service Bus
Enterprise message broker supporting publish/subscribe patterns, message routing with topic subscriptions, reliable delivery, and dead-letter queues.

### Azure SQL Database
Fully managed relational database service. Hosts control numbers database, event store, and SFTP tracking database.

### Application Insights
Application performance management (APM) service providing telemetry, distributed tracing, custom metrics, and dependency tracking.

### Log Analytics Workspace
Centralized repository for log data from Azure Monitor, enabling KQL queries and analysis across resources.

---

## 6. Platform Components

### Control Number Service
Generates sequential, unique control numbers for ISA (interchange), GS (functional group), and ST (transaction set) segments per trading partner ensuring X12 compliance.

### Event Store
Append-only database storing domain events for audit trails, event sourcing, and system state reconstruction. Implements 834 enrollment event sourcing pattern.

### Mapper Functions
Azure Functions that transform X12 transactions to internal formats and vice versa:
- 834 Mapper: Enrollment transactions
- 837 Mapper: Claims transactions
- 270/271 Mapper: Eligibility transactions
- 835 Mapper: Remittance advice transactions

### Orchestrator Function
Coordinates transaction processing workflow: validation, routing, transformation, acknowledgment generation, and error handling.

### Outbound Connectors
Integration functions that deliver transformed data to external systems:
- Claims system connector
- Enrollment system connector
- Eligibility API connector
- Partner-specific connectors

### Routing Service
Service Bus-based message routing using topic subscriptions with filters to direct transactions to appropriate handlers based on transaction type and partner ID.

### SFTP Connector
Azure Function that downloads files from partner SFTP servers and uploads acknowledgments/responses on configurable schedules.

### SFTP Tracking Database
EF Core database tracking downloaded and uploaded files with deduplication via SHA-256 file hash comparison.

---

## 7. Process and Workflow Terms

### Acknowledgment
Response transaction (TA1, 999, 997) confirming receipt and indicating acceptance or rejection of an X12 transmission.

### Deduplication
Process of detecting and preventing duplicate file processing using SHA-256 file hash comparison against tracking database.

### Event Sourcing
Architectural pattern where state changes are stored as a sequence of immutable events rather than current state snapshots. Used for 834 enrollment processing.

### Idempotency
Property ensuring repeated processing of the same input produces the same result without unintended side effects. Critical for reliable message processing.

### Routing Message
JSON message sent via Service Bus containing transaction metadata:
- Transaction type (834, 837, 270, 271, 835)
- Partner ID
- File path in blob storage
- Correlation ID
- Timestamp

**Example:**
```json
{
  "transactionType": "834",
  "partnerId": "PARTNER001",
  "filePath": "staging/PARTNER001/834/file_20250106_120000.edi",
  "correlationId": "12345678-1234-1234-1234-123456789abc",
  "timestamp": "2025-01-06T12:00:00Z"
}
```

### Trading Partner
External organization (payer, provider, clearinghouse, TPA) that exchanges EDI transactions with the platform.

### Transaction Control Number
Unique identifier for X12 transaction sets (ST02), functional groups (GS06), and interchanges (ISA13).

### Validation
Process of verifying X12 syntax correctness, required segment presence, data type conformance, and business rule compliance.

---

## 8. Acronyms

| Acronym | Full Term | Description |
|---------|-----------|-------------|
| ACH | Automated Clearing House | Electronic funds transfer network |
| ADF | Azure Data Factory | Cloud data integration service |
| ADR | Architecture Decision Record | Documented design decision |
| API | Application Programming Interface | Software communication interface |
| ASC X12 | Accredited Standards Committee X12 | Organization managing X12 standards |
| CARC | Claim Adjustment Reason Code | Standardized claim adjustment codes |
| CI/CD | Continuous Integration/Continuous Deployment | Automated software delivery pipeline |
| CMS | Centers for Medicare & Medicaid Services | Federal agency administering Medicare/Medicaid |
| COBRA | Consolidated Omnibus Budget Reconciliation Act | Law providing continued health coverage after job loss |
| DRG | Diagnosis Related Group | Hospital payment classification system |
| E2E | End-to-End | Complete workflow from start to finish |
| EDI | Electronic Data Interchange | Computer-to-computer exchange of business documents |
| EFT | Electronic Funds Transfer | Electronic money transfer |
| EIN | Employer Identification Number | Federal tax ID for businesses |
| EOB | Explanation of Benefits | Statement explaining claim payment/denial |
| ERA | Electronic Remittance Advice | Electronic payment notification (835) |
| HIPAA | Health Insurance Portability and Accountability Act | Healthcare privacy and transaction standards law |
| HL7 | Health Level Seven | Healthcare information exchange standards organization |
| IaC | Infrastructure as Code | Managing infrastructure via code (Bicep, Terraform) |
| ICD-10-CM | International Classification of Diseases, 10th Revision, Clinical Modification | Diagnosis coding system |
| ISA | Interchange Control Header | X12 envelope segment |
| KQL | Kusto Query Language | Azure log query language |
| MTTD | Mean Time To Detection | Average time to detect issues |
| MTTR | Mean Time To Recovery | Average time to resolve issues |
| NCPDP | National Council for Prescription Drug Programs | Pharmacy transaction standards organization |
| NPI | National Provider Identifier | Unique 10-digit provider ID |
| PHI | Protected Health Information | Patient health data protected by HIPAA |
| RARC | Remittance Advice Remark Code | Supplemental explanation codes |
| RBAC | Role-Based Access Control | Azure permission model |
| SLA | Service Level Agreement | Performance target commitments |
| SSN | Social Security Number | National identification number |
| TIN | Tax Identification Number | IRS-assigned tax ID |
| TPA | Third-Party Administrator | Organization managing health plan administration |
| UPIN | Unique Physician Identification Number | Legacy provider identifier (replaced by NPI) |
| WPC | Washington Publishing Company | Publisher of healthcare code sets (CARC/RARC) |
| X12 | ASC X12 | EDI standard for business transactions |

---

## Quick Reference Tables

### Transaction Type Summary

| Transaction | Name | Direction | Primary Use |
|-------------|------|-----------|-------------|
| 270 | Eligibility Inquiry | Provider → Payer | Check coverage |
| 271 | Eligibility Response | Payer → Provider | Return coverage details |
| 276 | Claim Status Inquiry | Provider → Payer | Check claim status |
| 277 | Claim Status Response | Payer → Provider | Return claim status |
| 278 | Prior Authorization | Bidirectional | Request/grant authorization |
| 834 | Enrollment | Sponsor → Payer | Add/change/terminate members |
| 835 | Remittance Advice | Payer → Provider | Communicate payments |
| 837 | Claim Submission | Provider → Payer | Submit claims for payment |
| 997 | Functional Acknowledgment | Receiver → Sender | Legacy acknowledgment (pre-5010) |
| 999 | Implementation Acknowledgment | Receiver → Sender | HIPAA 5010 acknowledgment |
| TA1 | Interchange Acknowledgment | Receiver → Sender | ISA-level acknowledgment |

### Common X12 Delimiters

| Delimiter | Character | Position in ISA | Purpose |
|-----------|-----------|-----------------|---------|
| Element Separator | * | ISA03 (position 4) | Separates elements within segment |
| Segment Terminator | ~ | End of each segment | Marks end of segment |
| Subelement Separator | : | ISA16 (position 105) | Separates subelements within composite |
| Repetition Separator | ^ | ISA11 (position 83) | Separates repeated elements |

### X12 Version Identifiers

| Code | Description | Usage |
|------|-------------|-------|
| 00401 | Version 4010 | Legacy, pre-HIPAA 5010 |
| 00501 | Version 5010 | Current HIPAA standard (ISA12) |
| 005010X220A1 | 834 Implementation Guide | GS08 for enrollment transactions |
| 005010X221A1 | 835 Implementation Guide | GS08 for remittance transactions |
| 005010X222A1 | 837P Implementation Guide | GS08 for professional claims |
| 005010X223A2 | 837I Implementation Guide | GS08 for institutional claims |
| 005010X224A2 | 837D Implementation Guide | GS08 for dental claims |
| 005010X279A1 | 270/271 Implementation Guide | GS08 for eligibility transactions |
| 005010X212 | 276/277 Implementation Guide | GS08 for claim status transactions |

---

**Document Version:** 1.0  
**Last Updated:** October 6, 2025  
**Next Review:** January 2026
