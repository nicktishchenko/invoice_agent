# EXTRACTION TASK TODO LIST

## Overview
This TODO list focuses exclusively on **extracting information from contract documents and invoices**.

The task is broken into two parts:
1. **Contract Extraction** - Extract metadata from 6 contract documents
2. **Invoice Extraction** - Extract metadata from invoices and link to contracts

---

## PART 1: EXTRACTION PARAMETERS DEFINITION

### ✅ COMPLETED: Contract Extraction Parameters

**Status:** COMPLETE - All 30 fields across 3 document types documented

**Deliverable:** `EXTRACTION_SPECIFICATION.md` - Section 1

**Fields by Document Type:**

| Document Type | Field Count | Critical Fields |
|---------------|------------|-----------------|
| Framework Agreement (MSA) | 14 | buyer_legal_name, vendor_legal_name, framework_amount, payment_terms |
| Purchase Order (PO) | 11 | po_number, po_amount, po_date, delivery_date |
| Statement of Work (SOW) | 15 | sow_number, start_date, end_date, services_description |

**What Needs to Be Extracted:**
- Document type (classification)
- Party information (buyer, vendor, addresses)
- Dates (contract, PO, SOW dates)
- Financial data (amounts, currency)
- Identifiers (PO numbers, SOW numbers, program codes)
- Relationships (links to parent framework)

---

### ✅ COMPLETED: Invoice Extraction Parameters

**Status:** COMPLETE - All 13 fields documented with linkage rules

**Deliverable:** `EXTRACTION_SPECIFICATION.md` - Section 2

**Invoice Fields:**

| Field | Type | Priority | Why Needed |
|-------|------|----------|-----------|
| invoice_id | String | CRITICAL | Unique identification |
| invoice_date | Date | CRITICAL | Date validation |
| vendor_name | String | CRITICAL | Party matching |
| buyer_name | String | CRITICAL | Party matching |
| po_number | String | CRITICAL | Primary linkage to contract |
| program_code | String | HIGH | Secondary linkage |
| sow_number | String | MEDIUM | Tertiary linkage |
| invoice_amount | Decimal | CRITICAL | Amount validation |
| currency | Enum | HIGH | Amount validation |
| payment_terms | String | MEDIUM | Due date calculation |
| payment_due_date | Date | HIGH | Derived field |
| line_items | List | MEDIUM | Detail breakdown |
| services_description | String | MEDIUM | Service validation |

---

## PART 2: EXTRACTION DEPENDENCIES

### ⏳ TODO: Map Extraction Dependencies

**Status:** NOT STARTED

**Task:** Create detailed mapping showing how contract fields enable invoice matching and validation

**Deliverables:**
1. Dependency matrix (Contract fields → Invoice fields)
2. Validation rules (which contract field enables which check)
3. Linkage rules (which fields are used for matching)

**Example Dependencies:**

```
Contract.po_number
  ↓
  Used by: Invoice matching (Priority 1)
  Validates: Invoice amount ≤ PO amount
  
Contract.po_date + Contract.delivery_date
  ↓
  Used by: Invoice date validation
  Validates: po_date ≤ invoice_date ≤ delivery_date + 30 days
  
Contract.sow_start_date + Contract.sow_end_date
  ↓
  Used by: Invoice date validation
  Validates: sow_start_date ≤ invoice_date ≤ sow_end_date
  
Contract.vendor_name + Contract.buyer_name
  ↓
  Used by: Invoice party matching (Priority 2)
  Validates: Invoice parties match contract parties
  
Contract.program_code
  ↓
  Used by: Invoice program matching (Priority 3)
  Validates: Invoice program matches contract program
```

**Output Document:** Create new file `EXTRACTION_DEPENDENCIES.md`

---

## PART 3: DOCUMENT EXTRACTION IMPLEMENTATION

### ⏳ TODO #1: Fix Date Format Extraction

**Status:** NOT STARTED

**Current Problem:**
- Regex pattern: `r"\d{4}[\s\-_]\d{2}[\s\-_]\d{2}"`
- Only matches: `YYYY-MM-DD`, `YYYY_MM_DD`, `YYYY MM DD`
- Does NOT match: `YYYY/MM/DD` (used in PDFs)
- Result: PO dates like `2022/01/14` are NOT extracted

**Task:** Update `_extract_dates()` method in `ContractRelationshipDiscoverer` class

**Location:** Demo_Invoice_Processing_Agent.ipynb, Cell 3 (Line ~245)

