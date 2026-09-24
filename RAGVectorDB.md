# Vector Database Search: Interview Preparation Guide

This guide breaks down the core concepts of vector search, comparing exact vs. approximate nearest neighbors, and details the three most common indexing strategies. 

---

## 1. The Fundamental Trade-off: KNN vs. ANN

At the heart of vector databases is a trade-off between **Accuracy (Recall)** and **Speed/Latency**.

### KNN (K-Nearest Neighbors) - Exact Search
KNN is a brute-force approach. It calculates the distance between the query vector and **every single vector** in the database to find the exact top-K closest matches.

*   **Pros:** 
    *   100% accurate (perfect recall).
    *   No complex index to build or maintain.
*   **Cons:** 
    *   $O(N)$ time complexity; latency grows linearly as the dataset grows.
    *   Extremely slow and computationally expensive for large datasets (millions/billions of vectors).
*   **Trade-off:** You sacrifice speed and scalability for absolute precision.

### ANN (Approximate Nearest Neighbors)
ANN uses specialized data structures (indexes) to narrow down the search space. It makes an educated guess to find the closest vectors without scanning the entire dataset.

*   **Pros:** 
    *   Sub-linear time complexity; massively scalable.
    *   Blazing fast search speeds with low latency.
*   **Cons:** 
    *   Less than 100% recall (might miss the true nearest neighbor).
    *   Requires memory overhead to store the index.
    *   Requires time and compute to build the index initially.
*   **Trade-off:** You sacrifice a small amount of accuracy (usually acceptable in AI applications) for massive gains in speed and scalability.

---

## 2. Indexing Strategies Deep Dive

When an interviewer asks how a vector database achieves ANN, they are asking about the indexing strategy. Here are the big three:

### Strategy 1: Flat Search (Brute Force KNN)
*This is the implementation of exact KNN.*

*   **How it searches:** Computes the distance from the query to every vector in the dataset, sorts the results, and returns the top K.
*   **Index Creation:** No index is created. Vectors are stored in a raw, flat array.
*   **Seeding new entries:** $O(1)$ complexity. The new vector is simply appended to the end of the array.
*   **Pros:** 100% exact recall; zero time spent building an index; instantaneous ingestion.
*   **Cons:** Terrible search latency on large datasets.
*   **When to use:** Datasets under ~10k-100k vectors, or when evaluating the accuracy baseline of an ANN algorithm.

### Strategy 2: IVF (Inverted File Index)
*Divides the vector space into clusters (Voronoi cells).*

*   **How it searches:** 
    1. Identifies the cluster centroid closest to the query vector.
    2. Searches only the vectors contained within that cluster (and optionally a few neighboring clusters).
*   **Index Creation:** Uses a clustering algorithm (like K-Means) to divide the dataset into `V` clusters. Calculates the centroid for each cluster and assigns all existing vectors to their nearest centroid.
*   **Seeding new entries:** Calculates the distance between the new vector and all centroids. Finds the closest centroid and adds the vector to that cluster's bucket.
*   **Pros:** Much faster than flat search; smaller memory footprint compared to graph-based indexes.
*   **Cons:** 
    *   **The "Edge Problem":** If a query falls near the boundary of two clusters, its true nearest neighbor might be in an adjacent cluster that wasn't searched.
*   **Crucial Trade-off (Interview Gold):** The `nprobe` parameter. 
    *   `nprobe` dictates how many clusters to search. 
    *   Low `nprobe` = faster search, lower recall. 
    *   High `nprobe` = slower search, higher recall.

### Strategy 3: HNSW (Hierarchical Navigable Small World)
*The industry standard for vector search (Pinecone, Milvus, Qdrant). It relies on a multi-layered graph, similar to a skip-list.*

*   **How it searches:** 
    1. The search starts at the top, sparse layer. 
    2. It greedily traverses the graph to find the closest node in that layer.
    3. It uses that node as an entry point to drop down to the denser layer below.
    4. Repeats this process until it reaches the base layer (Layer 0), which contains all vectors and their local links.
*   **Index Creation:** Constructs a multi-tier graph. The bottom layer has all vectors. Each layer above is exponentially sparser. Links (edges) are created between vectors that are close to each other at each specific layer.
*   **Seeding new entries:** 
    1. Assigns a maximum layer to the new vector based on exponentially decaying probability.
    2. Starts at the top layer and searches down the layers to find the entry point for the new vector's assigned layer.
    3. Inserts the node and connects it to its nearest neighbors from its maximum layer down to Layer 0.
*   **Pros:** Excellent performance; incredibly fast search times; highly accurate (often 95-99% recall).
*   **Cons:** 
    *   Massive memory consumption (storing all the graph edges takes up a lot of RAM).
    *   Slow index build times (inserting vectors and recalculating edges is computationally heavy).
*   **Crucial Trade-offs (Interview Gold):** 
    *   `efConstruction`: Controls the depth of search during index building. Higher = better index quality/recall, but slower ingestion.
    *   `efSearch`: Controls the depth of search during querying. Higher = better recall, but slower search.
    *   `M`: The number of bi-directional links created for every new element. Higher = better recall, but uses more memory.

---

## 3. Quick Summary for Interviews

*   **Need absolute truth and have small data?** Use Flat/KNN.
*   **Need good speed and want to save RAM?** Use IVF (tune `nprobe` for recall).
*   **Need the best speed/recall balance and have plenty of RAM?** Use HNSW (industry standard).
