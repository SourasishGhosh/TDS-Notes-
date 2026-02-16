# 📚 Semantic Search API - Implementation Notes

## Problem Statement

**Company**: InfoCore Solutions  
**Issue**: Extensive internal knowledge base, but traditional keyword-based search returns too many irrelevant results

**Solution**: Implement semantic search using OpenAI embeddings and cosine similarity

---

## API Endpoint

**URL**: `http://127.0.0.1:8000/similarity`  
**Method**: `POST`

### Request
```json
{
  "docs": ["document 1", "document 2", "document 3"],
  "query": "search query"
}
```

### Response
```json
{
  "matches": ["most relevant doc", "second match", "third match"]
}
```

---

## Architecture Overview

1. **Text Embedding**: Convert docs and query to vectors using OpenAI's `text-embedding-3-small`
2. **Similarity Calculation**: Compute cosine similarity between query and each doc
3. **Ranking**: Return top 3 documents in descending relevance order

---

## Code Breakdown

### Core Function: Cosine Similarity
```python
def cosine_similarity(a: np.ndarray, b: np.ndarray) -> float:
    """Cosine similarity: dot product / (norm_a * norm_b)"""
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
```

**Why cosine similarity?**
- Measures angle between vectors (0° = identical, 90° = orthogonal)
- Ignores magnitude (only direction matters for semantic similarity)
- Computationally efficient
- Returns value between -1 and 1

### Endpoint Logic
```python
@app.post("/similarity")
async def similarity_search(body: RequestBody):
    # 1. Embed query
    query_emb = np.array(get_embedding(body.query))
    
    # 2. Embed all documents
    doc_embs = [np.array(get_embedding(doc)) for doc in body.docs]
    
    # 3. Calculate similarities
    similarities = [cosine_similarity(query_emb, emb) for emb in doc_embs]
    
    # 4. Get top 3 indices in descending order
    top_indices = np.argsort(similarities)[-3:][::-1]
    
    # 5. Return documents
    matches = [body.docs[i] for i in top_indices]
    return {"matches": matches}
```

### The Top 3 Selection Explained
```
np.argsort(similarities)  →  Sorts indices by value (ascending)
[-3:]                     →  Takes last 3 (highest values)
[::-1]                    →  Reverses (descending order)
```

**Example**:
- Similarities: `[0.45, 0.92, 0.67]`
- argsort() → `[0, 2, 1]`
- [-3:] → `[0, 2, 1]`
- [::-1] → `[1, 2, 0]`
- **Result**: Index 1 (0.92), Index 2 (0.67), Index 0 (0.45) ✓

---

## Service Flow (Step-by-Step)

| Step | What Happens | Example |
|------|-------------|---------|
| 1 | Client sends JSON with docs and query | `{"docs": [...], "query": "AI"}` |
| 2 | API embeds the query into a vector | Query → 512D vector |
| 3 | API embeds each document into a vector | 3 docs → 3 vectors |
| 4 | API calculates similarity (query vs each doc) | [0.45, 0.92, 0.67] |
| 5 | API ranks by similarity (highest first) | [1st: 0.92, 2nd: 0.67, 3rd: 0.45] |
| 6 | API returns top 3 document contents | `{"matches": [doc1, doc2, doc3]}` |

---

## Key Design Decisions

| Decision | Why | Trade-off |
|----------|-----|-----------|
| **Return document text** | User-friendly; no extra lookup needed | Larger JSON response |
| **Top 3 results** | Balance recall vs precision | Not configurable |
| **Sequential API calls** | Simple; straightforward | Slower for many docs |
| **Async endpoint** | Future-proof; FastAPI best practice | No immediate benefit |
| **Open CORS** | Internal tool | Security risk if public |

---

## Performance & Optimization

### Current Bottleneck
Each embedding requires an API call to OpenAI (~200ms per call)
- N documents + 1 query = **N+1 API calls**
- Total time = ~(N+1) × 200ms

### Optimization #1: Batch Embeddings
```python
def batch_embeddings(texts: List[str]) -> List[List[float]]:
    response = client.embeddings.create(input=texts, model="text-embedding-3-small")
    return [item.embedding for item in response.data]

# Use: docs_embs = batch_embeddings(body.docs)
# Reduces N+1 calls to 2 calls (1 for all docs, 1 for query)
```

### Optimization #2: Pre-computed Embeddings
Store embeddings in database; only embed query at request time
```
Initial: embed 1000 docs (one time) → store in DB
Queries: embed query + search DB (much faster)
```

### Optimization #3: Vector Database (Production Scale)
For 1M+ documents, use Pinecone, Weaviate, or Milvus
- Pre-store all embeddings
- Query-time lookup: O(1) instead of O(N)

---

## Real-World Example

