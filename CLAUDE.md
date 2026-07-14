# POSTECH Tech Challenge - Fase 02

## Project Overview

Migrate the ToggleMaster monolith (built in Fase 1) into a distributed microservices ecosystem deployed on Kubernetes (AWS EKS). Worth 90% of the phase grade.

## Architecture: 5 Microservices

| Service | Language | Data Store |
|---|---|---|
| **auth-service** | Go | PostgreSQL (RDS) |
| **flag-service** | Python | PostgreSQL (RDS) |
| **targeting-service** | Python | PostgreSQL (RDS) |
| **evaluation-service** | Go | Redis (ElastiCache) |
| **analytics-service** | Python | AWS SQS (queue) + AWS DynamoDB |

## Cloud Environment

Personal AWS account with full IAM access. Using `eksctl`, IRSA, KEDA, and all standard tooling without restrictions.

---

## Step-by-Step Implementation

### Step 1 — Containerization (Docker)

- Create an optimized **Dockerfile** for each of the 5 microservices (multi-stage builds highly recommended)
- Create a single **`docker-compose.yml`** at the project root that starts:
  - 5 microservices
  - 4 local databases: 2x PostgreSQL, 1x Redis, 1x DynamoDB Local
- Verify the full stack runs locally before moving to cloud

### Step 2 — Provision Cloud Infrastructure

Collect all connection strings at the end of this step (RDS endpoints, ElastiCache endpoint, DynamoDB table name, SQS ARN).

#### EKS Cluster
- Use `eksctl create cluster` — auto-creates correct IAM roles.

#### Container Registry (ECR)
- Create **5 ECR repositories** (one per microservice: auth-service, flag-service, targeting-service, evaluation-service, analytics-service)
- Build and push Docker images to their respective ECR repos

#### Relational Databases (RDS)
- Create **3 independent AWS RDS for PostgreSQL** instances:
  - RDS 1 → auth-service
  - RDS 2 → flag-service
  - RDS 3 → targeting-service

#### Cache (ElastiCache)
- Create **1 AWS ElastiCache for Redis** cluster → evaluation-service

#### NoSQL Database (DynamoDB)
- Create **1 AWS DynamoDB table** → analytics-service (check source code for table name and primary key)

#### Message Queue (SQS)
- Create **1 AWS SQS Standard queue** → used by evaluation-service (producer) and analytics-service (consumer)

### Step 3 — Configure the Kubernetes Cluster

#### Metrics Server (both options)
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
Required for HPA to function.

#### Nginx Ingress Controller
- Install with IRSA so the controller pod has scoped permissions to manage Load Balancers (not full node permissions).

### Step 4 — Kubernetes Manifests

Create YAML manifests for each of the 5 microservices:

1. **Namespace** — logical separation per service
2. **Deployment** — manages Pods, references ECR images
3. **Service** — type `ClusterIP`
4. **Secret** — base64-encoded passwords, endpoints, and access keys from Step 2
5. **ConfigMap** — internal service URLs and non-sensitive config

#### Ingress
Create an Ingress manifest with path-based routing:
- `/auth` → auth-service
- `/flags` → flag-service
- `/targeting` → targeting-service
- `/evaluate` → evaluation-service
- `/analytics` → analytics-service

#### Best Practices
- Always set `resources.requests` and `resources.limits` on Deployments
- Secrets must always be base64-encoded
- Add `readinessProbe` and/or `livenessProbe` wherever possible
- Separate each application into its own Namespace

### Step 5 — Autoscaling

#### HPA — evaluation-service
```yaml
targetCPUUtilizationPercentage: 70
```

#### KEDA — analytics-service
- Install KEDA on the cluster
- Create a **ScaledObject** for analytics-service that monitors SQS `queueDepth` directly
- Scales pods from 0 to N based on queue depth (no CPU proxy needed)

---

## Deliverables

### Demo Video (max 20 min)
- [ ] `docker compose up` running all 9 containers (5 apps + 4 DBs) locally
- [ ] EKS/GKE/AKS cluster provisioned on the cloud
- [ ] `kubectl get pods` showing all 5 microservices running
- [ ] Nginx Ingress working (curl/Postman call to Load Balancer URL)
- [ ] Load test on evaluation-service (hey/ab/Postman) → HPA scaling up replicas (`kubectl get hpa`, `kubectl get pods`)
- [ ] Manual SQS messages sent → HPA/KEDA detecting load → analytics-service pods scaling
- [ ] Data appearing in DynamoDB table
- [ ] Architecture explanation + challenges faced (e.g., LabRole limitations)
- [ ] Explanation of analytics-service scaling choice (HPA by CPU vs KEDA by queue) with justification
- [ ] Explanation of the 3 data store purposes (RDS vs ElastiCache vs DynamoDB)

### Delivery Report (PDF or .txt)
- [ ] Team member names, RM, and Discord usernames
- [ ] Repository links
- [ ] Video link (YouTube or preferred platform)
- [ ] (Optional) Google Cloud Skills Boost badge link for +10 bonus points

---

## Bonus Points
- Max score per phase: **90 points**
- Complete the Google Cloud Skills Boost trail (invite will be sent), make the badge public, and include the link in the report → **+10 bonus points**
