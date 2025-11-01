# Production-Ready Scaleway Infrastructure for AgentOps

## Architecture Overview

**Design Philosophy**: Scaleway-native, production-grade, built for scale from day one.

**Inference Strategy**: Use external APIs (Groq, OpenAI) - Scaleway handles everything else.

## Core Services Stack

### 1. **Compute Layer** - Serverless Containers

**Why Serverless Containers over Kubernetes**:
- Zero infrastructure management
- Auto-scaling from 0 to 1000+ instances
- Pay only for actual usage (per-second billing)
- Built-in load balancing
- Faster deployment than K8s
- Perfect for agent workloads (event-driven, variable load)

**Components**:
- **AgentOps API**: FastAPI in Serverless Container (always-on, min scale: 1)
- **Chief of Staff Agent**: Strands agent in Serverless Container (always-on, min scale: 1)
- **Seed Agent**: Strands agent in Serverless Container (always-on, min scale: 1)
- **Specialist Agents**: Dynamically created Serverless Containers (scale to zero when idle)
- **Voice Interface**: voice-stack in Serverless Container (scale to zero)
- **Telegram Bot**: Python bot in Serverless Container (scale to zero)
- **Web Dashboard**: Next.js in Serverless Container (scale to zero)

**Configuration**:
- Region: fr-par (Paris - lowest latency to most of Europe)
- Memory: 512MB-2GB per container (adjustable)
- vCPU: 280-1120 mVCPU per container
- Concurrency: 10-80 requests per instance
- Timeout: No limit (unlike Functions)
- Protocol: HTTP/2 for gRPC support

**Cost Estimate**:
- Always-on containers (3): ~$30/month
- Dynamic containers: ~$20/month (average)
- **Total**: ~$50/month

### 2. **Data Layer** - Managed PostgreSQL

**Service**: Managed Database for PostgreSQL 15
**Instance Type**: DB-GP-S (4 vCPU, 16GB RAM) - Production-ready
**Storage**: 50GB Block Storage (auto-scaling enabled)
**High Availability**: Yes (multi-AZ, automatic failover)
**Backups**: Daily automated backups, 7-day retention
**Region**: fr-par

**Schema**:
- Projects, Tasks, Events, Agents, Artifacts, Task_Updates
- Full ACID compliance
- Connection pooling via PgBouncer (included)

**Cost**: ~$80/month

### 3. **Cache & Events Layer** - Managed Redis

**Service**: Managed Database for Redis 7
**Instance Type**: REDIS-GP-S (2 vCPU, 8GB RAM)
**Cluster Mode**: No (single node for MVP, can upgrade to cluster)
**Persistence**: RDB + AOF (data durability)
**Region**: fr-par

**Usage**:
- Real-time event pub/sub
- Agent state caching
- Rate limiting
- Session storage

**Cost**: ~$40/month

### 4. **Storage Layer** - Object Storage

**Service**: Scaleway Object Storage (S3-compatible)
**Bucket**: agentops-artifacts
**Region**: fr-par
**Features**:
- Versioning enabled
- Lifecycle policies (auto-delete old versions)
- Private by default
- Presigned URLs for temporary access

**Usage**:
- Agent-generated files (code, documents, images)
- Build artifacts
- Logs archive
- Backups

**Cost**: ~$5/month (50GB storage + minimal transfer)

### 5. **Container Registry**

**Service**: Scaleway Container Registry
**Namespace**: agentops
**Privacy**: Private
**Region**: fr-par

**Usage**:
- Store all Docker images for Serverless Containers
- Automatic vulnerability scanning
- Image versioning and tagging
- Seamless integration with Serverless Containers

**Cost**: ~$5/month (10GB images)

### 6. **Networking** - VPC & Private Networks

**Service**: Virtual Private Cloud (VPC)
**Private Network**: agentops-network
**Region**: fr-par

**Attached Resources**:
- PostgreSQL (private access only)
- Redis (private access only)
- Serverless Containers (can access private resources)

**Security**:
- No public access to databases
- All inter-service communication over private network
- Reduced latency
- No data transfer costs within VPC

**Cost**: Free

### 7. **Secrets Management** - Secret Manager

**Service**: Scaleway Secret Manager
**Region**: fr-par

**Secrets Stored**:
- Database credentials
- Redis credentials
- API keys (Groq, OpenAI, Cartesia, Twilio)
- Telegram bot token
- GitHub tokens
- S3 access keys

**Integration**:
- Serverless Containers can inject secrets as env vars
- Automatic rotation support
- Audit logs

**Cost**: ~$5/month

### 8. **Monitoring & Observability** - Cockpit

