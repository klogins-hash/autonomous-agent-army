# Implementation Specification: Autonomous Agent System
**Version**: 1.0  
**Target**: Agent Execution  
**Format**: Technical Specification

## System Overview

```
Voice Interface → CoS Agent → AgentOps → Execution Agents → Results
     ↓              ↓            ↓             ↓              ↓
  Telegram      Interface    PostgreSQL    Strands       Artifacts
     ↓           Gateway       Redis        Runtime         S3
   Web UI                                                    
```

## Component Specifications

### 1. AgentOps Core

**Purpose**: Agent-native task management and coordination system

**Technology Stack**:
- FastAPI 0.104+
- PostgreSQL 15+ (Scaleway Managed)
- Redis 7+ (Scaleway Managed)
- Python 3.11+

**Database Schema**:
```sql
-- File: schema.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL UNIQUE,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'active',
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE tasks (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    title TEXT NOT NULL,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'pending',
    priority TEXT NOT NULL DEFAULT 'medium',
    assigned_to TEXT,
    context JSONB DEFAULT '{}',
    result JSONB,
    version INTEGER DEFAULT 1,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_by TEXT,
    updated_by TEXT,
    CONSTRAINT valid_status CHECK (status IN ('pending', 'in_progress', 'blocked', 'completed', 'cancelled')),
    CONSTRAINT valid_priority CHECK (priority IN ('low', 'medium', 'high', 'critical'))
);

CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    event_type TEXT NOT NULL,
    entity_type TEXT NOT NULL,
    entity_id UUID NOT NULL,
    agent_name TEXT,
    payload JSONB DEFAULT '{}',
    timestamp TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE agents (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT UNIQUE NOT NULL,
    type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active',
    capabilities JSONB DEFAULT '[]',
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    last_seen_at TIMESTAMPTZ DEFAULT NOW(),
    CONSTRAINT valid_agent_status CHECK (status IN ('active', 'idle', 'busy', 'offline'))
);

CREATE TABLE artifacts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    task_id UUID REFERENCES tasks(id) ON DELETE CASCADE,
    type TEXT NOT NULL,
    storage_url TEXT NOT NULL,
    metadata JSONB DEFAULT '{}',
    created_by TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_tasks_priority ON tasks(priority);
CREATE INDEX idx_tasks_assigned ON tasks(assigned_to);
CREATE INDEX idx_tasks_project ON tasks(project_id);
CREATE INDEX idx_events_entity ON events(entity_type, entity_id);
CREATE INDEX idx_events_timestamp ON events(timestamp DESC);
CREATE INDEX idx_agents_status ON agents(status);
```

**API Endpoints**:
```python
# File: app/api/routes.py

# Projects
POST   /api/v1/projects                    # Create project
GET    /api/v1/projects                    # List projects
GET    /api/v1/projects/{id}               # Get project
PATCH  /api/v1/projects/{id}               # Update project
DELETE /api/v1/projects/{id}               # Delete project

# Tasks
POST   /api/v1/tasks                       # Create task
GET    /api/v1/tasks                       # Query tasks (with filters)
GET    /api/v1/tasks/{id}                  # Get task
PATCH  /api/v1/tasks/{id}                  # Update task
POST   /api/v1/tasks/{id}/claim            # Claim task (atomic)
POST   /api/v1/tasks/{id}/complete         # Complete task
POST   /api/v1/tasks/{id}/block            # Block task (needs input)

# Events
GET    /api/v1/events                      # Query events
WS     /api/v1/events/stream               # Real-time event stream

# Agents
POST   /api/v1/agents/register             # Register agent
POST   /api/v1/agents/{name}/heartbeat     # Update last_seen
GET    /api/v1/agents                      # List agents
GET    /api/v1/agents/{name}               # Get agent

# Artifacts
POST   /api/v1/artifacts                   # Upload artifact
GET    /api/v1/artifacts/{id}              # Get artifact metadata
```

**Event Types**:
```python
# File: app/models/events.py

class EventType:
    # Project events
    PROJECT_CREATED = "project.created"
    PROJECT_UPDATED = "project.updated"
    PROJECT_DELETED = "project.deleted"
    
    # Task events
    TASK_CREATED = "task.created"
    TASK_CLAIMED = "task.claimed"
    TASK_UPDATED = "task.updated"
    TASK_COMPLETED = "task.completed"
    TASK_BLOCKED = "task.blocked"
    TASK_CANCELLED = "task.cancelled"
    
    # Agent events
    AGENT_REGISTERED = "agent.registered"
    AGENT_ONLINE = "agent.online"
    AGENT_OFFLINE = "agent.offline"
    
    # Artifact events
    ARTIFACT_CREATED = "artifact.created"
```

