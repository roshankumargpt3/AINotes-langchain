# JLL AI Architect Interview Questions & Best Answers

## 1. RAG ETL Pipeline Steps
**Question:** What are the core pipeline steps for a RAG-based application ETL pipeline?

**Best Architect Answer:**
An enterprise RAG ETL pipeline consists of six sequential stages:
1. **Ingestion & Connectors:** Pulling structured and unstructured data from source systems (S3, Confluence, SharePoint, SQL) using batch or event-driven triggers.
2. **Pre-processing & Layout Parsing:** Detecting file types, removing noise, OCR for scanned images, and parsing complex layouts (tables, headers, footnotes) using specialized parsers (e.g., Unstructured, LlamaParse).
3. **Chunking Strategy:** Applying context-aware chunking based on document structure rather than simple character counts. Common patterns include **Parent-Child chunking** (small chunks for vector retrieval linked to larger context parent chunks) or **Semantic chunking** (splitting on embedding similarity shifts).
4. **Embedding Generation:** Batch-processing text chunks through an embedding model (e.g., `text-embedding-3-large`, `cohere-embed-v3`) with normalized outputs.
5. **Metadata Enrichment:** Appending critical filtering metadata to vectors, including Tenant ID, Role-Based Access Control (RBAC) groups, source URL, document version, and creation timestamps.
6. **Vector Indexing & Persistence:** Writing vectors and metadata into a vector database (e.g., Qdrant, Pinecone, PGVector) and building approximate nearest neighbor (ANN) indexes.

---

## 2. Embedding Dimensions & Trade-Offs
**Question:** If we increase embedding dimensions (e.g., from 512 to 768 or 1536), does it improve accuracy? What are the shortcomings?

**Best Architect Answer:**
* **Accuracy vs. Expressiveness:** Increasing dimensions allows the embedding space to represent fine-grained semantic nuances and domain-specific concepts, which can improve retrieval recall for complex datasets.
* **Shortcomings & Trade-Offs:**
  * **Memory Footprint:** Storage scales linearly (O(d)). A 1536-dimensional float32 vector consumes 6 KB of RAM per vector versus 2 KB for 512 dimensions. At scale (10M+ vectors), this significantly increases infrastructure costs.
  * **Query Latency:** Distance calculations (Cosine/Euclidean) take longer in higher-dimensional space.
  * **Curse of Dimensionality:** Higher dimensions can lead to sparse vector spaces where relative distances between points become less distinct.
* **Architectural Solution:** Instead of blindly inflating dimensions, use **Matryoshka Representation Learning (MRL)** embeddings (which allow truncation of vectors to smaller dimensions like 256 or 512 with minimal accuracy loss) or apply **Scalar/Product Quantization (PQ)** to compress high-dimensional vectors in memory.

---

## 3. Vector Indexes Beyond HNSW
**Question:** What are the main types of vector indexes, and when should we use indexes other than HNSW?

**Best Architect Answer:**
* **Flat (IndexFlatL2 / Exact Search):** Computes exact distance against all vectors. 
  * *When to use:* Small datasets (<10,000 vectors) or ground-truth benchmarking where 100% recall is strictly required.
* **HNSW (Hierarchical Navigable Small World):** Graph-based approximate nearest neighbor search. 
  * *When to use:* High-throughput, low-latency production applications requiring high recall (>95%). *Drawback:* High RAM usage for index graph construction.
* **IVF (Inverted File Index):** Partition-based clustering. Divides vector space into Voronoi cells and searches only nearest centroids.
  * *When to use:* Large datasets with memory constraints. Uses significantly less RAM than HNSW, though requires initial offline training/clustering.
* **PQ (Product Quantization):** Lossy compression technique that chunks vectors and maps them to centroids.
  * *When to use:* Multi-million to billion-scale vector stores to compress RAM footprint by 80–90%, typically combined with IVF (**IVF-PQ**).
* **DiskANN:** Graph-based index optimized for SSD storage.
  * *When to use:* Billion-scale datasets where storing the entire graph in RAM is cost-prohibitive.

---

## 4. Indexing Geospatial / Google Maps Data
**Question:** If storing geospatial data (latitude, longitude, etc.), which vector index or algorithm would you select?

**Best Architect Answer:**
* **Core Insight:** You should **not** use a vector database or embedding index for 2D coordinate search. Vector similarity measures (Cosine, Dot Product) fail on spherical Earth coordinate geometry and are computationally inefficient for 2D spatial queries.
* **Proper Approach:** Use spatial indexing structures designed for Euclidean or spherical geometry, such as **R-Trees**, **Quadtrees**, or **S2 Geometry / Geohash**:
  * Implement via native spatial databases like **PostgreSQL/PostGIS**, **Redis GEO**, or **Elasticsearch Geo-queries**.
* **Hybrid RAG Pattern:** If there is surrounding text (e.g., store reviews near a coordinate), run a two-pass query:
  1. Filter bounding-box coordinates via **PostGIS** spatial index.
  2. Perform vector search (HNSW) strictly over that filtered subset.

---

