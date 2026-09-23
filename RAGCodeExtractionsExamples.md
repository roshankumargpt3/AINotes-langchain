# Document Loaders in RAG Pipelines

Document loaders are the ingestion engines of a Retrieval-Augmented Generation (RAG) system. Before an AI can search your private data, that data must be translated into plain text. Loaders fetch raw data from a variety of sources and convert it into a standardized `Document` format.

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
For code, LangChain uses `GenericLoader` paired with a `LanguageParser` (powered by `tree-sitter`) to understand programming syntax and preserve logic blocks (like functions and classes) rather than flattening everything into one blob.

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

## Common Document Extraction Examples

These examples show how to use LangChain loaders for commonly used document types.

### 1. Plain Text Files (`.txt`)

```python
from langchain_community.document_loaders import TextLoader

loader = TextLoader("./notes.txt", encoding="utf-8")
documents = loader.load()

print(f"Loaded {len(documents)} document(s).")
print(documents[0].page_content)
print(documents[0].metadata)
```

### 2. Markdown Files (`.md`)

```python
from langchain_community.document_loaders import UnstructuredMarkdownLoader

loader = UnstructuredMarkdownLoader("./README.md")
documents = loader.load()

print(f"Loaded {len(documents)} document(s).")
print(documents[0].page_content)
```

### 3. Word Documents (`.docx`)

```python
from langchain_community.document_loaders import UnstructuredWordDocumentLoader

loader = UnstructuredWordDocumentLoader("./report.docx")
documents = loader.load()

print(f"Loaded {len(documents)} document(s).")
print(documents[0].page_content)
print(documents[0].metadata)
```

### 4. CSV Files (`.csv`)

```python
from langchain_community.document_loaders import CSVLoader

loader = CSVLoader(
    file_path="./employees.csv",
    csv_args={
        "delimiter": ",",
        "quotechar": '"'
    }
)

documents = loader.load()
print(f"Loaded {len(documents)} rows.")

for index, document in enumerate(documents[:3], start=1):
    print(f"\nRow {index}:")
    print(document.page_content)
    print(document.metadata)
```

### 5. JSON Files (`.json`)

```python
from langchain_community.document_loaders import JSONLoader

loader = JSONLoader(
    file_path="./data.json",
    jq_schema=".",
    text_content=False
)

documents = loader.load()
print(f"Loaded {len(documents)} document(s).")

for document in documents:
    print(document.page_content)
    print(document.metadata)
```

### 6. PowerPoint Files (`.pptx`)

```python
from langchain_community.document_loaders import UnstructuredPowerPointLoader

loader = UnstructuredPowerPointLoader("./presentation.pptx")
documents = loader.load()

print(f"Loaded {len(documents)} document(s).")
print(documents[0].page_content)
```

### 7. HTML Files (`.html`)

```python
from langchain_community.document_loaders import BSHTMLLoader

loader = BSHTMLLoader("./webpage.html")
documents = loader.load()

print(f"Loaded {len(documents)} document(s).")
print(documents[0].page_content)
print(documents[0].metadata)
```

### 8. Web Pages

```python
from langchain_community.document_loaders import WebBaseLoader

loader = WebBaseLoader("https://example.com")
documents = loader.load()

print(f"Loaded {len(documents)} document(s).")
print(documents[0].page_content)
print(documents[0].metadata)
```

### 9. EPUB Files (`.epub`)

```python
from langchain_community.document_loaders import UnstructuredEPubLoader

loader = UnstructuredEPubLoader("./book.epub")
documents = loader.load()

print(f"Loaded {len(documents)} document(s).")
print(documents[0].page_content)
```

### 10. Directory of Documents

```python
from langchain_community.document_loaders import DirectoryLoader, TextLoader

loader = DirectoryLoader(
    "./documents",
    glob="**/*.txt",
    loader_cls=TextLoader,
    loader_kwargs={"encoding": "utf-8"}
)

documents = loader.load()
print(f"Loaded {len(documents)} documents.")

for document in documents:
    print(document.metadata.get("source"))
    print(document.page_content[:300])
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
Docling excels at preserving complex layouts and tables, outputting the result as highly accurate Markdown. It requires a standalone package (`langchain-docling`) because it relies on heavy, state-of-the-art document parsing dependencies.

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
