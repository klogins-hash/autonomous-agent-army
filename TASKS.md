# Implementation Tasks

## Task Format

Each task follows this structure:
```yaml
id: task-XXX
title: Task title
status: pending | in_progress | completed
priority: low | medium | high | critical
dependencies: [task-YYY, task-ZZZ]
estimated_hours: N
acceptance_criteria:
  - Criterion 1
  - Criterion 2
implementation_notes: |
  Detailed notes for execution
```

---

## Phase 1: AgentOps Core (Week 1)

### task-001
```yaml
id: task-001
title: Set up PostgreSQL schema
status: pending
priority: critical
dependencies: []
estimated_hours: 2
acceptance_criteria:
  - PostgreSQL database created
  - Schema from IMPLEMENTATION_SPEC.md applied
  - All tables created with indexes
  - Migrations system in place (Alembic)
implementation_notes: |
  1. Install Alembic: pip install alembic
  2. Initialize: alembic init migrations
  3. Create initial migration with schema.sql
  4. Test migration up/down
  5. Verify all indexes created
  
  Files to create:
  - agentops/migrations/versions/001_initial_schema.py
  - agentops/alembic.ini
```

### task-002
```yaml
id: task-002
title: Implement AgentOps FastAPI server
status: pending
priority: critical
dependencies: [task-001]
estimated_hours: 8
acceptance_criteria:
  - FastAPI app structure created
  - All API endpoints from spec implemented
  - Database connection pooling configured
  - Redis client configured
  - Health check endpoint working
  - OpenAPI docs generated
implementation_notes: |
  Structure:
  agentops/
    app/
      __init__.py
      main.py
      config.py
      database.py
      redis.py
      api/
        __init__.py
        routes/
          projects.py
          tasks.py
          agents.py
          events.py
          artifacts.py
      models/
        __init__.py
        project.py
        task.py
        agent.py
        event.py
        artifact.py
      schemas/
        __init__.py
        (Pydantic models)
  
  Key dependencies:
  - fastapi
  - uvicorn
  - asyncpg
  - redis[asyncio]
  - pydantic
```

### task-003
```yaml
id: task-003
title: Implement event system (PostgreSQL + Redis pub/sub)
status: pending
priority: high
dependencies: [task-002]
estimated_hours: 4
acceptance_criteria:
  - Events written to PostgreSQL on every operation
  - Events published to Redis channels
  - WebSocket endpoint for event streaming
  - Subscribers can filter by event type
  - Event replay from PostgreSQL available
implementation_notes: |
  File: agentops/app/events.py
  
  class EventManager:
      async def publish(event_type, entity_type, entity_id, payload):
          # 1. Write to PostgreSQL
          # 2. Publish to Redis
          # 3. Notify WebSocket clients
  
  WebSocket endpoint: /api/v1/events/stream
  Query params: ?types=task.created,task.completed
```

### task-004
```yaml
id: task-004
title: Create AgentOps Python SDK
status: pending
priority: high
dependencies: [task-002, task-003]
estimated_hours: 6
acceptance_criteria:
  - SDK package created (agentops-sdk)
  - All API methods wrapped
  - Async/await support
  - Event subscription via WebSocket
  - Retry logic with exponential backoff
  - Type hints for all methods
  - Documentation with examples
implementation_notes: |
  Package structure:
  agentops-sdk/
    agentops/
      __init__.py
      client.py
      models.py
      exceptions.py
      events.py
    setup.py
    README.md
  
  Usage example:
  ```python
  from agentops import AgentOpsClient
  
  client = AgentOpsClient(
      url="http://localhost:8000",
      api_key="xxx"
  )
  
  # Create task
  task = await client.create_task({
      "title": "Build landing page",
      "project_id": "...",
      "priority": "high"
  })
  
  # Subscribe to events
  async for event in client.subscribe("tasks.created"):
      print(event)
  ```
```

