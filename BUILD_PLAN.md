# Layered Build Plan: AgentOps System on Scaleway

**Philosophy**: Build like creation - foundation first, each layer depends on the previous.

## Layer Architecture

```
┌─────────────────────────────────────────────────┐
│  Layer 7: Interfaces (Voice, Telegram, Web)     │  ← User interaction
├─────────────────────────────────────────────────┤
│  Layer 6: Agents (CoS, Seed, Specialists)       │  ← Intelligence
├─────────────────────────────────────────────────┤
│  Layer 5: SDK (Python Client Library)           │  ← Developer tools
├─────────────────────────────────────────────────┤
│  Layer 4: Events (Redis Pub/Sub, WebSocket)     │  ← Real-time comm
├─────────────────────────────────────────────────┤
│  Layer 3: API (FastAPI REST Endpoints)          │  ← Business logic
├─────────────────────────────────────────────────┤
│  Layer 2: Data (PostgreSQL Schema & Models)     │  ← State persistence
├─────────────────────────────────────────────────┤
│  Layer 1: Infrastructure (Scaleway Resources)   │  ← Foundation
└─────────────────────────────────────────────────┘
```

## Stage 0: Infrastructure Foundation (Scaleway)

**Purpose**: Provision all cloud resources before writing any code

### Resources to Create

**Database**:
- PostgreSQL 15 Managed Database
- Instance type: DB-DEV-S (2 vCPU, 2GB RAM) - $15/month
- High Availability: No (for MVP)
- Backups: Daily
- Region: fr-par

**Cache**:
- Redis Managed Database  
- Instance type: REDIS-DEV-S (1 vCPU, 1GB RAM) - $10/month
- Cluster mode: No (for MVP)
- Region: fr-par

**Storage**:
- Object Storage bucket
- Name: agentops-artifacts
- Region: fr-par
- Versioning: Enabled
- Public access: No

**Container Registry**:
- Registry namespace: agentops
- Privacy: Private
- Region: fr-par

**Networking**:
- Private Network: agentops-network
- Load Balancer: agentops-lb (for later)

### Validation Criteria
- [ ] PostgreSQL accessible and empty
- [ ] Redis accessible and empty
- [ ] S3 bucket created
- [ ] Container registry ready
- [ ] All connection strings saved to .env file

---

## Stage 1: Data Layer (PostgreSQL)

**Purpose**: Define the data model and schema

### Tasks

**1.1 Create Schema**
```sql
-- Tables: projects, tasks, events, agents, artifacts
-- Indexes: All performance-critical queries
-- Constraints: Data integrity rules
```

**1.2 Create Migration System**
- Alembic for schema migrations
- Initial migration with full schema
- Rollback capability

**1.3 Create SQLAlchemy Models**
- Python models for all tables
- Relationships defined
- Type hints

**1.4 Create Database Connection Pool**
- asyncpg connection pool
- Connection limits
- Health checks

### Validation Criteria
- [ ] Schema applied to Scaleway PostgreSQL
- [ ] All tables exist with correct structure
- [ ] Indexes created
- [ ] Can connect from Python
- [ ] Can perform basic CRUD operations

### Dependencies
- Requires: Stage 0 (PostgreSQL provisioned)
- Blocks: Stage 2 (API needs models)

---

## Stage 2: Core API Layer (FastAPI)

**Purpose**: REST API for all AgentOps operations

### Tasks

**2.1 FastAPI Application Structure**
```
agentops-api/
  app/
    __init__.py
    main.py          # FastAPI app
    config.py        # Settings
    database.py      # DB connection
    dependencies.py  # Dependency injection
```

**2.2 Implement Endpoints**
- Projects: CRUD operations
- Tasks: CRUD + claim/complete/block
- Agents: Register, heartbeat, list
- Artifacts: Upload, retrieve
- Health: /health endpoint

**2.3 Request/Response Models**
- Pydantic schemas for all endpoints
- Validation rules
- Error responses

**2.4 Authentication**
- API key authentication
- Rate limiting
- CORS configuration

**2.5 Deploy to Scaleway Serverless Containers**
- Dockerfile
- Build and push to registry
- Deploy container
- Configure environment variables
- Test endpoints

### Validation Criteria
- [ ] All endpoints respond correctly
- [ ] Can create/read/update/delete all entities
- [ ] Authentication works
- [ ] Deployed and accessible via HTTPS
- [ ] Health check passes

### Dependencies
- Requires: Stage 1 (Database models)
- Blocks: Stage 3 (Events need API)

---

## Stage 3: Event Layer (Redis + WebSocket)

**Purpose**: Real-time event broadcasting

### Tasks

**3.1 Event Manager**
- Publish events to PostgreSQL (audit log)
- Publish events to Redis (real-time)
- Event types enum

**3.2 Redis Pub/Sub**
- Channels for each event type
- Subscribe/unsubscribe logic
- Connection pooling

**3.3 WebSocket Endpoint**
- /api/v1/events/stream endpoint
- Filter by event types
- Authentication
- Heartbeat/keepalive

**3.4 Integrate with API**
- All API operations publish events
- Event payload includes full entity data

**3.5 Deploy Updates**
- Update container with event support
- Configure Redis connection
- Test WebSocket connection

### Validation Criteria
- [ ] Events written to PostgreSQL
- [ ] Events published to Redis
- [ ] WebSocket clients receive events
- [ ] Can filter events by type
- [ ] No events lost

### Dependencies
- Requires: Stage 2 (API endpoints)
- Blocks: Stage 4 (SDK needs events)

---

## Stage 4: SDK Layer (Python Client)

