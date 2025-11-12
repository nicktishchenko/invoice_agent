# Implementation Plan: Contract-to-Invoice Matching & Contract-Specific Rules

## Objective
Ensure that each invoice is validated using rules extracted from its specific source contract, not a generic set of rules.

---

## Current State Analysis

### Current Flow (INCORRECT):
```
1. Extract rules from ONE contract → saved to extracted_rules.json
2. Initialize InvoiceProcessor with those rules (all invoices use same rules)
3. Process all invoices → all validated against same contract's rules
```

### Target Flow (CORRECT):
```
1. Extract rules from MULTIPLE contracts → store per contract
2. For each invoice:
   a. Detect which contract it belongs to (matching)
   b. Load rules for that specific contract
   c. Validate invoice using contract-specific rules
```

---

## Implementation Plan

### Phase 1: Update Rules Storage Format

**Goal:** Support multiple contracts with their own rules

**Current Format:**
```json
{
  "contract_path": "docs/contracts/sample_contract_net30.pdf",
  "extracted_at": "2025-11-06T18:05:06",
  "rules": [...]
}
```

**New Format (Backward Compatible):**
```json
{
  "version": "2.0",
  "extracted_at": "2025-11-06T18:05:06",
  "contracts": [
    {
      "contract_id": "CONTRACT_1",
      "contract_path": "docs/contracts/sample_contract_net30.pdf",
      "parties": ["Vendor Corp", "Client Inc"],
      "program_code": "BCH",
      "extracted_at": "2025-11-06T18:05:06",
      "rules": [...]
    },
    {
      "contract_id": "CONTRACT_2",
      "contract_path": "docs/contracts/another_contract.pdf",
      "parties": ["Vendor Corp", "Client Inc"],
      "program_code": "ACME",
      "extracted_at": "2025-11-06T18:10:00",
      "rules": [...]
    }
  ]
}
```

**Backward Compatibility:**
- If file has old format (single `contract_path`), convert to new format
- If file has new format but only one contract, treat as single-contract mode

**Files to Modify:**
- `InvoiceProcessor._load_rules()` - Handle both formats
- Cell 15 (Rule Extraction) - Save in new format
- Cell 29 (Complete Pipeline) - Save in new format

---

### Phase 2: Add Contract-to-Invoice Matching Logic

**Goal:** Detect which contract an invoice belongs to before validation

**Matching Methods (Priority Order):**

1. **PO Number Matching** (Confidence: 0.95)
   - Extract PO number from invoice
   - Search contract CONTENT (not filenames) for matching PO references
   - Match if PO found in contract document content
   - Note: Requires parsing contract documents to search content

2. **Vendor + Program Code Matching** (Confidence: 0.85)
   - Extract vendor name from invoice
   - Extract program code from invoice (raw_text)
   - Match if BOTH vendor matches contract party AND program code matches
   - This handles cases where same parties have multiple contracts

3. **Vendor + Date Range Matching** (Confidence: 0.80)
   - Extract vendor name from invoice
   - Extract invoice date
   - Match if vendor matches contract party AND invoice date is within contract date range
   - Requires contract date range information (extract from contract documents or metadata)
   - Date ranges are helpful for distinguishing multiple contracts between same parties

4. **Program Code Matching** (Confidence: 0.70)
   - Extract program codes from invoice description/raw_text
   - Match if program code matches contract's program_code
   - Lower confidence because program codes can be ambiguous

5. **Vendor Only Matching** (Confidence: 0.60)
   - Extract vendor name from invoice
   - Match if vendor matches any party in contract
   - LOWEST confidence - only used if no other matches found
   - Will likely result in AMBIGUOUS status if multiple contracts share same parties

6. **No Fallback - Manual Review Required**
   - If no match found, status = "UNMATCHED"
   - Invoice cannot be processed automatically
   - Requires manual contract assignment
   - Log clear warning about unmatched invoice

**New Class: `InvoiceContractMatcher`**