### task-005
```yaml
id: task-005
title: Implement task claiming with optimistic locking
status: pending
priority: high
dependencies: [task-002]
estimated_hours: 3
acceptance_criteria:
  - POST /api/v1/tasks/{id}/claim endpoint works
  - Uses version field for optimistic locking
  - Returns 409 Conflict if already claimed
  - Updates assigned_to and status atomically
  - Publishes task.claimed event
implementation_notes: |
  SQL for atomic claim:
  ```sql
  UPDATE tasks
  SET 
      status = 'in_progress',
      assigned_to = $1,
      started_at = NOW(),
      version = version + 1,
      updated_by = $1
  WHERE 
      id = $2
      AND status = 'pending'
      AND version = $3
  RETURNING *;
  ```
  
  If no rows updated → 409 Conflict
```

---

## Phase 2: Chief of Staff Agent (Week 1)

### task-006
```yaml
id: task-006
title: Set up Strands framework for CoS agent
status: pending
priority: high
dependencies: [task-004]
estimated_hours: 4
acceptance_criteria:
  - Strands installed and configured
  - CoS agent project structure created
  - System prompt from spec implemented
  - Basic agent can respond to messages
  - AgentOps SDK integrated
implementation_notes: |
  Install Strands:
  pip install strands-agents
  
  Structure:
  agents/cos/
    __init__.py
    main.py
    tools.py
    prompts.py
    config.py
  
  System prompt in prompts.py
  Tools in tools.py (using AgentOps SDK)
```

### task-007
```yaml
id: task-007
title: Implement CoS agent tools (AgentOps integration)
status: pending
priority: high
dependencies: [task-006]
estimated_hours: 4
acceptance_criteria:
  - create_project tool works
  - create_task tool works
  - query_tasks tool works
  - update_task tool works
  - get_agent_status tool works
  - All tools have error handling
  - All tools log to AgentOps events
implementation_notes: |
  File: agents/cos/tools.py
  
  Each tool should:
  1. Validate inputs
  2. Call AgentOps SDK
  3. Handle errors gracefully
  4. Return structured response
  5. Log operation as event
  
  Use Strands @tool decorator
```

### task-008
```yaml
id: task-008
title: Create CoS agent HTTP API endpoint
status: pending
priority: high
dependencies: [task-007]
estimated_hours: 3
acceptance_criteria:
  - FastAPI server for CoS agent
  - POST /chat endpoint accepts message
  - Returns CoS response
  - Maintains conversation context
  - Handles concurrent requests
implementation_notes: |
  File: agents/cos/api.py
  
  Endpoint:
  POST /chat
  Body: {"message": "...", "session_id": "..."}
  Response: {"response": "...", "actions": [...]}
  
  Use Strands conversation management
  Store session state in Redis
```

---

## Phase 3: Seed Agent (Week 1)

### task-009
```yaml
id: task-009
title: Implement Seed agent with Strands
status: pending
priority: high
dependencies: [task-004]
estimated_hours: 6
acceptance_criteria:
  - Seed agent polls AgentOps for tasks
  - Claims tasks atomically
  - Executes tasks using Strands
  - Stores results as artifacts
  - Marks tasks complete
  - Handles errors gracefully
implementation_notes: |
  File: agents/seed/main.py
  
  Main loop:
  1. Subscribe to tasks.created events
  2. When new task: try to claim
  3. If claimed: execute
  4. Store result
  5. Mark complete
  
  Use Strands Agent class
  System prompt from spec
```

### task-010
```yaml
id: task-010
title: Implement meta-tooling for Seed agent
status: pending
priority: medium
dependencies: [task-009]
estimated_hours: 8
acceptance_criteria:
  - Seed agent can create new Python tools
  - Tools are tested before use
  - Tools are stored in AgentOps
  - Tools can be reused across tasks
  - Tool creation is logged
implementation_notes: |
  Use Strands meta-tooling capabilities
  
  Workflow:
  1. Agent identifies missing capability
  2. Generates tool code
  3. Tests tool in sandbox
  4. If works: register tool
  5. Use tool for task
  
  Store tools in:
  - File system: agents/seed/tools/generated/
  - AgentOps: as artifacts
```

### task-011
```yaml
id: task-011
title: Implement agent creation capability
status: pending
priority: low
dependencies: [task-009]
estimated_hours: 12
acceptance_criteria:
  - Seed agent can create specialist agents
  - New agents are deployed as containers
  - New agents register with AgentOps
  - New agents can execute tasks
  - Agent creation is logged
implementation_notes: |
  This is complex - may defer to Phase 3
  
  Workflow:
  1. Seed identifies need for specialist
  2. Generates agent spec (system prompt, tools)
  3. Creates Dockerfile
  4. Builds container
  5. Deploys to Scaleway Serverless Containers
  6. New agent registers with AgentOps
  
  Requires:
  - Scaleway API integration
  - Container registry access
  - Template system for agent creation
```

