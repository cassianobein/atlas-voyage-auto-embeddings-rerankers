# MongoDB Atlas Vector Search with Voyage AI: Embeddings, Automated Embedding, and Reranking

A self-contained Jupyter notebook that builds semantic product search on MongoDB Atlas with Voyage AI models. It compares two ways of producing vectors and two ways of reranking results, using a small e-commerce catalog.

## What the notebook covers

| Section | Topic | Key Atlas feature |
|---|---|---|
| 1 | Environment setup | Configuration loaded from `.env` with `python-dotenv` |
| 2 | Dataset preparation | 12 products, each combined into one `embedding_text` string |
| 3 | **Approach 1: Bring Your Own Embeddings (BYOE)** | Vectors generated through the Embedding API, `vector` index, `$vectorSearch` with `queryVector` |
| 4 | **Approach 2: Automated Embedding** | `autoEmbed` index, `$vectorSearch` with `query.text`; Atlas generates and syncs vectors |
| 5 | **Reranking Method 1: `$rerank` stage** | Native reranking inside the aggregation pipeline |
| 6 | **Reranking Method 2: Rerank API** | `POST https://ai.mongodb.com/v1/rerank` called from the application |
| 7 | Comparison and cleanup | Side-by-side rankings, optional removal of demo data |

### Approaches at a glance

| | Bring Your Own Embeddings | Automated Embedding |
|---|---|---|
| Who generates vectors | The application (Embedding API) | Atlas, at index time and on every insert/update |
| Index field type | `vector` (`numDimensions`, `similarity`) | `autoEmbed` (`model`, `modality`) |
| Query input | `queryVector` | `query.text` |

| | `$rerank` stage | Rerank API |
|---|---|---|
| Where it runs | Inside the aggregation pipeline | In application code |
| Requirements | MongoDB 9.0+, Native Reranking enabled | Model API key only |

## Project structure

```text
.
├── atlas_vector_search_autoembed_rerank.ipynb   # The notebook
├── .env.example                                 # Configuration template (safe to commit)
├── .env                                         # Your real configuration (git-ignored)
├── .gitignore
└── README.md
```

## Prerequisites

