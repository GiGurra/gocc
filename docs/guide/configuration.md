# Configuration

GoCC can be configured via CLI flags, environment variables, or a JSON config file.

## CLI Flags

| Flag | Env Var | Default | Description |
|------|---------|---------|-------------|
| `-m, --max-requests` | `MAX_REQUESTS` | 100 | Default max requests per window |
| `--max-requests-in-queue` | `MAX_REQUESTS_IN_QUEUE` | 400 | Default queue size |
| `-w, --window-millis` | `WINDOW_MILLIS` | 1000 | Window size in milliseconds |
| `-r, --requests-can-set-rate` | `REQUESTS_CAN_SET_RATE` | true | Allow clients to override rate |
| `--requests-can-mod-queue` | `REQUESTS_CAN_MOD_QUEUE` | true | Allow clients to override queue |
| `-c, --config-file` | `CONFIG_FILE` | "" | Path to JSON config file |
| `-p, --port` | `PORT` | 8080 | Listen port |
| `-l, --log-format` | `LOG_FORMAT` | "json" | json, text, or system-default |
| `--log-level` | `LOG_LEVEL` | "INFO" | DEBUG, INFO, WARN, ERROR |
| `-s, --server-type` | `SERVER_TYPE` | "echo-http2" | echo, echo-http2, or fast |
| `-i, --instance-urls` | `INSTANCE_URLS` | [] | For distributed mode |

## Environment Variables

All CLI flags can be set via environment variables:

```bash
export MAX_REQUESTS=200
export WINDOW_MILLIS=5000
export PORT=9090
gocc
```

## Config File

For per-key configuration, use a JSON config file:

```json
{
  "keys": [
    {
      "key_pattern": "^premium-.*",
      "key_pattern_is_regex": true,
      "max_requests_per_window": 1000,
      "max_requests_in_queue": 500,
      "window_millis": 1000
    },
    {
      "key_pattern": "^basic-.*",
      "key_pattern_is_regex": true,
      "max_requests_per_window": 100,
      "max_requests_in_queue": 50,
      "window_millis": 1000
    },
    {
      "key_pattern": "guest",
      "key_pattern_is_regex": false,
      "max_requests_per_window": 10,
      "max_requests_in_queue": 5,
      "window_millis": 60000
    }
  ]
}
```

### Config File Fields

| Field | Required | Description |
|-------|----------|-------------|
| `key_pattern` | Yes | Key to match (literal or regex) |
| `key_pattern_is_regex` | No | If true, pattern is a regex |
| `max_requests_per_window` | No | Override max requests |
| `max_requests_in_queue` | No | Override queue size |
| `window_millis` | No | Override window size |

### Rules

- Config file values override CLI/env defaults
- All parameters except `key_pattern` are optional
- Values set to 0 are ignored (defaults apply)
- If multiple patterns match, they apply in order
- **Hot reloading is supported** - changes apply without restart

### Using Config File

```bash
gocc --config-file /path/to/config.json
```

Or via environment:

```bash
export CONFIG_FILE=/path/to/config.json
gocc
```

## Client-Controlled Rates

By default, clients can override rates per-request:

```bash
# Set custom rate for this key
curl -X POST "http://localhost:8080/rate/my-key?maxRequests=500&maxRequestsInQueue=1000"
```

To disable this:

```bash
gocc --requests-can-set-rate=false --requests-can-mod-queue=false
```

## Server Types

| Type | Description |
|------|-------------|
| `echo-http2` | Default. Echo framework with HTTP/2. Best performance. |
| `echo` | Echo framework, HTTP/1.1 only |
| `fast` | FastHTTP (experimental, incomplete) |

## Logging

### Formats

- `json` (default) - Structured JSON logs
- `text` - Human-readable text
- `system-default` - Go's default slog format

### Levels

- `DEBUG` - Verbose debugging
- `INFO` - Normal operation
- `WARN` - Warnings
- `ERROR` - Errors only

### Response Logging

```bash
gocc --log2xx=true   # Log successful responses
gocc --log4xx=true   # Log 429 rate limit responses
gocc --log5xx=true   # Log errors (default)
```

## Example Configurations

### High-Throughput API

```bash
gocc \
  --max-requests 10000 \
  --window-millis 1000 \
  --max-requests-in-queue 50000 \
  --server-type echo-http2
```

### Strict Rate Limiting

```bash
gocc \
  --max-requests 100 \
  --window-millis 60000 \
  --max-requests-in-queue 0 \
  --requests-can-set-rate=false
```

### Debug Mode

```bash
gocc \
  --log-format text \
  --log-level DEBUG \
  --log2xx=true \
  --log4xx=true
```
