# Requirements Summary

## Overview

The `requirements.txt` file contains all Python packages used by the three invoice processing notebooks:

- **Demo_Invoice_Processing_Agent.ipynb** - Main demo with contract and invoice processing
- **Async_Contract_Invoice_Agent.ipynb** - Asynchronous processing with database support
- **Invoice_Processing_Agent.ipynb** - Core invoice validation system

---

## Python Packages (18 total)

### Core Dependencies
- **numpy==1.26.4** - Numerical computing
- **pandas>=2.0.0** - Data manipulation and analysis

### Document Processing
- **pdfplumber>=0.11.0** - PDF text extraction
- **python-docx>=1.1.0** - Word document processing
- **Pillow>=10.0.0** - Image processing
- **reportlab>=4.0.0** - PDF generation
- **matplotlib>=3.8.0** - Data visualization

### OCR (Optical Character Recognition)
- **pytesseract>=0.3.10** - Python wrapper for Tesseract OCR

### LangChain and RAG (Retrieval-Augmented Generation)
- **langchain==0.3.1** - LLM framework
- **langchain-core==0.3.6** - Core LangChain components
- **langchain-community==0.3.1** - Community integrations
- **langchain-ollama==0.2.0** - Ollama integration

### Vector Store
- **faiss-cpu>=1.8.0** - Fast similarity search (CPU version)

### Database (Async Agent)
- **sqlalchemy>=2.0.0** - SQL toolkit and ORM
- **psycopg2-binary>=2.9.0** - PostgreSQL adapter

### Utilities
- **beautifulsoup4>=4.12.0** - HTML/XML parsing
- **ipywidgets>=8.1.0** - Jupyter interactive widgets
- **pydantic==2.9.2** - Data validation

---

## External Requirements (NOT pip installable)

### 1. Ollama (Local LLM Runtime)
**Download:** https://ollama.ai

**Required Models:**
```bash
ollama pull gemma3:270m
ollama pull nomic-embed-text
```

**Start Ollama:**
```bash
ollama serve
```

### 2. Tesseract OCR Binary

**macOS:**
```bash
brew install tesseract
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt-get install tesseract-ocr
```

**Windows:**
Download installer from: https://github.com/UB-Mannheim/tesseract/wiki

---

## Installation Steps

### Step 1: Install Python Packages
```bash
pip install -r requirements.txt
```

### Step 2: Install Ollama
Visit https://ollama.ai and follow platform-specific instructions

### Step 3: Pull Ollama Models
```bash
ollama pull gemma3:270m
ollama pull nomic-embed-text
```

### Step 4: Install Tesseract OCR
- **macOS:** `brew install tesseract`
- **Linux:** `sudo apt-get install tesseract-ocr`
- **Windows:** Download from https://github.com/UB-Mannheim/tesseract/wiki

### Step 5: Start Ollama
```bash
ollama serve
```

### Step 6: Run Notebooks
```bash
jupyter notebook
```

---

## Package Breakdown by Notebook

### Demo_Invoice_Processing_Agent.ipynb
- Core: numpy, pandas
- Document: pdfplumber, python-docx, Pillow, reportlab, matplotlib
- OCR: pytesseract
- RAG: langchain, langchain-core, langchain-community, langchain-ollama
- Vector: faiss-cpu
- Utilities: beautifulsoup4, ipywidgets, pydantic

### Async_Contract_Invoice_Agent.ipynb
- All of above PLUS:
- Database: sqlalchemy, psycopg2-binary

### Invoice_Processing_Agent.ipynb
- All of above (core invoice validation system)

---

## Version Notes

- **Python:** 3.10+ required
- **numpy:** Pinned to 1.26.4 for compatibility
- **LangChain:** Pinned to 0.3.1 for stability
- **pydantic:** Pinned to 2.9.2 for compatibility

---

## Troubleshooting

### Ollama Connection Error
- Ensure Ollama is running: `ollama serve`
- Check models are installed: `ollama list`
- Should see: `gemma3:270m` and `nomic-embed-text`

### Tesseract Not Found
- Verify installation: `tesseract --version`
- macOS: `which tesseract`
- Linux: `which tesseract`
- Windows: Add to PATH if needed

### Import Errors
- Reinstall packages: `pip install --upgrade -r requirements.txt`
- Check Python version: `python --version` (should be 3.10+)

---

**Last Updated:** October 28, 2025