### Request
```bash
curl -X POST "http://127.0.0.1:8000/similarity" \
  -H "Content-Type: application/json" \
  -d '{
    "docs": [
      "Python loops with for and while statements",
      "Machine learning algorithms for classification",
      "Data structures like lists and dictionaries"
    ],
    "query": "algorithms for machine learning"
  }'
```

### Response
```json
{
  "matches": [
    "Machine learning algorithms for classification",
    "Data structures like lists and dictionaries",
    "Python loops with for and while statements"
  ]
}
```

**Why this order?**
- Doc 2: Contains "algorithms" AND "machine learning" → highest similarity
- Doc 3: Contains "data structures" (useful for ML) → medium similarity
- Doc 1: About Python loops (tangential) → lowest similarity

---

## Setup Instructions

### 1. Install Dependencies
```bash
pip install fastapi uvicorn openai python-dotenv numpy
```

### 2. Create `.env` File
```
OPENAI_API_KEY=sk-...your-key-here...
```

### 3. Run Server
```bash
python main.py
```

### 4. Test Endpoint
```bash
# Swagger UI
http://127.0.0.1:8000/docs

# Direct request
curl -X POST "http://127.0.0.1:8000/similarity" \
  -H "Content-Type: application/json" \
  -d '{"docs": ["doc1", "doc2"], "query": "test"}'
```

---

## Current Limitations

| Issue | Impact | Fix |
|-------|--------|-----|
| No error handling | Crashes on API failure | Add try-except blocks |
| No input validation | Accepts empty docs/query | Validate before processing |
| Fixed top-3 limit | Not configurable | Add `num_results` parameter |
| Slow for many docs | N+1 API calls | Use batch embeddings |
| No authentication | Anyone can call API | Add API key validation |

---

## Recommended Improvements (v2.0)

```python
from fastapi import HTTPException, Query

@app.post("/similarity")
async def similarity_search(
    body: RequestBody,
    num_results: int = Query(3, ge=1, le=10)
):
    # Input validation
    if not body.docs:
        raise HTTPException(status_code=400, detail="docs array required")
    if not body.query.strip():
        raise HTTPException(status_code=400, detail="query cannot be empty")
    if num_results > len(body.docs):
        num_results = len(body.docs)
    
    try:
        # Batch embed all texts
        all_texts = [body.query] + body.docs
        all_embeddings = batch_embeddings(all_texts)
        
        query_emb = np.array(all_embeddings[0])
        doc_embs = [np.array(e) for e in all_embeddings[1:]]
        
        # Calculate and rank
        similarities = [cosine_similarity(query_emb, emb) for emb in doc_embs]
        top_indices = np.argsort(similarities)[-num_results:][::-1]
        
        matches = [body.docs[i] for i in top_indices]
        return {"matches": matches, "scores": [similarities[i] for i in top_indices]}
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Error: {str(e)}")
```

---

## Understanding Text Embeddings

**What**: Vectors that represent semantic meaning of text

**How**: 
```
"Apple is a fruit" →  [0.12, -0.45, 0.78, ..., 0.33]  (512 dimensions)
"Orange is a fruit" → [0.11, -0.46, 0.79, ..., 0.34]  (very similar!)
"JavaScript is a language" → [0.91, 0.12, -0.55, ..., 0.02]  (very different)
```

**Why**: Similar words have similar vectors → cosine similarity works!

---

## Testing Checklist

- [ ] Server starts without errors
- [ ] Swagger UI opens at `http://127.0.0.1:8000/docs`
- [ ] POST request with valid JSON returns status 200
- [ ] Response contains `{"matches": [...]}`
- [ ] Returned docs are in descending relevance order
- [ ] Most semantically relevant doc is first
- [ ] Query similar to doc[0] returns doc[0] first
- [ ] Works with 3+ documents
- [ ] CORS headers allow cross-origin requests
- [ ] Invalid requests return appropriate errors (after improvements)

---

## Summary Table

| Aspect | Details |
|--------|---------|
| **Technology** | FastAPI + OpenAI Embeddings + NumPy |
| **Time Complexity** | O(N) where N = number of documents |
| **Space Complexity** | O(N) for storing embeddings |
| **API Calls** | N+1 (can optimize to 2) |
| **Response Time** | ~200ms * (N+1) |
| **Production Ready** | ✓ With error handling additions |
| **Scalable** | ✓ With vector database |

---

## References

1. **FastAPI Docs**: https://fastapi.tiangolo.com/
2. **OpenAI Embeddings**: https://platform.openai.com/docs/guides/embeddings
3. **Cosine Similarity**: https://en.wikipedia.org/wiki/Cosine_similarity
4. **NumPy Linear Algebra**: https://numpy.org/doc/stable/reference/routines.linalg.html
5. **Vector Databases**: 
   - Pinecone: https://www.pinecone.io/
   - Weaviate: https://weaviate.io/
   - Milvus: https://milvus.io/

---

**Last Updated**: February 2026  
**Status**: Production-Ready (with recommended improvements)  
---
