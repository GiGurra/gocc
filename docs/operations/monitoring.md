# Monitoring

GoCC provides debug endpoints and structured logging for observability.

## Debug Endpoints

### All Instances

```bash
curl http://localhost:8080/debug | jq
```

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
      "NumApprovedThisWindow": 87,
      "NumDeniedThisWindow": 3,
      "NumWaiting": 12,
      "Found": true
    },
    "api-key-456": {
      "Key": "api-key-456",
      "Config": {
        "WindowMillis": 1000,
        "MaxRequestsPerWindow": 100,
        "MaxRequestsInQueue": 400
      },
      "NumApprovedThisWindow": 100,
      "NumDeniedThisWindow": 50,
      "NumWaiting": 0,
      "Found": true
    }
  }
}
```

### Specific Key

```bash
curl http://localhost:8080/debug/user-123 | jq
```

```json
{
  "Key": "user-123",
  "Config": {
    "WindowMillis": 1000,
    "MaxRequestsPerWindow": 100,
    "MaxRequestsInQueue": 400
  },
  "NumApprovedThisWindow": 87,
  "NumDeniedThisWindow": 3,
  "NumWaiting": 12,
  "Found": true
}
```

## Key Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `NumApprovedThisWindow` | Requests approved in current window | - |
| `NumDeniedThisWindow` | Requests denied (429) | High = capacity issue |
| `NumWaiting` | Requests in queue | High = latency issue |

## Logging

### Configuration

```bash
gocc \
  --log-format json \
  --log-level INFO \
  --log2xx=false \
  --log4xx=true \
  --log5xx=true
```

### Log Levels

| Level | Use |
|-------|-----|
| `DEBUG` | Development, troubleshooting |
| `INFO` | Normal operation |
| `WARN` | Potential issues |
| `ERROR` | Errors only |

### Sample Logs

```json
{"time":"2024-01-15T10:30:00Z","level":"INFO","msg":"server started","port":8080,"server_type":"echo-http2"}
{"time":"2024-01-15T10:30:01Z","level":"INFO","msg":"request","method":"POST","path":"/rate/user-123","status":200,"latency_ms":0.5}
{"time":"2024-01-15T10:30:02Z","level":"INFO","msg":"request","method":"POST","path":"/rate/user-123","status":429,"latency_ms":0.3}
{"time":"2024-01-15T10:30:03Z","level":"INFO","msg":"config reloaded","keys_count":5}
```

## Health Checks

### Endpoint

```bash
curl http://localhost:8080/healthz
# OK
```

### HTTP Status

- `200 OK` - Service is healthy
- `5xx` - Service is unhealthy

## Building a Monitoring Dashboard

### Prometheus (Custom Scraper)

GoCC doesn't expose Prometheus metrics natively, but you can scrape the debug endpoint:

```python
# prometheus_gocc_exporter.py
import requests
import json
from prometheus_client import Gauge, start_http_server

approved = Gauge('gocc_approved_this_window', 'Requests approved', ['key'])
denied = Gauge('gocc_denied_this_window', 'Requests denied', ['key'])
waiting = Gauge('gocc_waiting', 'Requests in queue', ['key'])

def collect():
    resp = requests.get('http://gocc:8080/debug')
    data = resp.json()
    for key, instance in data.get('Instances', {}).items():
        approved.labels(key=key).set(instance['NumApprovedThisWindow'])
        denied.labels(key=key).set(instance['NumDeniedThisWindow'])
        waiting.labels(key=key).set(instance['NumWaiting'])
```

### Grafana Dashboard Queries

```promql
# Approval rate by key
sum(rate(gocc_approved_this_window[1m])) by (key)

# Denial rate by key
sum(rate(gocc_denied_this_window[1m])) by (key)

# Queue depth
gocc_waiting
```

## Alerting

### High Denial Rate

```yaml
# Prometheus alert rule
- alert: GoCCHighDenialRate
  expr: sum(rate(gocc_denied_this_window[5m])) > 100
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "GoCC high denial rate"
    description: "More than 100 requests/sec being denied"
```

### High Queue Depth

```yaml
- alert: GoCCHighQueueDepth
  expr: max(gocc_waiting) > 1000
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "GoCC high queue depth"
    description: "Queue depth exceeds 1000 requests"
```

### Service Down

```yaml
- alert: GoCCDown
  expr: up{job="gocc"} == 0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "GoCC is down"
```

## Log Aggregation

### Fluentd/Fluent Bit

```yaml
# fluent-bit config
[INPUT]
    Name              tail
    Path              /var/log/containers/gocc*.log
    Parser            json
    Tag               gocc.*

[OUTPUT]
    Name              elasticsearch
    Match             gocc.*
    Host              elasticsearch
    Index             gocc-logs
```

### Useful Log Queries

```
# Elasticsearch/Kibana
status:429 AND key:premium*

# Find slow requests
latency_ms:>100

# Error logs
level:ERROR
```

## Troubleshooting

### High Latency

1. Check `NumWaiting` - high values indicate queueing
2. Check window size - smaller windows = faster feedback
3. Check HTTP/2 usage - much faster than HTTP/1.1

### High Denial Rate

1. Check if limits are appropriate
2. Check for traffic spikes
3. Consider increasing `MaxRequestsPerWindow`

### Memory Growth

1. Check number of active keys via `/debug`
2. Keys expire after 3 windows of inactivity
3. Consider shorter windows for faster cleanup

### Instance Not Responding

1. Check `/healthz` endpoint
2. Check logs for errors
3. Check resource limits (CPU/memory)