**Redis Pub/Sub Channels**:
```
events:all              # All events
events:tasks            # Task-related events
events:agents           # Agent-related events
tasks:pending           # New pending tasks
tasks:completed         # Completed tasks
```

### 2. Chief of Staff Agent

**Purpose**: Voice-first strategic partner and task orchestrator

**Technology Stack**:
- Strands framework
- OpenAI GPT-4 (via Groq API for speed)
- AgentOps SDK (Python client)

**System Prompt**:
```
You are the Chief of Staff for a visionary entrepreneur with INFJ personality 
and combined-type ADHD. You communicate primarily via voice.

ROLE:
- Strategic partner, not just task executor
- Proactive, not reactive
- Organized but flexible
- Supportive and empathetic

CAPABILITIES:
- Create and manage projects and tasks in AgentOps
- Break down complex ideas into structured work
- Ask clarifying questions (concisely)
- Delegate to specialist agents
- Track progress and report back
- Make decisions within scope, escalate when needed

COMMUNICATION STYLE:
- Natural and conversational (you're on a phone call)
- Concise but warm (respect limited attention)
- Use "I" statements ("I'm creating..." not "Creating...")
- Acknowledge and validate ideas
- Provide clear next steps

CONSTRAINTS:
- Keep responses under 30 seconds of speech
- Ask max 3 questions at a time
- Summarize complex info into key points
- Default to action over discussion

TOOLS AVAILABLE:
- create_project(name, description)
- create_task(project, title, description, priority, context)
- query_tasks(filters)
- update_task(task_id, updates)
- get_agent_status()
- create_artifact(task_id, type, content)
```

**Tools Implementation**:
```python
# File: agents/cos/tools.py

from agentops import AgentOpsClient

class CoSTools:
    def __init__(self, agentops_url: str, api_key: str):
        self.ops = AgentOpsClient(url=agentops_url, api_key=api_key)
    
    async def create_project(self, name: str, description: str) -> dict:
        """Create a new project"""
        return await self.ops.create_project({
            "name": name,
            "description": description,
            "status": "active"
        })
    
    async def create_task(
        self,
        project: str,
        title: str,
        description: str = "",
        priority: str = "medium",
        context: dict = None
    ) -> dict:
        """Create a new task"""
        project_obj = await self.ops.get_project_by_name(project)
        return await self.ops.create_task({
            "project_id": project_obj["id"],
            "title": title,
            "description": description,
            "priority": priority,
            "context": context or {},
            "created_by": "cos-agent"
        })
    
    async def query_tasks(self, filters: dict = None) -> list:
        """Query tasks with filters"""
        return await self.ops.query_tasks(filters or {})
    
    async def update_task(self, task_id: str, updates: dict) -> dict:
        """Update a task"""
        return await self.ops.update_task(task_id, updates)
    
    async def get_agent_status(self) -> list:
        """Get status of all agents"""
        return await self.ops.list_agents()
```

### 3. Seed Agent

**Purpose**: General-purpose execution agent with self-improvement capabilities

**Technology Stack**:
- Strands framework
- AgentOps SDK
- Meta-tooling enabled

**System Prompt**:
```
You are the Seed Agent, the founding execution agent of an autonomous company.

ROLE:
- Execute tasks assigned to you
- Create tools when you lack capabilities
- Spawn specialist agents when tasks are complex
- Learn and improve over time

CAPABILITIES:
- Execute tasks from AgentOps queue
- Create Python tools using meta-tooling
- Create new specialist agents
- Store results as artifacts
- Report progress

WORKFLOW:
1. Poll AgentOps for pending tasks
2. Claim task (atomic operation)
3. Analyze task requirements
4. If you have tools: Execute
5. If you need tools: Create them, then execute
6. If task is complex: Create specialist agent and delegate
7. Store results as artifacts
8. Mark task complete with result

DECISION FRAMEWORK:
- Simple task (<30 min): Do it yourself
- Need new capability: Create tool first
- Complex task (>2 hours): Create specialist agent
- Blocked: Mark task as blocked with reason

TOOLS AVAILABLE:
- All AgentOps SDK methods
- create_tool(spec) - Meta-tooling
- create_agent(spec) - Agent creation
- execute_code(code) - Sandboxed execution
- web_search(query)
- web_scrape(url)
- file_operations (read, write, etc.)
```

