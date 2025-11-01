# AgentOps Deployment Completion Guide

**For**: Autonomous agents (Manus or deployed agents)  
**Status**: Build in progress, deployment pending  
**Last Updated**: November 1, 2025

---

## Current State

### ✅ Infrastructure Running
- Serverless SQL Database (PostgreSQL)
- Redis Cluster
- NATS Messaging
- Object Storage
- Container Registry (empty)
- Serverless Containers Namespace (no containers yet)

### 🔄 Build Instance Running
**Instance ID**: `1582b6b5-aaa3-4257-b597-60fe426c618c`  
**IP**: `212.47.244.116`  
**Type**: DEV1-M (3 vCPU, 4GB RAM)  
**Status**: Running automated build script  
**Cost**: ~$0.02/hour (~$14/month if kept running)

**Build Process**:
1. Install Docker ✓
2. Clone agentops-api repository ✓
3. Build Docker image (in progress)
4. Push to Container Registry (pending)

---

## Task 1: Verify Build Completion

**Objective**: Check if Docker image was successfully built and pushed

**Commands**:
```bash
# Check if image exists in Container Registry
scw registry image list \
  namespace-id=3af1f2d4-4253-464e-beef-eaf688b7342f \
  region=fr-par \
  -o json | python3 -c "import sys,json; images=json.load(sys.stdin); print('✅ Image found' if any('agentops-api' in img.get('name','') for img in images) else '❌ Image not found')"
```

**Expected Output**: `✅ Image found`

**If image not found**:
- Wait 5 more minutes (build takes 5-10 min total)
- Or check build instance logs (requires SSH access)

**Acceptance Criteria**:
- [ ] Docker image `agentops-api:latest` exists in Container Registry
- [ ] Image size is reasonable (~200-500MB)

---

## Task 2: Deploy AgentOps API Container

**Objective**: Deploy the API as a Serverless Container

**Prerequisites**:
- Task 1 completed successfully
- Scaleway CLI configured with credentials

**Commands**:
```bash
# Deploy Serverless Container
scw container container create \
  namespace-id=3af1f2d4-4253-464e-beef-eaf688b7342f \
  name=agentops-api \
  registry-image=rg.fr-par.scw.cloud/prod-agentops/agentops-api:latest \
  port=8000 \
  min-scale=1 \
  max-scale=5 \
  cpu-limit=1000 \
  memory-limit=2048 \
  timeout=300s \
  privacy=public \
  protocol=http1 \
  region=fr-par \
  -o json | python3 -c "import sys,json; d=json.load(sys.stdin); print(f\"Container ID: {d['id']}\nDomain: {d.get('domain_name', 'pending')}\")"
```

**Expected Output**:
```
Container ID: <uuid>
Domain: agentops-api-<namespace>.functions.fnc.fr-par.scw.cloud
```

**Acceptance Criteria**:
- [ ] Container created successfully
- [ ] Domain name assigned
- [ ] Container status is "ready"

---

## Task 3: Validate Deployment

**Objective**: Test that the deployed API is working

**Commands**:
```bash
# Get container endpoint
ENDPOINT=$(scw container container list \
  namespace-id=3af1f2d4-4253-464e-beef-eaf688b7342f \
  name=agentops-api \
  -o json | python3 -c "import sys,json; data=json.load(sys.stdin); print(data[0].get('domain_name','') if data else '')")

echo "API Endpoint: https://$ENDPOINT"

# Test health endpoint
curl -s "https://$ENDPOINT/health" | python3 -m json.tool

# Test projects endpoint
curl -s "https://$ENDPOINT/projects" | python3 -m json.tool | head -20

# Test tasks endpoint
curl -s "https://$ENDPOINT/tasks" | python3 -m json.tool | head -20
```

**Expected Output**:
- Health endpoint returns `{"status": "healthy"}`
- Projects endpoint returns list of projects
- Tasks endpoint returns list of tasks

**Acceptance Criteria**:
- [ ] Health check returns 200 OK
- [ ] API returns data from database
- [ ] No 500 errors
- [ ] Response time < 2 seconds

---

## Task 4: Update AgentOps SDK Configuration

**Objective**: Point SDK to deployed API instead of localhost

**File**: `/home/ubuntu/agentops-sdk/agentops/client.py`

**Change**:
```python
# OLD
self.api_url = api_url or os.getenv("AGENTOPS_API_URL", "http://localhost:8000")

# NEW
self.api_url = api_url or os.getenv("AGENTOPS_API_URL", "https://<ENDPOINT>")
```

**Or set environment variable**:
```bash
export AGENTOPS_API_URL="https://<ENDPOINT>"
```

**Acceptance Criteria**:
- [ ] SDK configured with production endpoint
- [ ] Simple agent can connect to deployed API
- [ ] Tasks can be created and claimed

---

## Task 5: Test End-to-End Workflow

**Objective**: Validate complete system with deployed API

**Commands**:
```bash
# Set API URL
export AGENTOPS_API_URL="https://<ENDPOINT>"

# Run simple agent
cd /home/ubuntu/agentops-sdk
python3 examples/simple_agent.py &
AGENT_PID=$!

# Wait for agent to register
sleep 5

# Create a test task
curl -X POST "https://<ENDPOINT>/tasks" \
  -H "Content-Type: application/json" \
  -d '{"title":"Production test task","description":"Validate deployed system","priority":100}'

# Wait for agent to process
sleep 10

# Check task status
curl -s "https://<ENDPOINT>/tasks?status=completed" | python3 -m json.tool

# Stop agent
kill $AGENT_PID
```