---

## Phase 4: Voice Interface (Week 2)

### task-012
```yaml
id: task-012
title: Integrate voice-stack with CoS agent
status: pending
priority: high
dependencies: [task-008]
estimated_hours: 4
acceptance_criteria:
  - voice-stack routes to CoS agent instead of direct LLM
  - Voice input → STT → CoS → TTS → Voice output works
  - Latency < 2 seconds end-to-end
  - Interruption handling still works
  - Conversation context maintained
implementation_notes: |
  Modify: voice-stack/app/pipeline/groq_llm.py
  
  Replace direct Groq call with HTTP call to CoS agent
  
  class CoSLLMService:
      async def generate(messages):
          # Call CoS agent
          response = await http_client.post(
              f"{COS_URL}/chat",
              json={"message": messages[-1]["content"]}
          )
          return response["response"]
```

### task-013
```yaml
id: task-013
title: Set up Twilio phone number for voice calls
status: pending
priority: medium
dependencies: [task-012]
estimated_hours: 2
acceptance_criteria:
  - Twilio account created
  - Phone number purchased
  - Webhook configured to voice-stack
  - Incoming calls work
  - Outgoing calls work (for proactive updates)
implementation_notes: |
  1. Sign up for Twilio
  2. Purchase phone number
  3. Configure webhook: https://voice.yourdomain.com/twilio/incoming
  4. Test incoming call
  5. Implement outgoing call function
  
  File: voice-stack/app/twilio_integration.py
```

---

## Phase 5: Telegram Interface (Week 2)

### task-014
```yaml
id: task-014
title: Implement Telegram bot
status: pending
priority: high
dependencies: [task-008]
estimated_hours: 6
acceptance_criteria:
  - Bot responds to messages
  - /task command creates tasks
  - /status command shows task status
  - Voice messages are transcribed
  - Inline keyboards for quick actions
  - Notifications for task completion
implementation_notes: |
  File: interfaces/telegram/bot.py
  
  Commands:
  - /task <description> - Create task
  - /status - Show active tasks
  - /agents - Show agent status
  - /help - Show help
  
  Use python-telegram-bot library
  Route messages to CoS agent
```

### task-015
```yaml
id: task-015
title: Implement Telegram notifications
status: pending
priority: medium
dependencies: [task-014]
estimated_hours: 3
acceptance_criteria:
  - Bot sends notification on task completion
  - Bot sends notification on task blocked
  - User can configure notification preferences
  - Notifications include relevant links
implementation_notes: |
  Subscribe to AgentOps events:
  - task.completed
  - task.blocked
  
  Send Telegram message with:
  - Task title
  - Result summary
  - Link to full result
  - Quick actions (approve, reject, etc.)
```

---

## Phase 6: Web Dashboard (Week 2)

### task-016
```yaml
id: task-016
title: Create Next.js web dashboard
status: pending
priority: medium
dependencies: [task-003]
estimated_hours: 12
acceptance_criteria:
  - Dashboard shows active tasks
  - Dashboard shows agent status
  - Dashboard shows recent completions
  - Real-time updates via WebSocket
  - Mobile responsive
  - Dark mode support
implementation_notes: |
  Create Next.js app:
  npx create-next-app@latest web-dashboard
  
  Pages:
  - / (overview)
  - /tasks (list)
  - /tasks/[id] (detail)
  - /agents (list)
  - /projects (list)
  
  Use TailwindCSS for styling
  Use WebSocket for real-time updates
```

---

## Phase 7: Deployment (Week 2)

### task-017
```yaml
id: task-017
title: Create Dockerfiles for all services
status: pending
priority: high
dependencies: [task-002, task-008, task-009, task-012, task-014]
estimated_hours: 4
acceptance_criteria:
  - Dockerfile for AgentOps API
  - Dockerfile for CoS agent
  - Dockerfile for Seed agent
  - Dockerfile for voice-stack
  - Dockerfile for Telegram bot
  - All images build successfully
  - Multi-stage builds for optimization
implementation_notes: |
  Use Python 3.11 slim base image
  Multi-stage build to reduce size
  
  Example:
  FROM python:3.11-slim as builder
  COPY requirements.txt .
  RUN pip install --user -r requirements.txt
  
  FROM python:3.11-slim
  COPY --from=builder /root/.local /root/.local
  COPY . .
  CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0"]
```