**Main Loop**:
```python
# File: agents/seed/main.py

import asyncio
from agentops import AgentOpsClient
from strands import Agent

class SeedAgent:
    def __init__(self, agentops_url: str, api_key: str):
        self.ops = AgentOpsClient(url=agentops_url, api_key=api_key)
        self.agent = Agent(name="seed", system_prompt=SEED_PROMPT)
        
    async def run(self):
        """Main execution loop"""
        # Register with AgentOps
        await self.ops.register_agent({
            "name": "seed",
            "type": "orchestrator",
            "capabilities": ["general-execution", "tool-creation", "agent-creation"]
        })
        
        # Subscribe to task events
        async for event in self.ops.subscribe("tasks.created", "tasks.pending"):
            if event.type == "tasks.created":
                await self.handle_new_task(event.payload["task_id"])
    
    async def handle_new_task(self, task_id: str):
        """Process a new task"""
        # Try to claim task
        try:
            task = await self.ops.claim_task(task_id, agent="seed")
        except ConflictError:
            # Another agent claimed it
            return
        
        # Execute task
        try:
            result = await self.execute_task(task)
            
            # Mark complete
            await self.ops.complete_task(
                task_id,
                result=result,
                artifacts=result.get("artifacts", [])
            )
        except Exception as e:
            # Mark as blocked
            await self.ops.block_task(
                task_id,
                reason=str(e),
                agent="seed"
            )
    
    async def execute_task(self, task: dict) -> dict:
        """Execute a task using Strands agent"""
        # Let the agent figure out how to do it
        response = await self.agent.run(
            f"""Execute this task:
            
            Title: {task['title']}
            Description: {task['description']}
            Context: {task['context']}
            
            Use your tools to complete this task and return the results.
            """
        )
        
        return response
```

### 4. Voice Interface

**Purpose**: Real-time voice interaction with CoS agent

**Technology Stack**:
- Existing voice-stack (Groq + Cartesia)
- Twilio for phone number
- WebRTC for browser calls

**Integration Point**:
```python
# File: voice-stack/app/pipeline/groq_llm.py

# Modify to route through CoS agent instead of direct LLM call

class CoSLLMService:
    def __init__(self, cos_agent_url: str):
        self.cos_url = cos_agent_url
    
    async def generate(self, messages: list) -> str:
        """Send to CoS agent instead of direct LLM"""
        # Extract user message
        user_message = messages[-1]["content"]
        
        # Call CoS agent
        response = await self.call_cos_agent(user_message)
        
        return response
    
    async def call_cos_agent(self, message: str) -> str:
        """Call CoS agent via HTTP"""
        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{self.cos_url}/chat",
                json={"message": message}
            ) as resp:
                data = await resp.json()
                return data["response"]
```

### 5. Telegram Interface

**Purpose**: Text/voice messaging interface

**Technology Stack**:
- python-telegram-bot 20+
- Async/await pattern

**Implementation**:
```python
# File: interfaces/telegram/bot.py

from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler
from agentops import AgentOpsClient

class TelegramInterface:
    def __init__(self, token: str, agentops_url: str, api_key: str):
        self.app = Application.builder().token(token).build()
        self.ops = AgentOpsClient(url=agentops_url, api_key=api_key)
        
        # Register handlers
        self.app.add_handler(CommandHandler("task", self.create_task))
        self.app.add_handler(CommandHandler("status", self.get_status))
        self.app.add_handler(MessageHandler(filters.TEXT, self.handle_message))
        self.app.add_handler(MessageHandler(filters.VOICE, self.handle_voice))
    
    async def create_task(self, update: Update, context):
        """Handle /task command"""
        # Parse: /task Build landing page @project-name !high
        text = " ".join(context.args)
        
        # Use CoS agent to parse and create task
        response = await self.call_cos_agent(f"Create task: {text}")
        
        await update.message.reply_text(response)
    
    async def get_status(self, update: Update, context):
        """Handle /status command"""
        # Query AgentOps
        tasks = await self.ops.query_tasks({"status": "in_progress"})
        
        status_text = f"📊 Active Tasks: {len(tasks)}\n\n"
        for task in tasks[:5]:
            status_text += f"🟡 {task['title']}\n"
            status_text += f"   Assigned: {task['assigned_to']}\n\n"
        
        await update.message.reply_text(status_text)
    
    async def handle_message(self, update: Update, context):
        """Handle text messages"""
        # Route to CoS agent
        response = await self.call_cos_agent(update.message.text)
        await update.message.reply_text(response)
    
    async def handle_voice(self, update: Update, context):
        """Handle voice messages"""
        # Download voice
        voice_file = await update.message.voice.get_file()
        voice_bytes = await voice_file.download_as_bytearray()
        
        # Transcribe (using Cartesia or Whisper)
        transcript = await self.transcribe(voice_bytes)
        
        # Process with CoS
        response = await self.call_cos_agent(transcript)
        
        # Send text response (or voice if preferred)
        await update.message.reply_text(f"You said: {transcript}\n\n{response}")
```

