# GoCC - Gopher Cruise Control

An in-memory rate limiter service with FIFO queueing, built on the actor model.

[![CI Status](https://github.com/gigurra/gocc/actions/workflows/ci.yml/badge.svg)](https://github.com/gigurra/gocc/actions/workflows/ci.yml)
[![Go Report Card](https://goreportcard.com/badge/github.com/gigurra/gocc)](https://goreportcard.com/report/github.com/gigurra/gocc)
[![Docs](https://img.shields.io/badge/docs-GitHub%20Pages-blue)](https://gigurra.github.io/gocc/)

## What is GoCC?

GoCC is a stand-alone rate limiting service. Applications query it via HTTP to check if they're allowed to perform an operation. If the rate limit is exceeded, clients can optionally wait in a FIFO queue.

```
POST /rate/my-key     → 200 OK (approved)
POST /rate/my-key     → 200 OK (approved)
POST /rate/my-key     → 429 Too Many Requests (denied)
POST /rate/my-key?canWait=true → [waits in queue] → 200 OK
```

**Performance**: ~450k req/s over HTTP/2, ~6M req/s internal processing

## Quick Start

```bash
# Install
go install github.com/GiGurra/gocc@latest

# Run with defaults (port 8080, 100 req/s per key)
gocc

# Or with custom settings
gocc --max-requests 200 --window-millis 5000 --port 9090
```

## Basic Usage

```bash
# Rate limit a key
curl -X POST "http://localhost:8080/rate/user-123"
# Returns: 200 OK with request ID, or 429 if limit exceeded

# Rate limit with queueing (waits if limit exceeded)
curl -X POST "http://localhost:8080/rate/user-123?canWait=true"

# Release a request early
curl -X DELETE "http://localhost:8080/rate/user-123/{request-id}"

# Debug: view all limiters
curl http://localhost:8080/debug | jq

# Health check
curl http://localhost:8080/healthz
```

## Key Features

- **Fixed time window** rate limiting with configurable window size
- **FIFO queueing** when limits are exceeded (`?canWait=true`)
- **Per-key configuration** via JSON config file with regex patterns
- **Hot-reloadable config** - no restart needed
- **Actor model** architecture - one goroutine per key
- **Distributed mode** - deploy as Kubernetes StatefulSet
- **HTTP/2 support** for better performance

## Configuration

### CLI Flags

| Flag | Default | Description |
|------|---------|-------------|
| `-m, --max-requests` | 100 | Max requests per window |
| `-w, --window-millis` | 1000 | Window size in ms |
| `--max-requests-in-queue` | 400 | Queue size after window full |
| `-p, --port` | 8080 | Listen port |
| `-s, --server-type` | echo-http2 | echo, echo-http2, or fast |
| `-c, --config-file` | "" | Path to JSON config |

All flags support environment variables (e.g., `MAX_REQUESTS`, `WINDOW_MILLIS`).

### Config File

Override settings per key with regex patterns:

```json
{
  "keys": [
    {
      "key_pattern": "^premium-.*",
      "key_pattern_is_regex": true,
      "max_requests_per_window": 1000,
      "window_millis": 1000
    },
    {
      "key_pattern": "guest",
      "max_requests_per_window": 10,
      "max_requests_in_queue": 5
    }
  ]
}
```

## Architecture

GoCC uses the actor model with sharded managers:

```
HTTP Request → Manager Shard → Rate Limiter Instance → Response
                  (1 of 25)       (per unique key)
```

- Each key gets its own goroutine
- Managers shard by key hash (FNV-1a)
- No shared state, no locks in hot path
- Instances auto-expire after 3 windows of inactivity

## Origin

Originally developed as a hack project at [Kivra](https://github.com/kivra/gocc), later open-sourced under MIT license.

The exploration of custom transport protocols for GoCC led to a separate project: [snail](https://github.com/GiGurra/snail) - achieving 300M+ req/s with batching.

## Documentation

Full documentation at **[gigurra.github.io/gocc](https://gigurra.github.io/gocc/)**

- [Getting Started](https://gigurra.github.io/gocc/guide/getting-started/)
- [Configuration](https://gigurra.github.io/gocc/guide/configuration/)
- [API Reference](https://gigurra.github.io/gocc/api/endpoints/)
- [Deployment](https://gigurra.github.io/gocc/operations/deployment/)
- [Architecture](https://gigurra.github.io/gocc/design/architecture/)

## License

MIT