```python
class InvoiceContractMatcher:
    """
    Matches invoices to their source contracts using multiple detection methods.
    
    Matching priority (stops at first successful match):
    1. PO Number (confidence: 0.95)
    2. Vendor + Program Code (confidence: 0.85)
    3. Vendor + Date Range (confidence: 0.80)
    4. Program Code only (confidence: 0.70)
    5. Vendor only (confidence: 0.60) - last resort, may be ambiguous
    6. No match → UNMATCHED status (requires manual review)
    """
    
    def __init__(self, contracts_data: Dict):
        """
        Args:
            contracts_data: Dict with 'contracts' list (from extracted_rules.json)
        """
        self.contracts = contracts_data.get("contracts", [])
        self.contract_index = self._build_contract_index()
    
    def match_invoice_to_contract(self, invoice_data: Dict) -> Dict:
        """
        Detect which contract an invoice belongs to.
        
        Returns:
            {
                "contract_id": "..." or None,
                "contract_path": "..." or None,
                "match_method": "PO_NUMBER|VENDOR_PROGRAM|VENDOR_DATE|PROGRAM_CODE|VENDOR_ONLY|UNMATCHED",
                "confidence": 0.0-1.0,
                "status": "MATCHED|AMBIGUOUS|UNMATCHED",
                "matching_details": {...}
            }
        """
        matches = []
        
        # 1. Try PO number matching (highest priority, unique identifier)
        po_matches = self._match_by_po_number(invoice_data)
        if po_matches:
            matches.extend(po_matches)
        
        # 2. Try vendor + program code (if no PO match)
        if not matches:
            vendor_program_matches = self._match_by_vendor_and_program(invoice_data)
            if vendor_program_matches:
                matches.extend(vendor_program_matches)
        
        # 3. Try vendor + date range (if no previous matches)
        if not matches:
            vendor_date_matches = self._match_by_vendor_and_date(invoice_data)
            if vendor_date_matches:
                matches.extend(vendor_date_matches)
        
        # 4. Try program code only (if no previous matches)
        if not matches:
            program_matches = self._match_by_program_code(invoice_data)
            if program_matches:
                matches.extend(program_matches)
        
        # 5. Try vendor only (last resort, lowest confidence)
        if not matches:
            vendor_matches = self._match_by_vendor_only(invoice_data)
            if vendor_matches:
                matches.extend(vendor_matches)
        
        # Build result
        result = {
            "contract_id": None,
            "contract_path": None,
            "match_method": None,
            "confidence": 0.0,
            "status": "UNMATCHED",
            "matching_details": {},
            "alternative_matches": [],
        }
        
        if len(matches) == 1:
            # Unique match
            contract_id, method, confidence = matches[0]
            contract_info = self.contract_index.get(contract_id, {})
            result["contract_id"] = contract_id
            result["contract_path"] = contract_info.get("contract_path", "")
            result["match_method"] = method
            result["confidence"] = confidence
            result["status"] = "MATCHED"
            result["matching_details"] = self._get_matching_details(invoice_data, contract_id)
        
        elif len(matches) > 1:
            # Multiple matches - ambiguous
            contract_id, method, confidence = matches[0]
            contract_info = self.contract_index.get(contract_id, {})
            result["contract_id"] = contract_id
            result["contract_path"] = contract_info.get("contract_path", "")
            result["match_method"] = method
            result["confidence"] = confidence
            result["status"] = "AMBIGUOUS"
            result["alternative_matches"] = [
                {"contract_id": m[0], "method": m[1], "confidence": m[2]}
                for m in matches[1:]
            ]
            result["matching_details"] = self._get_matching_details(invoice_data, contract_id)
        
        else:
            # No match - UNMATCHED (no fallback)
            result["status"] = "UNMATCHED"
            result["matching_details"] = {
                "reason": "No matching contract found. Manual review required.",
                "invoice_po": invoice_data.get("po_number"),
                "invoice_vendor": invoice_data.get("vendor_name"),
                "invoice_date": str(invoice_data.get("invoice_date")) if invoice_data.get("invoice_date") else None,
            }
        
        return result
    
    def _match_by_po_number(self, invoice_data: Dict) -> List[Tuple[str, str, float]]:
        """
        Match by PO number - searches contract CONTENT (not filenames).
        
        Requires parsing contract documents to search for PO references in content.
        """
        invoice_po = invoice_data.get("po_number")
        if not invoice_po:
            return []
        
        matches = []
        invoice_po_upper = invoice_po.upper()
        
        # Parse each contract and search content for PO number
        for contract in self.contracts:
            contract_id = contract.get("contract_id", "UNKNOWN")
            contract_path = contract.get("contract_path", "")
            
            if not contract_path:
                continue
            
            # Parse contract document to get text content
            contract_text = self._parse_contract_content(contract_path)
            
            # Search for PO number in contract content
            if invoice_po_upper in contract_text.upper():
                matches.append((contract_id, "PO_NUMBER", 0.95))
        
        return matches
    
    def _parse_contract_content(self, contract_path: str) -> str:
        """
        Parse contract document and extract text content.
        Reuses parsing logic from InvoiceRuleExtractorAgent.
        """
        # Implementation: Use pdfplumber for PDF, python-docx for DOCX
        # This method will be called during matching
        pass
    
    def _match_by_vendor_and_program(self, invoice_data: Dict) -> List[Tuple[str, str, float]]:
        """Match by vendor AND program code - handles multiple contracts between same parties"""
        pass
    
    def _match_by_vendor_and_date(self, invoice_data: Dict) -> List[Tuple[str, str, float]]:
        """
        Match by vendor AND date range - requires contract date range info.
        
        Checks if:
        1. Vendor matches contract party
        2. Invoice date falls within contract date range
        """
        invoice_vendor = invoice_data.get("vendor_name", "").lower()
        invoice_date = invoice_data.get("invoice_date")
        
        if not invoice_vendor or not invoice_date:
            return []
        
        matches = []
        
        for contract_id, contract_info in self.contract_index.items():
            # Check vendor match
            parties = contract_info.get("parties", [])
            vendor_matches = any(
                party in invoice_vendor or invoice_vendor in party
                for party in parties
            )
            
            if not vendor_matches:
                continue
            
            # Check date range
            date_range = contract_info.get("date_range")
            if date_range and self._date_in_range(invoice_date, date_range):
                matches.append((contract_id, "VENDOR_DATE", 0.80))
        
        return matches
    
    def _date_in_range(self, date: datetime, date_range: Dict) -> bool:
        """Check if date falls within contract date range."""
        start_date = date_range.get("start")
        end_date = date_range.get("end")
        
        if not start_date or not end_date:
            return False
        
        # Parse dates if strings
        if isinstance(start_date, str):
            start_date = datetime.fromisoformat(start_date.split("T")[0])
        if isinstance(end_date, str):
            end_date = datetime.fromisoformat(end_date.split("T")[0])
        
        return start_date <= date <= end_date
    
    def _match_by_program_code(self, invoice_data: Dict) -> List[Tuple[str, str, float]]:
        """Match by program code only"""
        pass
    
    def _match_by_vendor_only(self, invoice_data: Dict) -> List[Tuple[str, str, float]]:
        """Match by vendor only - last resort, low confidence"""
        pass
    
    def _build_contract_index(self) -> Dict:
        """
        Build searchable index of contract metadata.
        
        Includes:
        - contract_id
        - contract_path
        - parties (normalized, lowercase)
        - program_code (uppercase)
        - date_range (start, end dates)
        """
        index = {}
        for contract in self.contracts:
            contract_id = contract.get("contract_id", "UNKNOWN")
            index[contract_id] = {
                "contract_id": contract_id,
                "contract_path": contract.get("contract_path", ""),
                "parties": [p.lower() for p in contract.get("parties", [])],
                "program_code": contract.get("program_code", "").upper(),
                "date_range": contract.get("date_range"),  # {"start": "...", "end": "..."}
            }
        return index
    
    def _get_matching_details(self, invoice_data: Dict, contract_id: str) -> Dict:
        """Get details of why invoice matched this contract"""
        pass
```

