# 12 Invoice Processing Rules - Complete Reference

## Overview

The Invoice Processing Agent extracts **12 comprehensive invoice processing rules** from contracts using RAG (Retrieval-Augmented Generation). These rules are used to validate invoices and determine approval status.

---

## Location in Notebook

**Notebook:** `Demo_Invoice_Processing_Agent.ipynb`  
**Cell:** Cell 9 (Code Cell)  
**Class:** `InvoiceRuleExtractorAgent`  
**Method:** `extract_rules()`  
**Lines:** 818-832 (questions dictionary)

---

## The 12 Rules

### 1. **Payment Terms**
**Question:** "What are the payment terms (Net days, PO requirements)?"  
**Rule Key:** `payment_terms`  
**Type:** `payment_term`  
**Priority:** HIGH  
**Purpose:** Extract Net days (e.g., Net 30, Net 45) and PO requirements  
**Example:** "Net 30 days from invoice receipt"

### 2. **Approval Process**
**Question:** "What is the invoice approval process?"  
**Rule Key:** `approval_process`  
**Type:** `approval`  
**Priority:** MEDIUM  
**Purpose:** Define who must approve and within what timeframe  
**Example:** "Invoice must be approved by project manager within 5 business days"

### 3. **Late Payment Penalties**
**Question:** "What are the late payment penalties?"  
**Rule Key:** `late_penalties`  
**Type:** `penalty`  
**Priority:** HIGH  
**Purpose:** Define penalties for overdue invoices  
**Example:** "Late payment penalty: 1.5% per month on overdue amount"

### 4. **Invoice Submission Requirements**
**Question:** "What must be included on every invoice?"  
**Rule Key:** `submission_requirements`  
**Type:** `submission`  
**Priority:** MEDIUM  
**Purpose:** Define mandatory fields and references  
**Example:** "Invoice must reference MSA, SOW, and PO numbers"

### 5. **Dispute Resolution Process**
**Question:** "What is the dispute resolution process?"  
**Rule Key:** `dispute_resolution`  
**Type:** `dispute`  
**Priority:** MEDIUM  
**Purpose:** Define how disputes are handled  
**Example:** "Disputes must be raised within 30 days of invoice receipt"

### 6. **Tax Handling**
**Question:** "How are taxes handled in invoicing?"  
**Rule Key:** `tax_handling`  
**Type:** `tax`  
**Priority:** MEDIUM  
**Purpose:** Define tax treatment and requirements  
**Example:** "Invoices must include tax ID and GST number"

### 7. **Currency Requirements**
**Question:** "What currency requirements are specified?"  
**Rule Key:** `currency_requirements`  
**Type:** `currency`  
**Priority:** MEDIUM  
**Purpose:** Define acceptable currencies  
**Example:** "All invoices must be in USD"

### 8. **Invoice Format Requirements**
**Question:** "What invoice format or structure is required?"  
**Rule Key:** `invoice_format`  
**Type:** `format`  
**Priority:** LOW  
**Purpose:** Define format and structure requirements  
**Example:** "Invoices must be submitted as PDF or Word document"

### 9. **Supporting Documents Required**
**Question:** "What supporting documents are required?"  
**Rule Key:** `supporting_documents`  
**Type:** `documents`  
**Priority:** MEDIUM  
**Purpose:** Define required attachments  
**Example:** "Delivery notes, timesheets, or proof of service required"

### 10. **Delivery/Completion Terms**
**Question:** "What are the delivery or service completion terms?"  
**Rule Key:** `delivery_terms`  
**Type:** `delivery`  
**Priority:** MEDIUM  
**Purpose:** Define delivery or service completion requirements  
**Example:** "Services must be completed within 30 days of SOW start date"

### 11. **Warranty Terms**
**Question:** "What warranty or guarantee terms apply?"  
**Rule Key:** `warranty_terms`  
**Type:** `warranty`  
**Priority:** LOW  
**Purpose:** Define warranty and guarantee obligations  
**Example:** "Supplier warrants services for 90 days post-delivery"

### 12. **Rejection Criteria**
**Question:** "What are the invoice rejection criteria?"  
**Rule Key:** `rejection_criteria`  
**Type:** `rejection`  
**Priority:** HIGH  
**Purpose:** Define reasons for invoice rejection  
**Example:** "Reject if invoice date is after contract end date"

---

## Extraction Process

### Step 1: Question Formulation
Each of the 12 questions is designed to extract a specific aspect of invoice processing rules from the contract.

### Step 2: RAG Retrieval
For each question:
1. Convert question to embedding (using `nomic-embed-text`)
2. Search FAISS vector store for top-3 most relevant contract sections
3. Retrieve relevant text chunks

### Step 3: LLM Generation
1. Send question + retrieved context to Ollama (gemma3:270m)
2. LLM generates answer based on contract context
3. Answer is validated (must be >15 characters and substantive)