**Service**: Scaleway Cockpit (Grafana + Prometheus + Loki)
**Included**: Free with Scaleway account

**Features**:
- Metrics for all Serverless Containers
- Logs aggregation (stdout/stderr from containers)
- Custom dashboards
- Alerting (Grafana alerts)
- Traces (OpenTelemetry compatible)

**Dashboards**:
- AgentOps System Health
- Container Performance
- Database Metrics
- Cost Tracking
- Agent Activity

**Cost**: Free (included)

## Deployment Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    PUBLIC INTERNET                           │
└────────┬────────────────────────────────────────────────────┘
         │
    ┌────▼────┐
    │  CDN    │ (Optional - for static assets)
    └────┬────┘
         │
┌────────▼────────────────────────────────────────────────────┐
│              Serverless Containers (Public)                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ AgentOps │  │   CoS    │  │  Voice   │  │ Telegram │   │
│  │   API    │  │  Agent   │  │Interface │  │   Bot    │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
└───────┼─────────────┼─────────────┼─────────────┼──────────┘
        │             │             │             │
┌───────▼─────────────▼─────────────▼─────────────▼──────────┐
│                  Private Network (VPC)                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │PostgreSQL│  │  Redis   │  │   Seed   │  │Specialist│   │
│  │  (HA)    │  │          │  │  Agent   │  │ Agents   │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└──────────────────────────────────────────────────────────────┘
        │             │
┌───────▼─────────────▼──────────┐
│     Object Storage (S3)        │
│    (Artifacts, Backups)        │
└────────────────────────────────┘
```

## Infrastructure as Code

**Tool**: Terraform (Scaleway provider)

**Repository Structure**:
```
infrastructure/
├── terraform/
│   ├── main.tf              # Main configuration
│   ├── variables.tf         # Input variables
│   ├── outputs.tf           # Output values
│   ├── backend.tf           # State backend (S3)
│   ├── modules/
│   │   ├── database/        # PostgreSQL module
│   │   ├── redis/           # Redis module
│   │   ├── storage/         # Object Storage module
│   │   ├── registry/        # Container Registry module
│   │   ├── vpc/             # VPC and Private Network module
│   │   ├── containers/      # Serverless Containers module
│   │   └── secrets/         # Secret Manager module
│   └── environments/
│       ├── dev.tfvars
│       ├── staging.tfvars
│       └── prod.tfvars
└── scripts/
    ├── deploy.sh            # Deployment script
    ├── destroy.sh           # Cleanup script
    └── backup.sh            # Backup script
