# API Endpoints

GoCC exposes a simple REST API for rate limiting.

## Rate Limiting

### POST /rate/:key

Check rate limit for a key. Creates the key if it doesn't exist.

**Parameters:**

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `key` | path | string | Yes | Rate limit key |
| `canWait` | query | bool | No | Wait in queue if rate limited |
| `maxRequests` | query | int | No | Override max requests per window |
| `maxRequestsInQueue` | query | int | No | Override queue size |

**Responses:**

| Code | Description |
|------|-------------|
| 200 | Request approved |
| 429 | Rate limit exceeded |
| 499 | Client disconnected while waiting |

**Example:**

```bash
# Basic rate limit check
curl -X POST http://localhost:8080/rate/user-123

# With queueing
curl -X POST "http://localhost:8080/rate/user-123?canWait=true"

# With custom rate
curl -X POST "http://localhost:8080/rate/user-123?maxRequests=200&maxRequestsInQueue=500"
```

**Response Body (200 OK):**

```json
{
  "request_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

### GET /rate/:key

Same as POST. Useful for simple testing.

```bash
curl http://localhost:8080/rate/user-123
```

### DELETE /rate/:key/:requestId

Release a request before the window expires. Decrements the counter.

**Parameters:**

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `key` | path | string | Yes | Rate limit key |
| `requestId` | path | string | Yes | Request ID from POST response |

**Responses:**

| Code | Description |
|------|-------------|
| 200 | Request released |
| 404 | Request not found |

**Example:**

```bash
# Get a request ID
REQUEST_ID=$(curl -s -X POST http://localhost:8080/rate/user-123 | jq -r .request_id)

# Release it
curl -X DELETE "http://localhost:8080/rate/user-123/$REQUEST_ID"
```

## Debug

### GET /debug

View state of all rate limiter instances.

**Response:**

```json
{
  "Instances": {
    "user-123": {
      "Key": "user-123",
      "Config": {
        "WindowMillis": 1000,
        "MaxRequestsPerWindow": 100,
        "MaxRequestsInQueue": 400
      },
      "NumApprovedThisWindow": 50,
      "NumDeniedThisWindow": 10,
      "NumWaiting": 5,
      "Found": true
    }
  }
}
```

### GET /debug/:key

View state of a specific rate limiter instance.

**Parameters:**

| Name | In | Type | Required | Description |
|------|-----|------|----------|-------------|
| `key` | path | string | Yes | Rate limit key |

**Response (found):**

```json
{
  "Key": "user-123",
  "Config": {
    "WindowMillis": 1000,
    "MaxRequestsPerWindow": 100,
    "MaxRequestsInQueue": 400
  },
  "NumApprovedThisWindow": 50,
  "NumDeniedThisWindow": 10,
  "NumWaiting": 5,
  "Found": true
}
```

**Response (not found):**

```json
{
  "Key": "unknown-key",
  "Found": false
}
```

## Health

### GET /healthz

Health check endpoint.

**Response:**

```
OK
```

**Status Codes:**

| Code | Description |
|------|-------------|
| 200 | Service is healthy |

## Headers

### Request Headers

| Header | Description |
|--------|-------------|
| `X-Correlation-ID` | Passed through to logs for tracing |

### Response Headers

| Header | Description |
|--------|-------------|
| `Content-Type` | `application/json` for JSON responses |

## Error Responses

All error responses include a JSON body:

```json
{
  "error": "rate limit exceeded",
  "key": "user-123"
}
```

## Rate Limit Behavior

### Window-Based Counting

```
Time 0ms:      Window starts, counter = 0
Time 100ms:    Request → approved, counter = 1
Time 200ms:    Request → approved, counter = 2
...
Time 999ms:    Request → approved, counter = 100 (limit reached)
Time 1000ms:   Window resets, counter = 0
```

### Queue Behavior

When `canWait=true` and limit exceeded:

```
1. Request added to FIFO queue
2. Request waits for next window
3. On window reset, queued requests processed in order
4. Response sent when request is approved
```

### Key Expiration

Keys expire after 3 windows of inactivity:

```
Window 1:  Requests processed
Window 2:  No requests (idle)
Window 3:  No requests (idle)
Window 4:  No requests (idle) → Key expires
```

## Client Libraries

### Go

```go
import "net/http"

func rateLimit(key string, canWait bool) (string, error) {
    url := fmt.Sprintf("http://gocc:8080/rate/%s", key)
    if canWait {
        url += "?canWait=true"
    }
    resp, err := http.Post(url, "", nil)
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()

    if resp.StatusCode == 429 {
        return "", errors.New("rate limited")
    }

    var result struct {
        RequestID string `json:"request_id"`
    }
    json.NewDecoder(resp.Body).Decode(&result)
    return result.RequestID, nil
}
```

### Python

```python
import requests

def rate_limit(key: str, can_wait: bool = False) -> str:
    url = f"http://gocc:8080/rate/{key}"
    params = {"canWait": "true"} if can_wait else {}
    resp = requests.post(url, params=params)
    if resp.status_code == 429:
        raise Exception("rate limited")
    return resp.json()["request_id"]
```

### curl

```bash
#!/bin/bash
rate_limit() {
    local key=$1
    local can_wait=${2:-false}

    local url="http://gocc:8080/rate/$key"
    if [ "$can_wait" = "true" ]; then
        url="$url?canWait=true"
    fi

    response=$(curl -s -w "\n%{http_code}" -X POST "$url")
    status=$(echo "$response" | tail -1)
    body=$(echo "$response" | head -1)

    if [ "$status" = "429" ]; then
        echo "Rate limited" >&2
        return 1
    fi

    echo "$body" | jq -r .request_id
}
```
