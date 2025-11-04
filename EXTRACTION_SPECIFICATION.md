# EXTRACTION SPECIFICATION
## Contract Documents + Invoices

---

## PART 1: CONTRACT DOCUMENT EXTRACTION

### Overview

Contract documents fall into 3 categories:
1. **Framework Agreements (MSA)** - Master Service Agreements that define terms, parties, and authorization for derivative purchases
2. **Purchase Orders (PO)** - Binding purchase obligations issued under a Framework
3. **Statements of Work (SOW)** - Binding service obligations issued under a Framework (often issued as "Order Forms" in Bayer case)

### 1.1 FRAMEWORK AGREEMENT (MSA) Fields

**What:** Master Service Agreement or MSA document defining the business relationship

**Document Examples:** 
- `Bayer_CLMS_-_Action_required_Contract_JP0094.pdf` (Framework #1)
- `r4 MSA for BCH CAP 2021 12 10.docx` (Framework #2)

**Fields to Extract:**

| Field | Type | Example | Why Needed | Source |
|-------|------|---------|-----------|--------|
| `document_type` | Enum | "FRAMEWORK" | Classification for grouping and hierarchy | Document content keywords |
| `framework_id` | String | "BAYER_CLMS_001" | Unique identifier for this framework | Generated from parties + hash |
| `buyer_legal_name` | String | "Bayer Yakuhin, Ltd." | Full legal name of buyer/customer | PDF Annex D or DOCX header |
| `buyer_address` | String | "Breeze Tower 2-4-9, Umeda, 530-0001 Osaka" | Full address for contract verification | PDF Annex D or DOCX body |
| `vendor_legal_name` | String | "r4 Technologies, Inc." | Full legal name of vendor/contractor | PDF Annex D or DOCX header |
| `vendor_address` | String | "38 Grove Street, Building C, 06877, Ridgefield" | Full address for contract verification | PDF Annex D or DOCX body |
| `program_code` | String | "BCH", "CAP" | Program identifier for grouping SOWs/POs | Document content or filename |
| `contract_date` | Date (YYYY-MM-DD) | "2021-12-10" | When framework was executed | Document content or filename |
| `contract_end_date` | Date (YYYY-MM-DD) | "2024-12-10" | When framework expires | Annex P or termination clause |
| `framework_amount` | Decimal | 5000000 | Total authorized contract value | Annex P (Prices) section |
| `currency` | Enum | "USD", "EUR" | Currency of framework amount | Document content |
| `payment_terms` | String | "Net 45", "Net 60" | Default payment terms | Annex P or similar section |
| `binding_mechanism` | List[String] | ["PO", "SOW"] | How binding obligations are created | Article 2.1 or equivalent |
| `status` | Enum | "ACTIVE", "EXPIRED", "TERMINATED" | Current status of framework | Derived from dates |

**Why:**
- Buyer/vendor legal names are CRITICAL for grouping related POs/SOWs
- Framework amount caps total spending
- Program code groups SOWs/POs under same framework
- Binding mechanism determines which downstream docs are binding

---

### 1.2 PURCHASE ORDER (PO) Fields

**What:** Binding purchase order for specific goods/services

**Document Examples:**
- `Purchase Order No. 2151002393.pdf`

**Fields to Extract:**

| Field | Type | Example | Why Needed | Source |
|-------|------|---------|-----------|--------|
| `document_type` | Enum | "PURCHASE_ORDER" | Classification | "Purchase Order" in content |
| `po_number` | String | "2151002393" | HIGHEST PRIORITY for invoice linkage | "PO#" or "PO Number:" in content |
| `po_date` | Date (YYYY-MM-DD) | "2022-01-14" | When PO was issued | "dated" keyword in content |
| `delivery_date` | Date (YYYY-MM-DD) | "2022-05-31" | Expected delivery/completion | "delivery date" keyword |
| `buyer_name` | String | "Bayer Yakuhin, Ltd." | Same as framework buyer | Content header |
| `vendor_name` | String | "r4 Technologies, Inc." | Same as framework vendor | Content header |
| `po_amount` | Decimal | 60000.00 | Amount of this specific PO | "$60,000.00" in content |
| `currency` | Enum | "USD" | Currency of PO | Content or filename |
| `payment_terms` | String | "Net 45 after invoice" | Payment terms for this PO | Content (often repeats framework terms) |
| `framework_reference` | String | "BAYER_CLMS_001" | Link to parent framework | Cross-reference or parties match |
| `po_status` | Enum | "OPEN", "FULFILLED", "CANCELLED" | Current status | Derived or noted in document |

**Why:**
- `po_number` is the PRIMARY key for matching invoices to POs
- Amount validates invoice totals
- Delivery date helps validate invoice timing
- Framework reference ensures PO is under correct master agreement

---

### 1.3 STATEMENT OF WORK / ORDER FORM (SOW) Fields

**What:** Statement of Work or Order Form defining specific services under a Framework

**Document Examples:**
- `r4 Order Form for BCH CAP 2021 12 10.docx`
- `r4 SOW for BCH CAP 2021 12 10.docx`
- `r4 Order Form for BCH CAP 2022 11 01.docx`

**Fields to Extract:**

| Field | Type | Example | Why Needed | Source |
|-------|------|---------|-----------|--------|
| `document_type` | Enum | "SOW" or "ORDER_FORM" | Classification | "Statement of Work" or "Order Form" in content |
| `sow_number` | String | "11414-200" | Unique identifier for this SOW | "SOW Number" or similar in content |
| `sow_date` | Date (YYYY-MM-DD) | "2021-12-10" | When SOW was issued | Document date |
| `start_date` | Date (YYYY-MM-DD) | "2021-12-10" | When services start | "start date" or content |
| `end_date` | Date (YYYY-MM-DD) | "2022-04-10" | When services end | "end date" or duration + start |
| `duration_months` | Integer | 4 | How long the engagement | "4 months" in content |
| `buyer_name` | String | "Bayer Consumer Health China & Asia-Pacific" | Same as framework buyer | Content header |
| `vendor_name` | String | "r4 Technologies Inc." | Same as framework vendor | Content header |
| `sow_amount` | Decimal | NULL | Amount for this SOW (may be blank in template) | "$XXX" if present |
| `currency` | Enum | "USD" | Currency | Content |
| `program_code` | String | "BCH", "CAP" | Program identifier | Filename or content |
| `services_description` | String | "Implementation and Managed Services" | What services are included | SOW body/scope section |
| `framework_reference` | String | "BCH_CAP_MSA_001" | Link to parent framework | Cross-reference or parties match |
| `sow_status` | Enum | "ACTIVE", "COMPLETED", "CANCELLED" | Current status | Derived or noted |
| `renewal_version` | String | "2021", "2022" | Track if this is renewal/update | Compare dates for same program |

**Why:**
- SOW number needed for invoice SOW-based linkage
- Duration + dates validate invoice timing
- Program code groups multiple SOWs under same framework
- Services description helps validate invoice line items

---

### 1.4 Common Fields Across All Document Types

| Field | Type | Example | Extraction Method |
|-------|------|---------|-------------------|
| `file_path` | String | "./demo_contracts/Bayer_CLMS_...pdf" | Direct from filesystem |
| `file_format` | Enum | "PDF", "DOCX" | From file extension |
| `extraction_timestamp` | DateTime | "2025-11-02T10:30:00" | System timestamp |
| `extraction_confidence` | Float | 0.95 | 0.0-1.0 based on field extraction success |

---

## PART 2: INVOICE EXTRACTION

### 2.1 Invoice Fields

**What:** Invoice documents (PDF, DOCX) requesting payment for goods/services

**Fields to Extract:**

| Field | Type | Example | Why Needed | Source |
|-------|------|---------|-----------|--------|
| `invoice_id` | String | "INV-2022-001234" | Unique invoice identifier | Invoice content (NOT filename) |
| `invoice_date` | Date (YYYY-MM-DD) | "2022-02-15" | When invoice was issued | "Invoice Date:" or similar |
| `vendor_name` | String | "r4 Technologies, Inc." | Must match contract vendor | "FROM:" or "VENDOR:" section |
| `buyer_name` | String | "Bayer Yakuhin, Ltd." | Must match contract buyer | "TO:" or "BILL TO:" section |
| `po_number` | String | "2151002393" | PRIMARY linkage to contract | "PO#" or "Purchase Order:" |
| `program_code` | String | "BCH", "CAP" | SECONDARY linkage to framework | Invoice description |
| `sow_number` | String | "11414-200" | Link to SOW if applicable | "SOW#" or "Work Order:" |
| `invoice_amount` | Decimal | 15000.00 | Total amount due | "TOTAL:", "Amount:", "$XXX" |
| `currency` | Enum | "USD" | Currency of invoice | "$", "EUR", etc. |
| `invoice_status` | Enum | "DRAFT", "ISSUED", "PAID", "OVERDUE" | Payment status | Document state |
| `payment_due_date` | Date (YYYY-MM-DD) | "2022-03-02" | When payment is due | "DUE DATE:" (invoice_date + payment_terms) |
| `line_items` | List[Dict] | See 2.2 below | Breakdown of charges | Table in invoice body |
| `services_description` | String | "Q1 2022 Managed Services" | What was invoiced for | Invoice item descriptions |
| `notes` | String | "Includes travel costs" | Additional context | Notes section |

**Why:**
- Invoice must link to exactly ONE contract PO/SOW
- PO number is highest priority match
- Program code is secondary match if PO not found
- Amount must validate against contract/PO amounts
- Invoice date + payment terms → due date for aging analysis

---

### 2.2 Invoice Line Items

**What:** Individual line items/charges within an invoice

**Fields per Line Item:**

| Field | Type | Example | Purpose |
|-------|------|---------|---------|
| `line_number` | Integer | 1 | Sequence in invoice |
| `description` | String | "Implementation Services - Jan 2022" | What was charged |
| `quantity` | Decimal | 1 | Quantity of units |
| `unit_price` | Decimal | 10000.00 | Price per unit |
| `line_total` | Decimal | 10000.00 | quantity × unit_price |
| `service_code` | String | "IMPL", "SUPPORT" | Internal service category |

**Why:**
- Detailed line items help validate invoice accuracy
- Service codes can be matched to SOW scope
- Line item totals sum to invoice total (validation)

---

## PART 3: EXTRACTION DEPENDENCIES & LINKAGE

### 3.1 Contract → Invoice Linkage Map

```
PRIORITY 1 (HIGHEST - Use if present):
  Invoice.po_number = PO.po_number
  ✓ Example: Invoice references "PO 2151002393" → Links to Purchase Order 2151002393

PRIORITY 2 (HIGH - Use if PO# not found):
  Invoice.vendor_name ≈ Contract.vendor_name  AND
  Invoice.buyer_name ≈ Contract.buyer_name  AND
  (Invoice.sow_number = SOW.sow_number OR Invoice.program_code = Contract.program_code)
  ✓ Example: Invoice from "r4 Tech" to "Bayer BCH" with program "BCH CAP" 
            → Links to BCH CAP SOWs

PRIORITY 3 (MEDIUM - Use if above fail):
  Invoice.program_code = Contract.program_code
  ✓ Example: Invoice mentions "BCH CAP" → Links to any BCH CAP contract

PRIORITY 4 (LOW - Fallback):
  Similar vendor name + similar amount range
  ⚠️  Use only if above fail (high false positive risk)
```

### 3.2 Validation Rules

**Rule 1: Amount Validation**
```
IF Invoice.po_number is found:
  THEN Invoice.amount MUST be ≤ PO.po_amount
  ERROR: "Invoice amount ($X) exceeds PO amount ($Y)"
```

**Rule 2: Date Validation**
```
IF Invoice linked to PO:
  THEN Invoice.invoice_date MUST be ≥ PO.po_date
  ERROR: "Invoice dated before PO issued"
  
  AND Invoice.invoice_date MUST be ≤ PO.delivery_date (or soon after)
  ERROR: "Invoice dated after PO delivery date"
```

**Rule 3: Date Validation (SOW)**
```
IF Invoice linked to SOW:
  THEN Invoice.invoice_date MUST be BETWEEN SOW.start_date AND SOW.end_date
  ERROR: "Invoice outside SOW period"
```

**Rule 4: Vendor Validation**
```
IF Invoice.vendor_name is found:
  THEN Invoice.vendor_name MUST match (exactly or fuzzy) Contract.vendor_name
  ERROR: "Invoice vendor doesn't match contract vendor"
```

**Rule 5: Party Validation**
```
IF Invoice.buyer_name is found:
  THEN Invoice.buyer_name MUST match (exactly or fuzzy) Contract.buyer_name
  ERROR: "Invoice buyer doesn't match contract buyer"
```

---

## PART 4: EXTRACTION ARCHITECTURE

### 4.1 Contract Extraction Flow

```
INPUT: Contract Document File (PDF/DOCX)
  ↓
1. DETECT DOCUMENT TYPE
   - Look for: "Framework", "Master Service", "Purchase Order", "Statement of Work"
   - Fallback to filename analysis
   ↓
2. EXTRACT COMMON FIELDS
   - File metadata (path, format, size, timestamp)
   - All document types
   ↓
3. EXTRACT TYPE-SPECIFIC FIELDS
   ├─ IF Framework: Extract framework_id, payment_terms, amounts, binding_mechanism
   ├─ IF PO: Extract po_number, po_date, delivery_date, po_amount
   └─ IF SOW: Extract sow_number, duration, services_description
   ↓
4. LINK TO FRAMEWORK
   - Match buyer + vendor + program_code
   - Find parent framework
   ↓
OUTPUT: Structured Contract Dictionary with all fields
```

### 4.2 Invoice Extraction Flow

```
INPUT: Invoice Document File (PDF/DOCX)
  ↓
1. EXTRACT INVOICE METADATA
   - invoice_id, invoice_date, payment_due_date
   ↓
2. EXTRACT PARTIES
   - vendor_name, buyer_name
   ↓
3. EXTRACT LINKAGE KEYS (in priority order)
   - po_number (if present)
   - sow_number (if present)
   - program_code (if present)
   ↓
4. EXTRACT AMOUNTS
   - invoice_amount, currency, line_items
   ↓
5. EXTRACT SERVICES
   - services_description, line items with detail
   ↓
6. LINK TO CONTRACT
   - Use priority 1-4 linkage rules
   - Return: linked_contract_id, match_confidence, match_method
   ↓
7. VALIDATE
   - Amount validation
   - Date validation
   - Party validation
   ↓
OUTPUT: Invoice Dictionary with Contract Link + Validation Status
```

---

## PART 5: DATA STRUCTURES

### 5.1 Contract Document (Python Dict)

```python
{
    # Metadata
    "document_id": "DOC_1",
    "file_path": "./demo_contracts/Bayer_CLMS_...pdf",
    "file_format": "PDF",
    "extraction_timestamp": "2025-11-02T10:30:00",
    "extraction_confidence": 0.92,
    
    # Document Type
    "document_type": "FRAMEWORK",  # or "PO", "SOW"
    
    # Parties
    "buyer_legal_name": "Bayer Yakuhin, Ltd.",
    "buyer_address": "Breeze Tower...",
    "vendor_legal_name": "r4 Technologies, Inc.",
    "vendor_address": "38 Grove Street...",
    
    # Framework Fields (if applicable)
    "framework_id": "BAYER_CLMS_001",
    "contract_date": "2022-01-01",
    "contract_end_date": "2025-12-31",
    "framework_amount": 5000000,
    "currency": "USD",
    
    # PO Fields (if applicable)
    "po_number": "2151002393",
    "po_date": "2022-01-14",
    "delivery_date": "2022-05-31",
    "po_amount": 60000,
    
    # SOW Fields (if applicable)
    "sow_number": "11414-200",
    "sow_date": "2021-12-10",
    "duration_months": 4,
    "services_description": "Implementation and Managed Services",
    
    # Common Fields
    "program_code": "BCH",
    "payment_terms": "Net 45",
    "binding_mechanism": ["PO", "SOW"],
    
    # Relationships
    "framework_reference": "BAYER_CLMS_001",  # If PO/SOW, link to framework
    
    # Status
    "status": "ACTIVE"
}
```

### 5.2 Invoice (Python Dict)

```python
{
    # Metadata
    "invoice_id": "INV-2022-001234",
    "file_path": "./demo_invoices/INV-2022-001234.pdf",
    "invoice_date": "2022-02-15",
    "extraction_timestamp": "2025-11-02T10:30:00",
    
    # Parties
    "vendor_name": "r4 Technologies, Inc.",
    "buyer_name": "Bayer Yakuhin, Ltd.",
    
    # Linkage Keys
    "po_number": "2151002393",  # Primary linkage
    "program_code": "BCH",  # Secondary linkage
    "sow_number": "11414-200",  # Tertiary linkage
    
    # Amounts
    "invoice_amount": 15000.00,
    "currency": "USD",
    "line_items": [
        {
            "line_number": 1,
            "description": "Implementation Services - Jan 2022",
            "quantity": 1,
            "unit_price": 10000.00,
            "line_total": 10000.00
        },
        {
            "line_number": 2,
            "description": "Travel Expenses",
            "quantity": 1,
            "unit_price": 5000.00,
            "line_total": 5000.00
        }
    ],
    
    # Services
    "services_description": "Q1 2022 Managed Services including travel",
    "payment_terms": "Net 45",
    "payment_due_date": "2022-03-02",
    
    # Contract Linkage (Result of matching)
    "linked_contract": {
        "contract_id": "DOC_2",
        "document_type": "PURCHASE_ORDER",
        "po_number": "2151002393",
        "match_method": "PO_NUMBER",
        "match_confidence": 0.99
    },
    
    # Validation Results
    "validation": {
        "amount_valid": True,
        "date_valid": True,
        "party_valid": True,
        "overall_status": "VALID"
    }
}
```

---

## PART 6: SUCCESS METRICS

### For Contracts

| Field | Target Success Rate | Criticality |
|-------|-------------------|-------------|
| document_type | 95% | CRITICAL |
| po_number (if PO) | 98% | CRITICAL |
| buyer_legal_name | 90% | CRITICAL |
| vendor_legal_name | 90% | CRITICAL |
| contract_date | 85% | HIGH |
| po_amount (if PO) | 90% | CRITICAL |
| currency | 95% | HIGH |
| framework_amount | 85% | HIGH |
| program_code | 85% | MEDIUM |
| payment_terms | 80% | MEDIUM |

**Target Overall: 90%+ extraction success per critical field**

### For Invoices

| Field | Target Success Rate | Criticality |
|-------|-------------------|-------------|
| invoice_id | 98% | CRITICAL |
| invoice_date | 95% | CRITICAL |
| vendor_name | 90% | CRITICAL |
| po_number | 85% | CRITICAL |
| invoice_amount | 95% | CRITICAL |
| currency | 95% | HIGH |
| payment_due_date (derived) | 90% | HIGH |
| line_items | 80% | MEDIUM |
| services_description | 75% | MEDIUM |

**Target Overall: 90%+ extraction success per critical field**

### Linkage Success

| Linkage Type | Target Success Rate |
|--------------|-------------------|
| PO# matching | 95% |
| Program code matching | 85% |
| SOW# matching | 85% |
| False positive rate | < 5% |

---

## SUMMARY

**Contract Extraction:** 30 fields across 3 document types
- 6 critical fields (document_type, parties, dates, amounts, PO#)
- Must extract from both PDF and DOCX
- Must link POs/SOWs to Framework parent

**Invoice Extraction:** 13 fields
- 5 critical fields (invoice_id, parties, po_number, amount, date)
- Must link to contract via 4-priority matching
- Must validate against contract terms

**Linkage:** 4-priority system
1. PO# exact match (highest confidence)
2. Party + program matching (high confidence)
3. Program code alone (medium confidence)
4. Fuzzy party + amount (low confidence, fallback)