### Step 4: Rule Refinement
Raw answers are mapped to structured format:
```json
{
  "rule_id": "payment_terms",
  "type": "payment_term",
  "description": "Raw LLM answer",
  "priority": "high",
  "confidence": "medium"
}
```

---

## Code Implementation

### Location: Cell 9 - InvoiceRuleExtractorAgent.extract_rules()

```python
def extract_rules(self, text: str) -> Dict[str, str]:
    """
    Extract invoice processing rules from contract text using RAG.
    """
    # Comprehensive questions for rule extraction (not limited to 4)
    questions = {
        "payment_terms": "What are the payment terms (Net days, PO requirements)?",
        "approval_process": "What is the invoice approval process?",
        "late_penalties": "What are the late payment penalties?",
        "submission_requirements": "What must be included on every invoice?",
        "dispute_resolution": "What is the dispute resolution process?",
        "tax_handling": "How are taxes handled in invoicing?",
        "currency_requirements": "What currency requirements are specified?",
        "invoice_format": "What invoice format or structure is required?",
        "supporting_documents": "What supporting documents are required?",
        "delivery_terms": "What are the delivery or service completion terms?",
        "warranty_terms": "What warranty or guarantee terms apply?",
        "rejection_criteria": "What are the invoice rejection criteria?",
    }

    raw_rules = {}
    for key, question in questions.items():
        try:
            # Send question to RAG chain
            answer = rag_chain.invoke(question)
            
            # Validate answer
            if answer and len(answer.strip()) > 15 and "not specified" not in answer.lower():
                raw_rules[key] = answer.strip()
            else:
                raw_rules[key] = "Not found"
        except Exception as e:
            raw_rules[key] = "Not found"
    
    return raw_rules
```

---

## Rule Mapping

The extracted rules are mapped to standardized types:

| Rule Key | Type | Priority | Used For |
|----------|------|----------|----------|
| payment_terms | payment_term | HIGH | Due date calculation |
| approval_process | approval | MEDIUM | Approval workflow |
| late_penalties | penalty | HIGH | Penalty calculation |
| submission_requirements | submission | MEDIUM | Field validation |
| dispute_resolution | dispute | MEDIUM | Dispute handling |
| tax_handling | tax | MEDIUM | Tax validation |
| currency_requirements | currency | MEDIUM | Currency validation |
| invoice_format | format | LOW | Format validation |
| supporting_documents | documents | MEDIUM | Document validation |
| delivery_terms | delivery | MEDIUM | Delivery validation |
| warranty_terms | warranty | LOW | Warranty validation |
| rejection_criteria | rejection | HIGH | Rejection logic |

---

## Usage in Invoice Validation

These 12 rules are extracted once per contract and stored in `extracted_rules.json`. They are then used during Phase 2 (Invoice Validation) to:

1. **Validate required fields** (submission_requirements)
2. **Calculate due dates** (payment_terms)
3. **Check for overdue status** (late_penalties)
4. **Apply penalties** (late_penalties)
5. **Determine approval** (approval_process, rejection_criteria)
6. **Generate audit trail** (all rules applied)

---

## Example Output

When rules are extracted from a contract, the output looks like:

```json
{
  "payment_terms": "Net 30 days from invoice receipt",
  "approval_process": "Invoice must be approved by project manager within 5 business days",
  "late_penalties": "Late payment penalty: 1.5% per month on overdue amount",
  "submission_requirements": "Invoice must reference MSA, SOW, and PO numbers",
  "dispute_resolution": "Disputes must be raised within 30 days of invoice receipt",
  "tax_handling": "Invoices must include tax ID and GST number",
  "currency_requirements": "All invoices must be in USD",
  "invoice_format": "Invoices must be submitted as PDF or Word document",
  "supporting_documents": "Delivery notes or proof of service required",
  "delivery_terms": "Services must be completed within 30 days of SOW start date",
  "warranty_terms": "Supplier warrants services for 90 days post-delivery",
  "rejection_criteria": "Reject if invoice date is after contract end date"
}
```

---

## Key Characteristics

✓ **Comprehensive:** Covers all aspects of invoice processing  
✓ **Flexible:** Adapts to different contract types and industries  
✓ **RAG-Powered:** Uses semantic search for accurate extraction  
✓ **Structured:** Output is JSON for easy integration  
✓ **Reusable:** Extracted once, used for all invoices under that contract  
✓ **Deterministic:** Same contract always produces same rules  

---

## Reference

**Notebook:** `/Users/nikolay_tishchenko/Projects/codeium/invoice_agent/Demo_Invoice_Processing_Agent.ipynb`  
**Cell:** Cell 9 (Code)  
**Class:** `InvoiceRuleExtractorAgent`  
**Method:** `extract_rules()`  
**Lines:** 818-832 (questions)

**Related Cells:**
- Cell 9: Rule extraction logic
- Cell 16: Contract processing with rule extraction
- Cell 17: Rule saving to JSON
- Cell 18: Rule display

---

**Last Updated:** October 28, 2025