**Current Code:**
```python
filename_dates = re.findall(r"\d{4}[\s\-_]\d{2}[\s\-_]\d{2}", doc_path.name)
```

**Required Fix:**
```python
# Add forward slash to separator pattern
filename_dates = re.findall(r"\d{4}[\s\-_/]\d{2}[\s\-_/]\d{2}", doc_path.name)
```

**Test Case:**
- Input: `Purchase Order No. 2151002393.pdf` with content `2022/01/14`
- Expected: Extract date `2022/01/14`
- Current Result: FAIL (0% extraction)
- After Fix: PASS (100% extraction)

**Impact:** Fixes extraction for all PDF documents using `/` separator

---

### ⏳ TODO #2: Add PDF Content Extraction

**Status:** NOT STARTED

**Current Problem:**
- Only extracts from `.docx` files
- PDFs are processed but only filename is used
- Result: 33% of documents (PDFs) completely ignored
- Loses all content from PDFs

**Task:** Update `_extract_parties()` method to read PDF content

**Location:** Demo_Invoice_Processing_Agent.ipynb, Cell 3 (Line ~220)

**Current Code:**
```python
if doc_path.suffix.lower() == ".docx":
    doc = Document(doc_path)
    text = "\n".join([p.text for p in doc.paragraphs])
else:
    # For PDFs and other types, would need pdfplumber etc
    # For now, extract from filename
    text = doc_path.name
```

**Required Fix:**
```python
if doc_path.suffix.lower() == ".docx":
    doc = Document(doc_path)
    text = "\n".join([p.text for p in doc.paragraphs])
elif doc_path.suffix.lower() == ".pdf":
    import pdfplumber
    with pdfplumber.open(doc_path) as pdf:
        text = "\n".join([page.extract_text() or "" for page in pdf.pages])
else:
    text = doc_path.name
```

**Test Case:**
- Input: `Bayer_CLMS_-_Action_required_Contract_JP0094.pdf` (30 pages)
- Should Extract: Full party names, dates, Annex references
- Current Result: FAIL (only filename used)
- After Fix: PASS (all PDF content available)

**Impact:** Enables extraction from Framework Agreements and POs in PDF format

---

### ⏳ TODO #3: Implement Amount Extraction

**Status:** NOT STARTED

**Current Problem:**
- No amount extraction anywhere in code
- Field doesn't exist
- Contract amounts ($5M, $60K) completely lost
- Cannot validate invoices against PO amounts

**Task:** Create new method `_extract_amounts()` in `ContractRelationshipDiscoverer` class

**Location:** Demo_Invoice_Processing_Agent.ipynb, Cell 3 (after `_extract_dates()`)

**Specification:**
- Method signature: `def _extract_amounts(self, text: str) -> Dict[str, Optional[float]]`
- Input: Document text
- Output: `{"found": [list of amounts], "primary": amount_value, "currency": "USD"}`

**Patterns to Match:**
```python
patterns = [
    r'\$\s*([\d,]+(?:\.\d{2})?)',  # $XXX,XXX.XX or $5000
    r'€\s*([\d,]+(?:\.\d{2})?)',   # €XXX,XXX.XX
    r'USD\s*([\d,]+(?:\.\d{2})?)', # USD 5000000
    r'EUR\s*([\d,]+(?:\.\d{2})?)', # EUR 5000000
]
```

**Test Cases:**
- Input: `$60,000.00 USD` (PO 2151002393) → Output: `60000.00`
- Input: `$5,000,000` (Framework) → Output: `5000000.0`
- Input: `$5M` → Output: Not extracted (too approximate)

**Storage:**
- Add to document identifiers dictionary:
  ```python
  identifiers = {
      ...
      "amounts": {"found": [...], "primary": 60000.00, "currency": "USD"}
  }
  ```

**Impact:** Enables amount validation and contract value tracking

---

### ⏳ TODO #4: Implement PO Number Extraction

**Status:** NOT STARTED

**Current Problem:**
- No PO number extraction method exists
- Field not tracked anywhere
- Invoice-to-PO linkage completely broken
- Critical for invoice matching

**Task:** Create new method `_extract_po_number()` in `ContractRelationshipDiscoverer` class

**Location:** Demo_Invoice_Processing_Agent.ipynb, Cell 3 (after amount extraction)

**Specification:**
- Method signature: `def _extract_po_number(self, doc_path: Path, text: str) -> Optional[str]`
- Input: Document path + text content
- Output: PO number string or None

