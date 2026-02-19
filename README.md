# vector-db

Educational in-memory vector database in Python, with:
- pluggable embeddings
- vector indexes (brute-force, KD-tree, IVF-Flat, LSH)
- full-text search
- a chainable query builder
- basic table metadata and CRUD

This project is designed for learning and experimentation, not production workloads.

## What the project contains

- `src/vector_db/database.py`: `InMemoryVectorDB` for connection lifecycle, table management, and embedding registry.
- `src/vector_db/table.py`: `VectorTable` for records, indexes, CRUD, and search APIs.
- `src/vector_db/query.py`: `Query` builder with filtering, projection, pagination, reranking, and search mode selection.
- `src/vector_db/embeddings.py`: embedding protocol plus `CallableEmbedding` and a toy `HashEmbedding`.
- `src/vector_db/indexes/`: vector and scalar/text index implementations.
- `src/vector_db/examples/demo_basic.py`: end-to-end example.
- `tests/`: pytest suite covering distances, embeddings, indexes, table operations, and query behavior.

## Features

- In-memory DB with explicit `connect()` / `close()` lifecycle.
- Table creation with optional schema, tags, and text fields.
- CRUD operations:
  - `add`, `update`, `merge`, `upsert`, `delete`
- Vector search with configurable metric:
  - `cosine`, `euclidean`, `dot`
- Additional search modes:
  - full-text search over configured payload fields
  - hybrid search (weighted vector + text ranking)
- Query builder for chainable pipelines:
  - `.filter(...)`, `.where(...)`, `.vector_search(...)`, `.text_search(...)`, `.hybrid(...)`, `.use_index(...)`, `.limit(...)`, `.offset(...)`, `.select(...)`
- Index support:
  - `bruteforce` (default)
  - `kdtree` (euclidean only)
  - `ivfflat` (approximate)
  - `lsh` (cosine only, approximate)
  - scalar metadata with `BTreeIndex`
  - full-text inverted index (`FullTextIndex`)

## Installation

Python 3.9+ is required.

```bash
pip install -e .
```

For development tools:

```bash
pip install -e ".[dev]"
```

## Quick start

```python
from vector_db import InMemoryVectorDB, HashEmbedding, Query

db = InMemoryVectorDB("demo").connect()
db.register_embedding("hash-64", HashEmbedding(dim=64))

table = db.create_table(
    name="articles",
    embedding="hash-64",
    schema={"title": str, "text": str, "category": str, "views": int},
    text_fields=["title", "text"],
)

table.add({"title": "Intro to vectors", "text": "Vectors in linear algebra", "category": "math", "views": 10})
table.add({"title": "Neural networks", "text": "Deep learning with vectors and matrices", "category": "ml", "views": 20})

# Optional extra indexes
table.create_vector_index("ivf", index_type="ivfflat", n_lists=4, n_probe=2)
table.create_btree_index("views")

# Vector query
qvec = table.embedding.embed("vectors and matrices")
results = table.vector_search(qvec, k=5, index_name="default")

# Query builder example
rows = (
    Query(table)
    .filter(category="ml")
    .vector_search(qvec, k=10)
    .use_index("ivf")
    .select(["title", "category"])
    .limit(3)
    .execute()
)

for row in rows:
    print(row)
```

## How record vectors are created

When you call `table.add(...)`:
- if `vector` is provided, it is validated against the table embedding dimension.
- if `vector` is omitted, `payload["text"]` must be a string and the table embedding is used to generate a vector automatically.

## Running the example

```bash
python src/vector_db/examples/demo_basic.py
```

## Running tests

```bash
pytest -q
```

Current repository status: `15 passed`.

## Notes and limitations

- All data is in-memory only; nothing is persisted to disk.
- `HashEmbedding` is intentionally simplistic and not suitable for production.
- Approximate indexes (`ivfflat`, `lsh`) trade recall for speed.
- `KDTreeIndex` supports euclidean distance only.
