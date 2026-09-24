# Vector Database Search: Interview Preparation Guide

This guide breaks down the core concepts of vector search, comparing exact vs. approximate nearest neighbors, and details the three most common indexing strategies.

---

## 1. The Fundamental Trade-off: KNN vs. ANN

At the heart of vector databases is a trade-off between **accuracy (recall)** and **speed/latency**.

### KNN (K-Nearest Neighbors) — Exact Search

KNN is a brute-force approach. It calculates the distance between the query vector and **every single vector** in the database to find the exact top-K closest matches.

* **Pros:**
  * 100% accurate (perfect recall).
  * No complex index to build or maintain.
* **Cons:**
  * $O(N)$ time complexity; latency grows linearly as the dataset grows.
  * Extremely slow and computationally expensive for large datasets (millions/billions of vectors).
* **Trade-off:** You sacrifice speed and scalability for absolute precision.

### ANN (Approximate Nearest Neighbors)

ANN uses specialized data structures (indexes) to narrow down the search space. It makes an educated approximation to find the closest vectors without scanning the entire dataset.

* **Pros:**
  * Typically much faster than brute-force search and scalable to large datasets.
  * Low search latency.
* **Cons:**
  * Less than 100% recall (it might miss the true nearest neighbor).
  * Requires memory overhead to store the index.
  * Requires time and compute to build the index initially.
* **Trade-off:** You sacrifice a controllable amount of accuracy for large gains in speed and scalability.

---

## 2. Indexing Strategies Deep Dive

When an interviewer asks how a vector database achieves ANN, they are asking about the indexing strategy. Here are three important strategies:

### Strategy 1: Flat Search (Brute-Force KNN)

*This is the implementation of exact KNN.*

* **How it searches:** Computes the distance from the query to every vector in the dataset, then returns the top K closest vectors.
* **Index creation:** No search index is created. Vectors are stored in a raw, flat array.
* **Seeding new entries:** $O(1)$ amortized complexity: the new vector is appended to the array. (A resize or copy may occasionally be needed.)
* **Pros:** 100% exact recall; zero time spent building an index; simple ingestion.
* **Cons:** Search latency grows linearly with the number of vectors.
* **When to use:** Small datasets, or when evaluating the recall of an ANN algorithm.

### Strategy 2: IVF (Inverted File Index)

*Divides the vector space into clusters (Voronoi cells).* 

* **How it searches:**
  1. Finds the nearest cluster centroids to the query vector.
  2. Searches only the vectors assigned to those clusters (and optionally several nearby clusters).
* **Index creation:** Uses a clustering algorithm such as K-Means to create `V` clusters. It stores each centroid and assigns every existing vector to its nearest centroid's bucket.
* **Seeding new entries:** Calculates the distance from the new vector to the centroids, chooses the nearest centroid, and adds the vector to that centroid's bucket.
* **Pros:** Much faster than flat search; usually uses less memory than a graph-based index.
* **Cons:** If a query is near the boundary between clusters, its true nearest neighbor might be in a cluster that was not searched.
* **Crucial trade-off (interview point):** The `nprobe` parameter controls how many clusters are searched.
  * Low `nprobe` = faster search, lower recall.
  * High `nprobe` = slower search, higher recall.

### Strategy 3: HNSW (Hierarchical Navigable Small World)

HNSW is a graph-based ANN index used by systems such as Qdrant, Weaviate, and Milvus. It is based on a navigable small-world graph with multiple layers, conceptually similar to a skip list.

#### What the HNSW structure looks like

* **Layer 0** contains every vector and its local neighbor links.
* **Higher layers** contain progressively fewer vectors and longer-range links. These layers act like express lanes that quickly move the search toward the right region of the vector space.
* A vector may occur in Layer 0 and in one or more higher layers. The highest layer containing that vector is its **maximum layer**.

#### How it searches

Given a query vector and a requested top `K`:

1. The search starts at the index's entry point, normally at the highest layer currently present.
2. At each upper layer, it performs a greedy search: it visits neighboring nodes while a neighbor is closer to the query than the current node. When no neighbor improves the distance, it stops at that layer.
3. It moves down one layer, using the best node found so far as the starting point. This repeats until Layer 0.
4. At Layer 0, it normally performs a broader best-first search rather than a purely greedy search. It keeps a candidate queue and explores up to `efSearch` candidates before returning the best `K` results.

The upper layers provide fast navigation; the wider Layer 0 search provides recall. Increasing `efSearch` usually improves recall but increases latency.

