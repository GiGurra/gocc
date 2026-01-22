# Performance

GoCC is designed for high throughput while maintaining simplicity.

## Benchmarks

Measured on Apple M4 Pro laptop:

### Internal (No HTTP)

| Metric | Value |
|--------|-------|
| Throughput | ~6 million req/s |
| Latency | ~150 ns/request |

This represents the raw actor model performance without network overhead.

### With HTTP

| Protocol | Endpoint | Throughput |
|----------|----------|------------|
| HTTP/2 | `/rate/:key` | ~350-450k req/s |
| HTTP/2 | `/healthz` | ~500k req/s |
| HTTP/1.1 | `/rate/:key` | ~80k req/s |
| HTTP/1.1 | `/healthz` | ~85k req/s |

### Bottleneck Analysis

```
Component           Time per Request
─────────────────────────────────────
Rate limiter logic:     ~150 ns
HTTP/2 overhead:       ~2000 ns
HTTP/1.1 overhead:    ~12000 ns
Network (typical):    ~1-10 ms
```

The HTTP layer is the bottleneck, not the rate limiting logic.

## Why HTTP/2 is Faster

| Feature | HTTP/1.1 | HTTP/2 |
|---------|----------|--------|
| Connections | One request per connection (or keep-alive) | Multiplexed streams |
| Headers | Text, repeated | Binary, compressed (HPACK) |
| Pipelining | Not reliable | Native multiplexing |

Use HTTP/2 clients for best performance:
- Go: `http.Client` with HTTP/2 transport
- CLI: `h2load` for benchmarking

## Theoretical Limits

### Network Bandwidth

Assuming 100 Mbit/s per pod:
- Request size: ~300-500 bytes
- Max theoretical: ~25-33k req/s

In practice, local/loopback is much faster.

### Actor Model

With pipelining/batching (not implemented):
- Theoretical: ~300 million req/s
- See [snail](https://github.com/GiGurra/snail) for batched networking

GoCC uses strict request→response without batching for simplicity.

## Optimization Decisions

### What We Optimized

1. **HTTP/2 by default** - 4-5x faster than HTTP/1.1
2. **Sharded managers** - Parallel processing across 25 shards
3. **Direct response path** - Responses bypass manager
4. **Lazy instance creation** - No overhead for unused keys
5. **Auto-expiration** - Memory reclaimed for idle keys

### What We Didn't Optimize

1. **No batching** - Simplicity over maximum throughput
2. **No custom protocol** - HTTP compatibility
3. **No connection pooling** - Clients handle this
4. **No pre-allocation** - Go's GC handles it well

## Scaling Strategies

### Vertical Scaling

More CPU cores = more parallelism:
- Manager shards run in parallel
- Each instance is independent
- HTTP server handles concurrent requests

### Horizontal Scaling

Multiple GoCC instances:
- Deploy as Kubernetes StatefulSet
- Consistent hashing distributes keys
- No coordination overhead
- See [Kubernetes Deployment](../operations/kubernetes.md)

## Comparison with Alternatives

### vs. In-Process Rate Limiting

```go
// In-process (e.g., golang.org/x/time/rate)
limiter := rate.NewLimiter(100, 10)
if !limiter.Allow() {
    return errors.New("rate limited")
}
```

| Aspect | In-Process | GoCC |
|--------|------------|------|
| Latency | ~10 ns | ~2 μs (HTTP/2) |
| Deployment | Per-service | Centralized |
| Configuration | Code change | Hot-reload |
| Cross-service | No | Yes |

### vs. Redis-Based

```bash
# Redis rate limiting
INCR rate:user-123
EXPIRE rate:user-123 1
```

| Aspect | Redis | GoCC |
|--------|-------|------|
| Persistence | Yes | No (in-memory) |
| Latency | ~1 ms | ~2 μs (local) |
| Queueing | Limited | Native FIFO |
| Dependencies | Redis server | None |

### vs. API Gateway Rate Limiting

| Aspect | API Gateway | GoCC |
|--------|-------------|------|
| Latency | Varies | Low |
| Configuration | Gateway-specific | Universal |
| Queueing | Usually no | Yes |
| Protocol | HTTP only | HTTP (extensible) |

## Benchmarking Tools

### HTTP/1.1

```bash
wrk -d10s -t4 -c100 http://localhost:8080/rate/x
```

### HTTP/2

```bash
h2load -c100 -n1000000 -m100 http://localhost:8080/rate/x
```

### Internal

```bash
gocc benchmark actors
```

## Performance Tips

### For Clients

1. **Use HTTP/2** - 4-5x faster than HTTP/1.1
2. **Reuse connections** - Avoid connection setup overhead
3. **Set timeouts** - Don't wait forever for queued requests
4. **Batch where possible** - Combine related operations

### For Operators

1. **Deploy close to clients** - Minimize network latency
2. **Size instances appropriately** - More CPU = more throughput
3. **Monitor queue depth** - High queues indicate underprovisioning
4. **Use appropriate window sizes** - Smaller windows = faster feedback
