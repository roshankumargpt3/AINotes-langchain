# AI Concepts: Text Chunking Strategies

This document summarizes key concepts in Natural Language Processing (NLP) and Retrieval-Augmented Generation (RAG), specifically focusing on preparing text data through chunking.

## 1. The Importance of Chunking

Chunking is the process of breaking down large texts into smaller, manageable pieces before embedding them for search and retrieval. The size of your chunks directly determines what information gets retrieved effectively.

* **Too Large:** Contains mixed topics (diluting meaning), wastes context tokens on irrelevant text, and drops retrieval precision.
* **Too Small:** Fragments context, making individual chunks meaningless, and increases storage and search costs.
* **The Sweet Spot:** Typically 256-1024 tokens with a 10-20% overlap, depending heavily on the structure of the data.

## 2. Types of Chunking

1. **Fixed-Size (Token/Character):** Strict length divisions (e.g., exactly 500 characters) with slight overlap.
2. **Sentence-Based Chunking:** Splitting text specifically at sentence boundaries (using punctuation like periods, exclamation marks, or NLP libraries). 
3. **Structural:** Divided by natural document formatting (paragraphs, markdown headers, HTML/XML tags).
4. **Recursive:** A hierarchical approach that tries large structural separators first, and smaller ones only if needed to fit a size limit.
5. **Semantic:** Uses machine learning to group text by meaning, breaking chunks when the underlying topic shifts.
6. **Parent-Child (Hierarchical):** Creates multiple levels of chunks—large parent chunks for context and small child chunks for precise retrieval.

---

## 3. Deep Dive: Sentence-Based Chunking

Sentence-based chunking is a more refined version of fixed-size chunking. Instead of arbitrarily cutting off at a specific token count—which might slice a word or thought in half—this method uses natural language boundaries.

**How it works in practice:**
Most basic implementations just split on periods. However, enterprise systems use NLP libraries to avoid false splits on abbreviations (e.g., "Dr. Smith," "U.S.A.", or "Inc."). Sentences are often batched together to meet minimum size requirements before embedding.

---

## 4. Deep Dive: Recursive Chunking

Recursive chunking is the industry-standard starting point because it balances context preservation with strict size control.

**How it works:**
1. Establish a maximum chunk size (e.g., 500 tokens).
2. Define a hierarchy of separators (e.g., Paragraphs `\n\n` -> Line breaks `\n` -> Sentences `.` -> Words ` `).
3. Split the text using the highest-priority separator.
4. Evaluate resulting pieces. If a piece is under the limit, keep it. If it is over the limit, recursively split *only* that piece using the next separator in the hierarchy.

---

## 5. Deep Dive: Semantic Chunking 

Instead of relying on structural markers like paragraphs or word counts, semantic chunking uses AI to group text based on its *meaning*. It calculates the mathematical similarity between adjacent sentences to identify natural topic boundaries.

**How it works:**
1. **Initial Split:** The document is broken down into individual sentences.
2. **Embedding:** An embedding model converts each sentence into a vector (an array of numbers representing its meaning).
3. **Similarity Calculation:** The system calculates the *cosine similarity* between adjacent sentences (or rolling windows of sentences).
4. **Identify Breakpoints:** If the similarity score between Sentence A and Sentence B drops below a certain threshold (a "valley"), the system identifies a topic shift and places a chunk boundary there.

### Enterprise Level Example
Imagine a **Global Bank** processing a dense, 300-page internal compliance and risk management manual for its RAG-powered employee chatbot. 

In one continuous, poorly formatted section, the text discusses **Anti-Money Laundering (AML) reporting steps** and then seamlessly transitions into **Insider Trading blackout periods** without a clear separation.

* **Standard Chunking:** Might group the end of the AML rules and the beginning of the Insider Trading rules into the exact same chunk. If an employee asks the chatbot about AML, the LLM receives contaminated context mixing two unrelated compliance domains.
* **Semantic Chunking:** As the system compares the sentences, it notices the vocabulary and semantic meaning shift dramatically from *(transactions, monitoring, suspicious activity, reporting)* to *(trading, blackout, restricted persons, disclosure)* and automatically inserts a chunk boundary, keeping the two topics cleanly separated.