## 5. Multi-Language Policies, Region Filtering, and Security
**Question:** How do you handle RAG for multi-region, multi-language policy documents while enforcing enterprise-level security and encryption?

**Best Architect Answer:**
* **Language Strategy:** Use a **Native Multilingual Embedding Model** (e.g., `cohere-embed-multilingual-v3.0`). This maps concepts in different languages (English, Spanish, Japanese) into the same semantic vector space, allowing cross-lingual retrieval without managing 200 distinct physical indexes.
* **Region & Multi-Tenancy Strategy:** Apply **Pre-retrieval Metadata Filtering**. Tag each chunk with metadata attributes: `language`, `region_code`, and `allowed_roles`. Extract user claims from their JWT token during search and inject strict metadata filters into the vector query.
* **Encryption & Data Security Strategy:**
  * **At-Rest & In-Transit Encryption:** Standard AES-256 KMS key encryption for data stores and TLS 1.3 for transit.
  * **Field-Level Encryption / Tokenization:** For strict governance, run an **In-Flight PII Redaction / Anonymization Edge Service** (e.g., Microsoft Presidio) during ETL. Replace sensitive values with deterministic tokens before embedding. Keep the mapping in a secure, HSM-backed Key Vault to re-hydrate output only for authorized roles.

---

## 6. Challenges of Querying Across Multiple Index Partitions
**Question:** If an admin needs to query across multiple index partitions or 200 language indexes simultaneously, what technical challenges arise and how do you resolve them?

**Best Architect Answer:**
* **Challenges:**
  1. **Scatter-Gather Latency:** Querying 200 indexes in parallel creates network fan-out bottlenecks; the overall query response time matches the slowest partition (tail-latency problem).
  2. **Score Incomparability:** Cosine/L2 distance scores from different partitioned indexes are not directly comparable because vector density distributions differ. Merging raw top-K results leads to inaccurate rankings.
* **Solutions:**
  * **Unified Index with Metadata Filtering:** Avoid splitting into 200 physical indexes. Use a single unified index with metadata tags for language/region and utilize metadata filtering (e.g., Qdrant payload indices).
  * **Reciprocal Rank Fusion (RRF):** If physical partitioning is mandatory, do not merge by raw similarity scores. Merge returned items using RRF based on relative position/rank rather than score values.
  * **Parallel Execution with Timeouts:** Use asynchronous scatter-gather with strict SLAs per partition, dropping non-responsive index nodes if threshold timeouts are hit.

---

## 7. Vector Search on Encrypted Data
**Question:** If enterprise security mandates that data must be encrypted in storage, how do you handle search given that encryption destroys semantic vector relationships?

**Best Architect Answer:**
* **Architecture Pattern:** Decouple the **Embedding Space** from the **Raw Payload**.
* **Implementation Steps:**
  1. **Anonymized Embeddings:** Generate embeddings using non-sensitive or tokenized representations of the text. The vector representation itself is not reversible back to plain text.
  2. **Payload Encryption:** Encrypt the actual raw document text using customer-managed KMS keys before storing it in the Vector DB payload or document store.
  3. **Decryption at Access Time:** The vector database stores vector embeddings + encrypted payload blobs. Upon retrieval, the application validates user permissions, fetches the encrypted payload, and decrypts it in memory right before supplying context to the LLM.

---

## 8. Sparse (BM25) vs. Dense Retrieval on Anonymized Data
**Question:** Sparse retrieval (BM25) relies on exact keyword matching. If keywords are encrypted or anonymized, how do you preserve keyword search functionality?

**Best Architect Answer:**
* **Deterministic Tokenization:** Replace plain-text keywords with deterministic secure hashes or pseudo-tokens during ingest. If "John Doe" maps to `ID_98742`, the search query "John Doe" is similarly pre-processed to `ID_98742`, enabling exact-match BM25 against the tokenized index.
* **Learned Sparse Representations (SPLADE):** Transition from term-frequency BM25 to learned sparse neural models like **SPLADE**. SPLADE expands queries and documents into sparse term-weight vectors in a controlled vocabulary space *before* underlying storage encryption is applied, retaining sparse retrieval performance without storing raw plaintext.

---

## 9. Controlling LLM Hallucinations in RAG
**Question:** What concrete techniques do you use to eliminate or control hallucinations in a production RAG application?

**Best Architect Answer:**
1. **Retrieval Optimization:** Combine Dense and Sparse retrieval (**Hybrid Search**) paired with a **Cross-Encoder Re-ranker** (e.g., Cohere Rerank) to guarantee that top-ranked chunks have high semantic relevance.
2. **Prompt Grounding & Guardrails:** Use explicit system prompts instructing the LLM to restrict its answers strictly to the provided context block, declaring "Information not available in context" when unverified.
3. **Hyperparameter Tuning:** Set LLM `temperature = 0.0` or `top_p = 0.1` to enforce deterministic outputs.
4. **Automated Verification / Self-Correction:** Implement an output validation layer using frameworks like **Ragas** or **NVIDIA NeMo Guardrails** to evaluate *Faithfulness* (checking if every claim in the generated output is directly entailment-supported by the source context).

---