**Files to Create/Modify:**
- Add `InvoiceContractMatcher` class to Cell 19 (InvoiceProcessor class cell)
- Or create new cell before InvoiceProcessor

---

### Phase 3: Update InvoiceProcessor to Use Contract-Specific Rules

**Goal:** Load rules dynamically per invoice based on contract match

**Current `InvoiceProcessor.__init__()`:**
```python
def __init__(self, rules_file: str = "extracted_rules.json"):
    self.rules = self._load_rules(rules_file)  # Loads ALL rules once
    self.payment_terms = self._extract_payment_terms()
```

**New Approach:**

**Option A: Lazy Loading (Recommended)**
```python
def __init__(self, rules_file: str = "extracted_rules.json"):
    self.rules_file = rules_file
    self.rules_data = self._load_rules_data(rules_file)  # Load full structure
    self.matcher = InvoiceContractMatcher(self.rules_data)
    self.current_contract_id = None
    self.current_rules = []  # Loaded per invoice
    self.payment_terms = None  # Extracted per invoice

def process_invoice(self, invoice_path: str) -> Dict[str, Any]:
    # 1. Parse invoice
    invoice_data = self.parse_invoice(invoice_path)
    
    # 2. Match invoice to contract
    match_result = self.matcher.match_invoice_to_contract(invoice_data)
    
    # 3. Check if contract was matched
    if match_result["status"] == "UNMATCHED":
        # Cannot process without contract match
        return {
            "invoice_file": invoice_data["file"],
            "status": "ERROR",
            "action": "Manual review required - no contract match found",
            "issues": ["Invoice could not be matched to any contract. Manual assignment required."],
            "warnings": [],
            "contract_match": match_result,
            "validation_timestamp": datetime.now().isoformat(),
        }
    
    # 4. Load rules for matched contract
    if match_result["status"] == "AMBIGUOUS":
        # Log warning but proceed with first match
        logger.warning(
            f"Ambiguous contract match for invoice {invoice_data['file']}. "
            f"Using contract {match_result['contract_id']} (confidence: {match_result['confidence']})"
        )
    
    self._load_contract_rules(match_result["contract_id"])
    
    # 5. Validate using contract-specific rules
    result = self.validate_invoice(invoice_data)
    
    # 6. Add contract match info to result
    result["contract_match"] = match_result
    
    return result

def _load_contract_rules(self, contract_id: str):
    """Load rules for a specific contract"""
    for contract in self.rules_data.get("contracts", []):
        if contract["contract_id"] == contract_id:
            self.current_contract_id = contract_id
            self.current_rules = contract["rules"]
            self.payment_terms = self._extract_payment_terms()
            logger.info(f"Loaded {len(self.current_rules)} rules for contract {contract_id}")
            return
    
    # Should not happen if matching worked correctly
    logger.error(f"Contract {contract_id} not found in rules data")
    raise ValueError(f"Contract {contract_id} not found. Cannot load rules.")
```

