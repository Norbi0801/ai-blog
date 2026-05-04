+++
title = "Vector databases explained: Qdrant, Pinecone, pgvector and friends"
date = 2026-04-12
description = "What a vector database actually is, how the main options compare in practice, and the surprisingly common cases where you do not need one."

[taxonomies]
tags = ["vector-search", "databases", "rag", "ai"]
+++

A vector database is a boring piece of infrastructure with a loud marketing layer. At its core it does two things: store vectors with some metadata, and return the nearest neighbors of a query vector quickly. Everything else (filtering, hybrid search, quantization, replication, tenant isolation) is table stakes that a good general-purpose database also does, just on different data types.

This matters because "which vector DB should I use" is almost never answered by the vector DB itself. It is answered by how the rest of your stack looks, how much data you have, and how much operational pain you are willing to eat.

<!-- more -->

If you have not read [HNSW: how vector search actually works under the hood](/blog/hnsw-algorithm-how-vector-search-actually-works-under-the-hood), that post covers the index that sits inside almost all of these systems. This one is about what you are actually buying when you pick a product on top of that index.

## What a vector database is, minus the hype

Peel the labels off and the data model is simple:

```
record: (id, vector[f32; D], payload: JSON)
index:  ANN structure over the vectors (HNSW, IVF, DiskANN, or brute force)
query:  given a query vector and filters, return top-K by similarity
```

That is it. Three pieces. The interesting engineering lives in how each piece scales under pressure:

- **Storage**: vectors are large. 10M 768-dim float32 vectors are 30 GB. Add metadata and replicas and you are easily in TB territory before the first user logs in. How the system pages, compresses, and replicates that is most of the product.
- **Index**: [HNSW](/blog/hnsw-algorithm-how-vector-search-actually-works-under-the-hood) for low latency in RAM. IVF (and its quantized cousin IVFPQ) for disk-resident massive indexes. Brute force for anything small enough to fit that way. Most production systems run one of these by default and let you opt into the others.
- **Filtering**: "top 10 most similar products that are in stock, tagged `outdoor`, and belong to tenant 42." The hard part is not the ANN. It is making the ANN cooperate with the filter. Two broad strategies exist: post-filter (ANN first, drop filtered-out results) and pre-filter (restrict the graph walk to candidates that match). Pre-filter wins on selectivity but needs the index to know about the filter fields. Every mature vector DB has a story here, and it is the single feature that differs most between products.

## The algorithm cheat sheet

Three algorithms cover 95% of real deployments. The HNSW post goes deep on all of them; the short version:

- **Brute force**. Compute similarity against every vector. O(N*D) per query. Perfect recall, no build time, no memory overhead. Good up to ~100K vectors depending on dimensionality and latency budget. Modern CPUs with AVX2 or AVX-512 chew through tens of GFLOPS of dot products; on a laptop this gets you well under 100 ms for a million 128-dim vectors.
- **HNSW**. Graph-based, in-memory, log(N) query. Default for Qdrant, Weaviate, Milvus in-memory mode, pgvector, Pinecone's pod indexes, Elasticsearch, Redis, Vespa. Sub-millisecond queries at 95-99% recall are routine.
- **IVF / IVFPQ**. Clustered inverted file, disk-friendly, great for billion-scale when RAM is the binding constraint. The default for FAISS at scale and for Milvus in DiskANN mode.

You rarely pick the algorithm explicitly. You pick the database and it picks sensible defaults. But you do need to know which one is running, because the tuning knobs (M, efSearch, nprobe) leak through.

## The contenders, one by one

### Qdrant

Written in Rust, open source (Apache 2.0), self-hostable, also available as managed cloud. As of Qdrant 1.14 in early 2026, the differentiators are:

- **Payload-aware filtering**. Qdrant builds secondary indexes on payload fields (keyword, integer, float, geo, bool, datetime) and uses them during the HNSW walk. This is the feature that makes "top 10 nearest AND tenant_id=42 AND in_stock=true" fast even when the filter selectivity is low. Most competitors bolt filtering on top of ANN; Qdrant pushes it into the search.
- **Scalar and binary quantization**. 4x to 32x memory reduction with modest recall loss. Binary quantization turns each dim into one bit; on normalized embeddings you can often get away with it for the first-pass search and rerank the top-K with full precision.
- **Sparse vectors and hybrid search**. First-class support for BM25-style sparse retrieval alongside dense, with server-side fusion. You do not need a second system for lexical search.
- **Operations**. Single Go-free binary, runs in a container. Snapshot-based backups. Horizontal sharding via Raft. The TCO story is good if you already run your own infra.

Where it bites you: cluster mode is newer than the single-node path, and the ergonomics still assume you know what HNSW is. Documentation is solid but not as hand-holdy as managed services.

### Pinecone

The original managed vector database. Closed source, SaaS only. The core product split in 2024 from pod-based indexes to serverless, and by 2026 serverless is the default.

