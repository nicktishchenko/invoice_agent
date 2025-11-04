# Local Python Pipeline for Extracting Contract Parties and Document Types

## Requirements & Security
- **All processing is 100% local, no API calls, no data leaves your machines.**
- Suitable for sensitive contract/legal documents — GDPR/CCPA/HIPAA compliant when properly operated.

---

## Architecture Overview
1. **Text Extraction**: Extract plain text from PDF/DOCX/DOC using offline libraries.
2. **Entity Recognition** (NER): Extract party names using spaCy custom models or local LLMs.
3. **Document Classification**: Identify contract type (MSA, SOW, Purchase Order, etc.) via offline ML (scikit-learn) or local LLMs.
4. **Pattern/Rule Extraction**: Direct regex for quick prototypes.

---

## 1. Text Extraction Libraries

### For PDFs:
- **PyMuPDF (fitz)**: Fast, robust for digital PDFs
- **Docling**: Advanced PDF/document support, table-aware, works offline

### For DOCX/DOC:
- **python-docx**: Widely-used, pure offline
- **Docling**: Also supports DOC legacy files

### Universal (All formats):
- **Apache Tika**: Handles 1500+ formats, Java dependency, works offline

#### Example (Python):
```python
import pymupdf
from docx import Document
from docling.document_converter import DocumentConverter

def extract_text_from_file(file_path):
    ext = Path(file_path).suffix.lower()
    if ext == '.pdf':
        doc = pymupdf.open(str(file_path))
        text = ''.join(page.get_text() for page in doc)
        doc.close()
        return text
    elif ext == '.docx':
        doc = Document(str(file_path))
        return '\n'.join(p.text for p in doc.paragraphs)
    elif ext == '.doc':
        converter = DocumentConverter()
        result = converter.convert(str(file_path))
        return result.document.export_to_markdown()
```

---

## 2. Local NER (Party Extraction with spaCy)

### Training Custom NER (100% local, no cloud)
```python
import spacy
from spacy.training import Example
from spacy.util import minibatch

TRAIN_DATA = [
    ("This Master Service Agreement is entered into between Acme Corporation and TechVendor LLC.",
     {"entities": [(53, 69, "PARTY1"), (74, 89, "PARTY2")]}),
    # ... add more examples ...
]

def train_local_ner_model(training_data, output_dir):
    nlp = spacy.load("en_core_web_lg")
    ner = nlp.get_pipe("ner")
    for label in ["PARTY1", "PARTY2", "SUPPLIER", "CLIENT"]:
        ner.add_label(label)
    other_pipes = [p for p in nlp.pipe_names if p != "ner"]
    with nlp.disable_pipes(*other_pipes):
        optimizer = nlp.resume_training()
        for _ in range(30):
            random.shuffle(training_data)
            for batch in minibatch(training_data, size=8):
                examples = [Example.from_dict(nlp.make_doc(text), ann) for text, ann in batch]
                nlp.update(examples, drop=0.3, losses={})
    nlp.to_disk(output_dir)
```

#### Usage
```python
nlp = spacy.load('./models/contract_ner')
doc = nlp(input_text)
parties = [ent.text for ent in doc.ents if ent.label_ in ['PARTY1', 'PARTY2', 'ORG', 'PERSON']]
```

---

## 3. Document Classification (Offline)

### Option A: Traditional ML Model
```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.svm import LinearSVC
from sklearn.pipeline import Pipeline
import pickle

def train_local_classifier(texts, labels, save_path):
    clf = Pipeline([
        ('tfidf', TfidfVectorizer(max_features=5000, ngram_range=(1,3), min_df=2)),
        ('svc', LinearSVC())
    ])
    clf.fit(texts, labels)
    with open(save_path, 'wb') as f:
        pickle.dump(clf, f)
    return clf

def classify_document_local(text, clf_path):
    with open(clf_path, 'rb') as f:
        clf = pickle.load(f)
    return clf.predict([text])[0]
```

### Option B: Pattern-Based (Rule Extraction)
```python
import re
def classify_by_local_patterns(text):
    patterns = {
        'MSA': [r'master service agreement', r'\bm.?s.?a.?\b'],
        'SOW': [r'statement of work', r'\bs.?o.?w.?\b'],
        'Purchase Order': [r'purchase order', r'\bp.?o.?\b'],
        # ... more ...
    }
    text_lower = text.lower()
    for dtype, regexes in patterns.items():
        for pat in regexes:
            if re.search(pat, text_lower):
                return dtype
    return 'Unknown'
```

---

## 4. Local LLM Option (Optional)

### Using Ollama for Llama 3 / Mistral
- Download: https://ollama.com/
- After one-time setup, run:
    - `ollama pull llama3.2:3b`
- Example (Python):
```python
import ollama
prompt = "Extract parties and contract type: ...text..."
response = ollama.generate(
    model="llama3.2:3b",
    prompt=prompt,
    options={"temperature": 0.1}
)
print(response['response'])
```

---

## 5. Full Local Pipeline: Example Class

```python
from pathlib import Path
class SecureLocalContractExtractor:
    def __init__(self, spacy_model, clf_path):
        import spacy, pickle
        self.nlp = spacy.load(spacy_model)
        with open(clf_path, 'rb') as f:
            self.clf = pickle.load(f)
    def process(self, file_path):
        text = extract_text_from_file(file_path)
        doc_type = classify_document_local(text, self.clf)
        parties = [ent.text for ent in self.nlp(text).ents if ent.label_ in ['ORG','PERSON','PARTY1','PARTY2']]
        return {'file': file_path, 'doc_type': doc_type, 'parties': parties, 'text_len': len(text)}
```

---

## Security Benefits
- No cloud, API, or external I/O required
- All models run on hardware you control
- GDPR/CCPA/HIPAA/data-protection compliant

---

## References & Further Reading
- [spaCy Offline Training Guide](https://spacy.io/usage/training)
- [Ollama LLMs](https://ollama.com)
- [Docling project](https://github.com/docling-project/docling)
- [Pattern approaches (scikit-learn, regex)](https://scikit-learn.org/)