#### How the index is created

HNSW is usually built incrementally by inserting vectors one at a time:

1. For each vector, randomly choose its maximum layer using a distribution that strongly favors Layer 0.
2. Start from the current entry point at the highest layer. Greedily navigate downward until reaching the new vector's maximum layer.
3. At each layer from the new vector's maximum layer down to Layer 0, search for promising nearby existing nodes. During construction this search is controlled by `efConstruction`.
4. Select up to `M` neighbors for the new node, add bidirectional links between the new node and those neighbors, and prune excess links according to the implementation's neighbor-selection heuristic. Layer 0 often has a larger link limit (commonly `2M`) than upper layers.
5. If the new vector's maximum layer is higher than the current highest layer, make it the new entry point.

This is not the same as connecting every vector to every other vector. Each node has a bounded number of links, which keeps navigation efficient and limits memory usage.

#### How a new vector is inserted — concrete example

Assume an index has document embeddings for articles about **cats**, **dogs**, and **birds**, and its entry point is currently a high-level node about pets. A new document, *“How kittens learn to recognize their owners”*, arrives:

1. The algorithm assigns the new vector a maximum layer, for example Layer 1.
2. Starting at the top layer, it follows links to the node whose embedding is closest to the new document. It then drops to Layer 1.
3. At Layer 1, it explores candidate neighbors such as other cat-related documents and selects up to `M` good neighbors.
4. It repeats the neighbor search at Layer 0, where all documents exist, and creates links to the best local neighbors. The links are bidirectional, so the existing neighbors may also need to replace or prune one of their old links.
5. A future query about kitten behavior can use the sparse upper layer to reach the cat region quickly, then use Layer 0 to find the most similar documents.

#### What “exponentially decaying probability” means

Higher layers must be sparse; otherwise, the upper layers would be as large and expensive to search as Layer 0. HNSW therefore assigns high maximum layers rarely.

One common way to describe the distribution is:

\[
P(L \geq \ell) = e^{-\ell / m_L}
\]

where `L` is the randomly selected maximum layer and `m_L` controls how quickly the probability decays. In many implementations, `m_L` is related to `M`; a common approximation is `m_L = 1 / ln(M)`.

An equivalent implementation samples `U` uniformly from `(0, 1]` and computes something like:

\[
L = \lfloor -\ln(U) \times m_L \rfloor
\]

The exact formula and constants vary by implementation, but the important interview concept is the shape of the distribution:

* Almost every vector is placed in Layer 0.
* Roughly a smaller fraction, often about `1/M` under a common parameterization, reaches Layer 1.
* About `1/M²` reaches Layer 2, and so on.

For example, if the probability of reaching the next layer is approximately `1/16`, then out of 160,000 inserted vectors, about 10,000 might reach Layer 1, about 625 might reach Layer 2, and about 39 might reach Layer 3. These are expected values, not exact counts. This geometric decrease creates sparse “express lanes” while preserving a complete graph at Layer 0.

#### Pros and cons

* **Pros:** Excellent speed/recall trade-off; fast query performance; supports incremental insertion well.
* **Cons:** High memory consumption because it stores graph edges; index construction can be computationally expensive; updates and deletions may require extra maintenance depending on the implementation.

#### Crucial trade-offs (interview point)

* `efConstruction`: Number of candidates considered while building the graph. Higher values usually produce a better graph and higher recall, but slow indexing and increase build cost.
* `efSearch`: Number of candidates considered during a query. Higher values usually improve recall, but increase query latency. It should generally be at least `K`.
* `M`: Maximum number of graph connections per node at most layers. Higher values can improve connectivity and recall, but increase memory usage and build cost.

### A concise HNSW interview answer

> HNSW builds a multi-layer proximity graph. Every vector is in Layer 0, while only a geometrically decreasing subset is promoted to higher layers. During a query, the algorithm greedily navigates the sparse upper layers to get near the right region, then performs a wider best-first search at Layer 0. During insertion, it assigns a random maximum layer, searches for neighbors using `efConstruction`, and creates bounded bidirectional links. `M`, `efConstruction`, and `efSearch` trade memory/build time, index quality, and query recall/latency.

---

## 3. Quick Summary for Interviews

* **Need absolute truth and have small data?** Use Flat/KNN.
* **Need good speed and want to save RAM?** Use IVF (tune `nprobe` for recall).
* **Need a strong speed/recall balance and have sufficient RAM?** Use HNSW (tune `M`, `efConstruction`, and `efSearch`).
