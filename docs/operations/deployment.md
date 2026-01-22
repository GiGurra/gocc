# Deployment

GoCC can be deployed as a single instance or distributed across multiple nodes.

## Single Instance

### Binary

```bash
# Download or build
go install github.com/GiGurra/gocc@latest

# Run
gocc --port 8080
```

### Systemd

```ini
[Unit]
Description=GoCC Rate Limiter
After=network.target

[Service]
Type=simple
User=gocc
ExecStart=/usr/local/bin/gocc --port 8080 --config-file /etc/gocc/config.json
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Docker

```bash
docker run -d \
  --name gocc \
  -p 8080:8080 \
  -v /path/to/config.json:/config.json \
  ghcr.io/gigurra/gocc:latest \
  --config-file /config.json
```

### Docker Compose

```yaml
version: '3.8'
services:
  gocc:
    image: ghcr.io/gigurra/gocc:latest
    ports:
      - "8080:8080"
    volumes:
      - ./config.json:/config.json:ro
    command: ["--config-file", "/config.json"]
    restart: always
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/healthz"]
      interval: 10s
      timeout: 5s
      retries: 3
```

## Configuration

### Environment Variables

All CLI flags can be set via environment:

```bash
docker run -d \
  -e MAX_REQUESTS=200 \
  -e WINDOW_MILLIS=5000 \
  -e LOG_FORMAT=json \
  -p 8080:8080 \
  ghcr.io/gigurra/gocc:latest
```

### Config File with Hot Reload

Mount a config file and modify it without restarting:

```bash
# Update config
vim /path/to/config.json

# GoCC automatically reloads
# (using fsnotify file watcher)
```

## Resource Requirements

### Memory

- Base: ~10-20 MB
- Per active key: ~500 bytes + queue size
- Example: 10,000 keys, 10 queued each = ~15 MB additional

### CPU

- Scales linearly with request rate
- ~6M req/s internal = ~1 core saturated
- HTTP/2 at 450k req/s = ~0.5 core

### Recommended Starting Point

| Workload | CPU | Memory |
|----------|-----|--------|
| Light (<10k req/s) | 0.25 core | 64 MB |
| Medium (<100k req/s) | 0.5 core | 128 MB |
| Heavy (<500k req/s) | 1 core | 256 MB |

## Health Checks

### HTTP

```bash
curl http://localhost:8080/healthz
# OK
```

### Docker

```dockerfile
HEALTHCHECK --interval=10s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:8080/healthz || exit 1
```

### Kubernetes

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

## Logging

### Production Settings

```bash
gocc \
  --log-format json \
  --log-level INFO \
  --log2xx=false \
  --log4xx=false \
  --log5xx=true
```

### Log Output

```json
{"time":"2024-01-15T10:30:00Z","level":"INFO","msg":"server started","port":8080}
{"time":"2024-01-15T10:30:01Z","level":"ERROR","msg":"internal error","error":"..."}
```

## Security

### Network

- Deploy behind a load balancer/ingress
- Use TLS termination at load balancer
- Restrict access to internal network if possible

### Configuration

```bash
# Disable client-controlled rates in production
gocc \
  --requests-can-set-rate=false \
  --requests-can-mod-queue=false
```

### Container

The Docker image:
- Runs as non-root user
- Uses Alpine base (minimal attack surface)
- No shell access needed

## Backup and Recovery

GoCC is stateless - all state is in-memory:
- No backup needed
- On restart, all counters reset to 0
- Rate limit keys recreated on first request

For persistence across restarts:
- Use config file for key-specific settings
- Accept that counters will reset

## Monitoring

See [Monitoring](monitoring.md) for metrics and alerting.

## Next Steps

- [Kubernetes Deployment](kubernetes.md) - Deploy on Kubernetes
- [Monitoring](monitoring.md) - Metrics and alerting
