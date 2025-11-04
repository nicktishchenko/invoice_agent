# Party Extraction: Alternative Approaches

## Problem Statement
Current regex approach `r"\b([A-Za-z][A-Za-z0-9\s&\-]{2,40}),\s*(?:Ltd\.?|Inc\.?|Corp\.?|LLC|Limited)\b"` works for structured "Company Name, Suffix" format but:
- Misses companies without explicit suffix
- Matches garbage (addresses, fragments)
- Requires exact format

---

## Five Alternative Approaches

### 1️⃣ **Suffix-Based** (Current)
**Pattern:** `"Name, Suffix"`
```regex
\b([A-Za-z][A-Za-z0-9\s&\-]{2,40}),\s*(?:Ltd\.?|Inc\.?|Corp\.?|LLC|Limited)\b
```

**Pros:**
- Fast, deterministic
- High precision (low false positives)
- Works for formal contracts

**Cons:**
- Only matches "Company, Ltd" format
- Misses companies without suffix
- False positives on address fragments

**Best for:** Structured business documents (POs, MSAs)

---

### 2️⃣ **Frequency-Based**
**Theory:** Real company names appear multiple times; random text doesn't

**Implementation:**
1. Extract all capitalized multi-word phrases
2. Count occurrences
3. Keep phrases appearing 2+ times
4. Filter by length (5-80 chars)

**Pros:**
- No format requirement
- Self-adaptive (learns from document)
- Handles any company name format

**Cons:**
- Requires multi-page or repeated mentions
- Fails on single-mention parties
- Picks up common legal terms

**Best for:** Long contracts with multiple references

---

### 3️⃣ **Positional**
**Theory:** Companies appear in predictable locations:
- Page headers/titles
- "Between..." sections
- Signature blocks
- Opening paragraphs

**Implementation:**
1. Extract first/last N lines
2. Look for company patterns in those regions
3. Prioritize by position

**Pros:**
- Exploits document structure
- Fast (process fraction of doc)
- Intuitive

**Cons:**
- Format-dependent
- Varies by document type
- Fragile

**Best for:** POs, invoices (structured layouts)

---

### 4️⃣ **Context-Based**
**Theory:** Companies appear near specific semantic markers

**Patterns:**
- `"[Name], a [type]"` → "Company X, a corporation"
- `"between [X] and [Y]"` → "between Company X and Company Y"
- `"party is [Name]"` → "the party is Company LLC"
- `"For: [Name]"` → signature blocks

**Pros:**
- Semantic understanding
- Handles varied formats
- Lower false positive rate

**Cons:**
- Multiple patterns to maintain
- Requires good regex
- Context-dependent

**Best for:** Legal documents, contracts

---

### 5️⃣ **Named-Entity-Like**
**Theory:** Extract all proper noun sequences, then validate

**Implementation:**
1. Find all capitalized word sequences (2-5 words)
2. Count occurrences
3. Keep those appearing 2+ times (strong signal)
4. Filter out common English words/phrases

**Pros:**
- Comprehensive (catches all formats)
- Data-driven filtering
- Flexible

**Cons:**
- May need NER for best results
- Still requires filtering
- Computational cost

**Best for:** Diverse document types

---

## Recommended Hybrid Approach

Combine multiple strategies with priority:

```python
def extract_parties_hybrid(content):
    parties = set()
    
    # Step 1: High confidence (Suffix-Based)
    suffix_matches = extract_by_suffix(content)
    parties.update(suffix_matches)
    
    # Step 2: If few results, use Context
    if len(parties) < 2:
        context_matches = extract_by_context(content)
        parties.update(context_matches)
    
    # Step 3: Validate with Frequency
    if len(parties) < 2:
        freq_matches = extract_by_frequency(content)
        parties.update(freq_matches)
    
    return list(parties)
```

**Benefits:**
- ✅ Fast (suffix-based first)
- ✅ Flexible (fallback strategies)
- ✅ Reliable (multi-layer validation)
- ✅ No hardcoding (patterns only)

---

## Implementation Priority

1. **Now:** Keep Approach 1 (suffix) as primary
2. **Next:** Add Approach 4 (context patterns)
3. **Later:** Add Approach 2 (frequency) as fallback
4. **Optional:** Add Approach 5 (NER-style) for robust cases

---

## Testing Strategy

For each approach, measure:
- **Precision:** How many extracted names are real?
- **Recall:** How many real companies are found?
- **F1-Score:** Balance of precision + recall

```
Suffix-Based:     Precision: High, Recall: Low
Context-Based:    Precision: Medium, Recall: Medium
Frequency-Based:  Precision: Low, Recall: High
Hybrid:           Precision: High, Recall: High
```

---

## Next Steps

Which approach interests you most?
- Stick with Approach 1 + add context validation?
- Implement Approach 4 (context patterns)?
- Combine Approaches 1+4+2 (hybrid)?
- Explore something completely different?