**Purpose**: Easy-to-use client library for agents

### Tasks

**4.1 SDK Package Structure**
```
agentops-sdk/
  agentops/
    __init__.py
    client.py        # Main client class
    models.py        # Data models
    events.py        # Event subscription
    exceptions.py    # Custom exceptions
  setup.py
  README.md
```

**4.2 Implement Client Methods**
- All API operations wrapped
- Async/await support
- Automatic retries
- Type hints

**4.3 Event Subscription**
- WebSocket connection
- Async iterator for events
- Reconnection logic

**4.4 Error Handling**
- Retry with exponential backoff
- Custom exceptions
- Logging

**4.5 Documentation**
- Usage examples
- API reference
- Installation guide

**4.6 Publish Package**
- Upload to PyPI (or private registry)
- Version 0.1.0

### Validation Criteria
- [ ] Can install via pip
- [ ] All API operations work
- [ ] Event subscription works
- [ ] Retries work on failure
- [ ] Documentation complete

### Dependencies
- Requires: Stage 3 (Events API)
- Blocks: Stage 5 (Agents need SDK)

---

## Stage 5: Agent Layer (CoS + Seed)

**Purpose**: Intelligent agents that use AgentOps

### Tasks

**5.1 Chief of Staff Agent**
- Strands framework setup
- System prompt implementation
- Tools (create_project, create_task, etc.)
- HTTP API endpoint (/chat)
- Conversation context management
- Deploy to Scaleway Serverless Container

**5.2 Seed Agent**
- Strands framework setup
- System prompt implementation
- Main loop (poll tasks, execute)
- Task claiming logic
- Result storage
- Deploy to Scaleway Serverless Container

**5.3 Meta-Tooling (Seed)**
- Tool creation capability
- Tool testing
- Tool storage

**5.4 Agent Communication**
- CoS can create tasks for Seed
- Seed reports progress to CoS
- Event-driven coordination

### Validation Criteria
- [ ] CoS responds to messages
- [ ] CoS creates tasks in AgentOps
- [ ] Seed polls for tasks
- [ ] Seed executes simple tasks
- [ ] Seed stores results
- [ ] Both agents deployed and running

### Dependencies
- Requires: Stage 4 (SDK for agents)
- Blocks: Stage 6 (Interfaces need agents)

---

## Stage 6: Interface Layer (Voice, Telegram, Web)

**Purpose**: Human interaction with the system

### Tasks

**6.1 Voice Interface**
- Integrate voice-stack with CoS agent
- Modify LLM pipeline to call CoS
- Test voice → task creation flow
- Deploy voice-stack to Scaleway
- Set up Twilio (optional for now)

**6.2 Telegram Bot**
- Bot implementation
- Commands: /task, /status, /agents
- Message routing to CoS
- Voice message transcription
- Notifications (task complete, blocked)
- Deploy to Scaleway Serverless Container

**6.3 Web Dashboard**
- Next.js application
- Pages: Dashboard, Tasks, Agents, Projects
- Real-time updates (WebSocket)
- Mobile responsive
- Deploy to Vercel or Scaleway

**6.4 Interface Gateway**
- Unified authentication
- Intent parsing (NLU)
- Response formatting
- Failover logic

### Validation Criteria
- [ ] Can create tasks via voice
- [ ] Can create tasks via Telegram
- [ ] Can view tasks in web dashboard
- [ ] Real-time updates work
- [ ] All interfaces deployed

### Dependencies
- Requires: Stage 5 (Agents to interact with)
- Blocks: Stage 7 (Deployment validation)

---

## Stage 7: Deployment & Validation

**Purpose**: Production-ready system

### Tasks

**7.1 Monitoring**
- Prometheus metrics
- Grafana dashboards
- Error tracking (Sentry)
- Cost monitoring

**7.2 Security Hardening**
- HTTPS everywhere
- API key rotation
- Database encryption
- S3 bucket policies

**7.3 Documentation**
- System architecture diagram
- API documentation
- User guide
- Agent development guide
- Deployment runbook

**7.4 Testing**
- E2E test: Voice → Task → Execution → Completion
- Load testing
- Failure recovery testing

**7.5 Backup & Recovery**
- Database backups configured
- Disaster recovery plan
- Restore testing

### Validation Criteria
- [ ] All services monitored
- [ ] Security audit passed
- [ ] Documentation complete
- [ ] E2E tests passing
- [ ] Backups working

### Dependencies
- Requires: Stage 6 (All components)
- Blocks: Nothing (system complete)

---

## Execution Plan

### Current Status
- ✅ Scaleway CLI configured
- ✅ Credentials set
- ⏳ Ready to start Stage 0

### Next Steps
1. Execute Stage 0: Provision infrastructure
2. Execute Stage 1: Create database schema
3. Execute Stage 2: Build and deploy API
4. Continue through stages sequentially

### Time Estimates
- Stage 0: 1 hour
- Stage 1: 2 hours
- Stage 2: 4 hours
- Stage 3: 3 hours
- Stage 4: 3 hours
- Stage 5: 6 hours
- Stage 6: 8 hours
- Stage 7: 4 hours

**Total**: ~31 hours (can be done over 1-2 weeks)

### Principles
1. **Foundation first**: Never skip a stage
2. **Validate each layer**: Don't proceed until validation passes
3. **Deploy continuously**: Every stage deploys to Scaleway
4. **Document as you go**: Update docs after each stage
5. **Test immediately**: Validate before moving to next layer

---

## Current Stage: Stage 0

**Status**: Ready to begin  
**Next Action**: Provision Scaleway infrastructure

Shall I proceed with Stage 0?