- **Python 3.9 or later.**
- **A MongoDB Atlas cluster** with Vector Search ([Get started with Atlas](https://www.mongodb.com/docs/atlas/getting-started/)).
  - Automated Embedding on dedicated clusters (`M10+`) requires **storage auto-scaling** to be enabled ([Automated Embedding overview](https://www.mongodb.com/docs/vector-search/crud-embeddings/automated-embedding/), [cluster auto-scaling](https://www.mongodb.com/docs/atlas/cluster-autoscaling/)).
  - The `$rerank` stage requires **MongoDB 9.0+** and **Native Reranking** turned on in **Project Settings** ([manage Native Reranking](https://www.mongodb.com/docs/vector-search/query/native-reranking/managing/)). The `rerank-3` and `rerank-3-lite` models also need a dedicated (`M10+`) cluster ([`$rerank` requirements](https://www.mongodb.com/docs/vector-search/query/aggregation-stages/rerank/)).
- **A model API key** created in Atlas, used for the Atlas Embedding and Reranking API (`https://ai.mongodb.com/v1`) ([manage model API keys](http://dochub.mongodb.org/core/voyage-api-keys)).
- **Network access:** your machine's IP address added to the Atlas IP access list (**Atlas → Network Access**) ([IP access list](https://www.mongodb.com/docs/atlas/security/ip-access-list/)).

## Setup

### 1. Create a virtual environment and install dependencies

```bash
cd atlas-voyage-auto-embeddings-rerankers
python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install --upgrade pip
pip install jupyterlab ipykernel pymongo python-dotenv requests
```

### 2. Configure credentials

```bash
cp .env.example .env
```

Edit `.env` and replace the placeholders:

| Variable | Required | Description | Example |
|---|---|---|---|
| `MONGODB_URI` | Yes | Atlas [connection string](https://www.mongodb.com/docs/manual/reference/connection-string/) | `mongodb+srv://<user>:<password>@<cluster-host>/` |
| `VOYAGE_API_KEY` | Yes | Atlas [model API key](http://dochub.mongodb.org/core/voyage-api-keys) | — |
| `VOYAGE_EMBEDDING_MODEL` | Yes | Model for Bring Your Own Embeddings ([embedding models](https://www.mongodb.com/docs/api/doc/atlas-embedding-and-reranking-api/operation/operation-createembedding)) | `voyage-4` |
| `VOYAGE_RERANK_MODEL` | Yes | Reranker model ([reranker models](https://www.mongodb.com/docs/vector-search/query/aggregation-stages/rerank/)) | `rerank-3` or `rerank-2.5` |
| `MONGODB_DB_NAME` | No | Database name (default `ecommerce_demo`) | `ecommerce_demo` |
| `AUTOEMBED_MODEL` | No | Model for the `autoEmbed` index, defaults to `VOYAGE_EMBEDDING_MODEL` ([supported models](https://www.mongodb.com/docs/vector-search/crud-embeddings/automated-embedding/models/)) | `voyage-4` |
| `EMBEDDING_DIMENSIONS` | No | 256, 512, 1024 (default), or 2048 ([`output_dimension`](https://www.mongodb.com/docs/api/doc/atlas-embedding-and-reranking-api/operation/operation-createembedding)) | `1024` |

`.env` is listed in `.gitignore`. Never commit real credentials.

## Running the notebook

**VS Code:** open `atlas_vector_search_autoembed_rerank.ipynb`, click **Select Kernel** (top-right), and choose **Python Environments → .venv**. Install the Jupyter extension if prompted.

**Jupyter Lab:** with the virtual environment active, run:

```bash
jupyter lab
```

Then open the notebook in the browser tab that appears.

Run the cells **from top to bottom** (Shift+Enter, or **Run All**). What to expect:

1. **Configuration** prints the non-secret settings and stops if a required variable is missing.
2. **Connection** pings the cluster and prints the server version.
3. **Embedding API check** makes one test call to confirm the API key and model.
4. **Index creation** waits until each index is `READY`. The Automated Embedding index can take several minutes, because Atlas generates the vectors during the build.
5. **`$rerank` cells** print a clear message and are skipped if the cluster does not meet the requirements; the Rerank API cells still run.
6. **Cleanup** is off by default. Set `CLEANUP = True` in the last code cell to drop the demo collections.

### Resources created in Atlas

| Collection | Search index | Contents |
|---|---|---|
| `products_byoe` | `byoe_vector_index` (`vector` + filters) | Products with an `embedding` field |
| `products_autoembed` | `autoembed_vector_index` (`autoEmbed` + filters) | Products without vectors; Atlas manages them |

Both indexes include `filter` fields for `category`, `price`, and `in_stock`. See [How to index fields for Vector Search](https://www.mongodb.com/docs/vector-search/indexes/vector-search-type/) for the `vector`, `autoEmbed`, and `filter` field types, and [Manage search indexes with PyMongo](https://www.mongodb.com/docs/languages/python/pymongo-driver/current/indexes/atlas-search-index/) for the Python index API.

## Example query (mongosh)

Automated Embedding search followed by native reranking. Field reference: [`$vectorSearch`](https://www.mongodb.com/docs/vector-search/query/aggregation-stages/vector-search-stage/), [`$rerank`](https://www.mongodb.com/docs/vector-search/query/aggregation-stages/rerank/), [`$meta`](https://www.mongodb.com/docs/manual/reference/operator/aggregation/meta/).

```javascript
db.products_autoembed.aggregate([
  { $vectorSearch: {
      index: "autoembed_vector_index",
      path: "embedding_text",
      query: { text: "comfortable shoes for long-distance trail running" },
      numCandidates: 100,
      limit: 10
  } },
  { $rerank: {
      query: { text: "comfortable shoes for long-distance trail running" },
      path: "embedding_text",
      numDocsToRerank: 10,
      model: "rerank-3"
  } },
  { $limit: 3 },
  { $project: { _id: 0, name: 1, price: 1, rerankScore: { $meta: "score" } } }
])
```

## Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| `Missing required environment variables` | `.env` not found or incomplete. Run the notebook from the project folder and check the variable names. |
| `Could not connect to MongoDB Atlas` | Wrong URI or credentials, or your IP is not on the [Atlas IP access list](https://www.mongodb.com/docs/atlas/security/ip-access-list/). |
| HTTP `401` from the API | Invalid model API key ([manage model API keys](http://dochub.mongodb.org/core/voyage-api-keys)). |
| HTTP `403` from the API | Your IP address is not allowed for this API key. |
| HTTP `429` from the API | Rate limit reached ([rate limits](http://dochub.mongodb.org/core/voyage-rate-limits)). The notebook retries automatically with backoff. |
| Index stays `PENDING`/`BUILDING` | Normal for the first few minutes. For `autoEmbed`, check that storage auto-scaling is enabled on `M10+` clusters ([manage Automated Embedding](https://www.mongodb.com/docs/vector-search/crud-embeddings/automated-embedding/management/)). |
| `$rerank pipeline failed` | Cluster below MongoDB 9.0, Native Reranking disabled, or `rerank-3` on a cluster smaller than `M10` ([`$rerank` requirements](https://www.mongodb.com/docs/vector-search/query/aggregation-stages/rerank/), [enable Native Reranking](https://www.mongodb.com/docs/vector-search/query/native-reranking/managing/)). Try `VOYAGE_RERANK_MODEL="rerank-2.5"`. |
| Searches return no results | The index is not `READY` yet ([index status](https://www.mongodb.com/docs/manual/reference/method/db.collection.getSearchIndexes/)), or Atlas is still generating embeddings for new documents. |

## References

**Vector Search**
- [How to index fields for Vector Search](https://www.mongodb.com/docs/vector-search/indexes/vector-search-type/): `vector`, `autoEmbed`, and `filter` field types
- [`$vectorSearch` stage](https://www.mongodb.com/docs/vector-search/query/aggregation-stages/vector-search-stage/): query syntax, `numCandidates`, and [pre-filter operators](https://www.mongodb.com/docs/vector-search/query/aggregation-stages/vector-search-stage/#std-label-vectorSearch-agg-pipeline-filter)
- [`db.collection.getSearchIndexes()`](https://www.mongodb.com/docs/manual/reference/method/db.collection.getSearchIndexes/) and [`db.collection.updateSearchIndex()`](https://www.mongodb.com/docs/manual/reference/method/db.collection.updateSearchIndex/)

**Automated Embedding**
- [Automated Embedding overview](https://www.mongodb.com/docs/vector-search/crud-embeddings/automated-embedding/)
- [How Automated Embedding works](https://www.mongodb.com/docs/vector-search/crud-embeddings/automated-embedding/overview/)
- [Supported models](https://www.mongodb.com/docs/vector-search/crud-embeddings/automated-embedding/models/)
- [Manage Automated Embedding](https://www.mongodb.com/docs/vector-search/crud-embeddings/automated-embedding/management/)

**Reranking**
- [`$rerank` stage](https://www.mongodb.com/docs/vector-search/query/aggregation-stages/rerank/)
- [Get started with Native Reranking](https://www.mongodb.com/docs/vector-search/query/native-reranking/quickstart/)
- [Manage Native Reranking settings](https://www.mongodb.com/docs/vector-search/query/native-reranking/managing/)

**Atlas Embedding and Reranking API**
- [API reference](https://www.mongodb.com/docs/api/doc/atlas-embedding-and-reranking-api/) and [API overview](http://dochub.mongodb.org/core/voyage-api-overview)
- [`POST /embeddings`](https://www.mongodb.com/docs/api/doc/atlas-embedding-and-reranking-api/operation/operation-createembedding) and [`POST /rerank`](https://www.mongodb.com/docs/api/doc/atlas-embedding-and-reranking-api/operation/operation-rerankdocuments)
- [Model API keys](http://dochub.mongodb.org/core/voyage-api-keys) and [rate limits](http://dochub.mongodb.org/core/voyage-rate-limits)

**Atlas and drivers**
- [Get started with Atlas](https://www.mongodb.com/docs/atlas/getting-started/), [IP access list](https://www.mongodb.com/docs/atlas/security/ip-access-list/), [cluster auto-scaling](https://www.mongodb.com/docs/atlas/cluster-autoscaling/)
- [Connection strings](https://www.mongodb.com/docs/manual/reference/connection-string/)
- [PyMongo `MongoClient`](https://www.mongodb.com/docs/languages/python/pymongo-driver/current/connect/mongoclient/) and [search index management in PyMongo](https://www.mongodb.com/docs/languages/python/pymongo-driver/current/indexes/atlas-search-index/)