## 10. RAG vs. Fine-Tuning (LoRA) Selection Criteria
**Question:** When should you choose Fine-Tuning (e.g., LoRA) versus a RAG-based approach?

**Best Architect Answer:**
* **Primary Purpose:** RAG is for fetching **Knowledge** & dynamic facts. Fine-Tuning is for learning **Style, Structure, Tone, or Syntax**.
* **Data Freshness:** RAG handles Real-time / Dynamic data (instant DB update). Fine-Tuning handles Static data (requires re-training/adapter update).
* **Auditability:** RAG offers high auditability (direct citation of retrieved chunks). Fine-Tuning offers low auditability (knowledge is absorbed into model weights).
* **Use Cases:** Use RAG for Policy Q&A, Enterprise search, HR bots. Use Fine-Tuning for formatting outputs (JSON/SQL), specialized domain terminology, niche language style.
* **Architectural Rule:** Use **RAG for Knowledge** and **Fine-Tuning for Behavior**. In advanced enterprise apps, combine both: fine-tune a smaller open-weights LLM to follow precise JSON output formats, and feed it dynamic context via RAG.

---

## 11. Agent Harnessing & AI Harness Concepts
**Question:** What is an "Agent Harness" in the context of modern AI architectures?

**Best Architect Answer:**
An **Agent Harness** is the infrastructure, runtime container, and control plane that encapsulates an autonomous LLM agent. It acts as the bridge between raw LLM reasoning and real-world system execution:
* **Sandboxing & Isolation:** Executes agent-generated code or tool actions inside isolated, stateless containers (e.g., AWS Lambda, Docker, Modal) to protect host systems.
* **State & Memory Management:** Tracks multi-turn conversational state, execution call stacks, and checkpointing.
* **Tool & MCP Gateway:** Exposes API integrations, database connectors, and Model Context Protocol (MCP) servers safely with rate-limiting and authorization controls.
* **Governance & Human-in-the-Loop (HITL):** Intercepts high-risk tool calls (e.g., database writes, wire transfers) for approval before execution.
* **Observability:** Logs full agent execution traces (inputs, outputs, latency, tool calls, token costs) to platforms like Langfuse or Arize Phoenix.

---

## 12. Memory Taxonomy: Short-Term, Long-Term, and Episodic
**Question:** How do you define and implement Short-Term, Long-Term, and Episodic memory for conversational agents?

**Best Architect Answer:**
* **Short-Term Memory (Conversational State):**
  * *Definition:* In-session window containing the last N turns of the current conversation.
  * *Implementation:* High-speed Key-Value stores (Redis) using sliding windows or token-bounded summarization.
* **Long-Term Memory (User Profile / Fact Store):**
  * *Definition:* Persistent user preferences, attributes, and explicit knowledge across all historical sessions.
  * *Implementation:* Vector stores and relational DBs tagged with `user_id`. Extracted asynchronously post-session.
* **Episodic Memory (Experience Tracking):**
  * *Definition:* Log of past task executions and workflows, allowing the agent to recall *how* it successfully solved a similar problem in the past.
  * *Implementation:* Stored as structured execution trajectories (Goal -> Plan -> Tool Calls -> Outcome) in a vector DB, retrieved during planning stages via semantic similarity.

---

## 13. Managing User Memory When Underlying Data Refreshes (e.g., 8-Hour Updates)
**Question:** If enterprise policies refresh every 8 hours, how do you handle user memory and long-term context to ensure stale information is not served?

**Best Architect Answer:**
* **Decouple Policy Knowledge from User Preferences:** Never store policy facts directly inside the user's long-term memory. User memory stores user traits (e.g., "User lives in Region X"), while policy documents live in the versioned Vector Knowledge Base.
* **Metadata Versioning & Watermarking:** Every chunk in the policy vector store carries a `policy_version` and `ingestion_timestamp`.
* **Dynamic Re-verification at Inference:**
  1. Retrieve policy details strictly from the fresh 8-hour policy index.
  2. Retrieve user preferences from the user memory store.
  3. If a cached user preference contradicts the newly updated policy, an **Input Guardrail** flags the conflict, invalidates the stale user memory key, and prompts the user for re-confirmation.
* **Session Invalidations:** Flush active session caches or update cache keys using an automated event trigger (e.g., EventBridge) whenever a new policy version deployment completes.

---

## 14. Agentic Session Management
**Question:** How does session management work in an agentic workflow compared to standard web applications?

**Best Architect Answer:**
In standard web apps, sessions store simple user claims and auth state. In **Agentic Architectures**, session management requires an Execution State Machine:
* **Graph State Persistence:** Persisting the node execution state (e.g., in LangGraph or AutoGen) so an agent can pause execution while waiting for asynchronous events or human input.
* **Checkpointing:** Saving a snapshot of state (variables, message history, tool results) after every step into a persistent store (PostgreSQL/Redis checkpointing).
* **Time-to-Live & Thread Cleanup:** Applying strict TTLs on active execution threads to clean up abandoned or looping agent instances.
* **Replayability:** Storing deterministic state logs so failed agent executions can be restarted from the exact step of failure without re-running previous tool calls.
