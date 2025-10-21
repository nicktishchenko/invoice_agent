# Claude Development Guidelines - RAG LLM Invoice Processing Project

**Project:** Invoice Processing Agent with RAG (Retrieval-Augmented Generation)  
**Author:** r4 Technologies, Inc 2025  
**Last Updated:** October 2025

---

## 1. PROJECT OVERVIEW

### Core Purpose
This project implements an AI Agent system for:
- **Rule Extraction**: Extract invoice processing rules from contract documents using RAG
- **Invoice Processing**: Validate and process invoices against extracted rules
- **Local LLM Integration**: Uses Ollama (gemma3:270m) for local processing (no API keys)
- **Vector Store**: FAISS for fast semantic search and retrieval

### Key Files
- `Invoice_Processing_Agent.ipynb` - Main notebook (RAG-powered agent)
- `Invoice_Rule_Extractor.ipynb` - Rule extraction workflow
- `Invoice_Processing_Agent_wip.ipynb` - Work in progress version
- `Chapter_02-04/` - RAG pipeline examples and tutorials
- `data/contracts/` - Sample contract documents for testing

### Technology Stack
- **LLM Framework**: LangChain (0.3.1)
- **Local LLM**: Ollama with gemma3:270m
- **Embeddings**: nomic-embed-text (768 dimensions)
- **Vector Store**: FAISS (CPU)
- **Document Processing**: pdfplumber, python-docx, pytesseract
- **OCR**: pytesseract (requires Tesseract binary)
- **Python**: 3.11+

---

## 2. GIT WORKFLOW

### Branch Strategy
```
main (production)
├── feature/invoice-validation
├── feature/rag-optimization
├── feature/ocr-improvements
└── bugfix/embedding-errors
```

### Commit Standards
- **Feature commits**: `feat: [description]` - New features or functionality
- **Bug fixes**: `fix: [description]` - Bug corrections
- **Documentation**: `docs: [description]` - README, comments, guides
- **Refactoring**: `refactor: [description]` - Code improvements without behavior change
- **Tests**: `test: [description]` - Test additions or modifications
- **Performance**: `perf: [description]` - Performance optimizations

### Workflow Rules
- ✅ **ALWAYS** create feature branches for changes
- ✅ **ALWAYS** commit frequently with descriptive messages
- ✅ **NEVER** push directly to main branch
- ✅ **ALWAYS** add and commit automatically when tasks complete
- ✅ Use pull requests for code review before merging

### Example Workflow
```bash
# Create feature branch
git checkout -b feature/improve-rag-retrieval

# Make changes, test, commit frequently
git add .
git commit -m "feat: implement adaptive k-value for retriever"

# Push to feature branch
git push origin feature/improve-rag-retrieval

# Create PR and merge after review
```

---

## 3. CODE QUALITY STANDARDS (CRITICAL)

### IDE Diagnostics (MUST NOT SKIP)
- ✅ **ALWAYS** run IDE diagnostics after editing files
- ✅ **ALWAYS** fix all linting errors before completing tasks
- ✅ **ALWAYS** fix all type errors before completing tasks
- ✅ **ALWAYS** verify no warnings in output

### Python Code Standards
- **Style**: PEP 8 compliance
- **Type Hints**: Use type annotations for all functions
  ```python
  def extract_rules(self, text: str) -> Dict[str, str]:
      """Extract invoice rules from text."""
      pass
  ```
- **Docstrings**: Google-style docstrings for all classes and functions
  ```python
  def parse_document(self, file_path: str) -> str:
      """
      Parse contract document and extract text.
      
      Args:
          file_path (str): Path to PDF or DOCX file.
          
      Returns:
          str: Extracted text content.
          
      Raises:
          FileNotFoundError: If file doesn't exist.
          ValueError: If file format not supported.
      """
  ```
- **Error Handling**: Use specific exceptions, not bare `except`
  ```python
  try:
      # code
  except FileNotFoundError as e:
      logger.error(f"File not found: {e}")
      raise
  except ValueError as e:
      logger.error(f"Invalid value: {e}")
      raise
  ```

### Logging Standards
- Use `logging` module, not `print()` for production code
- Log levels:
  - `INFO`: Normal operation milestones
  - `WARNING`: Recoverable issues
  - `ERROR`: Errors that need attention
  - `DEBUG`: Detailed diagnostic info

