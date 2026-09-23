# Document Loaders in RAG Pipelines

Document loaders are the ingestion engines of a Retrieval-Augmented Generation (RAG) system. Before an AI can search your private data, that data must be translated into plain text. Loaders fetch raw, unstructured, or semi-structured data from various sources (PDFs, websites, APIs) and convert it into a standardized format for Large Language Models (LLMs).

## The RAG Pipeline
1. **Ingestion (Document Loaders):** Extract text and metadata from the source.
2. **Chunking (Text Splitters):** Break text into manageable pieces.
3. **Embedding:** Convert chunks into numerical vectors.
4. **Storage:** Save vectors in a database for retrieval.

A good document loader extracts both **Page Content** (the raw text) and **Metadata** (source URL, author, page numbers) to allow for filtering and accurate source citation.

---

## LangChain Document Loaders

LangChain provides hundreds of pre-built loaders to act as a standardized bridge between raw data sources and your AI application. 

### The `Document` Object
Every LangChain loader outputs a standardized `Document` object with two properties:
* `page_content`: The extracted text string.
* `metadata`: A Python dictionary of contextual data (e.g., `{"source": "report.pdf", "page": 4}`).

### Basic PDF Loader Example
`PyPDFLoader` is the standard tool for extracting text from PDFs. It automatically splits the document so each page becomes its own `Document` object.

```python
from langchain_community.document_loaders import PyPDFLoader

# Initialize the loader
loader = PyPDFLoader("./financial_report.pdf")

# Execute and load pages
pages = loader.load()

# Access content and metadata
print(pages[0].page_content)
print(pages[0].metadata)
```

### Source Code Loader Example
For code, LangChain uses `GenericLoader` paired with a `LanguageParser` (powered by `tree-sitter`) to understand programming syntax and preserve logic blocks (like functions and classes) rather than blindly chopping text.

```python
from langchain_community.document_loaders.generic import GenericLoader
from langchain_community.document_loaders.parsers import LanguageParser

# Load Python files, keeping functions/classes intact
loader = GenericLoader.from_filesystem(
    path="./my_project_code",
    glob="**/*",
    suffixes=[".py"],
    parser=LanguageParser(language="python", parser_threshold=50)
)

docs = loader.load()
```

---

## Advanced Parsing: Unstructured vs. Docling

Basic loaders struggle with complex PDF layouts, multi-column formats, and data tables. For these, you need advanced, ML-backed parsers.

### 1. Unstructured.io
Unstructured is a versatile tool supporting over 30 file formats. It categorizes text into specific elements like `Title`, `NarrativeText`, or `Table`.

```python
# pip install langchain-unstructured unstructured[all-docs]
from langchain_unstructured import UnstructuredLoader

loader = UnstructuredLoader(
    file_path="./annual_report.pdf",
    strategy="hi_res", # Uses ML layout detection
    split_pdf_page=True 
)

docs = loader.load()
print(docs[0].metadata['category']) # e.g., "Table" or "Title"
```

### 2. IBM Docling
Docling excels at preserving complex layouts and tables, outputting the result as highly accurate Markdown. It requires a standalone package (`langchain-docling`) because it relies on heavy, state-of-the-art vision models (like DocLayNet).

```python
# pip install "langchain-docling[local]"
from langchain_docling import DoclingLoader
from langchain_docling.loader import ExportType

loader = DoclingLoader(
    file_path="./research_paper.pdf",
    export_type=ExportType.MARKDOWN 
)

docs = loader.load()
# Output will be beautifully formatted Markdown, preserving tables
print(docs[0].page_content) 
```

### Comparison
| Feature | Unstructured.io | IBM Docling |
| :--- | :--- | :--- |
| **Best For** | Multi-format enterprise pipelines. | Research papers and highly complex tables. |
| **Output Style** | Distinct, categorized elements. | Continuous, structured Markdown/JSON. |
| **Ecosystem** | Mature, offers hosted serverless APIs. | Open-source, excellent for local, private processing. |
