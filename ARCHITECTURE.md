# ToggleMaster — Full Architecture Reference

## What the application does

ToggleMaster is a **Feature Flag platform** (like LaunchDarkly/Unleash). It lets engineering teams enable/disable features for specific users without deploying new code.

---

## The 5 Services

### auth-service (Go → PostgreSQL)
- Creates and validates API keys
- Stores only the SHA256 hash of keys, never the plain text
- `POST /keys` — create key (requires MASTER_KEY)
- `GET /validate` — validate key (called internally by flag-service and targeting-service)

### flag-service (Python → PostgreSQL)
- CRUD for feature flag definitions (name, description, is_enabled boolean)
- Every request calls auth-service/validate via `require_auth` middleware

### targeting-service (Python → PostgreSQL)
- CRUD for rollout rules tied to flags
- Supports PERCENTAGE rollout: "enable this flag for 30% of users"
- Same `require_auth` middleware as flag-service

### evaluation-service (Go → Redis)
The hot path — called on every user action to answer "is this flag on for this user?"

Flow:
1. Check Redis cache (30s TTL) for flag+rule data
2. Cache HIT → skip to evaluation logic
3. Cache MISS → call flag-service + targeting-service **concurrently** (2 goroutines, sync.WaitGroup), store result in Redis
4. Run evaluation logic:
   - Flag disabled → `false`
   - Flag enabled, no rule → `true` (everyone gets it)
   - PERCENTAGE rule → SHA1(userID+flagName) % 100 < threshold → deterministic, same user always gets same result
5. Fire-and-forget goroutine sends event to SQS
6. Return result immediately

### analytics-service (Python → SQS + DynamoDB)
Background consumer. Long-polls SQS (20s wait, batches of 10), writes each event to DynamoDB, deletes message only after successful write.

DynamoDB schema: `event_id` (PK, UUID), `user_id`, `flag_name`, `result` (bool), `timestamp`

---

## Redis vs DynamoDB — why different databases

|  | Redis | DynamoDB |
|---|---|---|
| Type | In-memory cache | Persistent NoSQL database |
| Data survives restart? | No | Yes, forever |
| Use here | Temporary cache, 30s TTL | Permanent event log |
| Why chosen | Microsecond reads, perfect for cache | Scales to billions of rows, no schema |
| What if lost? | Cache rebuilds itself on next miss | Data gone permanently |

Redis is **intentionally throwaway** — just a speedup. DynamoDB is the **source of truth for analytics**.

---

## Why SQS sits between evaluation-service and analytics-service

Producer/consumer pattern. If DynamoDB is slow or down, it does not affect evaluation latency. SQS buffers events. analytics-service consumes at its own pace. If analytics-service is down for 10 minutes, messages wait in the queue — no data loss. A message is deleted from the queue only after a successful DynamoDB write.

---

## Communication map

```
Client App
    │
    ▼
evaluation-service ──(cache miss)──► flag-service ──► auth-service ──► PostgreSQL (auth)
    │                           └──► targeting-service ──► auth-service
    │                                   flag-service ──► PostgreSQL (flags)
    │                                   targeting-service ──► PostgreSQL (targeting)
    ├──► Redis (cache, 30s TTL)
    └──► SQS (async, fire-and-forget)
              │
              ▼
       analytics-service ──► DynamoDB
```

---

## Cloud Infrastructure (AWS)

### Local → Production mapping

| Local (docker-compose) | Production (AWS) |
|---|---|
| docker containers | EKS pods |
| docker images | ECR repositories (5 repos, one per service) |
| postgres containers | 3 RDS PostgreSQL instances (auth, flag, targeting) |
| redis container | ElastiCache Redis cluster |
| dynamodb-local | AWS DynamoDB (ToggleMasterAnalytics table) |
| SQS | AWS SQS Standard queue |
| localhost routing | Nginx Ingress → AWS Load Balancer |

### EKS Cluster
Provisioned with `eksctl create cluster`. Runs all 5 services as pods.

### ECR
5 repositories. Images built locally, tagged, and pushed to ECR. Kubernetes Deployments reference ECR image URLs.

```bash
docker build -t auth-service ./auth-service
docker tag auth-service 108101918154.dkr.ecr.us-east-1.amazonaws.com/auth-service:latest
docker push 108101918154.dkr.ecr.us-east-1.amazonaws.com/auth-service:latest
```

### 3 RDS PostgreSQL Instances
Separate instances (not separate DBs on the same server) — each service owns its data independently. If one is overloaded, others are not affected.

### ElastiCache Redis
Managed Redis for evaluation-service cache. AWS handles availability and failover.

---

## Kubernetes Manifests (per service)

### 1. Namespace
Logical isolation — one namespace per service. Problems in one do not affect others.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: evaluation
```

### 2. Secret
Sensitive config (RDS passwords, Redis URL, SQS URL) — always base64-encoded.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: evaluation-secrets
  namespace: evaluation
data:
  redis-url: cmVkaXM6Ly9jbHVzdGVyLmFiYzEyMy5jYWNoZS5hbWF6b25hd3MuY29tOjYzNzk=
  sqs-url: aHR0cHM6Ly9zcXMudXMtZWFzdC0xLmFtYXpvbmF3cy5jb20vMTA4Li4u
```

### 3. ConfigMap
Non-sensitive config — internal service URLs, AWS region. Plain text, no encoding needed.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: evaluation-config
  namespace: evaluation
data:
  FLAG_SERVICE_URL: "http://flag-service.flag.svc.cluster.local:8002"
  TARGETING_SERVICE_URL: "http://targeting-service.targeting.svc.cluster.local:8003"
  AWS_REGION: "us-east-1"
