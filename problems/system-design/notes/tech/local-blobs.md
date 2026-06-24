If you extend the same constraint set (“SQLite-like, but for blobs”), the target becomes:

> an **embedded object store for large binary payloads**, with optional indexing + metadata querying

So instead of KV/document-first, you now optimize for:

* large contiguous bytes (images, videos, embeddings, files)
* streaming / partial reads
* content addressing or metadata lookup

---

# 1. First-principles split: what “local blob store” actually decomposes into

A blob system always separates into two layers:

### (A) Blob plane (bytes)

* raw storage of binary chunks
* append/read/seek operations
* minimal interpretation

### (B) Index plane (metadata)

* filename → blob id
* hash → blob
* tags → blob
* time/version → blob

SQLite already hints at this split:

* BLOB column = blob plane
* SQL tables = index plane

So “SQLite for blobs” is really:

> embedded storage engine + metadata indexer + optional chunking layer

---

# 2. Closest real embedded blob-capable systems

## 🟦 1. SQLite itself (surprisingly strong baseline)

SQLite

SQLite is often *already the correct answer* if:

* blobs are < a few GB total
* you want ACID + simplicity

### Pattern: BLOB column storage

```sql id="blob1"
CREATE TABLE files (
  id TEXT PRIMARY KEY,
  data BLOB,
  mime TEXT,
  created_at INTEGER
);
```

### Use case

* app assets
* cached media
* offline datasets

### Limitation

* not optimized for streaming huge blobs (though WAL helps)
* entire blob often loaded unless carefully chunked

---

## 🟨 2. LMDB (memory-mapped blob/KV hybrid)

LMDB

* KV store backed by mmap
* extremely fast reads
* good for binary payloads

### Model

* key → value (value can be blob)

### Use case

* embedded browsers
* ML feature stores
* mobile apps

### Tradeoff

* no built-in query layer
* you design metadata indexing yourself

---

## 🟧 3. RocksDB (log-structured blob KV store)

RocksDB

* designed for large write throughput
* supports large values (blobs)
* compaction-based storage

### Pattern

```
key = file_id
value = raw bytes
```

### Good for:

* event logs
* media caches
* streaming ingestion systems

---

## 🟩 4. UnQLite (closest “SQLite-for-NoSQL + blobs” conceptually)

UnQLite

* supports KV + document store
* supports binary values
* embedded, single-file design

This is one of the closest conceptual matches to:

> “SQLite but non-relational + blob-friendly”

---

# 3. The often-overlooked best answer: SQLite + chunking

Even for large blobs, the strongest general solution is:

### Chunked blob table

```sql id="blob2"
CREATE TABLE blob_chunks (
  file_id TEXT,
  chunk_index INTEGER,
  data BLOB,
  PRIMARY KEY(file_id, chunk_index)
);
```

### Why this works well

* avoids memory spikes
* supports streaming reads
* allows partial updates
* index-friendly

This effectively turns SQLite into a **mini object storage engine**.

---

# 4. When you outgrow embedded blob stores

If your intent shifts toward:

* multi-device sync
* CDN-like access
* distributed storage

Then you move to object storage systems like:

* MinIO (local S3-compatible server)
* IPFS-style systems

But those are no longer “SQLite-equivalent”—they are **networked storage systems**, not embedded engines.

---

# 5. Decision funnel (practical selection logic)

Start from constraints:

### If you want simplest system

→ use **SQLite BLOB column**

### If you want high-performance KV blobs

→ use **RocksDB**

### If you want embedded + object-like access

→ use **LMDB or UnQLite**

### If you want scalable local-first design

→ SQLite + chunking layer (most robust pattern)

---

# Key insight

A “local blob store equivalent to SQLite” doesn’t really exist as a separate category because:

> SQLite already *is* the canonical embedded blob store; everything else is either a KV engine or a specialized optimization around it.

The real design choice is not “which blob DB”, but:

* **Do you want relational indexing (SQLite)?**
* **or raw KV throughput (RocksDB/LMDB)?**
* **or document abstraction (UnQLite/Realm)?**

---

If you want, I can map this into a **full local-first architecture (KV + blob + cache + WAL + sync layer)** like what browsers or mobile offline systems actually implement.