### task-018
```yaml
id: task-018
title: Set up Scaleway infrastructure
status: pending
priority: high
dependencies: [task-017]
estimated_hours: 6
acceptance_criteria:
  - PostgreSQL database created
  - Redis cluster created
  - Object Storage bucket created
  - Load Balancer configured
  - Private Network created
  - All services can communicate
implementation_notes: |
  Use Scaleway CLI or Terraform
  
  Resources:
  - scw rdb instance create (PostgreSQL)
  - scw redis cluster create
  - scw object bucket create
  - scw lb create
  - scw vpc private-network create
  
  Save connection strings to .env
```

### task-019
```yaml
id: task-019
title: Deploy services to Scaleway
status: pending
priority: high
dependencies: [task-018]
estimated_hours: 4
acceptance_criteria:
  - AgentOps API deployed and accessible
  - CoS agent deployed
  - Seed agent deployed
  - Voice-stack deployed
  - Telegram bot deployed
  - All services healthy
  - Monitoring configured
implementation_notes: |
  Use Scaleway Serverless Containers
  
  Deploy:
  1. Push images to Scaleway Container Registry
  2. Create serverless containers
  3. Configure environment variables
  4. Set up auto-scaling
  5. Configure health checks
  
  Commands:
  scw registry namespace create name=agentops
  docker tag agentops-api registry.scaleway.com/agentops/api
  docker push registry.scaleway.com/agentops/api
  scw container container create ...
```

---

## Phase 8: Testing & Polish (Week 3)

### task-020
```yaml
id: task-020
title: Write integration tests
status: pending
priority: medium
dependencies: [task-019]
estimated_hours: 8
acceptance_criteria:
  - E2E test: Voice → Task → Execution → Completion
  - E2E test: Telegram → Task → Notification
  - Integration test: CoS → AgentOps
  - Integration test: Seed → Task execution
  - All tests passing
implementation_notes: |
  Use pytest + pytest-asyncio
  
  File: tests/integration/test_e2e.py
  
  Test scenarios:
  1. User calls → CoS creates task → Seed executes → User notified
  2. User sends Telegram message → Task created → Completed → Notification
  3. CoS creates project → Creates tasks → Seed executes all
```

### task-021
```yaml
id: task-021
title: Set up monitoring and alerting
status: pending
priority: medium
dependencies: [task-019]
estimated_hours: 4
acceptance_criteria:
  - Prometheus metrics exposed
  - Grafana dashboards created
  - Alerts configured for errors
  - Uptime monitoring configured
  - Cost tracking dashboard
implementation_notes: |
  Metrics to track:
  - Task creation rate
  - Task completion rate
  - Agent utilization
  - API latency
  - Error rates
  - Database connections
  
  Use Scaleway Cockpit + Grafana
```

### task-022
```yaml
id: task-022
title: Write documentation
status: pending
priority: low
dependencies: [task-019]
estimated_hours: 4
acceptance_criteria:
  - README.md updated
  - API documentation complete
  - User guide created
  - Agent development guide created
  - Deployment guide created
implementation_notes: |
  Documents to create:
  - README.md (overview)
  - docs/API.md (API reference)
  - docs/USER_GUIDE.md (how to use)
  - docs/AGENT_GUIDE.md (how to create agents)
  - docs/DEPLOYMENT.md (how to deploy)
  
  Use clear examples and screenshots
```

---

## Summary

**Total estimated hours**: 115 hours (~3 weeks for 1 agent)

**Critical path**:
1. AgentOps Core (tasks 001-005)
2. CoS Agent (tasks 006-008)
3. Voice Integration (task 012)
4. Deployment (tasks 017-019)

**Can be parallelized**:
- Seed Agent (tasks 009-011)
- Telegram Interface (tasks 014-015)
- Web Dashboard (task 016)

**Optional/Future**:
- Agent creation (task 011)
- Advanced monitoring (task 021)
- Documentation (task 022)