**Option B: Per-Contract Processor Instances**
- Create separate InvoiceProcessor instance per contract
- More memory but cleaner separation

**Recommendation:** Option A (Lazy Loading) - more efficient

**Files to Modify:**
- `InvoiceProcessor.__init__()` - Load full rules structure
- `InvoiceProcessor.process_invoice()` - Add contract matching step
- `InvoiceProcessor._load_rules()` → `_load_rules_data()` - Return full structure
- `InvoiceProcessor.validate_invoice()` - Use `self.current_rules` instead of `self.rules`

---

### Phase 4: Update Rule Extraction to Support Multiple Contracts

**Goal:** Extract rules from multiple contracts and store per contract

**Current Cell 15 (Single Contract):**
```python
contract_path = "docs/contracts/sample_contract_net30.pdf"
rules = rag_agent.run(contract_path)
rules_data = {
    "contract_path": contract_path,
    "extracted_at": datetime.now().isoformat(),
    "rules": rules
}
```

**New Approach:**

**Option A: Process All Contracts in Directory**
```python
# Scan contracts directory
contracts_dir = Path("docs/contracts")
contract_files = list(contracts_dir.glob("*.pdf")) + list(contracts_dir.glob("*.docx"))

all_contracts = []
for contract_path in contract_files:
    # Extract rules for this contract
    rules = rag_agent.run(str(contract_path))
    
    # Parse contract to extract metadata (parties, dates, program codes)
    contract_text = rag_agent.parse_document(str(contract_path))
    parties = extract_parties(contract_text)  # Extract party names
    date_range = extract_date_range(contract_text)  # Extract contract date range
    program_code = extract_program_code(contract_text)  # Extract program code
    
    # Generate contract_id from parties and program code
    contract_id = generate_contract_id(parties, program_code)
    
    contract_data = {
        "contract_id": contract_id,
        "contract_path": str(contract_path),
        "parties": parties,
        "program_code": program_code,
        "date_range": date_range,  # {"start": "YYYY-MM-DD", "end": "YYYY-MM-DD"}
        "extracted_at": datetime.now().isoformat(),
        "rules": rules
    }
    all_contracts.append(contract_data)

# Save all contracts
rules_data = {
    "version": "2.0",
    "extracted_at": datetime.now().isoformat(),
    "contracts": all_contracts
}
```

**Option B: Process One Contract at a Time (Append to Existing)**
- Load existing rules_data
- Add new contract to contracts list
- Save updated rules_data

**Recommendation:** Option A for batch processing, Option B for incremental

**Files to Modify:**
- Cell 15 - Update to process multiple contracts
- Cell 29 (Complete Pipeline) - Update to process multiple contracts

---

### Phase 5: Update Pipeline Cells

**Cell 29 (Complete Pipeline Test):**

**Current:**
```python
# Step 1: Extract rules from ONE contract
contract_path = "docs/contracts/sample_contract_net30.pdf"
rules = rag_agent.run(contract_path)
# Save rules

# Step 2: Process ALL invoices with same rules
processor = InvoiceProcessor(rules_file="extracted_rules.json")
for invoice_file in invoice_files:
    result = processor.process_invoice(invoice_file)
```

