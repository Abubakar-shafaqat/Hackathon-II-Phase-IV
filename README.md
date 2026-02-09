# Phase IV: Local Kubernetes Deployment

## Project Overview

Phase IV transitions the AI-powered todo application from development environment to a production-grade local Kubernetes cluster. This phase containerizes all application components using Docker and deploys them to Minikube using Helm charts, integrating AI-assisted DevOps tools for operations.

### What This Phase Does

This phase implements:
- **Docker Containerization**: Multi-stage builds for frontend and backend
- **Minikube Cluster Setup**: Local Kubernetes with required addons
- **Kubernetes Manifests**: Deployments, Services, ConfigMaps, Secrets, Ingress
- **Helm Charts**: Templated deployment packages for both services
- **AI DevOps Integration**: kubectl-ai, kagent, Docker AI Gordon
- **Health Monitoring**: Liveness and readiness probes

### Why This Phase is Important

Phase IV enables production-ready deployment capabilities:
1. Containerization ensures consistent environments
2. Kubernetes provides orchestration and scaling
3. Helm charts enable reproducible deployments
4. AI tools simplify operations and troubleshooting
5. Health probes ensure application reliability

### Connection to Other Phases

- **From Phase III**: Containerizes complete AI-powered application
- **To Phase V**: Foundation for production-grade features and monitoring

---

## Objectives

| Objective | Description | Status |
|-----------|-------------|--------|
| Containerization | Docker images for frontend and backend | Implemented |
| Minikube Setup | Local Kubernetes cluster configuration | Implemented |
| Kubernetes Manifests | All required resource definitions | Implemented |
| Helm Charts | Templated deployment packages | Implemented |
| AI DevOps Tools | kubectl-ai, kagent, Gordon integration | Implemented |
| Health Monitoring | Probes and endpoints | Implemented |
| Database Integration | External Neon PostgreSQL connection | Implemented |

---

## Detailed Explanation

### Purpose and Problem Solved

**Problem**: Development environment differs from production, causing "works on my machine" issues. Manual deployment is error-prone and doesn't scale.

**Solution**: A containerized, orchestrated deployment that:
- Runs identically in any environment
- Scales horizontally with demand
- Self-heals through health monitoring
- Simplifies operations with AI assistance
- Enables infrastructure as code

### Functional Responsibilities

#### 1. Docker Containerization

**Frontend Dockerfile (Multi-stage):**
```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:20-alpine AS runner
WORKDIR /app
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

**Backend Dockerfile (Multi-stage):**
```dockerfile
# Stage 1: Build
FROM python:3.13-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Stage 2: Production
FROM python:3.13-slim AS runner
WORKDIR /app
RUN useradd --create-home appuser
COPY --from=builder /usr/local/lib/python3.13/site-packages /usr/local/lib/python3.13/site-packages
COPY . .
USER appuser
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Requirements:**
- Images under 500MB each
- Non-root user execution
- Multi-stage builds for optimization
- .dockerignore files configured

#### 2. Minikube Cluster Setup

**Cluster Configuration:**
```bash
# Start cluster with resources
minikube start --cpus=4 --memory=8192

# Enable required addons
minikube addons enable ingress          # External access
minikube addons enable metrics-server   # HPA support
minikube addons enable storage-provisioner  # Persistent volumes
```

**Required Addons:**
- **Ingress Controller**: Routes external traffic to services
- **Metrics Server**: Provides metrics for Horizontal Pod Autoscaler
- **Storage Provisioner**: Enables persistent volume claims

#### 3. Kubernetes Manifests

**Deployments:**
- Frontend: 2+ replicas for high availability
- Backend: 2+ replicas with MCP server
- Resource limits and requests defined
- Liveness and readiness probes configured

**Services:**
- ClusterIP for internal communication
- NodePort for development access (optional)

**ConfigMaps:**
- Non-sensitive environment variables
- API URLs, feature flags

**Secrets:**
- DATABASE_URL
- JWT_SECRET
- OPENAI_API_KEY / GEMINI_API_KEY