- **Serverless is the actual pitch**. You pay for storage and reads, not for keeping nodes warm. For spiky workloads (a chatbot that sits idle overnight) this is dramatically cheaper than running your own always-on cluster.
- **Zero ops**. You do not see the index type, the shard count, or the node sizes. You get a collection, you write to it, you query it. For teams without infra headcount this is a real feature.
- **Filtering**. Namespaces for hard tenant separation, metadata filters for soft filtering. The filter story is fine but less sophisticated than Qdrant's payload-aware walk at high selectivity.

Where it bites you: cost. At scale serverless stops being cheap. The public pricing sits around $0.33 per GB-month of storage and per-read fees that add up fast on RAG workloads doing many queries per user session. A back-of-the-envelope for a 10M vector index doing 50 req/s round the clock is several thousand dollars a month, and the equivalent self-hosted Qdrant on a mid-tier VM is an order of magnitude cheaper. You are paying for someone else to run it, and that trade is real for small teams and real in the other direction once you have scale.

Lock-in is also a thing. The API is Pinecone-specific. Moving off means re-embedding into another store and changing client code.

### pgvector

An extension for PostgreSQL. Not really "a vector database", more "Postgres with a new column type and two index types". This is usually the right answer when you already run Postgres.

- **HNSW index since 0.5** (July 2023), ivfflat before that. Current version (0.7+) supports halfvec (float16), sparsevec, binary vectors, and parallel index builds.
- **One database, one connection pool, one transaction**. Your vectors, your user rows, your orders. Join them. Filter on them. This is the killer feature. No dual-write problems, no separate index to keep in sync, no "which system is authoritative when they disagree".
- **Filtering is just SQL**. The query planner decides whether to apply a predicate before or after the ANN. It is not always right (the planner has limited stats on vector distances), but for most workloads the combination of a B-tree index on the filter columns and HNSW on the vector column works well.

Where it bites you:

- **Performance ceiling is lower than dedicated systems**. pgvector's HNSW is good, but not tuned the way Qdrant's or Milvus's is. Expect roughly 2-5x higher query latency on the same hardware at the same recall target. For most apps this does not matter.
- **Index build time**. Building HNSW on 10M vectors in pgvector 0.7 takes tens of minutes on a beefy box, an order of magnitude slower than FAISS or hnswlib. Parallel builds help, as do the `maintenance_work_mem` settings, but you feel it.
- **Scale**. Past a few hundred million vectors you start fighting Postgres, not using it. Partitioning, read replicas, extra disk, all the usual Postgres-at-scale tricks.

Rough rule: if your data fits on one Postgres instance (with room to grow), pgvector wins. If it does not, pick something that was built to shard vectors.

### Milvus

C++ core, distributed from day one, Zilliz is the company behind it. The heavyweight open-source option.

- **Horizontal scale out of the box**. Coordinator nodes, query nodes, index nodes, data nodes, object storage. It looks like a distributed OLAP system because that is what it is. If you know you are going to 10^9+ vectors, Milvus and Vespa are where you end up.
- **DiskANN and GPU indexes**. Milvus 2.5 supports GPU-accelerated index building and querying (CUDA, NVIDIA cuVS integration). For billion-scale where query latency is paramount, this matters.
- **Multi-tenancy**. Collections, partitions, dynamic fields. More flexible than Qdrant, less SQL-like than pgvector.

Where it bites you: operational surface. The component zoo means you are running six kinds of service plus MinIO or S3 plus etcd. "Milvus Lite" exists for local dev but the real thing is a cluster. If you do not need the scale, the complexity is expensive.

### Weaviate

Written in Go, open source, also offered as managed cloud.

- **Module system**. Built-in vectorizer modules (OpenAI, Cohere, Hugging Face, local transformers) mean you can hand it raw text and it embeds for you. Nice for prototyping; arguably less nice once you want to control your embedding pipeline.
- **GraphQL query language**, with a REST API alongside. Opinionated but consistent.
- **Multi-tenancy is first class**. Tenants are isolated at the index level, which makes SaaS workloads with thousands of tenants tractable without creating thousands of collections.

Where it bites you: the opinionated surface. If your pipeline does not match Weaviate's module assumptions, you end up fighting the framing. Performance is competitive but not class-leading.

### Honorable mentions

- **Vespa**. Yahoo's search engine, open source. Absurdly scalable, supports dense vectors, sparse vectors, tensors, and full-text search in one expression language. Steep learning curve. Used at places that need billion-scale retrieval with complex ranking.
- **LanceDB**. Embedded, file-based (like SQLite for vectors). Lance format stores vectors and metadata in columnar Parquet-like files. Great for local-first, single-process workloads and for notebook-style analytics on embedding datasets.
- **Chroma**. Python-first, batteries included, extremely easy to get started. Fine for prototypes and small apps; less suitable as a production backbone.
- **Redis with RediSearch**. Vector search bolted onto Redis. Works if you already run Redis and want to avoid a second system. Single-process limits apply.
- **Elasticsearch / OpenSearch**. Both have kNN search via Lucene's HNSW implementation. Decent option if you are already running them for text search; less competitive as a greenfield choice.

