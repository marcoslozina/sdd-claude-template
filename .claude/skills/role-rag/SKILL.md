---
name: role-rag
description: RAG pipeline design: chunking sizes, embedding model choice, vector search, summary indexing, contextual embeddings, Claude re-ranking, precision/recall/MRR evaluation, and prompt caching for static docs. Use when building or debugging retrieval over a vector store, or tuning answer quality and grounding.
---

# Skill: RAG (Retrieval-Augmented Generation)

> Based on the official Anthropic documentation: https://docs.anthropic.com/en/docs/build-with-claude/rag

## What RAG is and why it exists

LLMs have two limitations: knowledge frozen at the training cutoff date,
and no access to private or up-to-date information. RAG solves both:

```
Without RAG: "What's the status of ticket PROJ-4521?" → it guesses or doesn't know

With RAG:
  1. Look the ticket up in Jira
  2. Pass the content + the question to the model
  → It answers with real information
```

---

## RAG pipeline — 3 components

### 1. Ingestion and chunking

```python
def chunk_document(text: str, chunk_size: int = 300, overlap: int = 50) -> list[str]:
    words = text.split()
    chunks = []
    for i in range(0, len(words), chunk_size - overlap):
        chunk = " ".join(words[i:i + chunk_size])
        chunks.append(chunk)
    return chunks
```

**Recommended sizes by content type:**
| Content | Chunk size | Overlap |
|-----------|-----------|---------|
| Technical documentation | 200-400 tokens | 50 tokens |
| Source code | Per function/class | Minimal |
| Conversations | Per turn | No overlap |
| Articles | Per paragraph | 1-2 sentences |

**Rule:** small chunks lose context, large chunks waste the window. Test with your own data.

---

### 2. Embeddings and semantic search

```python
import voyageai
import numpy as np

class VectorDB:
    def __init__(self):
        self.client = voyageai.Client()
        self.embeddings: list = []
        self.metadata: list = []
        self.query_cache: dict = {}

    def index(self, chunks: list[dict]) -> None:
        texts = [f"{c['heading']}\n{c['text']}" for c in chunks]
        # Batches of 128 for efficiency
        all_embeddings = []
        for i in range(0, len(texts), 128):
            batch = self.client.embed(texts[i:i+128], model="voyage-3").embeddings
            all_embeddings.extend(batch)
        self.embeddings = all_embeddings
        self.metadata = chunks

    def search(self, query: str, k: int = 5, threshold: float = 0.75) -> list[dict]:
        # Cache for repeated queries
        if query not in self.query_cache:
            self.query_cache[query] = self.client.embed([query], model="voyage-3").embeddings[0]

        q_emb = self.query_cache[query]
        similarities = np.dot(self.embeddings, q_emb)
        top_indices = np.argsort(similarities)[::-1]

        return [
            {"text": self.metadata[i]["text"], "score": similarities[i]}
            for i in top_indices
            if similarities[i] >= threshold
        ][:k]
```

**Recommended embedding models:**
- General: `voyage-3` (Voyage AI, recommended by Anthropic)
- Code: `voyage-code-3`
- Multilingual: `voyage-multilingual-2`

---

### 3. Generation with context

```python
import anthropic

client = anthropic.Anthropic()
db = VectorDB()

def answer(question: str) -> str:
    # Retrieve relevant context
    results = db.search(question, k=3)
    context = "\n\n---\n\n".join([r["text"] for r in results])

    response = client.messages.create(
        model="claude-sonnet-5",
        max_tokens=1024,
        system="""You answer questions ONLY with the provided context.
If the information is not in the context, say explicitly that you don't have it.
Do not make up answers or use outside knowledge.""",
        messages=[{
            "role": "user",
            "content": f"<context>\n{context}\n</context>\n\nQuestion: {question}"
        }]
    )
    return response.content[0].text
```

---

## Advanced techniques (they improve precision)

### Summary Indexing
Index summaries and full chunks separately.
Search over the summaries, return the full chunk.
This narrows the search space before ranking, which tends to help precision on large
corpora — measure it on your own data rather than assuming a fixed improvement number.

### Contextual Embeddings
Use Claude to generate situational context before embedding each chunk:

```python
def add_context_to_chunk(chunk: str, doc_summary: str) -> str:
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",  # cheap for this step
        max_tokens=200,
        messages=[{
            "role": "user",
            "content": f"""Document: {doc_summary}
Chunk: {chunk}
Write one sentence that situates this chunk within the document."""
        }]
    )
    return f"{response.content[0].text}\n\n{chunk}"
```

### Re-ranking with Claude
Retrieve the top-10, reorder by actual relevance before passing them to the model:

```python
def rerank(query: str, candidates: list[str], top_k: int = 3) -> list[str]:
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=200,
        messages=[{
            "role": "user",
            "content": f"""Query: {query}
Candidates: {json.dumps(list(enumerate(candidates)))}
Return the indices of the {top_k} most relevant ones, in order, as a JSON array."""
        }]
    )
    indices = json.loads(response.content[0].text)
    return [candidates[i] for i in indices]
```

---

## Evaluation metrics

```python
# Precision: of what I retrieved, how much was correct?
# Recall: of what was correct, how much did I retrieve?
# MRR: how high up does the first correct result appear?

def mrr(retrieved: list[str], relevant: set[str]) -> float:
    for i, item in enumerate(retrieved, 1):
        if item in relevant:
            return 1 / i
    return 0.0

def precision_recall(retrieved: list[str], relevant: set[str]) -> tuple[float, float]:
    hits = len(set(retrieved) & relevant)
    return hits / len(retrieved), hits / len(relevant)
```

**What Anthropic actually published (Contextual Retrieval, Sept 2024 — measured as
top-20-chunk retrieval failure rate, not precision/recall/MRR — treat the numbers
above as your own metrics to compute, not as this benchmark's):**
- Baseline (embeddings only): 5.7% failure rate
- + Contextual Embeddings: 3.7% (-35% relative)
- + Contextual Embeddings + Contextual BM25: 2.9% (-49% relative)
- + Contextual Embeddings + Contextual BM25 + Reranking: 1.9% (-67% relative)

Numbers depend heavily on your corpus and query distribution — re-run this kind of
evaluation on your own data before trusting any published figure, including this one.

---

## When RAG is not the solution

RAG works well for information retrieval.
It works badly when:
- The question requires reasoning over information that isn't in the docs
- The knowledge base holds contradictory information (a governance problem)
- The question requires real-time data (use tool use + an API instead)

---

## Integration with Prompt Caching

For RAG with repeated context (the same document base):

```python
system = [
    {"type": "text", "text": "You answer based only on the provided context."},
    {
        "type": "text",
        "text": f"<knowledge_base>{static_docs}</knowledge_base>",
        "cache_control": {"type": "ephemeral"}  # caches the static docs
    }
]
```

**Saves up to 90% of input tokens for docs that repeat across requests.**

---

## Common RAG decisions

Apply the decision protocol (this project's CLAUDE.md if it defines one, otherwise dev-harness's docs/DECISION_PROTOCOL.md) when facing:
- **Vector DB:** Pinecone vs Weaviate vs Chroma vs pgvector
- **Embedding model:** Voyage AI vs OpenAI Ada vs Cohere
- **Chunk strategy:** fixed vs semantic vs by document structure
- **Reranking:** cross-encoder vs Claude vs hybrid BM25
- **Doc governance:** versioning, updates, deduplication
