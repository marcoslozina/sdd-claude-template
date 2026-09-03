---
name: role-app-performance
description: Finding and fixing real application bottlenecks — profiling (Python, Go, Java, Node), Big O red flags, N+1 queries, SQL indexing, caching strategies, connection pooling, memory management and HTTP/pagination tuning. Use when an app is slow, when designing for latency or throughput targets, or when reviewing code that touches the DB, the network, or hot loops.
---

# Skill: Application Performance

## Role
Identify and eliminate real bottlenecks, not perceived ones. Without profiling there is no optimization —
performance intuition is wrong 90% of the time. Measure first, optimize afterwards.

## When to activate this skill
- The application is slow and nobody knows why
- You're about to design a system with latency or throughput requirements
- Code review of code that touches the DB, the network, or intensive processing
- N+1 queries, missing caching, or wrong data structures are spotted

---

## Golden rule

```
NEVER optimize without measuring first.
Profile → identify the real bottleneck → optimize → measure again.
```

Premature optimization is the root of all evil (Knuth). Slow and correct code
beats fast and incorrect code.

---

## Algorithmic complexity — Big O

| Complexity | Name | Scale at N=1M |
|-------------|--------|-----------------|
| O(1) | Constant | 1 op |
| O(log n) | Logarithmic | ~20 ops |
| O(n) | Linear | 1,000,000 ops |
| O(n log n) | Linearithmic | ~20,000,000 ops |
| O(n²) | Quadratic | 1,000,000,000,000 ops — DANGER |
| O(2ⁿ) | Exponential | Does not scale |

**Warning signs in code:**
```python
# ❌ O(n²) — loop inside a loop over collections
for user in users:
    for order in orders:  # if orders grows, this blows up
        if order.user_id == user.id: ...

# ✅ O(n) — O(1) lookup with a dict/map
orders_by_user = {o.user_id: o for o in orders}
for user in users:
    order = orders_by_user.get(user.id)
```

---

## Profiling per language

### Python
```bash
# CPU profiling
python -m cProfile -s cumulative app.py
python -m cProfile -o output.prof app.py && snakeviz output.prof

# Memory profiling
pip install memory-profiler
@profile  # decorator on the function to measure
python -m memory_profiler app.py

# Line-by-line
pip install line_profiler
@profile
def hot_function(): ...
kernprof -l -v app.py
```

### Go
```go
import _ "net/http/pprof"
// Expose on :6060/debug/pprof
go tool pprof http://localhost:6060/debug/pprof/profile
go tool pprof http://localhost:6060/debug/pprof/heap
```

### Java
```bash
# JVM flags for profiling
-XX:+FlightRecorder -XX:StartFlightRecording=duration=60s,filename=app.jfr
# Visualize with JDK Mission Control
```

### TypeScript/Node
```bash
node --prof app.js
node --prof-process isolate-*.log > processed.txt
# Or use clinic.js
npx clinic doctor -- node app.js
```

---

## Database performance

### N+1 — the most common problem

```python
# ❌ N+1: 1 query for users + N queries for orders
users = db.query(User).all()
for user in users:
    print(user.orders)  # one query per user

# ✅ Eager loading: 2 queries total
users = db.query(User).options(joinedload(User.orders)).all()
```

### Indexing — base rules

```sql
-- Index columns used in WHERE, JOIN, ORDER BY
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

-- Composite index: most selective column first
CREATE INDEX idx_orders_status_user ON orders(status, user_id);

-- EXPLAIN to verify the index is actually used
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123;
```

### Query optimization

```sql
-- ❌ SELECT * brings back unnecessary columns
SELECT * FROM users WHERE active = true;

-- ✅ Only the columns you need
SELECT id, email, name FROM users WHERE active = true;

-- ❌ LIKE with a leading wildcard cannot use an index
SELECT * FROM users WHERE email LIKE '%@gmail.com';

-- ✅ Wildcard only at the end uses the index
SELECT * FROM users WHERE email LIKE 'marco%';
```

---

## Caching — strategies

| Strategy | When | Recommended TTL |
|-----------|--------|-----------------|
| **Cache-aside** | Frequent reads, data changes rarely | Minutes-hours |
| **Write-through** | Consistency is critical | No TTL, invalidate on write |
| **Write-behind** | Frequent writes, eventual consistency is fine | Seconds |
| **Read-through** | Transparent to the client | Minutes |

```python
# Cache-aside with Redis
def get_user(user_id: str) -> User:
    cached = redis.get(f"user:{user_id}")
    if cached:
        return User.from_json(cached)

    user = db.query(User).filter_by(id=user_id).first()
    redis.setex(f"user:{user_id}", ttl=300, value=user.to_json())
    return user

# Invalidate on writes
def update_user(user_id: str, data: dict) -> User:
    user = db.update(user_id, data)
    redis.delete(f"user:{user_id}")  # invalidate cache
    return user
```

---

## Connection pooling

```python
# ❌ A new connection per request — expensive
def get_user(id):
    conn = psycopg2.connect(DATABASE_URL)  # costly
    ...
    conn.close()

# ✅ Pool of reusable connections
from sqlalchemy import create_engine
engine = create_engine(DATABASE_URL, pool_size=10, max_overflow=20)
```

```go
// Go: configure the pool on sql.DB
db, _ := sql.Open("postgres", dsn)
db.SetMaxOpenConns(25)
db.SetMaxIdleConns(10)
db.SetConnMaxLifetime(5 * time.Minute)
```

---

## Memory management

### Python
```python
# Generators instead of lists for large volumes
# ❌ Loads everything into memory
def get_all_users():
    return db.query(User).all()  # 1M users in RAM

# ✅ Streaming with a generator
def get_all_users():
    yield from db.query(User).yield_per(1000)
```

### Go
```go
// Reuse buffers with sync.Pool
var bufPool = sync.Pool{
    New: func() interface{} { return new(bytes.Buffer) },
}

func process() {
    buf := bufPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufPool.Put(buf)
    }()
}
```

---

## HTTP performance

```typescript
// Compression — reduce response size
app.use(compression())

// HTTP caching headers
res.set('Cache-Control', 'public, max-age=3600')
res.set('ETag', hash(data))

// Cursor pagination (more efficient than offset on large tables)
// ❌ OFFSET scales badly
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 10000;

// ✅ Cursor-based pagination
SELECT * FROM orders WHERE id > :last_id ORDER BY id LIMIT 20;
```

---

## Performance review checklist

- [ ] Are there nested loops over large collections? (sign of O(n²))
- [ ] Do the queries have indexes on the WHERE and JOIN columns?
- [ ] Is SELECT * being used where it shouldn't?
- [ ] Are there N+1 queries in operations over lists?
- [ ] Is frequently read data cached?
- [ ] Is the cache invalidated correctly on writes?
- [ ] Is connection pooling configured?
- [ ] Are I/O-bound operations asynchronous?
- [ ] Did you profile before optimizing?
