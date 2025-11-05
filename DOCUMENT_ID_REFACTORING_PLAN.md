# Document ID Extraction - Comprehensive Refactoring Plan

## Current State (Program Code)
The system currently extracts **Program Code** (BCH, CAP, R4, THE, PAGE, etc.) which:
- ❌ Is unreliable (gets garbage like "THE", "PAGE")
- ❌ Is not useful for invoice matching
- ❌ Used as grouping key: `(parties, program_code)`

## Target State (Document ID)
Replace with **Document ID** extraction that:
- ✅ Extracts `(type, id)` tuples: `("PURCHASE_ORDER", "2151002393")` or `("AGREEMENT", "JP0094")` or `("SOW", "BCH-2021")`
- ✅ From **content only** (no filename parsing)
- ✅ With clear semantic meaning
- ✅ Used as grouping key: `(parties, document_type, document_id)` instead of `(parties, program_code)`

---

## Implementation Plan

### Phase 1: Method Replacement
**What:** Replace `_extract_program_code_from_content()` with `_extract_document_id_from_content()`

**Old Method Returns:**
```python
"program_code": "BCH"  # or None, or garbage like "THE"
```

**New Method Returns:**
```python
"document_id": {
    "type": "PURCHASE_ORDER",  # or "AGREEMENT" or "SOW" or None
    "id": "2151002393"          # or None
}
```

**Extraction Logic (Content Only):**
1. Priority 1: Purchase Order patterns
   - "Purchase Order No. 2151002393" → `{"type": "PURCHASE_ORDER", "id": "2151002393"}`
   - "PO# 12345" → `{"type": "PURCHASE_ORDER", "id": "12345"}`
   - "Order No. ABC123" → `{"type": "PURCHASE_ORDER", "id": "ABC123"}`

2. Priority 2: Agreement/Contract patterns
   - "Agreement No. JP0094" → `{"type": "AGREEMENT", "id": "JP0094"}`
   - "Contract ID: CLMS-1234" → `{"type": "AGREEMENT", "id": "CLMS-1234"}`

3. Priority 3: SOW patterns
   - "SOW No. BCH-2021" → `{"type": "SOW", "id": "BCH-2021"}`
   - "Statement of Work ID: SOW-001" → `{"type": "SOW", "id": "SOW-001"}`

---

### Phase 2: Data Structure Updates
**What:** Update all data structures that reference `program_code`

**Locations to Update:**

1. **In `discover_contracts()`:**
   ```python
   # BEFORE:
   program_code = self._extract_program_code_from_content(content)
   contract = {..., "program_code": program_code, ...}
   
   # AFTER:
   document_id = self._extract_document_id_from_content(content)
   contract = {..., "document_id": document_id, ...}
   ```

2. **In `discover_contracts()` - Grouping Logic:**
   ```python
   # BEFORE:
   for contract in self.contracts:
       parties = contract["parties"]
       code = contract["program_code"]
       key = (parties, code)
       self.grouped_contracts[key] = [...]
   
   # AFTER:
   for contract in self.contracts:
       parties = contract["parties"]
       doc_id = contract["document_id"]
       # key now has 3 components: (parties, doc_type, doc_id)
       key = (parties, doc_id["type"], doc_id["id"])
       self.grouped_contracts[key] = [...]
   ```

3. **In `discover_contracts()` - Return Statement:**
   ```python
   # BEFORE:
   "contract_id": f"{'_'.join(parties)}_{code or 'UNKNOWN'}"
   "program_code": code
   
   # AFTER:
   "contract_id": f"{'_'.join(parties)}_{doc_id['type'] or 'UNKNOWN'}_{doc_id['id'] or 'UNKNOWN'}"
   "document_id": doc_id  # Include full dict
   ```

4. **In `_build_hierarchy()`:**
   ```python
   # BEFORE:
   for (parties_tuple, code), contracts in self.grouped_contracts.items():
       "program_code": code
   
   # AFTER:
   for (parties_tuple, doc_type, doc_id), contracts in self.grouped_contracts.items():
       "document_id": {"type": doc_type, "id": doc_id}
   ```

---

### Phase 3: Logging & Display Updates
**What:** Update all output/logging to show document ID instead of program code

**Locations to Update:**

1. **In `discover_contracts()` - Log Line:**
   ```python
   # BEFORE:
   self.logger.info(f"✓ {filepath.name}: {len(parties)} parties, code={program_code}, type={doc_types}")
   
   # AFTER:
   self.logger.info(f"✓ {filepath.name}: {len(parties)} parties, doc_id={document_id['type']}:{document_id['id']}, type={doc_types}")
   ```

2. **In Test Cells - Output Display:**
   ```python
   # BEFORE:
   f"  Program Code:   {contract['program_code']}"
   
   # AFTER:
   f"  Document ID:    Type={contract['document_id']['type']}, ID={contract['document_id']['id']}"
   ```

---

## Complete List of Changes Required

| Location | Change | Impact |
|----------|--------|--------|
| `_extract_program_code_from_content()` | Replace with `_extract_document_id_from_content()` | Method definition |
| `discover_contracts()` - line ~150 | Change extraction call | Extract call |
| `discover_contracts()` - line ~155 | Change contract dict key | Data structure |
| `discover_contracts()` - line ~165 | Update logging | Logging |
| `discover_contracts()` - line ~180 | Update grouping loop | Grouping logic |
| `discover_contracts()` - line ~190 | Update grouping key | Grouping key |
| `discover_contracts()` - return dict | Update contract_id & add document_id | Return structure |
| `_build_hierarchy()` - line ~320 | Update loop unpacking | Loop structure |
| `_build_hierarchy()` - line ~330 | Update hierarchy dict | Hierarchy structure |
| Test cell - display loop | Update console output | Display |

---

## Expected Results After Changes

### Before:
```
✓ Purchase Order No. 2151002393.pdf: 2 parties, code=None, type=['PURCHASE_ORDER']
✓ r4 SOW for BCH CAP 2021 12 10.docx: 34 parties, code=PAGE, type=['SOW']

Document ID:    None
Document ID:    PAGE
```

### After:
```
✓ Purchase Order No. 2151002393.pdf: 2 parties, doc_id=PURCHASE_ORDER:2151002393, type=['PURCHASE_ORDER']
✓ r4 SOW for BCH CAP 2021 12 10.docx: 34 parties, doc_id=SOW:?, type=['SOW']

Document ID:    Type=PURCHASE_ORDER, ID=2151002393
Document ID:    Type=SOW, ID=? (to be determined from content)
```

---

## Testing Strategy

After implementation:
1. Run Phase A test on all 6 demo contracts
2. Verify each contract shows proper document ID type + ID
3. Verify grouping works correctly with new key structure
4. Check that hierarchy is built correctly

---

## Risk Assessment

**Low Risk:** Changes are isolated to one class (`ContractRelationshipDiscoverer`)
**Medium Risk:** Multiple locations need updates (need to verify all are found and updated)
**Mitigation:** Complete list provided above; verify each change; test end-to-end