```

Internal URL format inside the cluster: `http://<service-name>.<namespace>.svc.cluster.local:<port>`

### 4. Deployment
References the ECR image, sets resource limits, and defines health checks.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: evaluation-service
  namespace: evaluation
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: evaluation-service
        image: 108101918154.dkr.ecr.us-east-1.amazonaws.com/evaluation-service:latest
        resources:
          requests:
            cpu: "250m"       # guaranteed minimum — used for scheduling
            memory: "128Mi"
          limits:
            cpu: "500m"       # hard ceiling — prevents starving other pods
            memory: "256Mi"
        envFrom:
        - secretRef:
            name: evaluation-secrets
        - configMapRef:
            name: evaluation-config
        readinessProbe:        # pod only receives traffic when /health returns 200
          httpGet:
            path: /health
            port: 8004
        livenessProbe:         # pod is restarted if /health fails 3 times
          httpGet:
            path: /health
            port: 8004
```

### 5. Service (ClusterIP)
Makes the Deployment reachable inside the cluster by a stable DNS name. Not exposed to the internet directly.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: evaluation-service
  namespace: evaluation
spec:
  type: ClusterIP
  selector:
    app: evaluation-service
  ports:
  - port: 8004
    targetPort: 8004
```

### 6. Ingress
One Ingress for all 5 services. Nginx reads this and creates path-based routing on the Load Balancer.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: togglemaster-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - http:
      paths:
      - path: /auth
        backend:
          service: { name: auth-service, port: { number: 8001 } }
      - path: /flags
        backend:
          service: { name: flag-service, port: { number: 8002 } }
      - path: /targeting
        backend:
          service: { name: targeting-service, port: { number: 8003 } }
      - path: /evaluate
        backend:
          service: { name: evaluation-service, port: { number: 8004 } }
      - path: /analytics
        backend:
          service: { name: analytics-service, port: { number: 8005 } }
```

AWS creates one Load Balancer with one public URL (e.g. `a1b2c3.us-east-1.elb.amazonaws.com`) that routes all traffic by path.

---

## Autoscaling

### HPA on evaluation-service (CPU-based)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: evaluation-hpa
  namespace: evaluation
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: evaluation-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilizationPercentage: 70
```

**Why CPU for evaluation-service?** It is a synchronous HTTP service — high traffic directly means high CPU. CPU is the right proxy for load. Requires Metrics Server installed on the cluster.

### KEDA on analytics-service (SQS queue depth)

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: analytics-scaledobject
  namespace: analytics
spec:
  scaleTargetRef:
    name: analytics-service
  minReplicaCount: 0    # scales to ZERO when queue is empty — pay nothing idle
  maxReplicaCount: 20
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/108101918154/togglemaster-evaluation-service
      queueLength: "10"  # 1 pod per 10 messages in queue
      awsRegion: us-east-1
```

**Why KEDA not HPA for analytics-service?** analytics-service is a consumer — its load is defined by the SQS backlog, not its own CPU. When 10,000 messages pile up, CPU is still near zero (the pod is idle, waiting). HPA would never scale it up. KEDA reads the queue depth directly and scales accordingly. It also scales to **zero pods** when the queue is empty.

### IRSA (IAM Roles for Service Accounts)
Nginx Ingress Controller gets a scoped IAM role to manage Load Balancers — not full node-level AWS permissions. Secure by principle of least privilege.

---

## Full production architecture

```
Internet
    │
    ▼
AWS Load Balancer (1 public URL)
    │
    ▼
Nginx Ingress Controller (EKS pod, IRSA)
    │
    ├── /auth      → auth-service pods       (ns: auth,      fixed replicas) → RDS PostgreSQL
    ├── /flags     → flag-service pods       (ns: flag,      fixed replicas) → RDS PostgreSQL
    ├── /targeting → targeting-service pods  (ns: targeting, fixed replicas) → RDS PostgreSQL
    ├── /evaluate  → evaluation-service pods (ns: evaluation, HPA 2–10 pods) → ElastiCache Redis
    │                    └──(async, fire-and-forget)──► SQS
    └── /analytics → analytics-service pods (ns: analytics, KEDA 0–20 pods) → DynamoDB
                         └── consumes from SQS
```

---

## Production usage examples

```bash
# 1. Create an API key (one-time setup)
curl -X POST https://lb.us-east-1.elb.amazonaws.com/auth/keys \
  -H "Authorization: Bearer admin-secreto-123" \
  -d '{"name": "backend-app"}'
# Returns the key once: tm_key_abc123...

# 2. Create a feature flag
curl -X POST https://lb.us-east-1.elb.amazonaws.com/flags/flags \
  -H "Authorization: Bearer tm_key_abc123" \
  -d '{"name": "new-checkout", "is_enabled": true}'

# 3. Create a 30% rollout rule
curl -X POST https://lb.us-east-1.elb.amazonaws.com/targeting/rules \
  -H "Authorization: Bearer tm_key_abc123" \
  -d '{"flag_name": "new-checkout", "rules": {"type": "PERCENTAGE", "value": 30}}'

# 4. Evaluate at runtime (called millions of times by your app)
curl "https://lb.us-east-1.elb.amazonaws.com/evaluate/evaluate?user_id=alice&flag_name=new-checkout"
# {"flag_name": "new-checkout", "user_id": "alice", "result": true}

curl "https://lb.us-east-1.elb.amazonaws.com/evaluate/evaluate?user_id=bob&flag_name=new-checkout"
# {"flag_name": "new-checkout", "user_id": "bob", "result": false}
# Behind the scenes: both events written async to SQS → DynamoDB
```

Alice always gets `true` and Bob always gets `false` — the SHA1 bucket for each user+flag combination is deterministic.
