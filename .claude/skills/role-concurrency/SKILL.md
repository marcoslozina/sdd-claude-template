---
name: role-concurrency
description: Writing correct concurrent code — race conditions and detection (`go test -race`), deadlock prevention via lock ordering, I/O-bound vs CPU-bound choices, async/await in Python, TypeScript, Go and Java, worker pools and backpressure, safe concurrency patterns, and loop-closure bugs. Use when working with goroutines, threads, async tasks, shared state, queues, background workers, or reviewing code with locks.
---

# Skill: Concurrency and Parallelism

## Role
Write correct concurrent code, not just fast concurrent code.
A silent race condition in production is worse than slow code.
Correctness first, performance second.

## When to activate this skill
- The system handles multiple simultaneous requests
- There is shared state between goroutines, threads, or async tasks
- Workers, queues, or background processing are in play
- You're designing a system with I/O-bound or CPU-bound operations
- Code review of code with `async/await`, goroutines, threads, or locks

---

## Fundamental concepts

### Concurrency vs Parallelism
```
Concurrency:  multiple tasks in progress at the same time (can be on 1 CPU)
Parallelism:  multiple tasks executing simultaneously (requires multiple CPUs)

Concurrency is about DESIGN (program structure).
Parallelism is about EXECUTION (hardware).
```

### I/O-bound vs CPU-bound
| Type | Bottleneck | Solution |
|------|-----------|----------|
| **I/O-bound** | Waiting on network, disk, DB | `async/await`, threads, goroutines |
| **CPU-bound** | Intensive processing | Multiprocessing, real multiple CPUs |

> In Python: the GIL prevents real parallelism with threads for CPU-bound work → use `multiprocessing`.
> In Go/Java: threads/goroutines give real parallelism for both types.

---

## Race Conditions — detection and prevention

```go
// ❌ Race condition: two goroutines read-modify-write
var counter int
go func() { counter++ }()
go func() { counter++ }()
// counter may end up as 1 instead of 2

// ✅ Option A: atomic operations
var counter atomic.Int64
go func() { counter.Add(1) }()

// ✅ Option B: mutex
var mu sync.Mutex
var counter int
go func() {
    mu.Lock()
    counter++
    mu.Unlock()
}()
```

```bash
# Detect race conditions in Go
go test -race ./...
go run -race main.go
```

---

## Deadlocks — patterns and prevention

```go
// ❌ Classic deadlock: A waits on B, B waits on A
func transfer(from, to *Account, amount int) {
    from.mu.Lock()
    to.mu.Lock()   // goroutine A locks from, waits on to
                   // goroutine B locks to, waits on from → DEADLOCK
    from.balance -= amount
    to.balance += amount
    to.mu.Unlock()
    from.mu.Unlock()
}

// ✅ Order the locks consistently (always in the same order)
func transfer(from, to *Account, amount int) {
    first, second := from, to
    if from.id > to.id {
        first, second = to, from
    }
    first.mu.Lock()
    second.mu.Lock()
    // ...
}
```

---

## Async/Await per language

### Python (asyncio)
```python
import asyncio
import httpx

# ❌ Sequential — 3 requests = sum of the times
async def fetch_all_slow():
    a = await fetch("url_a")
    b = await fetch("url_b")  # waits for a to finish
    c = await fetch("url_c")  # waits for b to finish

# ✅ Parallel — 3 requests = time of the slowest one
async def fetch_all_fast():
    async with httpx.AsyncClient() as client:
        a, b, c = await asyncio.gather(
            client.get("url_a"),
            client.get("url_b"),
            client.get("url_c"),
        )

# CPU-bound: use ProcessPoolExecutor (not threads — GIL)
from concurrent.futures import ProcessPoolExecutor
async def cpu_heavy_task(data):
    loop = asyncio.get_event_loop()
    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, process, data)
```