### Notebook Standards
- Each cell should have a descriptive comment: `# Cell X: [Description]`
- Cells should be logically grouped (setup, processing, output)
- Use markdown cells for section headers
- Include error handling with try-except blocks
- Suppress unnecessary warnings and logs

---

## 4. DOCUMENTATION STANDARDS

### README Updates
- Update `README.md` when adding new features
- Include:
  - Feature description
  - Usage examples
  - Configuration requirements
  - Known limitations

### Inline Comments
- Add comments for complex logic (RAG chains, vector operations)
- Explain WHY, not WHAT (code shows what it does)
- Example:
  ```python
  # Adaptive k-value: use fewer retrievals if document has few chunks
  # This prevents empty retrieval results and improves performance
  k_value = min(3, self.num_chunks)
  self.retriever = self.vectorstore.as_retriever(search_kwargs={"k": k_value})
  ```

### API Documentation
- Document all public methods with docstrings
- Include parameter types and return types
- Document exceptions that can be raised

### Change Logs
- Maintain `CHANGELOG.md` with:
  - Version number
  - Date
  - Features added
  - Bugs fixed
  - Breaking changes

### Example CHANGELOG Entry
```markdown
## [2.1.0] - 2025-10-21
### Added
- Adaptive k-value for FAISS retriever based on document size
- Better error handling for OCR failures
- Support for scanned PDF documents

### Fixed
- Fixed embedding timeout issues with large documents
- Corrected vector store initialization for empty documents

### Changed
- Improved performance: reduced embedding requests by 40%
- Updated FAISS to latest version
```

---

## 5. TESTING REQUIREMENTS

### Testing Strategy
- ✅ **ALWAYS** write tests for new features
- ✅ **ALWAYS** run existing tests before completing tasks
- ✅ Focus on end-to-end tests over unit tests
- ✅ Use test-driven development for complex features

### Test Structure
```
tests/
├── test_rule_extraction.py
├── test_invoice_processing.py
├── test_rag_pipeline.py
├── test_document_parsing.py
└── fixtures/
    ├── sample_contract.pdf
    ├── sample_invoice.pdf
    └── expected_rules.json
```

### Test Examples

#### End-to-End Test
```python
def test_invoice_processing_workflow():
    """Test complete invoice processing from contract to validation."""
    # Setup
    agent = InvoiceRuleExtractorAgent()
    contract_path = "tests/fixtures/sample_contract.pdf"
    invoice_path = "tests/fixtures/sample_invoice.pdf"
    
    # Extract rules
    rules = agent.run(contract_path)
    assert len(rules) > 0, "Should extract at least one rule"
    
    # Process invoice
    processor = InvoiceProcessor(rules)
    result = processor.validate_invoice(invoice_path)
    assert result["status"] in ["approved", "rejected", "review"]
    
    # Verify audit trail
    assert len(result["audit_trail"]) > 0
```

#### RAG Pipeline Test
```python
def test_rag_retrieval_accuracy():
    """Test that RAG retriever finds relevant contract sections."""
    agent = InvoiceRuleExtractorAgent()
    agent.parse_document("tests/fixtures/sample_contract.pdf")
    
    # Test retrieval
    query = "What are the payment terms?"
    docs = agent.retriever.get_relevant_documents(query)
    
    assert len(docs) > 0, "Should retrieve relevant documents"
    assert any("payment" in doc.page_content.lower() for doc in docs)
```

### Running Tests
```bash
# Run all tests
pytest tests/

# Run specific test file
pytest tests/test_rule_extraction.py

# Run with coverage
pytest --cov=src tests/

# Run end-to-end tests only
pytest tests/ -m "e2e"
```

### Test Coverage Goals
- Minimum 80% code coverage
- 100% coverage for critical paths (rule extraction, validation)
- All public methods tested

---

## 6. PROJECT STRUCTURE & CONVENTIONS

### Directory Organization
```
rag_llm/
├── Invoice_Processing_Agent.ipynb          # Main notebook
├── Invoice_Rule_Extractor.ipynb            # Rule extraction
├── Chapter_02-04/                          # RAG tutorials
├── data/
│   ├── contracts/                          # Sample contracts
│   ├── invoices/                           # Sample invoices
│   └── extracted_rules.json                # Output rules
├── src/                                    # Python modules (if extracted)
│   ├── agents/
│   │   ├── rule_extractor.py
│   │   └── invoice_processor.py
│   ├── utils/
│   │   ├── document_parser.py
│   │   └── validators.py
│   └── rag/
│       ├── vector_store.py
│       └── retriever.py
├── tests/                                  # Test files
│   ├── test_rule_extraction.py
│   ├── test_invoice_processing.py
│   └── fixtures/
├── requirements.txt                        # Dependencies
├── claude.md                               # This file
├── README.md                               # Project overview
└── CHANGELOG.md                            # Version history
```