## Which problem calls for which product

Three use cases account for the vast majority of "do we need a vector DB" conversations:

**RAG** ([retrieval-augmented generation](/blog/fine-tuning-vs-rag-vs-prompt-engineering)). You have docs, you embed them, you fetch relevant chunks at query time. The load profile is write-once, read-many, with moderate throughput (unless you are a consumer product). Most RAG systems have fewer than 10M chunks. pgvector handles this without breaking a sweat, and keeping your docs and their embeddings in the same database removes an entire class of bugs. Reach for Qdrant when the filter story gets complex (per-tenant corpora, time-based filtering, hybrid sparse + dense) or when you need binary quantization to fit a large corpus in RAM.

**Recommendations**. User and item embeddings, fetch nearest items given a user vector or a seed item. Latency budget is usually 10-50 ms end to end, traffic is heavy and steady. This workload rewards a dedicated system: Qdrant or Milvus self-hosted, or Pinecone if you want zero ops. pgvector works if the corpus is small and the request rate is modest, but the per-query overhead of Postgres (connection setup, parser, planner) eats more of the budget than it does for RAG.

**Deduplication and near-duplicate detection**. Embed everything, for each new item check its nearest neighbor, flag if similarity is above a threshold. This is often a batch job. For batch workloads you do not need a server at all. FAISS in a Python script, or LanceDB over a Parquet file, or even brute-force NumPy on a laptop, wins against a networked vector DB. The round trip latency of a query to a remote service is the dominant cost when you have no user waiting.

Other common cases that look like vector search: anomaly detection (distance from the nearest cluster), classification by nearest labelled example, semantic caching (hash LLM responses by prompt embedding). Same three algorithms, same three products.

## When a vector database is not the answer

The honest version: most projects that reach for a vector DB do not need one yet.

If your corpus is under 100K vectors and your query rate is under a few QPS, brute force in memory is faster than a network round trip to a dedicated vector database. A NumPy array and a one-line `argpartition` beats any ANN index below that scale, because the ANN needs its own RAM, its own process, and its own network hop. You are paying for three layers of abstraction to save work that takes five milliseconds.

Checks before you add a vector DB to your stack:

- **Do you have the vectors yet?** A surprising number of vector DB evaluations happen before anyone has embedded the actual data. Embed first. Often the top-K results are good enough without any index at all.
- **Is the query rate high enough to matter?** At 1 QPS, a 50 ms brute force and a 5 ms HNSW are both fine. At 1000 QPS they are not.
- **Are you already running something that does this?** If you have Postgres, start with pgvector. If you have Elasticsearch, start with its kNN. The operational savings usually beat a marginal performance win from a dedicated system.
- **Do you need filtering?** If yes, pick a system with a good filter story (Qdrant, pgvector via SQL) and test it with your real filter selectivity. Filter performance is where benchmarks lie and real workloads bleed.

## A quick selection matrix

| You are | Start with | Consider if |
|---|---|---|
| Already on Postgres, < 50M vectors | pgvector | Query latency is a problem |
| Small team, want zero ops, bursty traffic | Pinecone serverless | Cost becomes painful |
| Self-hosting, need good filtering and cost control | Qdrant | |
| Heading for billions of vectors | Milvus or Vespa | |
| Need text + vector in one query language | Weaviate or Vespa | |
| Prototyping, notebook or local-first | LanceDB or Chroma | |
| Batch dedup or analytics job | FAISS or brute force | |

None of these are wrong defaults. The wrong default is "Pinecone because it is famous" or "pgvector because it is there" without asking what the workload actually looks like.

## What to take away

The vector DB category is mature enough that the products are mostly similar in the middle and differ at the edges. The edges (filtering model, quantization, operational surface, pricing curve) are what you should evaluate, not the ANN algorithm. HNSW versus IVF is a tuning choice, not a vendor choice.

Pick the system that matches your stack and your scale. Do not let "vector" get treated as a special data type that needs its own database until you have measured why. Most RAG projects are a Postgres table and a CREATE INDEX away from done. The ones that need more usually know it because something specific is on fire, and the fire points directly at which feature to buy.

## References

- [pgvector](https://github.com/pgvector/pgvector) with HNSW, halfvec and sparsevec support
- [Qdrant docs on filtering and payload indexes](https://qdrant.tech/documentation/concepts/filtering/)
- [Pinecone serverless architecture](https://www.pinecone.io/learn/serverless/)
- [Milvus architecture](https://milvus.io/docs/architecture_overview.md)
- [Weaviate concepts](https://weaviate.io/developers/weaviate/concepts)
- [FAISS](https://github.com/facebookresearch/faiss) for batch and baseline comparisons
- [LanceDB](https://github.com/lancedb/lancedb) for embedded workloads
- [ann-benchmarks](https://ann-benchmarks.com/) for algorithm-level comparisons across implementations