---

## 6. Deep Dive: Parent-Child (Hierarchical) Chunking

Parent-Child chunking creates a multi-level hierarchy where **large parent chunks** preserve broad context and **small child chunks** enable precise retrieval. This strategy solves a critical problem: small chunks are precise but lack context; large chunks have context but dilute meaning.

### How It Works

**Two-Stage Chunking Process:**
1. **Create Parent Chunks:** Use structural divisions (sections, subsections) or semantic boundaries to create larger chunks (e.g., 2,000–4,000 tokens) that capture complete ideas or topics.
2. **Create Child Chunks:** Break each parent chunk into smaller, retrievable pieces (e.g., 256–512 tokens) using recursive, sentence-based, or semantic methods.

**Retrieval Strategy:**
- **Search Phase:** Query embeddings are matched against child chunk embeddings for high precision.
- **Context Expansion Phase:** When a relevant child chunk is found, retrieve its associated parent chunk to provide richer context to the LLM.

### Practical Example

Consider a **medical research paper** with sections:
- **Parent Chunk 1:** Entire "Methods" section (~3,000 tokens)
  - **Child Chunk 1a:** "Study Design" (~300 tokens)
  - **Child Chunk 1b:** "Participant Selection" (~300 tokens)
  - **Child Chunk 1c:** "Data Collection Protocol" (~300 tokens)

When a user queries "What was the sample size?", the system retrieves the precise child chunk 1b (with minimal noise), but the LLM also receives the full parent chunk 1 (Methods section) for methodological context.

### Variants of Parent-Child Strategies

#### 1. **Sliding Window Parent-Child**
Creates overlapping hierarchies where multiple overlapping parent chunks each own their child chunks. This enables chunks to be connected through multiple contexts.

**Use Case:** Dense legal documents where a paragraph might relate to multiple sections.

#### 2. **Multi-Level Hierarchies (3+ Levels)**
Extends beyond parent-child to grandparent-parent-child structures:
- **Grandparent:** Entire document (~10,000 tokens)
- **Parent:** Section (~2,000 tokens)
- **Child:** Subsection (~300 tokens)

**Use Case:** Books, dissertations, or comprehensive reference materials where retrieval spans multiple abstraction levels.

**Example:** A programming textbook:
- Grandparent: "Data Structures and Algorithms" (entire textbook)
- Parent: "Chapter 3: Trees" (chapter)
- Child: "Binary Search Trees - Insertion" (subsection)

#### 3. **Hybrid Parent-Child with Semantic Grouping**
Combines semantic similarity with hierarchical relationships. Parents are created by semantic boundaries, and children are created by recursive splits within each semantic cluster.

**Use Case:** Technical documentation where concepts are tightly grouped by topic but may span traditional sections.

#### 4. **Metadata-Enriched Parent-Child**
Adds metadata tags to the relationship, such as:
- `parent_id`: Links child to parent
- `hierarchy_level`: 1 (parent) or 2 (child)
- `topic`: Semantic category
- `importance_score`: Which chunks matter most

**Use Case:** RAG systems requiring flexible filtering, reranking, or context-aware retrieval.

### Advantages & Disadvantages

**Pros:**
- ✅ **Best of Both Worlds:** Combines retrieval precision of small chunks with context richness of large chunks.
- ✅ **Flexible Retrieval:** Supports both fine-grained and broad-context queries without manual expansion.
- ✅ **Reduced Hallucination:** Richer context reduces LLM hallucination on out-of-context queries.
- ✅ **Scalable:** Works across documents of any size and complexity.
- ✅ **Supports Knowledge Graphs:** Child-to-parent links enable relationship-based traversal.

**Cons:**
- ❌ **Increased Complexity:** Requires maintaining parent-child relationships in the vector database.
- ❌ **Higher Storage Costs:** Multiple levels of chunks stored and indexed simultaneously.
- ❌ **Slower Retrieval:** Two-stage retrieval (child search → parent expansion) adds latency.
- ❌ **Dependency Management:** Breaking a parent chunk can fragment child chunks unexpectedly.

### Implementation Example (Pseudocode)