### Naming Conventions
- **Classes**: PascalCase (e.g., `InvoiceRuleExtractorAgent`)
- **Functions**: snake_case (e.g., `extract_rules()`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `MAX_CHUNK_SIZE = 800`)
- **Private methods**: Leading underscore (e.g., `_create_vectorstore()`)
- **Files**: snake_case (e.g., `rule_extractor.py`)

---

## 7. RAG PIPELINE SPECIFICS

### Vector Store Configuration
- **Type**: FAISS (CPU)
- **Embeddings**: nomic-embed-text (768 dims)
- **Chunk Size**: 800 characters
- **Chunk Overlap**: 200 characters
- **Retriever k**: Adaptive (min(3, num_chunks))

### RAG Chain Flow
```
Document → Parse → Chunk → Embed → FAISS Index
                                       ↓
Query → Embed → Retrieve → Format → LLM → Output
```

### Performance Optimization
- Use `redirect_stderr()` to suppress Ollama logging
- Limit LLM response length: `num_predict=100`
- Adaptive retriever k-value based on document size
- Batch processing for multiple documents

### Error Handling in RAG
```python
try:
    with redirect_stderr(io.StringIO()):
        vectorstore = FAISS.from_documents(
            documents=splits, 
            embedding=self.embeddings
        )
except Exception as e:
    raise ValueError(f"Failed to create FAISS vector store: {str(e)}")
```

---

## 8. OLLAMA & LOCAL LLM SETUP

### Required Models
```bash
ollama pull gemma3:270m      # LLM for generation
ollama pull nomic-embed-text # Embeddings model
```

### Ollama Configuration
- **Host**: localhost:11434 (default)
- **LLM Model**: gemma3:270m
- **Temperature**: 0 (deterministic)
- **Response Limit**: num_predict=100 (faster generation)
- **Embeddings Dimension**: 768

### Troubleshooting Ollama
```python
# Test connection
from langchain_ollama import OllamaEmbeddings, ChatOllama

embeddings = OllamaEmbeddings(model="nomic-embed-text")
test = embeddings.embed_query("test")
print(f"Embedding dimension: {len(test)}")  # Should be 768

# Test LLM
llm = ChatOllama(model="gemma3:270m", temperature=0)
response = llm.invoke("Hello")
print(response)
```

---

## 9. COMMON ISSUES & SOLUTIONS

### Issue: FAISS Vector Store Creation Fails
**Cause**: Ollama embedding requests timing out or failing  
**Solution**:
1. Restart Ollama: `ollama serve`
2. Reduce chunk size: `chunk_size=500`
3. Use adaptive k-value: `k_value = min(3, num_chunks)`

### Issue: OCR Produces Garbled Text
**Cause**: Low-quality scanned PDF  
**Solution**:
1. Enhance image contrast before OCR
2. Use `--psm 6` config for tesseract
3. Validate with `is_garbled_text()` function

### Issue: Rule Extraction Returns "Not Found"
**Cause**: Query not matching document content  
**Solution**:
1. Check document was parsed correctly
2. Verify vector store has chunks
3. Use broader query terms
4. Increase retriever k-value

### Issue: Embedding Dimension Mismatch
**Cause**: Using different embedding models  
**Solution**: Ensure consistent model: `nomic-embed-text` (768 dims)

---

## 10. DEVELOPMENT WORKFLOW CHECKLIST

### Before Starting a Feature
- [ ] Create feature branch: `git checkout -b feature/[name]`
- [ ] Update `.gitignore` if needed
- [ ] Review existing tests

### During Development
- [ ] Write tests first (TDD approach)
- [ ] Implement feature
- [ ] Run IDE diagnostics
- [ ] Fix all linting/type errors
- [ ] Run all tests: `pytest tests/`
- [ ] Update docstrings
- [ ] Add inline comments for complex logic

### Before Committing
- [ ] Run full test suite
- [ ] Check code coverage
- [ ] Verify no debug prints/logs
- [ ] Format code: `black .` and `isort .`
- [ ] Commit with descriptive message

