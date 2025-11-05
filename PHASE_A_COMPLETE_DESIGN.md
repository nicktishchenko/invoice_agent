# Phase A: Complete Design - Document Discovery, Grouping & Matching

## Table of Contents
1. [Problem Overview](#problem-overview)
2. [Critical Design Principles](#critical-design-principles)
3. [Component 1: Document Type Detection](#component-1-document-type-detection)
4. [Component 2: Document ID Extraction](#component-2-document-id-extraction)
5. [Component 3A: Document Grouping by Relationships](#component-3a-document-grouping-by-relationships)
6. [Component 3B: Invoice Matching to Groups](#component-3b-invoice-matching-to-groups)
7. [Data Flow](#data-flow)

---

## Problem Overview

### What We're Building
A system to process 18 documents (6 contracts + 12 invoices) from `docs/` folder that:
1. Identifies all document types in each file (a file can contain multiple types)
2. Extracts document IDs with semantic meaning
3. Groups related documents together (MSA + SOW + Order Forms)
4. Matches invoices to correct agreement groups

### Original Flaws (All Corrected)

❌ **Flaw 1: Single Type Per File**
- Old: Return "MSA" → MISSES that file also references SOW and ORDER_FORM
- New: Return ["MSA" (primary), "SOW" (ref), "ORDER_FORM" (ref)]

❌ **Flaw 2: Filename-Dependent Detection**
- Old: Look at filename first → "r4 MSA for BCH CAP"
- New: Scan CONTENT first → Filename is last resort

❌ **Flaw 3: Direct ID Match for Grouping**
- Old: MSA "11414-1" matches SOW "11414-1" → WRONG (they have different IDs)
- New: MSA mentions SOW's ID, SOW mentions MSA's ID → CROSS-REFERENCE

❌ **Flaw 4: Garbage ID Extraction**
- Old: "program_code" returns "ANY", "THE", "PAGE" (meaningless substrings)
- New: Type-specific pattern matching extracts ACTUAL document identifiers found in content
  - For INVOICE: Extract what appears after "Invoice No." or "INV-"
  - For MSA: Extract what appears after "MSA" or "Agreement Number"
  - For SOW: Extract what appears after "SOW" or "Statement of Work Number"
  - For PO: Extract what appears after "PO" or "Purchase Order No."
  - **Format varies by document** - could be "11414-1", "INV-001", "2151002393", "JP0094", or other patterns found in actual content

---

## Critical Design Principles

### Principle 1: Content-First, Filename-Last
```
Priority Order:
1. CONTENT KEYWORDS (Primary) → "MASTER SERVICES AGREEMENT" found in text
2. SUPPORTING KEYWORDS (Boost) → "Terms and Conditions", "Services"
3. FILENAME PATTERNS (Fallback) → Only if content is empty
```

### Principle 2: Detect ALL Document Types in File
- Don't stop after finding first type
- Scan for every possible type: INVOICE, MSA, SOW, PO, ORDER_FORM
- Mark each as PRIMARY (if in first 1000 chars) or SECONDARY (referenced)
- Each file can have 1-5 types detected

### Principle 3: Cross-References Are The Link
```
Files belong together if they MENTION each other's IDs:
- MSA says: "SOW 11414-200"
- SOW says: "MSA 11414-1"
→ They reference each other → confidence 0.95

NOT because they have the same ID (they don't).
```

### Principle 4: Hierarchical Document Structure
```
agreement_group:
├─ primary_document: MSA 11414-1
├─ related_documents:
│  ├─ SOW 11414-200 (cross_reference, confidence 0.95)
│  ├─ ORDER_FORM (party+date+naming match, confidence 0.75)
│  └─ PO 2151002393 (cross_reference, confidence 0.95)
└─ key_identifiers: [11414-1, 11414-200, 2151002393]
```

---

## Component 1: Document Type Detection

### Input
- File content (full text extracted from PDF/DOCX)
- Filename (secondary signal only)

### Output
```python
[
    {"type": "MSA", "primary": True, "confidence": 0.99, "evidence": "MASTER SERVICES AGREEMENT"},
    {"type": "SOW", "primary": False, "confidence": 0.85, "evidence": "Statement of Work for Professional Services"},
    {"type": "ORDER_FORM", "primary": False, "confidence": 0.75, "evidence": "mutually agreed upon Product Order Form"}
]
```

### Algorithm

```python
def _detect_all_document_types_in_file(content, filename):
    """
    Detect ALL document types in file.
    CRITICAL: Do NOT stop after finding first type - scan ALL types.
    """
    
    types_found = []
    
    # Define ALL possible types with patterns
    patterns = {
        "INVOICE": {
            "primary_keywords": r"Invoice|INVOICE",
            "supporting_keywords": [r"Invoice No\.|Invoice Number", r"Due Date", r"Amount Due", r"Total Amount"],
            "base_confidence": 0.95
        },
        "MSA": {
            "primary_keywords": r"Master Services Agreement|MASTER SERVICES AGREEMENT|MSA \d+",
            "supporting_keywords": [r"Terms and Conditions", r"Services", r"Effective Date", r"SOW"],
            "base_confidence": 0.90
        },
        "SOW": {
            "primary_keywords": r"Statement of Work|Scope of Work|SOW \d+|SOW\s",
            "supporting_keywords": [r"Implementation", r"Deliverables", r"Timeline", r"scope"],
            "base_confidence": 0.90
        },
        "PURCHASE_ORDER": {
            "primary_keywords": r"Purchase Order|PO \d+|PURCHASE ORDER",
            "supporting_keywords": [r"Item", r"Quantity", r"Unit Price", r"Total Price"],
            "base_confidence": 0.90
        },
        "ORDER_FORM": {
            "primary_keywords": r"Order Form|ORDER FORM|services ordered",
            "supporting_keywords": [r"Services Ordered", r"Term", r"Term Fee"],
            "base_confidence": 0.85
        }
    }
    
    # CRITICAL: Scan ALL types, don't break/return early
    for doc_type, patterns_config in patterns.items():
        if re.search(patterns_config["primary_keywords"], content, re.I):
            # Type found in content
            
            # Determine if PRIMARY or SECONDARY
            # PRIMARY = mentioned in first 1000 chars (is the focus of this file)
            first_section = content[:1000]
            is_primary = bool(re.search(patterns_config["primary_keywords"], first_section, re.I))
            
            # Calculate confidence
            base_conf = 0.95 if is_primary else 0.80
            
            # Boost with supporting keywords
            supporting_count = sum(
                1 for pattern in patterns_config["supporting_keywords"]
                if re.search(pattern, content, re.I)
            )
            confidence = min(base_conf + (supporting_count * 0.02), 0.99)
            
            types_found.append({
                "type": doc_type,
                "primary": is_primary,
                "confidence": confidence,
                "evidence": _extract_evidence(content, doc_type)
            })
    
    # If no types found in content, fall back to filename (lowest priority)
    if not types_found:
        filename_patterns = {
            "MSA": r"MSA|Master.*Service",
            "SOW": r"SOW|Statement.*Work",
            "INVOICE": r"INV|Invoice",
            "PURCHASE_ORDER": r"PO|Purchase.*Order",
            "ORDER_FORM": r"Order.*Form"
        }
        
        for doc_type, pattern in filename_patterns.items():
            if re.search(pattern, filename, re.I):
                types_found.append({
                    "type": doc_type,
                    "primary": True,
                    "confidence": 0.60,  # Lower confidence for filename-only
                    "evidence": f"Filename: {filename}"
                })
    
    # Sort: primary types first, then by confidence
    return sorted(
        types_found,
        key=lambda x: (x["primary"], x["confidence"]),
        reverse=True
    )
```

### Real Example
```
File: "r4 MSA for BCH CAP 2021 12 10.docx"
Content: "MASTER SERVICES AGREEMENT...Statement of Work...Order Form..."

Scan results:
✅ "MASTER SERVICES AGREEMENT" found in first 200 chars → MSA (primary, 0.99)
✅ "Statement of Work" found in middle → SOW (secondary, 0.85)
✅ "Product Order Form" found → ORDER_FORM (secondary, 0.75)
✅ No INVOICE or PO keywords

Return:
[
    {"type": "MSA", "primary": True, "confidence": 0.99},
    {"type": "SOW", "primary": False, "confidence": 0.85},
    {"type": "ORDER_FORM", "primary": False, "confidence": 0.75}
]
```

---

## Component 2: Document ID Extraction

### Input
- File content
- List of document types detected by Component 1

### Output
```python
[
    {"type": "MSA", "primary": True, "id": "11414-1", "confidence": 0.95},
    {"type": "SOW", "primary": False, "id": "11414-200", "confidence": 0.90},
    {"type": "ORDER_FORM", "primary": False, "id": None, "confidence": 0.0}
]
```

### Algorithm

```python
def _extract_document_ids_for_types(content, detected_types):
    """
    Extract ID for EACH type found.
    Type-specific pattern matching to find actual document identifiers in content.
    
    IMPORTANT: Do NOT assume ID format. The patterns below are starting points.
    Different documents will have different ID formats and locations.
    Adjust patterns based on actual document content - IDs can be:
    - "11414-1" (dash-separated numbers)
    - "INV-001" (prefix-number)
    - "2151002393" (pure numbers)
    - "JP0094" (letters+numbers)
    - Or any other format found in actual documents
    
    If a pattern doesn't match, id=None, and that's OK - not all document types
    will have extractable IDs in every file.
    """
    
    ids_found = []
    
    # Type-specific ID patterns (in order of reliability)
    # These are STARTING PATTERNS - adjust based on actual document content
    id_patterns = {
        "INVOICE": [
            r"Invoice\s*(?:No|Number|#)[\s:]*([A-Z0-9\-]+)",
            r"INV-?([0-9A-Z\-]+)",
            r"INVOICE\s+([A-Z0-9\-]+)"
        ],
        "MSA": [
            r"MSA\s+([0-9\-]+)",
            r"(?:Agreement|MSA).*?[Nn]umber[\s:]*([0-9\-]+)",
            r"MSA\s+[Nn]o\.?[\s:]*([0-9\-]+)"
        ],
        "SOW": [
            r"SOW\s+(?:Number|No)[\s:]*([0-9\-]+)",
            r"SOW\s+([0-9\-]+)",
            r"STATEMENT OF WORK.*?(?:Number|No)[\s:]*([0-9\-]+)"
        ],
        "PURCHASE_ORDER": [
            r"(?:Purchase\s+)?Order\s+No\.?[\s:]*([0-9\-]+)",
            r"PO\s+([0-9\-]+)",
            r"Purchase Order.*?Number[\s:]*([0-9\-]+)"
        ],
        "ORDER_FORM": [
            r"Order\s+Form.*?[Nn]umber[\s:]*([0-9\-]+)",
            r"Order\s+Form\s+[Nn]o\.?[\s:]*([0-9\-]+)"
        ]
    }
    
    for type_info in detected_types:
        doc_type = type_info["type"]
        patterns = id_patterns.get(doc_type, [])
        
        doc_id = None
        confidence = 0.0
        
        # Try each pattern for this type
        for pattern in patterns:
            match = re.search(pattern, content, re.I | re.MULTILINE)
            if match:
                doc_id = match.group(1)
                confidence = 0.95 if type_info["primary"] else 0.90
                break
        
        ids_found.append({
            "type": doc_type,
            "primary": type_info["primary"],
            "id": doc_id,
            "confidence": confidence
        })
    
    return ids_found
```

### Real Example
```
File: "r4 MSA for BCH CAP 2021 12 10.docx"
Detected types: [MSA(primary), SOW, ORDER_FORM]

Component 2 extraction:
(Examples shown - actual IDs will vary based on what's found in documents)

- MSA: Scan for patterns "MSA ###", "Agreement Number", etc. 
        → Found in content: "MSA 11414-1" → ID: "11414-1" (0.95)
- SOW: Scan for patterns "SOW ###", "Statement of Work Number", etc.
        → Found in content: "SOW 11414-200" → ID: "11414-200" (0.90)
- ORDER_FORM: Scan for patterns "Order Form Number", etc.
        → Not found in content → ID: None (0.0)

Return:
[
    {"type": "MSA", "id": "11414-1", "confidence": 0.95},
    {"type": "SOW", "id": "11414-200", "confidence": 0.90},
    {"type": "ORDER_FORM", "id": None, "confidence": 0.0}
]

NOTE: Actual IDs extracted will depend on what identifiers exist in the
actual document content. Different document types and sources may use
different ID formats (numbers, dashes, letters, etc.).
```

---

## Component 3A: Document Grouping by Relationships

### Input
- All 18 files processed through Components 1 & 2
- Each file: detected_types[], extracted_ids[], parties[], filename

### Output
```python
{
    "agreements": {
        "agreement_BCH_CAP_2021": {
            "primary_document": {...MSA 11414-1...},
            "related_documents": [
                {"file": {...SOW 11414-200...}, "relationship": "cross_reference", "confidence": 0.95},
                {"file": {...ORDER_FORM...}, "relationship": "party_date_naming", "confidence": 0.75}
            ],
            "key_identifiers": ["11414-1", "11414-200"]
        }
    },
    "invoices": [INV-001, INV-002, ...]
}
```

### Relationship Signals (Corrected)

**Signal 1: Cross-References (PRIMARY - Confidence 0.95)**
```
CORRECT approach (NOT direct ID match):

File 1 (MSA):
- Contains ID: 11414-1
- Content says: "SOW 11414-200" ← REFERENCES other document's ID

File 2 (SOW):
- Contains ID: 11414-200
- Content says: "MSA 11414-1" ← REFERENCES other document's ID

Relationship detected:
✅ MSA mentions SOW's ID (11414-200) in content
✅ SOW mentions MSA's ID (11414-1) in content
→ CROSS-REFERENCE confirmed (confidence 0.95)

Why this works:
- They reference each other's IDs explicitly
- This is semantic connection, not coincidence
- Works even if filenames are different
```

**Signal 2: Party + Date Proximity (SECONDARY - Confidence 0.75)**
```
Check if documents have:
- Same parties (>80% overlap)
- Creation dates within 30 days

Example:
✅ Both list: [Bayer Consumer Health, r4 Technologies, Inc.]
✅ Both dated: 2021-12-10
→ Party+date match (confidence 0.75)

Note: Weaker than cross-reference, but useful confirmation
```

**Signal 3: Naming Pattern (TERTIARY - Confidence 0.60)**
```
Check if filenames share project identifiers:

File 1: "r4 MSA for BCH CAP 2021 12 10.docx"
File 2: "r4 SOW for BCH CAP 2021 12 10.docx"

Common tokens after removing type keywords: {"r4", "BCH", "CAP", "2021", "12", "10"}
→ Naming pattern match (confidence 0.60)

Note: Weakest signal, only for confirmation
```

### Algorithm

```python
def _group_documents_by_relationships(all_documents):
    """
    Group documents by detected relationships.
    
    Process:
    1. Separate invoices from agreements
    2. Find primary documents (marked primary=true in Component 1)
    3. For each primary, find related documents using signals
    4. Create hierarchical groups
    """
    
    invoices = []
    agreement_groups = {}
    grouped_files = set()
    
    # Separate invoices
    invoices = [d for d in all_documents if d["primary_type"] == "INVOICE"]
    non_invoices = [d for d in all_documents if d["primary_type"] != "INVOICE"]
    
    # Find primary documents (cores of agreements)
    primary_docs = [d for d in non_invoices if d["is_primary_in_file"]]
    
    # Group around each primary
    for primary_doc in primary_docs:
        
        if primary_doc["filename"] in grouped_files:
            continue
        
        group_key = _create_group_key(primary_doc)
        related_docs = []
        
        # Scan all other documents for relationships
        for candidate in non_invoices:
            
            if candidate["filename"] in grouped_files or candidate["filename"] == primary_doc["filename"]:
                continue
            
            # SIGNAL 1: Cross-references (PRIMARY)
            if _documents_cross_reference(primary_doc, candidate):
                related_docs.append({
                    "file": candidate,
                    "relationship": "cross_reference",
                    "confidence": 0.95,
                    "evidence": _extract_cross_reference_evidence(primary_doc, candidate)
                })
                grouped_files.add(candidate["filename"])
                continue
            
            # SIGNAL 2+3: Party + Date + Naming (SECONDARY)
            if (_party_and_date_match(primary_doc, candidate) and 
                _naming_pattern_match(primary_doc, candidate)):
                related_docs.append({
                    "file": candidate,
                    "relationship": "party_date_naming",
                    "confidence": 0.75,
                    "evidence": "Same parties, close dates, similar naming"
                })
                grouped_files.add(candidate["filename"])
                continue
        
        # Create group
        agreement_groups[group_key] = {
            "primary_document": primary_doc,
            "related_documents": related_docs,
            "key_identifiers": _collect_all_ids(
                [primary_doc] + [r["file"] for r in related_docs]
            ),
            "parties": primary_doc["parties"]
        }
        
        grouped_files.add(primary_doc["filename"])
    
    return {
        "agreements": agreement_groups,
        "invoices": invoices
    }


def _documents_cross_reference(doc1, doc2):
    """
    Check if documents reference each other's IDs.
    This is the PRIMARY relationship signal.
    """
    
    # Get all IDs extracted from each document
    doc1_ids = [id_obj["id"] for id_obj in doc1["extracted_ids"] if id_obj["id"]]
    doc2_ids = [id_obj["id"] for id_obj in doc2["extracted_ids"] if id_obj["id"]]
    
    # Check: Does doc1's content mention any of doc2's IDs?
    doc1_mentions_doc2 = any(str(id_val) in doc1["content"] for id_val in doc2_ids)
    
    # Check: Does doc2's content mention any of doc1's IDs?
    doc2_mentions_doc1 = any(str(id_val) in doc2["content"] for id_val in doc1_ids)
    
    # Both must mention each other (mutual cross-reference)
    return doc1_mentions_doc2 and doc2_mentions_doc1
```

### Real Example
```
File 1: "r4 MSA for BCH CAP 2021 12 10.docx"
- primary_type: MSA
- extracted_ids: [{type: MSA, id: 11414-1}, {type: SOW, id: 11414-200}, ...]
- content contains: "SOW 11414-200", "Statement of Work"
- parties: [Bayer Consumer Health, r4 Technologies, Inc.]

File 2: "r4 SOW for BCH CAP 2021 12 10.docx"
- primary_type: SOW
- extracted_ids: [{type: SOW, id: 11414-200}, {type: MSA, id: 11414-1}, ...]
- content contains: "MSA 11414-1", "Master Services Agreement"
- parties: [Bayer Consumer Health, r4 Technologies, Inc.]

Grouping process:
1. File 1 is primary (MSA is primary type)
2. Check relationship with File 2:
   - Signal 1 (Cross-ref): File 1 mentions "11414-200" (File 2's ID) ✅
   - Signal 1 (Cross-ref): File 2 mentions "11414-1" (File 1's ID) ✅
   → Relationship: cross_reference (confidence 0.95)
3. File 3 (ORDER_FORM):
   - Signal 1 (Cross-ref): No explicit ID references ❌
   - Signal 2+3: Same parties ✅, Same date ✅, Shared naming "BCH CAP 2021" ✅
   → Relationship: party_date_naming (confidence 0.75)

Result:
agreement_BCH_CAP_2021:
├─ primary: File 1 (MSA 11414-1)
├─ related: File 2 (SOW 11414-200, cross_ref, 0.95)
├─ related: File 3 (ORDER_FORM, party_date_naming, 0.75)
└─ key_identifiers: [11414-1, 11414-200]
```

---

## Component 3B: Invoice Matching to Groups

### Input
- Grouped documents from Component 3A
- Invoice files

### Output
```python
{
    "INV-001.docx": {
        "matched_agreement": "agreement_BCH_CAP_2021",
        "confidence": 0.85,
        "match_reasons": ["party_match", "po_reference"]
    },
    "INV-002.docx": {
        "matched_agreement": "agreement_BCH_CAP_2021",
        "confidence": 0.60,
        "match_reasons": ["party_match"]
    }
}
```

### Matching Signals

| Signal | Weight | Description | Example |
|--------|--------|-------------|---------|
| Party match | 0.40 | Invoice parties match agreement parties | Invoice from "Bayer" → Agreement with "Bayer + r4" |
| PO reference | 0.35 | Invoice mentions PO number in agreement group | Invoice says "PO 2151002393" → Found in group's POs |
| Agreement ID | 0.20 | Invoice mentions MSA or SOW ID | Invoice says "SOW 11414-200" → Found in group |
| Naming pattern | 0.05 | Filename patterns match | "BCH" in invoice → "BCH" in agreement group |

### Algorithm

```python
def _match_invoices_to_agreement_groups(grouped_docs):
    """
    Match invoices to agreement groups.
    
    Strategy: Use multiple signals to find the best matching agreement group.
    Signals accumulate confidence (max 0.99).
    Threshold: Only report matches with confidence >= 0.40
    """
    
    matches = {}
    
    for invoice in grouped_docs["invoices"]:
        best_match = None
        best_score = 0
        
        for group_key, agreement_group in grouped_docs["agreements"].items():
            score = 0
            reasons = []
            
            # Signal 1: Party match (0.40)
            # Highest confidence - parties are rarely shared across unrelated agreements
            if _parties_match(invoice["parties"], agreement_group["parties"]):
                score += 0.40
                reasons.append("party_match")
            
            # Signal 2: PO/Document ID reference (0.35)
            # Invoice mentions a PO, MSA, or SOW ID that exists in this group
            invoice_refs = _extract_document_references(invoice["content"])
            group_ids = agreement_group["key_identifiers"]  # All IDs from all docs in group
            if invoice_refs and group_ids and (invoice_refs & set(group_ids)):
                score += 0.35
                reasons.append("document_id_reference")
            
            # Signal 3: Agreement ID reference (0.20)
            # Invoice explicitly mentions an agreement/MSA/SOW ID
            agreement_refs = _extract_agreement_ids(invoice["content"])
            group_agreement_ids = [id for id in agreement_group["key_identifiers"]
                                   if id]  # Any ID found in group
            if agreement_refs and group_agreement_ids and (agreement_refs & set(group_agreement_ids)):
                score += 0.20
                reasons.append("agreement_id_reference")
            
            # Signal 4: Naming pattern (0.05)
            # Filename patterns suggest same project (weakest signal)
            if _naming_pattern_match(invoice["filename"], group_key):
                score += 0.05
                reasons.append("naming_pattern")
            
            if score > best_score:
                best_score = score
                best_match = {
                    "group_key": group_key,
                    "confidence": min(score, 0.99),
                    "match_reasons": reasons
                }
        
        if best_match and best_match["confidence"] >= 0.40:  # Minimum threshold
            matches[invoice["filename"]] = best_match
    
    return matches
```

---

## Data Flow

```
┌─────────────────────────────────────┐
│ 18 Files (6 contracts + 12 invoices)│
│ docs/demo_contracts/                │
│ docs/demo_invoices/                 │
└────────────────┬────────────────────┘
                 │
                 ▼
    ┌────────────────────────────────┐
    │ COMPONENT 1:                   │
    │ Detect ALL Document Types      │
    │                                │
    │ Input: File content + filename │
    │ Output: [type1, type2, ...]    │
    │         with primary flag      │
    └────────────────┬───────────────┘
                     │
    Each file processed:
    ✅ MSA file → [MSA(p), SOW(ref), ORDER_FORM(ref)]
    ✅ SOW file → [SOW(p), MSA(ref), ORDER_FORM(ref)]
    ✅ INV file → [INVOICE(p)]
                     │
                     ▼
    ┌────────────────────────────────┐
    │ COMPONENT 2:                   │
    │ Extract Document IDs           │
    │                                │
    │ Input: Content + Type list     │
    │ Output: [{type, id, conf}, ...] │
    └────────────────┬───────────────┘
                     │
    Each file processed:
    ✅ MSA → [{MSA: 11414-1}, {SOW: 11414-200}]
    ✅ SOW → [{SOW: 11414-200}, {MSA: 11414-1}]
    ✅ INV → [{INVOICE: INV-001}]
                     │
                     ▼
    ┌────────────────────────────────┐
    │ COMPONENT 3A:                  │
    │ Group by Relationships         │
    │                                │
    │ Input: All files + extracted   │
    │ Output: Hierarchical groups    │
    └────────────────┬───────────────┘
                     │
    Groups formed:
    ✅ agreement_BCH_CAP_2021
       ├─ MSA 11414-1 (primary)
       ├─ SOW 11414-200 (cross_ref, 0.95)
       └─ ORDER_FORM (party_date, 0.75)
    ✅ agreement_JP0094
       └─ ...
    ✅ invoices: [INV-001, ..., INV-012]
                     │
                     ▼
    ┌────────────────────────────────┐
    │ COMPONENT 3B:                  │
    │ Match Invoices to Groups       │
    │                                │
    │ Input: Groups + invoices       │
    │ Output: Matches with scores    │
    └────────────────────────────────┘
                     │
    Matches created:
    ✅ INV-001 → agreement_BCH_CAP_2021 (0.85)
    ✅ INV-002 → agreement_BCH_CAP_2021 (0.60)
    ✅ ...
```

---

## Success Criteria

✅ **Component 1: Type Detection**
- MSA file returns [MSA, SOW, ORDER_FORM] - NOT just MSA
- SOW file returns [SOW, MSA, ORDER_FORM] - captures all types
- Filename never influences detection if content matches

✅ **Component 2: ID Extraction**
- MSA ID: "11414-1" extracted correctly
- SOW ID: "11414-200" extracted correctly
- Invoice IDs: "INV-001" through "INV-012" extracted

✅ **Component 3A: Grouping**
- MSA and SOW grouped together via cross-reference signal
- Cross-reference confidence: 0.95 (mutual mention of IDs)
- ORDER_FORM grouped via party+date+naming signals
- Result: 2-3 agreement groups formed

✅ **Component 3B: Invoice Matching**
- All 12 invoices matched to correct agreement groups
- Confidence scores reflect match strength
- PO references correctly link invoices to agreement POs

---

## Implementation Notes

### Code Structure in Notebook
```
Cell N: class ContractRelationshipDiscoverer
  ├─ __init__()
  ├─ discover_contracts()  [MAIN ENTRY POINT]
  ├─ _detect_all_document_types_in_file()  [Component 1]
  ├─ _extract_document_ids_for_types()  [Component 2]
  ├─ _group_documents_by_relationships()  [Component 3A]
  ├─ _match_invoices_to_agreement_groups()  [Component 3B]
  └─ [Helper methods]
```

### Path Configuration
```python
DOCS_DIR = Path("docs")
CONTRACTS_DIR = DOCS_DIR / "demo_contracts"
INVOICES_DIR = DOCS_DIR / "demo_invoices"
```

### Testing Strategy
1. Test Component 1 on 1 file (MSA) → Verify all 3 types detected
2. Test Component 2 on same file → Verify IDs extracted
3. Test Component 3A on 3 files (MSA, SOW, ORDER_FORM) → Verify grouping
4. Test Component 3B on all 18 files → Verify invoice matching
5. Full end-to-end test on all 18 files