```python
class ParentChildChunker:
    def __init__(self, parent_size=2000, child_size=300, overlap=50):
        self.parent_size = parent_size
        self.child_size = child_size
        self.overlap = overlap
    
    def chunk(self, document):
        # Step 1: Create parent chunks using semantic boundaries
        parent_chunks = self.create_semantic_parents(document, self.parent_size)
        
        # Step 2: For each parent, create child chunks
        chunks_hierarchy = []
        for parent_id, parent_content in enumerate(parent_chunks):
            parent_chunk = {
                'id': f'parent_{parent_id}',
                'content': parent_content,
                'children': []
            }
            
            # Recursively chunk the parent
            child_chunks = self.recursive_chunk(parent_content, self.child_size, self.overlap)
            
            for child_id, child_content in enumerate(child_chunks):
                child_chunk = {
                    'id': f'parent_{parent_id}_child_{child_id}',
                    'content': child_content,
                    'parent_id': f'parent_{parent_id}'
                }
                parent_chunk['children'].append(child_chunk)
            
            chunks_hierarchy.append(parent_chunk)
        
        return chunks_hierarchy
    
    def retrieve_with_context(self, query_embedding, vector_db):
        # Find most relevant child chunk
        child_chunk = vector_db.search(query_embedding, top_k=1)[0]
        parent_id = child_chunk['parent_id']
        
        # Retrieve parent for additional context
        parent_chunk = vector_db.get_by_id(parent_id)
        
        return {
            'primary': child_chunk,
            'context': parent_chunk
        }
```

---

## 7. Comparison: Which Chunking Type is Best?

There is no universal "best," but different methods excel in specific scenarios:

| Chunking Method | Best Use Case | Pros & Cons |
| :--- | :--- | :--- |
| **Recursive Chunking** | **The Best Default.** Standard documents like PDFs, articles, and reports. | **Pros:** Excellent balance of context and size limits. <br> **Cons:** Requires structurally sound documents. |
| **Semantic Chunking** | **Best for High Accuracy.** Dense, complex materials (legal contracts, compliance manuals). | **Pros:** Highest quality retrieval; prevents topic blending. <br> **Cons:** Very computationally expensive; requires embedding model. |
| **Structural Chunking** | **Best for Code/Formatted Data.** Python scripts, HTML, XML, nested Markdown. | **Pros:** Prevents fracturing of logical blocks or nested structures. <br> **Cons:** Can result in imbalanced chunk sizes. |
| **Sentence-Based Chunking** | **Best for High Granularity.** Transcripts, conversational data, or as a RAG baseline. | **Pros:** Guarantees complete thoughts are preserved. <br> **Cons:** Individual chunks often too small for rich context. |
| **Fixed-Size Chunking** | **Best for Raw Text.** Completely unstructured text, quick pipeline testing. | **Pros:** Computationally simple and extremely fast. <br> **Cons:** High risk of cutting sentences or ideas mid-thought. |
| **Parent-Child Chunking** | **Best for Complex Retrieval.** Multi-document systems, hierarchical data, knowledge bases. | **Pros:** Combines precision with context; supports multi-level retrieval. <br> **Cons:** Higher complexity; requires careful hierarchy design; increased storage. |

---

## 8. Choosing Your Strategy: Decision Framework

1. **Document Structure:** Is your content well-formatted? → Use **Recursive** or **Structural**.
2. **Retrieval Precision:** Do you need exact answers? → Use **Semantic** or **Parent-Child**.
3. **Computational Budget:** Can you afford embeddings/LLM calls? → **Semantic** and **Parent-Child** are expensive.
4. **Context Importance:** Does the LLM need broad context? → Use **Parent-Child** or larger chunks.
5. **Scalability:** Must it work across thousands of documents? → **Fixed-Size** or **Recursive** scale best.
6. **Knowledge Complexity:** Is your domain hierarchical or interconnected? → **Parent-Child** with metadata relationships.

---

## Summary

- **Simple documents:** Recursive chunking.
- **Complex topics:** Semantic or parent-child chunking.
- **Speed-critical systems:** Fixed-size chunking.
- **Production RAG systems:** Parent-child chunking with semantic boundaries.