**Acceptance Criteria**:
- [ ] Agent registers successfully
- [ ] Task is created
- [ ] Agent claims and processes task
- [ ] Task status changes to "completed"
- [ ] Results are stored

---

## Task 6: Cleanup Build Instance

**Objective**: Remove temporary build instance to save costs

**Commands**:
```bash
# Stop the build instance
scw instance server stop 1582b6b5-aaa3-4257-b597-60fe426c618c zone=fr-par-1

# Wait for it to stop
sleep 15

# Delete the instance
scw instance server delete 1582b6b5-aaa3-4257-b597-60fe426c618c zone=fr-par-1 with-ip=true with-volumes=true
```

**Expected Output**: `✅ Success`

**Acceptance Criteria**:
- [ ] Build instance stopped
- [ ] Build instance deleted
- [ ] IP address released
- [ ] Volumes deleted

**Cost Savings**: ~$14/month

---

## Task 7: Deploy Agent Containers (Future)

**Objective**: Deploy Chief of Staff and Seed agents

**Status**: Code not yet complete (Stage 5 in progress)

**When ready**:
1. Complete agent implementation
2. Add Dockerfiles to agent repositories
3. Build agent images (same process as API)
4. Deploy as Serverless Containers
5. Configure to use deployed API

**Estimated Time**: 2-3 hours of development

---

## Monitoring and Maintenance

### Check Container Status
```bash
scw container container list namespace-id=3af1f2d4-4253-464e-beef-eaf688b7342f -o json
```

### View Container Logs
```bash
scw container container logs <container-id>
```

### Check Costs
```bash
scw billing consumption list
```

### Update Container Image
```bash
# After pushing new image to registry
scw container container update <container-id> \
  registry-image=rg.fr-par.scw.cloud/prod-agentops/agentops-api:latest

scw container container deploy <container-id>
```

---

## Troubleshooting

### Issue: Image not in registry after 10 minutes

**Diagnosis**:
```bash
# Check if build instance is still running
scw instance server get 1582b6b5-aaa3-4257-b597-60fe426c618c zone=fr-par-1
```

**Solution**: Build instance may have failed. Create new instance with build script.

### Issue: Container deployment fails

**Diagnosis**:
```bash
# Check container logs
scw container container logs <container-id>
```

**Common causes**:
- Database connection failure (check credentials in Secret Manager)
- Redis/NATS connection timeout (expected, should gracefully degrade)
- Port mismatch (ensure container exposes port 8000)

### Issue: API returns 500 errors

**Diagnosis**:
```bash
# Check database connection
export DB_HOST="postgres://6838bdb4-a8db-453f-84e5-0b9025abd987.pg.sdb.fr-par.scw.cloud:5432"
export DB_PASSWORD="<from terraform output>"
export IAM_USER_ID="7f9d2892-bd2b-48f9-9f37-67a737607307"

psql "$DB_HOST/rdb?sslmode=require" -U "$IAM_USER_ID" -c "SELECT COUNT(*) FROM projects;"
```

**Solution**: Verify environment variables in container configuration.

---

## Success Criteria

**Deployment is complete when**:
- [x] Infrastructure provisioned (Terraform applied)
- [x] Database schema deployed
- [ ] AgentOps API container deployed and healthy
- [ ] API accessible via public endpoint
- [ ] SDK can connect to deployed API
- [ ] End-to-end task execution works
- [ ] Build instance cleaned up

**System is production-ready when**:
- [ ] All above criteria met
- [ ] Chief of Staff agent deployed
- [ ] Seed agent deployed
- [ ] Voice interface integrated
- [ ] Monitoring configured
- [ ] Cost alerts set up

---

## Cost Summary

### Current Monthly Cost: ~$50-60
- Serverless SQL: ~$15
- Redis: ~$10
- NATS: ~$10
- Storage/Registry/Secrets: ~$15
- **Build instance**: ~$14 (DELETE after build!)

### After API Deployment: ~$60-80
- Infrastructure: ~$50
- AgentOps API container: ~$10-30 (depends on usage)

### After Full Deployment: ~$100-150
- Infrastructure: ~$50
- API + Agents: ~$50-100

---

## Next Session Commands

**Quick status check**:
```bash
# Check if build complete
scw registry image list namespace-id=3af1f2d4-4253-464e-beef-eaf688b7342f region=fr-par | grep agentops

# Check if API deployed
scw container container list namespace-id=3af1f2d4-4253-464e-beef-eaf688b7342f | grep agentops-api

# Get API endpoint
scw container container list namespace-id=3af1f2d4-4253-464e-beef-eaf688b7342f name=agentops-api -o json | python3 -c "import sys,json; data=json.load(sys.stdin); print(f\"https://{data[0].get('domain_name','')}\" if data else 'not deployed')"
```

**Resume deployment**:
```bash
# If build complete but not deployed, run Task 2
# If deployed but not tested, run Task 3
# If tested and working, run Task 6 (cleanup)
```

---

## References

- **Infrastructure Repo**: https://github.com/klogins-hash/serverless-gitops
- **API Repo**: https://github.com/klogins-hash/agentops-api
- **SDK Repo**: https://github.com/klogins-hash/agentops-sdk
- **Planning Repo**: https://github.com/klogins-hash/autonomous-agent-army

**Scaleway Resources**:
- Project ID: `077c9bc7-a333-4912-ae07-cd6d997485a0`
- Region: `fr-par`
- Zone: `fr-par-1`
- Container Registry: `rg.fr-par.scw.cloud/prod-agentops`
- Namespace ID: `3af1f2d4-4253-464e-beef-eaf688b7342f`

---

**This document is agent-executable. Each task can be completed autonomously by running the provided commands and checking acceptance criteria.**
