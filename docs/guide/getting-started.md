# Getting Started

## Installation

### Using Go Install

```bash
go install github.com/GiGurra/gocc@latest
```

### From Source

```bash
git clone https://github.com/GiGurra/gocc.git
cd gocc
go build .
```

### Docker

```bash
docker pull ghcr.io/gigurra/gocc:latest
docker run -p 8080:8080 ghcr.io/gigurra/gocc:latest
```

## Running GoCC

### Basic Usage

Start with default settings (100 requests/second per key):

```bash
gocc
```

### Custom Settings

```bash
gocc --max-requests 200 --window-millis 5000 --port 9090
```

### View All Options

```bash
gocc --help
```

## Your First Rate Limit

With GoCC running, let's test rate limiting:

```bash
# First request - approved
curl -X POST http://localhost:8080/rate/test-key
# {"request_id":"abc123..."}

# Make many requests quickly
for i in {1..150}; do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:8080/rate/test-key
done
# First 100: 200
# Rest: 429
```

## Using the Queue

When rate limited, clients can wait in a queue:

```bash
# This will wait if rate limited (instead of returning 429)
curl -X POST "http://localhost:8080/rate/test-key?canWait=true"
```

## Debug Endpoints

View the state of all rate limiters:

```bash
curl http://localhost:8080/debug | jq
```

View a specific key:

```bash
curl http://localhost:8080/debug/test-key | jq
```

## Health Check

```bash
curl http://localhost:8080/healthz
# OK
```

## Next Steps

- [Configuration](configuration.md) - CLI flags, environment variables, and config files
- [Queueing](queueing.md) - How FIFO queueing works
- [Architecture](../design/architecture.md) - How GoCC works internally