**Patterns to Match:**
```python
patterns = [
    r'(?:PO|Purchase Order)\s*#:?\s*(\d+)',      # PO #2151002393 or PO: 2151002393
    r'(?:PO|PO\.)\s*(\d{10})',                    # PO 2151002393 (10 digits)
    r'purchase\s+order\s+(?:number|#)\s*(\d+)',  # Purchase Order Number 2151002393
    r'Po(?:_|-)?Number:?\s*(\d+)',                # Po_Number: 2151002393
]
```

**Test Case:**
- Input: `Purchase Order No. 2151002393.pdf` with "Purchase Order No. 2151002393 dated..."
- Expected: `"2151002393"`
- Current Result: FAIL (method doesn't exist)
- After Fix: PASS

**Storage:**
- Add to document identifiers dictionary:
  ```python
  identifiers = {
      ...
      "po_number": "2151002393"
  }
  ```

**Impact:** Enables primary-priority invoice linkage (highest confidence)

---

### ⏳ TODO #5: Fix Party Name Extraction

**Status:** NOT STARTED

**Current Problem:**
- Only captures abbreviated names ("BAYER", "R4")
- Misses full legal names ("Bayer Yakuhin, Ltd.", "r4 Technologies, Inc.")
- Cannot distinguish different legal entities (Bayer Yakuhin vs Bayer Consumer Health)
- Results in wrong grouping of frameworks

**Task:** Improve `_extract_parties()` method to extract full legal names

**Location:** Demo_Invoice_Processing_Agent.ipynb, Cell 3 (Line ~223)

**Specification:**
- Extract from multiple sources:
  1. PDF Annex D (Definitions section)
  2. DOCX headers/footers
  3. Document body introductions
  4. "This Agreement is between..." clauses

**Current Code:**
```python
if "bayer" in text.lower():
    parties.add("BAYER")
if "r4" in text.lower():
    parties.add("R4")
```

**Required Improvements:**
```python
# Pattern 1: Legal entity definitions (Annex D style)
definition_pattern = r'(\w[\w\s&,\.]*(?:Ltd\.|Inc\.|Inc|LLC|GmbH)[\w\s]*?)(?:\s+(?:or|"|\n|$))'

# Pattern 2: "This Agreement is between..." clauses
agreement_pattern = r'This\s+(?:Agreement|Contract)\s+(?:is\s+)?(?:entered into\s+)?(?:between|by\s+and)[\s\n]+"?([^"]+?)"?\s+and\s+"?([^"]+?)"?'

# Pattern 3: Headers/introductions
header_pattern = r'^(Bayer[\w\s,\.]+?)$'  # Multiline, beginning of line
```

**Test Cases:**
- Input: "Annex D: Bayer Yakuhin, Ltd., Breeze Tower 2-4-9, Umeda..."
- Expected: `["Bayer Yakuhin, Ltd.", "Breeze Tower 2-4-9, Umeda, 530-0001 Osaka"]`
- Current Result: Only `["BAYER"]`

- Input: "Bayer Consumer Health, China & Asia-Pacific"
- Expected: Full legal name
- Current Result: Only `["BAYER"]`

**Impact:**
- Fixes grouping of 2 different frameworks
- Enables proper BCH CAP invoice linkage
- Resolves "orphaned PO" issue

---

### ⏳ TODO #6: Fix Program Code Extraction

**Status:** NOT STARTED

**Current Problem:**
- Only extracts from filename
- Filename-dependent fragility
- If filename changes, grouping breaks
- Only filters 5 common words

**Task:** Update `_extract_program_code()` to search document content

**Location:** Demo_Invoice_Processing_Agent.ipynb, Cell 3 (Line ~238)

**Current Code:**
```python
match = re.search(r"\b([A-Z]{2,4})\b", filename)
if match:
    code = match.group(1)
    if code not in ["FOR", "PDF", "SOW", "MSA", "THE"]:
        return code
return None
```

**Required Improvement:**
```python
# Search content first for known program codes
known_codes = ["BCH", "CAP", "CLMS"]
for code in known_codes:
    if code in text.upper():
        return code

# Fallback to filename
match = re.search(r"\b([A-Z]{2,4})\b", filename)
if match:
    code = match.group(1)
    if code not in ["FOR", "PDF", "SOW", "MSA", "THE", "AND"]:
        return code

return None
```

**Test Cases:**
- Input: Document containing "BCH CAP program"
- Expected: `"BCH"`
- Current Result: Only works if in filename

**Impact:** More robust program code extraction, less dependent on filename format

---

### ⏳ TODO #7: Implement Document Type Detection from Content

**Status:** NOT STARTED

**Current Problem:**
- Only detects from filename keywords
- Cannot distinguish frameworks from related docs
- Content-based keywords ignored

**Task:** Update `_detect_document_type()` to read document content

**Location:** Demo_Invoice_Processing_Agent.ipynb, Cell 3 (Line ~203)

**Specification:**
- Search for keywords in document text
- Priority order for detection
- Fallback to filename if content unclear

**Keywords by Type:**
```python
framework_keywords = ["master service agreement", "msa", "framework agreement", "terms and conditions"]
po_keywords = ["purchase order", "po number", "po #", "order number"]
sow_keywords = ["statement of work", "sow", "scope of work", "sow number"]
order_form_keywords = ["order form", "order for services"]
```

**Required Code:**
```python
def _detect_document_type(self, doc_path: Path, text: str) -> str:
    """Detect document type from content first, then filename"""
    text_lower = text.lower()
    
    # Check content for keywords
    if any(kw in text_lower for kw in ["master service agreement", "framework"]):
        return "FRAMEWORK"
    if any(kw in text_lower for kw in ["purchase order", "po number"]):
        return "PURCHASE_ORDER"
    if any(kw in text_lower for kw in ["statement of work", "sow number"]):
        return "SOW"
    if "order form" in text_lower:
        return "ORDER_FORM"
    
    # Fallback to filename
    filename_upper = doc_path.name.upper()
    if "MSA" in filename_upper:
        return "FRAMEWORK"
    if "PO" in filename_upper or "PURCHASE" in filename_upper:
        return "PURCHASE_ORDER"
    # ... etc
    
    return "OTHER"
```

**Test Cases:**
- Input: PDF with "Master Service Agreement" in content
- Expected: `"FRAMEWORK"` (even if filename doesn't say so)

**Impact:** Proper document classification enables correct hierarchy building

---

### ⏳ TODO #8: Build Contract Relationship Hierarchy

**Status:** NOT STARTED

**Current Problem:**
- Groups by (parties, program_code) only
- Cannot identify Framework → PO relationships
- Cannot identify Framework → SOW relationships
- PO 2151002393 marked as "orphaned" when it should link to Framework

**Task:** Redesign `_verify_contract_hierarchies()` to build explicit parent-child relationships

**Location:** Demo_Invoice_Processing_Agent.ipynb, Cell 3 (Line ~260)

**Specification:**

**Current Broken Logic:**
```
Groups contracts by: (parties, program_code)
Result: Cannot trace PO to Framework
```

**Required New Logic:**
```
1. Identify Framework documents (document_type == "FRAMEWORK")
2. For each PO/SOW:
   a. Find Framework with matching parties + program_code
   b. Create explicit link: PO/SOW.framework_reference = Framework.id
   c. Add to Framework's children list

3. Store hierarchy:
   {
       "framework_id": "BAYER_CLMS_001",
       "framework_doc": "Bayer_CLMS_...pdf",
       "purchase_orders": [
           {"po_number": "2151002393", "doc": "PO 2151002393.pdf"}
       ],
       "sows": [
           {"sow_number": "11414-200", "doc": "SOW...docx"}
       ]
   }
```

**Test Case:**
- Input: 2 frameworks + 1 PO + 3 SOWs
- Expected Output:
  ```
  Framework 1 (Bayer_CLMS)
    └── PO 2151002393
  
  Framework 2 (BCH_CAP)
    ├── SOW 11414-200 (2021)
    ├── Order Form (2021)
    └── Order Form (2022)
  ```
- Current Output: PO marked as "orphaned"
- After Fix: Proper hierarchy with no orphans

**Impact:** Resolves the "orphaned PO" issue, enables proper invoice linkage

---

## PART 4: INVOICE EXTRACTION IMPLEMENTATION

### ⏳ TODO #9: Create Invoice Extraction Class

**Status:** NOT STARTED

**Current Status:** InvoiceParser class exists but is incomplete

**Task:** Complete or replace `InvoiceParser` class with all invoice extraction logic

**Specification:**
- Extract all 13 invoice fields (see EXTRACTION_SPECIFICATION.md Section 2)
- Support PDF and DOCX formats
- Extract from document content (NOT filename)
- Calculate payment_due_date from invoice_date + payment_terms

**Required Methods:**
1. `parse_invoices_directory()` - Main entry point
2. `_extract_invoice_id()` - From content, not filename
3. `_extract_dates()` - invoice_date
4. `_extract_vendors()` - vendor_name, buyer_name
5. `_extract_amounts()` - invoice_amount, currency, line_items
6. `_extract_linkage_keys()` - po_number, sow_number, program_code
7. `_extract_services()` - services_description
8. `_calculate_payment_due_date()` - Derived from date + terms

**Test Cases:**
- Parse sample invoices from demo_invoices/
- Extract all 13 fields
- Calculate due dates correctly
- Handle multiple line items

**Impact:** Enables invoice-to-contract linkage

---

### ⏳ TODO #10: Implement Invoice-to-Contract Linkage

**Status:** NOT STARTED

**Current Status:** `InvoiceLinkageDetector` class exists but relies on broken extraction

**Task:** Implement 4-priority matching system

**Specification:**

**Priority 1 (99% confidence):**
```
IF invoice.po_number EXISTS:
    FIND: Contract where po_number == invoice.po_number
    LINK: Direct match
    CONFIDENCE: 0.99
```

**Priority 2 (85% confidence):**
```
IF invoice.po_number NOT FOUND:
    FIND: Contract where
        - vendor_name fuzzy_matches invoice.vendor_name AND
        - buyer_name fuzzy_matches invoice.buyer_name AND
        - (sow_number == invoice.sow_number OR program_code == invoice.program_code)
    LINK: Party + Program match
    CONFIDENCE: 0.85
```

**Priority 3 (70% confidence):**
```
IF priorities 1-2 fail:
    FIND: Contract where program_code == invoice.program_code
    LINK: Program code only
    CONFIDENCE: 0.70
```

**Priority 4 (50% confidence):**
```
IF priorities 1-3 fail:
    FIND: Contract where
        - vendor_name similar to invoice.vendor_name AND
        - invoice.amount within 20% of contract.amount
    LINK: Fuzzy match
    CONFIDENCE: 0.50
    WARNING: High false positive risk
```

**Implementation Location:** `InvoiceLinkageDetector._detect_single_invoice()`

**Test Cases:**
- Invoice with PO# → Should match Priority 1
- Invoice with program code but no PO# → Should match Priority 2/3
- Invoice with only vendor match → Should match Priority 4

**Impact:** Enables invoice-contract connection

---

### ⏳ TODO #11: Implement Invoice Validation

**Status:** NOT STARTED

**Task:** Create validation engine that checks invoice against linked contract

**Validation Rules:**

**Rule 1: Amount Validation**
```
IF invoice linked to PO:
    IF invoice.amount > po.amount:
        FAIL: "Invoice amount exceeds PO authorization"
        SEVERITY: ERROR
    ELSE IF invoice.amount > po.amount * 0.9:
        WARN: "Invoice near PO limit"
        SEVERITY: WARNING
```

**Rule 2: Date Validation (PO)**
```
IF invoice linked to PO:
    IF invoice.date < po.date:
        FAIL: "Invoice dated before PO issued"
        SEVERITY: ERROR
    ELSE IF invoice.date > po.delivery_date + 30 days:
        WARN: "Invoice long after delivery date"
        SEVERITY: WARNING
```

**Rule 3: Date Validation (SOW)**
```
IF invoice linked to SOW:
    IF NOT (sow.start_date <= invoice.date <= sow.end_date):
        FAIL: "Invoice outside SOW period"
        SEVERITY: ERROR
```

**Rule 4: Vendor Validation**
```
IF invoice.vendor_name exists:
    IF NOT fuzzy_match(invoice.vendor_name, contract.vendor_name):
        FAIL: "Invoice vendor mismatch"
        SEVERITY: ERROR
```

**Rule 5: Buyer Validation**
```
IF invoice.buyer_name exists:
    IF NOT fuzzy_match(invoice.buyer_name, contract.buyer_name):
        FAIL: "Invoice buyer mismatch"
        SEVERITY: ERROR
```

**Output:**
```python
{
    "validation": {
        "amount_valid": True/False,
        "date_valid": True/False,
        "vendor_valid": True/False,
        "buyer_valid": True/False,
        "overall_status": "VALID" / "WARNING" / "REJECTED",
        "issues": [list of validation errors]
    }
}
```

**Impact:** Prevents invalid invoices from being paid

---

## PART 5: TESTING & VALIDATION

### ⏳ TODO #12: Test Extraction Against Real Documents

**Status:** NOT STARTED

**Task:** Run all extraction methods against 6 real contract documents and sample invoices

**Test Documents:**
1. `Bayer_CLMS_-_Action_required_Contract_JP0094.pdf` (Framework #1, 30 pages)
2. `Purchase Order No. 2151002393.pdf` (PO, 2 pages)
3. `r4 MSA for BCH CAP 2021 12 10.docx` (Framework #2, DOCX)
4. `r4 Order Form for BCH CAP 2021 12 10.docx` (SOW #1, DOCX)
5. `r4 Order Form for BCH CAP 2022 11 01.docx` (SOW #2, DOCX)
6. `r4 SOW for BCH CAP 2021 12 10.docx` (SOW #3, DOCX)

**Success Criteria:**

| Field | Document | Target | Acceptable |
|-------|----------|--------|-----------|
| document_type | All 6 | 100% | 95%+ |
| po_number | PO only | 100% | 100% |
| po_amount | PO only | 100% | 100% |
| buyer_legal_name | All 6 | 100% | 90%+ |
| vendor_legal_name | All 6 | 100% | 90%+ |
| dates | All 6 | 95%+ | 90%+ |
| program_code | BCH CAP docs | 100% | 90%+ |
| sow_number | SOW docs | 100% | 90%+ |

**Test Results Documentation:**
- Create file: `EXTRACTION_TEST_RESULTS.md`
- Include:
  - Test date and time
  - Documents processed
  - Per-field success rates
  - Failures and root causes
  - Recommendations for fixes

**Impact:** Validates that extraction meets 90%+ target

---

## Success Metrics Summary

| Metric | Target | Why Important |
|--------|--------|--------------|
| Contract extraction success | 90%+ per critical field | Core functionality |
| Invoice extraction success | 90%+ per critical field | Core functionality |
| Invoice-to-PO linkage accuracy | 95%+ | Primary linkage method |
| Invoice-to-SOW linkage accuracy | 85%+ | Secondary linkage method |
| False positive rate (wrong linkage) | < 5% | Prevent payment errors |
| Document type detection | 95%+ | Enable hierarchy building |
| Party name extraction | 90%+ | Framework identification |
| Amount extraction | 90%+ | Invoice validation |
| Date extraction | 90%+ | Timing validation |

---

## Document References

**Specification Documents:**
- `EXTRACTION_SPECIFICATION.md` - Complete parameter definitions
- `EXTRACTION_BUGS_ANALYSIS.md` - Current bugs and root causes
- `EXTRACTION_PLAN_SUMMARY.md` - Visual overview with examples

**Implementation Document:**
- Demo_Invoice_Processing_Agent.ipynb - Notebook containing pipeline classes

**Test Data:**
- `/demo_contracts/` - 6 real contract documents
- `/demo_invoices/` - Sample invoice documents (if available)

---

## Status Dashboard

| Task | Status | Priority | Est. Effort |
|------|--------|----------|------------|
| Contract parameters | ✅ DONE | - | - |
| Invoice parameters | ✅ DONE | - | - |
| Dependency mapping | ⏳ TODO | MEDIUM | 2 hours |
| Fix date format | ⏳ TODO | CRITICAL | 0.5 hours |
| Add PDF support | ⏳ TODO | CRITICAL | 1 hour |
| Amount extraction | ⏳ TODO | CRITICAL | 2 hours |
| PO number extraction | ⏳ TODO | CRITICAL | 1.5 hours |
| Party name extraction | ⏳ TODO | CRITICAL | 2.5 hours |
| Program code fix | ⏳ TODO | HIGH | 1 hour |
| Document type detection | ⏳ TODO | HIGH | 1.5 hours |
| Contract hierarchy | ⏳ TODO | CRITICAL | 3 hours |
| Invoice extraction | ⏳ TODO | CRITICAL | 4 hours |
| Invoice-contract linkage | ⏳ TODO | CRITICAL | 3 hours |
| Invoice validation | ⏳ TODO | CRITICAL | 2.5 hours |
| Testing & validation | ⏳ TODO | HIGH | 2 hours |

**Total Estimated Effort:** ~26 hours

**Current Progress:** 7.7% (2 of 26 hours completed)

---

## Next Steps

1. **Immediate:** Start with TODO #1 (Fix Date Format) - quickest win, 0.5 hours
2. **Then:** TODO #2 (Add PDF Support) - enables other fixes, 1 hour
3. **Parallel:** TODO #3-#5 (Extraction methods) - 5.5 hours
4. **Then:** TODO #8 (Contract Hierarchy) - critical for grouping
5. **Parallel:** TODO #9-#11 (Invoice extraction) - 9.5 hours
6. **Final:** TODO #12 (Testing) - validate everything works