**New:**
```python
# Step 1: Extract rules from ALL contracts
contracts_dir = Path("docs/contracts")
contract_files = [f for f in contracts_dir.glob("*.*") 
                  if f.suffix.lower() in [".pdf", ".docx"]]

all_contracts = []
for contract_path in contract_files:
    rules = rag_agent.run(str(contract_path))
    contract_id = generate_contract_id(contract_path)
    all_contracts.append({
        "contract_id": contract_id,
        "contract_path": str(contract_path),
        "rules": rules,
        ...
    })

# Save all contracts
rules_data = {"version": "2.0", "contracts": all_contracts}
with open("extracted_rules.json", "w") as f:
    json.dump(rules_data, f, indent=2)

# Step 2: Process invoices with contract-specific rules
processor = InvoiceProcessor(rules_file="extracted_rules.json")
for invoice_file in invoice_files:
    result = processor.process_invoice(invoice_file)
    # Result now includes contract_match info
```

**Cell 30 (Report Generation):**
- Update to show which contract each invoice was matched to
- Show contract match confidence and method

---

## Implementation Steps (Order)

1. **Step 1:** Create `InvoiceContractMatcher` class
   - Add to Cell 19 or new cell
   - Implement matching methods
   - Test with sample data

2. **Step 2:** Update `InvoiceProcessor._load_rules()` → `_load_rules_data()`
   - Handle both old and new formats
   - Return full structure with contracts list

3. **Step 3:** Update `InvoiceProcessor.__init__()`
   - Initialize matcher
   - Store rules_data instead of rules list

4. **Step 4:** Update `InvoiceProcessor.process_invoice()`
   - Add contract matching step
   - Load contract-specific rules
   - Add contract_match to result

5. **Step 5:** Update `InvoiceProcessor.validate_invoice()`
   - Use `self.current_rules` instead of `self.rules`

6. **Step 6:** Update Cell 15 (Rule Extraction)
   - Process multiple contracts
   - Save in new format

7. **Step 7:** Update Cell 29 (Complete Pipeline)
   - Extract rules from all contracts
   - Process invoices with contract matching

8. **Step 8:** Update Cell 30 (Report)
   - Show contract match information

9. **Step 9:** Test & Verify
   - Test with single contract (backward compatibility)
   - Test with multiple contracts
   - Test invoice matching accuracy

---

## Data Flow Diagram

```
┌─────────────────┐
│ Contract Files  │
│ (docs/contracts)│
└────────┬────────┘
         │
         ▼
┌─────────────────────────┐
│ Rule Extraction (RAG)   │
│ - Process each contract │
│ - Extract rules         │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ extracted_rules.json     │
│ {                        │
│   "contracts": [        │
│     {contract_id,       │
│      rules, ...},       │
│     ...                  │
│   ]                      │
│ }                        │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ InvoiceProcessor         │
│ - Loads rules_data      │
│ - Initializes Matcher   │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ Invoice File             │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ 1. Parse Invoice         │
│    (extract fields)     │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ 2. Match to Contract     │
│    (PO/Vendor/Code)     │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ 3. Load Contract Rules  │
│    (contract-specific)  │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ 4. Validate Invoice     │
│    (using matched rules)│
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ Result + Contract Match │
│ Info                    │
└─────────────────────────┘
```

---

## Backward Compatibility Strategy

1. **Old Format Detection:**
   - Check if file has `contract_path` key (old format)
   - Convert to new format automatically

2. **Single Contract Mode:**
   - If only one contract in new format, behave like old format
   - No breaking changes for existing workflows

3. **Migration:**
   - Old `extracted_rules.json` files will auto-convert on first load
   - No manual migration needed

---

## Testing Strategy

1. **Unit Tests:**
   - Test `InvoiceContractMatcher` matching methods
   - Test format conversion (old → new)
   - Test rule loading per contract

2. **Integration Tests:**
   - Test complete pipeline with single contract
   - Test complete pipeline with multiple contracts
   - Test invoice matching accuracy

3. **Edge Cases:**
   - Invoice with no matching contract (fallback)
   - Invoice matching multiple contracts (ambiguous)
   - Empty contracts list
   - Missing contract metadata

---

## Success Criteria

✅ Each invoice is matched to its source contract  
✅ Rules are loaded per invoice based on contract match  
✅ Validation uses contract-specific rules  
✅ Backward compatibility maintained  
✅ Clear logging of contract matches  
✅ Report shows which contract each invoice belongs to  

---

## Estimated Implementation Time

- Phase 1: 30 minutes
- Phase 2: 1-2 hours
- Phase 3: 1-2 hours
- Phase 4: 1 hour
- Phase 5: 30 minutes
- Testing: 1 hour

**Total: ~5-7 hours**