### 6. Web Dashboard

**Purpose**: Visual overview and batch operations

**Technology Stack**:
- Next.js 14
- React 18
- TailwindCSS
- WebSocket for real-time updates

**Pages**:
```
/                       # Dashboard (overview)
/tasks                  # Task list
/tasks/[id]             # Task detail
/agents                 # Agent status
/projects               # Project list
/projects/[id]          # Project detail
```

**Real-time Updates**:
```typescript
// File: web/lib/agentops.ts

export class AgentOpsClient {
  private ws: WebSocket;
  
  constructor(url: string) {
    this.ws = new WebSocket(`${url}/api/v1/events/stream`);
    
    this.ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      this.handleEvent(data);
    };
  }
  
  handleEvent(event: Event) {
    switch (event.type) {
      case 'task.created':
        // Update UI
        break;
      case 'task.completed':
        // Show notification
        break;
    }
  }
}
```

## Deployment Architecture

### Scaleway Resources

**Compute**:
- 1x GP1-S instance (AgentOps API server)
- 1x Serverless Container (CoS Agent)
- 1x Serverless Container (Seed Agent)
- 1x Serverless Container (Voice Stack)

**Databases**:
- 1x Managed PostgreSQL (HA cluster, 2GB RAM)
- 1x Managed Redis (3-node cluster, 2GB RAM)

**Storage**:
- 1x Object Storage bucket (artifacts)

**Networking**:
- 1x Load Balancer (public endpoints)
- 1x Private Network (inter-service communication)

**Estimated Cost**: ~$150/month

### Docker Compose (Development)

```yaml
# File: docker-compose.yml

version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: agentops
      POSTGRES_USER: agentops
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./schema.sql:/docker-entrypoint-initdb.d/schema.sql
    ports:
      - "5432:5432"
  
  redis:
    image: redis:7
    ports:
      - "6379:6379"
  
  agentops-api:
    build: ./agentops
    environment:
      DATABASE_URL: postgresql://agentops:${DB_PASSWORD}@postgres:5432/agentops
      REDIS_URL: redis://redis:6379
    ports:
      - "8000:8000"
    depends_on:
      - postgres
      - redis
  
  cos-agent:
    build: ./agents/cos
    environment:
      AGENTOPS_URL: http://agentops-api:8000
      AGENTOPS_API_KEY: ${AGENTOPS_API_KEY}
    depends_on:
      - agentops-api
  
  seed-agent:
    build: ./agents/seed
    environment:
      AGENTOPS_URL: http://agentops-api:8000
      AGENTOPS_API_KEY: ${AGENTOPS_API_KEY}
    depends_on:
      - agentops-api
  
  voice-stack:
    build: ./voice-stack
    environment:
      COS_AGENT_URL: http://cos-agent:8001
      CARTESIA_API_KEY: ${CARTESIA_API_KEY}
      GROQ_API_KEY: ${GROQ_API_KEY}
    ports:
      - "8080:8080"
    depends_on:
      - cos-agent

volumes:
  postgres_data:
```

## Implementation Tasks

See TASKS.md for detailed task breakdown.

## Testing Strategy

**Unit Tests**:
- AgentOps API endpoints
- Database operations
- Event publishing/subscribing

**Integration Tests**:
- CoS agent → AgentOps flow
- Seed agent → Task execution
- Voice → CoS → AgentOps chain

**E2E Tests**:
- Voice call → Task creation → Execution → Completion
- Telegram message → Task creation → Notification

## Monitoring

**Metrics to Track**:
- Task creation rate
- Task completion rate
- Average task duration
- Agent utilization
- API latency
- Error rates

**Tools**:
- Prometheus (metrics)
- Grafana (dashboards)
- Sentry (error tracking)

## Security

**Authentication**:
- API keys for agent-to-AgentOps communication
- JWT for web dashboard
- Telegram user ID verification
- Twilio webhook signature verification

**Authorization**:
- Agents can only update tasks they've claimed
- Read-only access for dashboard
- Admin API key for system operations

**Data Protection**:
- TLS for all HTTP communication
- Encrypted database connections
- S3 bucket encryption
- No sensitive data in logs