### Before Pushing
- [ ] Verify all tests pass
- [ ] Update README if needed
- [ ] Update CHANGELOG
- [ ] Create pull request with description

### After Merging
- [ ] Delete feature branch
- [ ] Update version number
- [ ] Create release notes
- [ ] Tag commit: `git tag v2.1.0`

---

## 11. PERFORMANCE TARGETS

### RAG Pipeline
- Rule extraction: < 30 seconds per contract
- Invoice validation: < 5 seconds per invoice
- Vector store creation: < 10 seconds for 50KB document
- Embedding requests: < 2 seconds per batch

### Resource Usage
- Memory: < 2GB for typical operations
- CPU: Efficient with Ollama local processing
- Disk: FAISS index ~100MB per 1000 documents

### Optimization Strategies
1. **Batch Processing**: Process multiple invoices in parallel
2. **Caching**: Cache vector stores for repeated documents
3. **Chunking**: Optimal chunk_size=800, overlap=200
4. **Retriever**: Adaptive k-value based on document size

---

## 12. SECURITY & BEST PRACTICES

### API Keys & Secrets
- ✅ Use environment variables for sensitive data
- ✅ Never commit API keys or credentials
- ✅ Use `.gitignore` to exclude sensitive files
- ✅ Rotate keys regularly

### Data Privacy
- ✅ Don't log sensitive invoice data
- ✅ Sanitize PII before processing
- ✅ Implement audit trails for compliance
- ✅ Use secure file handling

### Error Handling
- ✅ Never expose stack traces to users
- ✅ Log errors for debugging
- ✅ Provide user-friendly error messages
- ✅ Implement graceful degradation

---

## 13. DEPLOYMENT GUIDELINES

### Pre-Deployment Checklist
- [ ] All tests passing
- [ ] Code coverage > 80%
- [ ] No linting errors
- [ ] Documentation updated
- [ ] CHANGELOG updated
- [ ] Version number bumped
- [ ] Performance benchmarks met

### Deployment Steps
1. Create release branch: `git checkout -b release/v2.1.0`
2. Update version numbers
3. Update CHANGELOG
4. Create pull request
5. After approval, merge to main
6. Tag release: `git tag v2.1.0`
7. Push tags: `git push origin --tags`

---

## 14. USEFUL COMMANDS

### Git
```bash
# Create and switch to feature branch
git checkout -b feature/[name]

# Commit with message
git commit -m "feat: [description]"

# Push to remote
git push origin feature/[name]

# Create pull request (GitHub CLI)
gh pr create --title "Feature: [name]" --body "Description"
```

### Testing
```bash
# Run all tests
pytest tests/

# Run with verbose output
pytest tests/ -v

# Run specific test
pytest tests/test_rule_extraction.py::test_extract_payment_terms

# Generate coverage report
pytest --cov=src tests/
```

### Code Quality
```bash
# Format code
black .

# Sort imports
isort .

# Lint
flake8 .

# Type checking
mypy src/
```

---

## 15. RESOURCES & REFERENCES

### Documentation
- [LangChain Docs](https://python.langchain.com/)
- [FAISS Documentation](https://github.com/facebookresearch/faiss)
- [Ollama Guide](https://ollama.ai/)
- [pytesseract](https://github.com/madmaze/pytesseract)

### Learning
- RAG Architecture: See `Chapter_02-04/` notebooks
- Vector Stores: `Chapter_10/CHAPTER10-1_VECTORSTORES.ipynb`
- Retrievers: `Chapter_10/CHAPTER10-2_RETRIEVERS.ipynb`
- Document Loaders: `Chapter_11/CHAPTER11-1_DOCUMENT_LOADERS.ipynb`

### Community
- GitHub Issues: Report bugs and feature requests
- Discussions: Ask questions and share ideas
- Pull Requests: Contribute improvements

---

## 16. VERSION HISTORY

### v2.0 (Current)
- RAG-powered rule extraction with FAISS
- Local Ollama integration (gemma3:270m)
- Improved OCR with pytesseract
- Adaptive retriever k-value

### v1.3
- Initial invoice processing
- Basic rule extraction
- Document parsing

---

**Last Updated**: October 21, 2025  
**Maintained By**: r4 Technologies, Inc  
**Questions?** Check existing issues or create a new one.