```

## Scaling Strategy

### Horizontal Scaling (Serverless Containers)

**Automatic**:
- Containers scale from min to max instances based on load
- AgentOps API: min 1, max 10
- CoS Agent: min 1, max 5
- Seed Agent: min 1, max 3
- Specialists: min 0, max 100 (per specialist type)

**Triggers**:
- HTTP requests (immediate)
- Queue messages (batched)
- CRON schedules (time-based)

### Vertical Scaling (Databases)

**PostgreSQL**:
- Start: DB-GP-S (4 vCPU, 16GB RAM)
- Scale to: DB-GP-M (8 vCPU, 32GB RAM) if needed
- Zero-downtime migration

**Redis**:
- Start: REDIS-GP-S (2 vCPU, 8GB RAM)
- Scale to: REDIS-GP-M (4 vCPU, 16GB RAM) if needed
- Or switch to cluster mode for horizontal scaling

### Storage Scaling

**Object Storage**:
- Unlimited (pay per GB)
- Auto-scaling enabled

**PostgreSQL Storage**:
- Start: 50GB
- Auto-scaling enabled (up to 10TB)

## High Availability & Disaster Recovery

### Database HA

**PostgreSQL**:
- Multi-AZ deployment (fr-par-1, fr-par-2)
- Automatic failover (<30 seconds)
- Synchronous replication
- Daily backups (7-day retention)
- Point-in-time recovery

**Redis**:
- Single node (MVP)
- RDB + AOF persistence
- Can upgrade to cluster mode for HA

### Container HA

**Serverless Containers**:
- Automatically distributed across AZs
- Health checks every 30 seconds
- Automatic restart on failure
- No single point of failure

### Backup Strategy

**Databases**:
- Automated daily backups
- Manual backup before major changes
- Backup retention: 7 days
- Stored in Object Storage

**Object Storage**:
- Versioning enabled
- Lifecycle policies
- Cross-region replication (optional)

**Infrastructure**:
- Terraform state in S3 (versioned)
- Git repository for IaC

## Security

### Network Security

- Databases in Private Network (no public access)
- Serverless Containers: Public endpoints (HTTPS only)
- VPC isolation
- No SSH access needed (serverless)

### Access Control

**IAM**:
- Principle of least privilege
- Separate API keys for each service
- Rotate keys every 90 days

**Secrets**:
- All sensitive data in Secret Manager
- No secrets in code or environment variables
- Automatic injection into containers

### Encryption

**At Rest**:
- PostgreSQL: AES-256 encryption
- Redis: AES-256 encryption
- Object Storage: AES-256 encryption
- Secrets: AES-256 encryption

**In Transit**:
- HTTPS/TLS 1.3 for all public endpoints
- TLS for database connections
- Private Network for internal communication

### Compliance

- GDPR compliant (EU region)
- SOC 2 Type II (Scaleway certified)
- ISO 27001 (Scaleway certified)

## Cost Breakdown

| Service | Configuration | Monthly Cost |
|---------|--------------|--------------|
| PostgreSQL (HA) | DB-GP-S, 50GB | $80 |
| Redis | REDIS-GP-S | $40 |
| Serverless Containers (always-on) | 3 containers, min scale 1 | $30 |
| Serverless Containers (dynamic) | Variable load | $20 |
| Object Storage | 50GB + transfer | $5 |
| Container Registry | 10GB images | $5 |
| Secret Manager | 20 secrets | $5 |
| VPC & Private Network | - | Free |
| Cockpit (Monitoring) | - | Free |
| **Total** | | **$185/month** |

**Additional Costs** (external):
- Groq API: ~$10/month (1M tokens)
- Twilio (voice): ~$20/month (100 calls)
- **Grand Total**: ~$215/month

## Deployment Process

### Stage 0: Infrastructure Provisioning

1. **Create Terraform configuration**
2. **Initialize Terraform**: `terraform init`
3. **Plan deployment**: `terraform plan -out=plan.tfplan`
4. **Apply infrastructure**: `terraform apply plan.tfplan`
5. **Validate**:
   - PostgreSQL accessible via private network
   - Redis accessible via private network
   - Object Storage bucket created
   - Container Registry namespace created
   - Secrets stored in Secret Manager

### Stage 1: Container Images

1. **Build AgentOps API image**
2. **Build CoS Agent image**
3. **Build Seed Agent image**
4. **Build Voice Interface image**
5. **Build Telegram Bot image**
6. **Build Web Dashboard image**
7. **Push all images to Container Registry**

### Stage 2: Deploy Containers

1. **Deploy AgentOps API** (min scale: 1)
2. **Deploy CoS Agent** (min scale: 1)
3. **Deploy Seed Agent** (min scale: 1)
4. **Deploy Voice Interface** (min scale: 0)
5. **Deploy Telegram Bot** (min scale: 0)
6. **Deploy Web Dashboard** (min scale: 0)

### Stage 3: Configure & Test

1. **Apply database schema** (migrations)
2. **Configure secrets** (inject into containers)
3. **Set up monitoring** (Cockpit dashboards)
4. **Configure alerts** (Grafana)
5. **End-to-end test** (voice → task → execution)

## Monitoring & Alerts

### Key Metrics

**System Health**:
- Container uptime
- Request latency (p50, p95, p99)
- Error rate
- Database connections
- Redis memory usage

**Business Metrics**:
- Tasks created per day
- Tasks completed per day
- Average task completion time
- Agent utilization
- Cost per task

### Alerts

**Critical**:
- Database down
- Redis down
- Container crash loop
- Error rate > 5%
- Latency > 5s

**Warning**:
- Database CPU > 80%
- Redis memory > 80%
- Cost spike > 20%
- Task queue backlog > 100

## Advantages of This Architecture

### vs. Kubernetes

**Simpler**:
- No cluster management
- No node scaling
- No pod orchestration
- No Helm charts

**Cheaper**:
- No always-on nodes
- Pay per second of execution
- No control plane costs

**Faster**:
- Deploy in seconds (not minutes)
- Auto-scaling is instant
- No cluster warm-up

### vs. Traditional VMs

**More Scalable**:
- Auto-scale from 0 to 1000+
- No capacity planning
- No over-provisioning

**More Reliable**:
- Automatic failover
- No single points of failure
- Self-healing

**More Cost-Effective**:
- Pay only for usage
- No idle resources
- Granular billing (per second)

## Next Steps

1. **Provision infrastructure** (Stage 0)
2. **Build and deploy containers** (Stages 1-2)
3. **Configure and test** (Stage 3)
4. **Monitor and optimize** (ongoing)

**Estimated Time**: 2-3 days to production-ready system.
