# Kubernetes Deployment

GoCC can be deployed on Kubernetes as a single instance or distributed across multiple pods.

## Single Instance

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gocc
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gocc
  template:
    metadata:
      labels:
        app: gocc
    spec:
      containers:
      - name: gocc
        image: ghcr.io/gigurra/gocc:latest
        ports:
        - containerPort: 8080
        env:
        - name: MAX_REQUESTS
          value: "100"
        - name: WINDOW_MILLIS
          value: "1000"
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 500m
            memory: 256Mi
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
---
apiVersion: v1
kind: Service
metadata:
  name: gocc
spec:
  selector:
    app: gocc
  ports:
  - port: 8080
    targetPort: 8080
```

### With ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gocc-config
data:
  config.json: |
    {
      "keys": [
        {
          "key_pattern": "^premium-.*",
          "key_pattern_is_regex": true,
          "max_requests_per_window": 1000
        }
      ]
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gocc
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gocc
  template:
    metadata:
      labels:
        app: gocc
    spec:
      containers:
      - name: gocc
        image: ghcr.io/gigurra/gocc:latest
        args: ["--config-file", "/config/config.json"]
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: config
          mountPath: /config
      volumes:
      - name: config
        configMap:
          name: gocc-config
```

## Distributed Mode

For higher availability and throughput, deploy multiple instances with consistent hashing.

### StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: gocc
spec:
  serviceName: gocc-headless
  replicas: 3
  selector:
    matchLabels:
      app: gocc
  template:
    metadata:
      labels:
        app: gocc
    spec:
      containers:
      - name: gocc
        image: ghcr.io/gigurra/gocc:latest
        ports:
        - containerPort: 8080
        env:
        - name: INSTANCE_URLS
          value: "http://gocc-0.gocc-headless:8080,http://gocc-1.gocc-headless:8080,http://gocc-2.gocc-headless:8080"
        resources:
          requests:
            cpu: 200m
            memory: 128Mi
          limits:
            cpu: 1000m
            memory: 512Mi
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
---
apiVersion: v1
kind: Service
metadata:
  name: gocc-headless
spec:
  clusterIP: None
  selector:
    app: gocc
  ports:
  - port: 8080
    targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: gocc
spec:
  selector:
    app: gocc
  ports:
  - port: 8080
    targetPort: 8080
```

### How Distributed Mode Works

1. **Stable DNS Names**: StatefulSet provides `gocc-0`, `gocc-1`, `gocc-2`, etc.
2. **Consistent Hashing**: Each key hashes to a specific instance
3. **Request Forwarding**: If a request hits the wrong instance, it's forwarded
4. **No Database**: State is distributed across instances, no coordination needed

```
Client → gocc-1 → hash(key) = gocc-2 → forward → gocc-2 → response
```

### Scaling

```bash
# Scale up
kubectl scale statefulset gocc --replicas=5

# Update INSTANCE_URLS env var to include new instances
# Or use init container/sidecar to discover instances
```

## Ingress

### NGINX Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gocc
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  rules:
  - host: ratelimit.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gocc
            port:
              number: 8080
```

### With TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gocc
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - ratelimit.example.com
    secretName: gocc-tls
  rules:
  - host: ratelimit.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gocc
            port:
              number: 8080
```

## Pod Disruption Budget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: gocc-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: gocc
```

## Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: gocc-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: gocc
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

Note: For StatefulSet in distributed mode, scaling requires updating `INSTANCE_URLS`.

## Network Policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: gocc-network-policy
spec:
  podSelector:
    matchLabels:
      app: gocc
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: my-apps
    ports:
    - protocol: TCP
      port: 8080
```

## Troubleshooting

### Check Logs

```bash
kubectl logs -l app=gocc -f
```

### Check Instance State

```bash
kubectl exec -it gocc-0 -- curl http://localhost:8080/debug | jq
```

### Check Connectivity (Distributed)

```bash
# From gocc-0, check connectivity to gocc-1
kubectl exec -it gocc-0 -- curl http://gocc-1.gocc-headless:8080/healthz
```
