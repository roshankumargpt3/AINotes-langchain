# AI Concepts: Text Chunking Strategies

This document summarizes key concepts in Natural Language Processing (NLP) and Retrieval-Augmented Generation (RAG), specifically focusing on preparing text data through chunking.

## 1. The Importance of Chunking

Chunking is the process of breaking down large texts into smaller, manageable pieces before embedding them for search and retrieval. The size of your chunks directly determines what information gets retrieved in a RAG system.

* **Too Large:** Contains mixed topics (diluting meaning), wastes context tokens on irrelevant text, and drops retrieval precision.
* **Too Small:** Fragments context, making individual chunks meaningless, and increases storage and search costs.
* **The Sweet Spot:** Typically 256-1024 tokens with a 10-20% overlap, depending heavily on the structure of the data.

## 2. Types of Chunking

1. **Fixed-Size (Token/Character):** Strict length divisions (e.g., exactly 500 characters) with slight overlap.
2. **Sentence-Based Chunking:** Splitting text specifically at sentence boundaries (using punctuation like periods, exclamation marks, or NLP libraries). 
3. **Structural:** Divided by natural document formatting (paragraphs, markdown headers, HTML/XML tags).
4. **Recursive:** A hierarchical approach that tries large structural separators first, and smaller ones only if needed to fit a size limit.
5. **Semantic:** Uses machine learning to group text by meaning, breaking chunks when the underlying topic shifts.

---

## 3. Deep Dive: Sentence-Based Chunking

Sentence-based chunking is a more refined version of fixed-size chunking. Instead of arbitrarily cutting off at a specific token count—which might slice a word or thought in half—this method uses punctuation (`.`, `?`, `!`) or advanced NLP tokenizers (like spaCy or NLTK) to ensure chunks represent complete thoughts.

**How it works in practice:**
Most basic implementations just split on periods. However, enterprise systems use NLP libraries to avoid false splits on abbreviations (e.g., "Dr. Smith," "U.S.A.", or "Inc."). Sentences are often batched together (e.g., 3-5 sentences per chunk) with a 1-sentence overlap to maintain context.

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

Instead of relying on structural markers like paragraphs or word counts, semantic chunking uses AI to group text based on its *meaning*. It calculates the mathematical similarity between adjacent sentences and creates a break only when it detects a shift in the topic.

**How it works:**
1. **Initial Split:** The document is broken down into individual sentences.
2. **Embedding:** An embedding model converts each sentence into a vector (an array of numbers representing its meaning).
3. **Similarity Calculation:** The system calculates the *cosine similarity* between adjacent sentences (or rolling windows of sentences).
4. **Identify Breakpoints:** If the similarity score between Sentence A and Sentence B drops below a certain threshold (a "valley"), the system identifies a topic shift and places a chunk boundary there.

### Enterprise Level Example
Imagine a **Global Bank** processing a dense, 300-page internal compliance and risk management manual for its RAG-powered employee chatbot. 

In one continuous, poorly formatted section, the text discusses **Anti-Money Laundering (AML) reporting steps** and then seamlessly transitions into **Insider Trading blackout periods** without a clear paragraph break or header.

* **Standard Chunking:** Might group the end of the AML rules and the beginning of the Insider Trading rules into the exact same chunk. If an employee asks the chatbot about AML, the LLM receives context about both and might hallucinate, telling the employee they cannot trade personal stock during an AML investigation.
* **Semantic Chunking:** As the system compares the sentences, it notices the vocabulary and semantic meaning shift dramatically from *(transactions, monitoring, suspicious activity, reporting)* to *(personal accounts, stock, blackout periods, securities)*. The similarity score plummets between these two sentences. The system automatically slices the chunk exactly at that shift, creating one pure AML chunk and one pure Insider Trading chunk. 

---

## 6. Comparison: Which Chunking Type is Best?

There is no universal "best," but different methods excel in specific scenarios:

| Chunking Method | Best Use Case | Pros & Cons |
| :--- | :--- | :--- |
| **Recursive Chunking** | **The Best Default.** Standard documents like PDFs, articles, and reports. | **Pros:** Excellent balance of context and size limits. <br> **Cons:** Requires structurally sound text to work optimally. |
| **Semantic Chunking** | **Best for High Accuracy.** Dense, complex materials (legal contracts, compliance manuals). | **Pros:** Highest quality retrieval; prevents topic blending. <br> **Cons:** Very slow and computationally expensive. |
| **Structural Chunking** | **Best for Code/Formatted Data.** Python scripts, HTML, XML, nested Markdown. | **Pros:** Prevents fracturing of logical blocks or nested structures. <br> **Cons:** Can result in wildly varying chunk sizes. |
| **Sentence-Based Chunking** | **Best for High Granularity.** Transcripts, conversational data, or as a RAG baseline. | **Pros:** Guarantees complete thoughts are preserved. <br> **Cons:** Individual sentences often lack broader surrounding context. |
| **Fixed-Size Chunking** | **Best for Raw Text.** Completely unstructured text, quick pipeline testing. | **Pros:** Computationally simple and extremely fast. <br> **Cons:** High risk of cutting sentences/thoughts in half. |