### TypeScript/Node
```typescript
// ❌ Sequential
const user = await getUser(id)
const orders = await getOrders(id)  // waits for getUser to finish

// ✅ Parallel with Promise.all
const [user, orders] = await Promise.all([
  getUser(id),
  getOrders(id),
])

// ✅ With individual error handling
const results = await Promise.allSettled([
  getUser(id),
  getOrders(id),
])
results.forEach(r => {
  if (r.status === 'fulfilled') use(r.value)
  else log(r.reason)
})
```

### Go (goroutines + channels)
```go
// ✅ Fan-out: spread work across goroutines
func processAll(items []Item) []Result {
    results := make(chan Result, len(items))

    for _, item := range items {
        go func(i Item) {
            results <- process(i)
        }(item)  // pass by value — avoids the closure bug
    }

    out := make([]Result, 0, len(items))
    for range items {
        out = append(out, <-results)
    }
    return out
}

// ✅ Context for cancellation
func fetchWithTimeout(ctx context.Context, url string) ([]byte, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    resp, err := http.DefaultClient.Do(req)
    // If the context is cancelled, the request is interrupted automatically
}
```

### Java (CompletableFuture)
```java
// ✅ Parallel with CompletableFuture
CompletableFuture<User> userFuture = CompletableFuture
    .supplyAsync(() -> userRepo.findById(id));

CompletableFuture<List<Order>> ordersFuture = CompletableFuture
    .supplyAsync(() -> orderRepo.findByUserId(id));

CompletableFuture.allOf(userFuture, ordersFuture).join();
User user = userFuture.get();
List<Order> orders = ordersFuture.get();

// ✅ Virtual threads (Java 21+) — I/O-bound without callbacks
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    Future<User> user = executor.submit(() -> userRepo.findById(id));
    Future<List<Order>> orders = executor.submit(() -> orderRepo.findByUserId(id));
}
```

---

## Work Queues and Backpressure

```python
# ✅ Worker pool with asyncio — cap the concurrency
async def process_with_limit(items, max_workers=10):
    semaphore = asyncio.Semaphore(max_workers)

    async def bounded_process(item):
        async with semaphore:  # 10 in parallel at most
            return await process(item)

    return await asyncio.gather(*[bounded_process(i) for i in items])
```

```go
// ✅ Worker pool in Go
func workerPool(jobs <-chan Job, results chan<- Result, workers int) {
    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- process(job)
            }
        }()
    }
    wg.Wait()
    close(results)
}
```

---

## Safe concurrency patterns

| Pattern | When | Mechanism |
|---------|--------|-----------|
| **Immutability** | State that never changes | No locks needed |
| **Message passing** | Communication between workers | Channels (Go), queues |
| **Actor model** | State isolated per actor | Akka (Java), erlang-style |
| **Thread-local storage** | Per-thread state without locks | `threading.local()` (Python) |
| **Copy-on-write** | Frequent reads, rare writes | Snapshot before modifying |

---

## Closures in loops — the classic bug

```typescript
// ❌ Every callback captures the same `i`
for (var i = 0; i < 5; i++) {
  setTimeout(() => console.log(i), 100)  // prints 5,5,5,5,5
}

// ✅ let creates a scope per iteration
for (let i = 0; i < 5; i++) {
  setTimeout(() => console.log(i), 100)  // prints 0,1,2,3,4
}
```

```go
// ❌ The goroutine captures a reference to the loop variable
for _, v := range items {
    go func() { process(v) }()  // every goroutine sees the last v
}

// ✅ Pass by value
for _, v := range items {
    go func(item Item) { process(item) }(v)
}
```

---

## Concurrency checklist

- [ ] Is all shared state protected with a mutex, atomics, or channels?
- [ ] Is `go test -race` (or the equivalent) being run?
- [ ] Do the goroutines/tasks have a way to be cancelled via context/signal?
- [ ] Are I/O-bound operations async? Do CPU-bound ones use real workers?
- [ ] Do loops with goroutines/closures pass variables by value?
- [ ] Is there a concurrency limit (semaphore/pool) to avoid saturating resources?
- [ ] Are locks always acquired in the same order? (deadlock prevention)
- [ ] Are locks released with `defer` to guarantee release on panics/errors?
