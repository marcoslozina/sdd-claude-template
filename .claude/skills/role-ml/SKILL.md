---
name: role-ml
description: Production ML engineering: problem-to-approach mapping, data validation with pandera, embeddings-based classifiers, evaluation metrics, FastAPI model serving, and drift monitoring. Use when training, evaluating, deploying, or monitoring a model, or choosing a feature store, experiment tracker, or serving stack.
---

# Skill: Machine Learning Engineering

## Role
Integrate ML into production systems. Not just training models — deploying, monitoring, and maintaining them.

---

## Problem types → approach

| Problem | Recommended approach | When to scale up |
|----------|--------------------|--------------:|
| Text classification | LLM fine-tuning or embeddings + classifier | > 10k examples |
| Entity extraction | LLM with structured outputs | Always |
| Recommendation | Collaborative filtering + embeddings | > 100k users |
| Anomaly detection | Isolation Forest / Autoencoder | Depends on the domain |
| Forecasting | ARIMA / Prophet / LSTM | Depends on seasonality |
| Similarity search | Embeddings + vector DB | > 1k documents |

**Rule:** before training a model, check whether an LLM plus a good prompt solves the problem. It's faster and more maintainable.

---

## ML pipeline (production)

```
Raw Data → Preprocessing → Feature Engineering → Training → Evaluation → Serving
    ↓             ↓                ↓                ↓            ↓           ↓
  S3/DB       Validation       Feature Store     MLflow      Metrics     FastAPI/Lambda
```

### Project structure

```
ml/
  data/
    raw/               # original data, immutable
    processed/         # transformed data
    features/          # computed features
  notebooks/           # exploration (not production)
  src/
    data/              # loaders, preprocessors
    features/          # feature engineering
    models/            # training, evaluation
    serving/           # inference API
  tests/
  configs/             # hyperparameters, paths
```

---

## Data validation (before training)

```python
import pandera as pa
from pandera import Column, DataFrameSchema

schema = DataFrameSchema({
    "user_id": Column(str, nullable=False),
    "label": Column(int, pa.Check.isin([0, 1])),
    "text": Column(str, pa.Check(lambda x: x.str.len() > 0)),
    "created_at": Column("datetime64[ns]"),
})

def validate_training_data(df):
    try:
        schema.validate(df)
    except pa.errors.SchemaError as e:
        raise ValueError(f"Invalid training data: {e}")
```

---

## Embeddings for ML with the Claude API

```python
import voyageai

voyage = voyageai.Client()

def embed_texts(texts: list[str], batch_size: int = 128) -> list[list[float]]:
    embeddings = []
    for i in range(0, len(texts), batch_size):
        batch = voyage.embed(texts[i:i+batch_size], model="voyage-3").embeddings
        embeddings.extend(batch)
    return embeddings

# For downstream classification
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import cross_val_score

X = embed_texts(texts)
clf = LogisticRegression(max_iter=1000)
scores = cross_val_score(clf, X, labels, cv=5, scoring='f1_weighted')
print(f"F1: {scores.mean():.3f} ± {scores.std():.3f}")
```

---

## Evaluation — metrics per problem type

```python
from sklearn.metrics import (
    classification_report,
    roc_auc_score,
    mean_absolute_error,
    ndcg_score
)

# Classification
print(classification_report(y_true, y_pred))
print(f"AUC-ROC: {roc_auc_score(y_true, y_prob):.3f}")

# Regression
print(f"MAE: {mean_absolute_error(y_true, y_pred):.3f}")

# Ranking / Recommendation
print(f"NDCG@10: {ndcg_score(y_true, y_scores, k=10):.3f}")
```

**Don't report accuracy alone.** For imbalanced data, use F1 or AUC-ROC.

---

## Model serving

### FastAPI + in-memory model

```python
from fastapi import FastAPI
from pydantic import BaseModel
import joblib

app = FastAPI()
model = joblib.load("model.pkl")
embedder = VoyageEmbedder()

class PredictRequest(BaseModel):
    text: str

class PredictResponse(BaseModel):
    label: int
    confidence: float

@app.post("/predict", response_model=PredictResponse)
async def predict(req: PredictRequest):
    embedding = embedder.embed([req.text])[0]
    proba = model.predict_proba([embedding])[0]
    return PredictResponse(
        label=int(proba.argmax()),
        confidence=float(proba.max())
    )
```

### When to use which serving option

| Scale | Latency | Solution |
|--------|---------|----------|
| < 100 req/s | < 200ms | FastAPI on ECS/Lambda |
| > 100 req/s | < 100ms | FastAPI + uvicorn workers |
| Async batch | Not critical | Lambda + SQS + S3 |
| Large models | Variable | SageMaker Endpoints |

---

## Production monitoring

```python
# Log predictions to detect drift
import logging

logger = logging.getLogger("ml.inference")

def predict_with_logging(text: str, model_version: str) -> dict:
    result = model.predict(text)
    logger.info({
        "input_length": len(text),
        "prediction": result["label"],
        "confidence": result["confidence"],
        "model_version": model_version,
        "latency_ms": result["latency"]
    })
    return result
```

**Metrics to monitor:**
- Feature distribution drift (PSI, KL divergence)
- Metric degradation in production vs holdout
- Inference latency (p50, p95, p99)
- Rate of low-confidence predictions

---

## Common ML Engineering decisions

Apply the decision protocol from CLAUDE.md when facing:
- **Build vs Buy:** train your own model vs LLM + prompting vs third-party API
- **Feature store:** Feast vs Tecton vs custom on Redis
- **Experiment tracking:** MLflow vs W&B vs Comet
- **Serving:** custom FastAPI vs SageMaker vs BentoML vs Ray Serve
- **Vector DB:** pgvector vs Pinecone vs Weaviate vs Chroma
- **Retraining:** scheduled vs drift-triggered vs online learning