**Ingress:**
- Routes /app/* to frontend
- Routes /api/* to backend
- TLS configuration (optional)

#### 4. Helm Charts

**Chart Structure:**
```
helm/
├── frontend/
│   ├── Chart.yaml          # Chart metadata
│   ├── values.yaml         # Default values
│   ├── values-local.yaml   # Minikube values
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── ingress.yaml
│       └── configmap.yaml
│
└── backend/
    ├── Chart.yaml
    ├── values.yaml
    ├── values-local.yaml
    └── templates/
        ├── deployment.yaml
        ├── service.yaml
        ├── configmap.yaml
        ├── secrets.yaml
        └── migration-job.yaml
```

**Benefits:**
- Templated resource generation
- Environment-specific values
- Version-controlled releases
- Easy upgrades and rollbacks

#### 5. AI DevOps Integration

**kubectl-ai:**
```bash
# Natural language Kubernetes commands
kubectl-ai "show all pods in todo namespace"
kubectl-ai "scale backend deployment to 3 replicas"
kubectl-ai "get logs from failing pod"
```

**kagent:**
```bash
# Cluster health analysis
kagent analyze cluster-health
kagent diagnose pod-crash todo-backend-xxx
kagent suggest optimization
```

**Docker AI (Gordon):**
```bash
# Container optimization
docker ai "optimize this Dockerfile"
docker ai "why is my image so large"
docker ai "security scan this image"
```

#### 6. Health Monitoring

**Liveness Probe:**
- Checks if application is alive
- Restarts container if unhealthy
- Endpoint: /health/live

**Readiness Probe:**
- Checks if ready to receive traffic
- Removes from load balancer if not ready
- Endpoint: /health/ready

**Probe Configuration:**
```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8000
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8000
  initialDelaySeconds: 5
  periodSeconds: 5
```

### Internal Workflow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    KUBERNETES DEPLOYMENT ARCHITECTURE                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│                        ┌─────────────────────┐                          │
│                        │   External Traffic   │                          │
│                        └──────────┬──────────┘                          │
│                                   │                                      │
│                                   ▼                                      │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │                     INGRESS CONTROLLER                          │     │
│  │                     (nginx-ingress)                             │     │
│  │  ┌─────────────────────────────────────────────────────────┐   │     │
│  │  │  Rules:                                                  │   │     │
│  │  │    /app/*  →  frontend-service:3000                     │   │     │
│  │  │    /api/*  →  backend-service:8000                      │   │     │
│  │  └─────────────────────────────────────────────────────────┘   │     │
│  └─────────────────────────┬────────────────┬─────────────────────┘     │
│                            │                │                            │
│              ┌─────────────┘                └─────────────┐              │
│              │                                            │              │
│              ▼                                            ▼              │
│  ┌─────────────────────────┐            ┌─────────────────────────┐     │
│  │   FRONTEND SERVICE      │            │   BACKEND SERVICE       │     │
│  │   (ClusterIP)           │            │   (ClusterIP)           │     │
│  │   Port: 3000            │            │   Port: 8000            │     │
│  └───────────┬─────────────┘            └───────────┬─────────────┘     │
│              │                                      │                    │
│              ▼                                      ▼                    │
│  ┌─────────────────────────┐            ┌─────────────────────────┐     │
│  │  FRONTEND DEPLOYMENT    │            │  BACKEND DEPLOYMENT     │     │
│  │  ┌─────────┐┌─────────┐ │            │  ┌─────────┐┌─────────┐ │     │
│  │  │ Pod 1   ││ Pod 2   │ │            │  │ Pod 1   ││ Pod 2   │ │     │
│  │  │ Next.js ││ Next.js │ │            │  │ FastAPI ││ FastAPI │ │     │
│  │  │         ││         │ │            │  │ + MCP   ││ + MCP   │ │     │
│  │  └─────────┘└─────────┘ │            │  └─────────┘└─────────┘ │     │
│  │  replicas: 2            │            │  replicas: 2            │     │
│  └─────────────────────────┘            └───────────┬─────────────┘     │
│                                                     │                    │
│  ┌─────────────────────────┐                       │                    │
│  │      CONFIGMAP          │◄──────────────────────┤                    │
│  │  - NEXT_PUBLIC_API_URL  │                       │                    │
│  │  - NODE_ENV             │                       │                    │
│  └─────────────────────────┘                       │                    │
│                                                     │                    │
│  ┌─────────────────────────┐                       │                    │
│  │       SECRETS           │◄──────────────────────┘                    │
│  │  - DATABASE_URL         │                                            │
│  │  - JWT_SECRET           │                                            │
│  │  - OPENAI_API_KEY       │                                            │
│  └─────────────────────────┘                                            │
│                                                                          │
└──────────────────────────────────────────┬──────────────────────────────┘
                                           │
                                           ▼
                            ┌─────────────────────────┐
                            │   NEON POSTGRESQL       │
                            │   (External Service)    │
                            │   SSL/TLS Connection    │
                            └─────────────────────────┘
```

### Inputs → Processing → Outputs

| Operation | Input | Processing | Output |
|-----------|-------|------------|--------|
| Build Images | Source code, Dockerfiles | Multi-stage build | Docker images (<500MB) |
| Start Cluster | Minikube config | Initialize K8s | Running cluster |
| Deploy Helm | Charts + values | Template + apply | Running pods |
| Health Check | Probe requests | Verify endpoints | Ready/NotReady status |
| Migration | Job manifest | Run migration | Updated DB schema |

### Dependency on Phase III

| Phase III Component | Phase IV Handling |
|--------------------|-------------------|
| Frontend application | Containerized in frontend image |
| Backend + MCP Server | Containerized in backend image |
| Environment variables | ConfigMaps and Secrets |
| Database schema | Migration Job |
| API keys | Kubernetes Secrets |

### Integration and Stabilization

| Aspect | Development | Kubernetes |
|--------|-------------|------------|
| Service Discovery | localhost:port | Service DNS names |
| Configuration | .env files | ConfigMaps/Secrets |
| Scaling | Manual restart | Replica count |
| Health Checks | Manual testing | Automated probes |
| Load Balancing | None | Service load balancing |
| Recovery | Manual restart | Auto pod restart |

---

## Technology Stack

### Containerization

| Technology | Version | Purpose |
|------------|---------|---------|
| Docker | 24+ | Container runtime |
| Docker Buildx | Latest | Multi-stage builds |
| .dockerignore | - | Exclude files from build |

### Orchestration

| Technology | Version | Purpose |
|------------|---------|---------|
| Kubernetes | 1.28+ | Container orchestration |
| Minikube | Latest | Local K8s cluster |
| kubectl | 1.28+ | K8s CLI |

### Package Management

| Technology | Version | Purpose |
|------------|---------|---------|
| Helm | 3.12+ | K8s package manager |

### AI DevOps

| Technology | Purpose |
|------------|---------|
| kubectl-ai | Natural language K8s commands |
| kagent | Cluster health analysis |
| Docker AI Gordon | Container optimization |

### Runtime

| Technology | Version | Purpose |
|------------|---------|---------|
| Node.js | 20+ | Frontend runtime |
| Python | 3.13+ | Backend runtime |
| PostgreSQL | 15+ | Database (Neon) |

---

## Folder Structure

```
Phase-IV/
├── docker/                             # Docker configurations
│   ├── frontend/
│   │   ├── Dockerfile                  # Frontend multi-stage build
│   │   └── .dockerignore               # Excluded files
│   └── backend/
│       ├── Dockerfile                  # Backend multi-stage build
│       └── .dockerignore               # Excluded files
│
├── k8s/                                # Kubernetes resources
│   ├── manifests/                      # Raw manifests (alternative to Helm)
│   │   ├── namespace.yaml              # Namespace definition
│   │   ├── frontend-deployment.yaml    # Frontend pods
│   │   ├── frontend-service.yaml       # Frontend service
│   │   ├── backend-deployment.yaml     # Backend pods
│   │   ├── backend-service.yaml        # Backend service
│   │   ├── configmap.yaml              # Environment config
│   │   ├── secrets.yaml                # Sensitive data (template)
│   │   ├── ingress.yaml                # Routing rules
│   │   └── migration-job.yaml          # DB migration
│   │
│   └── helm/                           # Helm charts
│       ├── frontend/
│       │   ├── Chart.yaml              # Chart metadata
│       │   ├── values.yaml             # Default values
│       │   ├── values-local.yaml       # Minikube-specific
│       │   └── templates/
│       │       ├── _helpers.tpl        # Template helpers
│       │       ├── deployment.yaml     # Deployment template
│       │       ├── service.yaml        # Service template
│       │       ├── ingress.yaml        # Ingress template
│       │       ├── configmap.yaml      # ConfigMap template
│       │       └── hpa.yaml            # HPA template (optional)
│       │
│       └── backend/
│           ├── Chart.yaml
│           ├── values.yaml
│           ├── values-local.yaml
│           └── templates/
│               ├── _helpers.tpl
│               ├── deployment.yaml
│               ├── service.yaml
│               ├── configmap.yaml
│               ├── secrets.yaml
│               └── migration-job.yaml
│
├── scripts/                            # Automation scripts
│   ├── setup-minikube.sh               # Cluster setup
│   ├── build-images.sh                 # Image building
│   ├── deploy.sh                       # Helm deployment
│   ├── ai-deploy.sh                    # AI-assisted deployment
│   └── troubleshoot.sh                 # AI troubleshooting
│
├── frontend/                           # Application code (from Phase III)
├── backend/                            # Application code (from Phase III)
│
├── specs/
│   └── phase4/
│       ├── constitution.md             # Phase IV principles
│       ├── spec.md                     # Deployment specification
│       └── plan.md                     # Architecture plan
│
├── CLAUDE.md                           # Development rules
└── README.md                           # This documentation
```

### Folder Details

| Folder | Purpose |
|--------|---------|
| `docker/` | Dockerfiles and ignore files for containerization |
| `k8s/manifests/` | Raw Kubernetes YAML files (alternative to Helm) |
| `k8s/helm/` | Helm charts for templated deployment |
| `scripts/` | Automation scripts for common operations |
| `frontend/`, `backend/` | Application code from Phase III |
| `specs/` | Deployment specifications |

---

## Setup & Installation

### Prerequisites

- Docker Desktop installed
- Minikube installed
- kubectl CLI installed
- Helm 3.12+ installed
- Phase III codebase ready

### Step 1: Start Minikube Cluster

```bash
# Start cluster with resources
minikube start --cpus=4 --memory=8192

# Enable required addons
minikube addons enable ingress
minikube addons enable metrics-server

# Verify cluster
kubectl cluster-info
kubectl get nodes
```

### Step 2: Build Docker Images

```bash
# Navigate to Phase IV
cd Phase-IV

# Build frontend image
docker build -t todo-frontend:latest -f docker/frontend/Dockerfile ./frontend

# Build backend image
docker build -t todo-backend:latest -f docker/backend/Dockerfile ./backend

# Load images into Minikube
minikube image load todo-frontend:latest
minikube image load todo-backend:latest

# Verify images
minikube image ls | grep todo
```

### Step 3: Configure Secrets

```bash
# Create secrets file from template
cp k8s/helm/backend/secrets-template.yaml k8s/helm/backend/secrets.yaml

# Edit with actual values (base64 encoded)
# DATABASE_URL, JWT_SECRET, OPENAI_API_KEY
```

### Step 4: Deploy with Helm

```bash
# Deploy backend
helm install todo-backend k8s/helm/backend -f k8s/helm/backend/values-local.yaml

# Deploy frontend
helm install todo-frontend k8s/helm/frontend -f k8s/helm/frontend/values-local.yaml

# Verify deployment
kubectl get pods
kubectl get services
kubectl get ingress
```

### Step 5: Access Application

```bash
# Start tunnel for ingress
minikube tunnel

# Get ingress IP
kubectl get ingress

# Access application at ingress IP or localhost
```

---

## Usage Instructions

### Checking Deployment Status

```bash
# View all resources
kubectl get all

# View pods with details
kubectl get pods -o wide

# View pod logs
kubectl logs -f deployment/todo-backend
kubectl logs -f deployment/todo-frontend
```

### Scaling Deployments

```bash
# Scale backend to 3 replicas
kubectl scale deployment todo-backend --replicas=3

# Or with Helm
helm upgrade todo-backend k8s/helm/backend --set replicaCount=3
```

### Updating Application

```bash
# Rebuild image with new tag
docker build -t todo-backend:v2 -f docker/backend/Dockerfile ./backend
minikube image load todo-backend:v2

# Update deployment
helm upgrade todo-backend k8s/helm/backend --set image.tag=v2
```

### Rollback

```bash
# View release history
helm history todo-backend

# Rollback to previous version
helm rollback todo-backend 1
```

### Using AI DevOps Tools

```bash
# kubectl-ai examples
kubectl-ai "list all pods in default namespace"
kubectl-ai "why is todo-backend pod crashing"
kubectl-ai "show resource usage for all pods"

# kagent examples
kagent analyze cluster
kagent diagnose deployment todo-backend
```

---

## Future Scope (Addressed in Phase V)

| Feature | Description |
|---------|-------------|
| Production Values | Environment-specific configurations |
| HPA Configuration | Auto-scaling based on metrics |
| Pod Disruption Budget | Availability during updates |
| Advanced Monitoring | Prometheus/Grafana integration |
| AI Operations | Enhanced troubleshooting scripts |

---

## Conclusion

Phase IV successfully containerizes and orchestrates the AI-powered todo application. Key achievements:

1. **Docker Containerization**: Optimized multi-stage builds under 500MB
2. **Kubernetes Deployment**: High-availability with 2+ replicas
3. **Helm Charts**: Reproducible, version-controlled deployments
4. **AI DevOps Tools**: Natural language infrastructure management
5. **Health Monitoring**: Automated recovery through probes
6. **External Database**: Secure SSL connection to Neon PostgreSQL

The infrastructure foundation established here enables Phase V to add production-grade monitoring, auto-scaling, and advanced operational capabilities.
