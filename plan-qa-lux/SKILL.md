---
name: plan-qa-lux
description: Generates feature-specific end-to-end smoke test plans for the Commander MMS platform with concrete gRPC calls, SQL queries, Kafka checks, and validation steps
---

# plan-qa

Guide developers through local end-to-end smoke testing of the Commander MMS platform with the precision of a senior QA engineer. Produce step-by-step validation plans that exercise the full distributed flow: Client Gateway -> Agent Gateway -> Agent -> Miner and back, with real gRPC calls, real database checks, and real event flows.

This is not a replacement for unit or integration tests. The goal is happy-path confidence that the feature works as a whole across the distributed system.

## How to Use This Skill

The user will describe the feature they want to test. This could be:

- A description of what they built
- A ticket or requirement document
- A branch name or PR description
- A plain English explanation

Assess what the feature touches across all services and produce a step-by-step testing guide specific to that feature.

## Core Principles

1. **Feature-specific, not generic** - Every step must be tailored to what the developer is actually testing. No boilerplate filler.
2. **Concrete commands** - Give copy-pasteable gRPC calls, SQL queries, and Kafka inspection commands. The developer should be able to follow without thinking about infrastructure.
3. **Correct ordering** - Steps must respect the distributed dependency chain (infrastructure before services, seed data before API calls, agent connected before execution plans, etc.).
4. **Verify everything** - Every action should have a corresponding validation step. Don't just execute the feature, confirm it worked at each layer.
5. **Clean up after** - Always include cleanup steps so the local databases stay usable.
6. **End-to-end when possible** - Prefer testing the full flow (client -> server -> agent -> miner) over testing individual services in isolation.

---

## Stack Reference

This is the fixed stack. All guidance must reference these specifics.

### Services

| Service        | Port  | Protocol | Description                                                  |
| -------------- | ----- | -------- | ------------------------------------------------------------ |
| Client Gateway | 10001 | gRPC     | Client-facing API for agents, execution plans                |
| Agent Gateway  | 10000 | gRPC     | Agent connections, bidirectional streaming                   |
| Data Collector | 9090  | gRPC     | Metrics, configs, events ingestion from agents               |
| Agent          | N/A   | gRPC     | Runs locally via `make dev-agent`, connects to Agent Gateway |

### Infrastructure

| Component        | Port(s)    | Description                                |
| ---------------- | ---------- | ------------------------------------------ |
| PostgreSQL       | 5432       | Operational data (agents, execution plans) |
| ClickHouse       | 9000, 8123 | Time-series metrics, miner configs, events |
| Redpanda (Kafka) | 19092      | Event streaming between services           |
| Schema Registry  | 18081      | Protobuf schema registry                   |
| Redpanda Console | 8088       | Kafka topic browser UI                     |

### Database Credentials

| Database   | User    | Password | Database  | Connection                                           |
| ---------- | ------- | -------- | --------- | ---------------------------------------------------- |
| PostgreSQL | admin   | admin    | commander | `psql -h localhost -p 5432 -U admin -d commander`    |
| ClickHouse | default | (empty)  | default   | `docker exec commander-clickhouse clickhouse-client` |

**Important:** All PostgreSQL tables are in the `commander` schema, not `public`. Queries must use the schema prefix (e.g., `commander.agent`, `commander.execution_plan`).

### Kafka Topics

| Topic                                           | Purpose                      | Partitions |
| ----------------------------------------------- | ---------------------------- | ---------- |
| `commander.data-collector.miner-metrics`        | Miner metrics from agents    | 3          |
| `commander.data-collector.miner-configs`        | Miner config snapshots       | 1          |
| `commander.data-collector.miner-events`         | Miner events                 | 1          |
| `commander.agent-gateway.agent-update`          | Agent config changes         | 1          |
| `commander.agent-gateway.miner-update`          | Miner config updates         | 1          |
| `commander.agent-gateway.execution-plan-cancel` | Execution plan cancellations | 1          |
| `commander.agent-gateway.agent-restart`         | Agent restart commands       | 1          |
| `commander.agent-gateway.agent-logs-upload`     | Log upload requests          | 1          |

### Real Agent (Local Dev)

The real agent (`cmd/agent`) runs the full agent codebase locally via `make dev-agent`. It connects to Agent Gateway, authenticates, and runs all components (collectors, scanner, execution plan handler, etc.).

**Setup**: Edit `dev-resources/agent/agent.config.toml` to match a seeded agent:

```toml
AgentId = "10000000-0000-0000-0000-000000000001"
AgentSecret = "quetal"
```

The agent will scan IPs from `dev-resources/agent/ips.json`. Without real miners, collectors will run but find nothing — this is expected for testing config propagation, interval changes, and other agent-side logic.

### Key Commands

| Action                    | Command                    |
| ------------------------- | -------------------------- |
| Start full stack          | `docker compose up -d`     |
| Stop full stack           | `docker compose down`      |
| Hot-reload Agent Gateway  | `make dev-agentgateway`    |
| Hot-reload Client Gateway | `make dev-clientgateway`   |
| Hot-reload Data Collector | `make dev-data-collector`  |
| Hot-reload Agent          | `make dev-agent`           |
| PostgreSQL migrations     | `make pg-deploy`           |
| ClickHouse migrations     | `make ch-deploy`           |
| List Kafka topics         | `make redpanda-topic-list` |
| ClickHouse CLI            | `make clickhouse`          |
| Unit tests                | `make test`                |
| Integration tests         | `make test-integration`    |
| All checks                | `make check`               |

---

## System Architecture Knowledge

### Service Communication Flows

Understanding these flows is critical for designing test plans:

#### Flow 1: Execution Plan (Client -> Miner)

```
Client
  -> Client Gateway (gRPC: CreateExecutionPlan)
    -> PostgreSQL (persist plan, steps, details with status=WAITING)
    -> Kafka (commander.clientgateway.execution-plan-created)
      -> Agent Gateway (consumes event)
        -> Agent (via bidirectional gRPC stream: AgentExecutionPlanRequest)
          -> Miner (REST/SSH commands per step type)
        <- Agent (AgentExecutionPlanResponse with per-miner results)
      <- Agent Gateway (updates PostgreSQL: step details, statuses)
    <- Client Gateway (gRPC: GetExecutionPlans to query results)
<- Client
```

#### Flow 2: Metrics Collection (Miner -> ClickHouse)

```
Miner
  <- Agent (minerStatsCollector polls miners periodically)
    -> Data Collector (gRPC: SubmitMinerMetric)
      -> Kafka (commander.data-collector.miner-metrics)
        -> Redpanda Connect (consumes, transforms, writes)
          -> ClickHouse (miner_metric table)
```

#### Flow 3: Config Snapshots (Miner -> ClickHouse)

```
Miner
  <- Agent (minerConfigCollector polls miners periodically)
    -> Data Collector (gRPC: SubmitMinerConfig)
      -> Kafka (commander.data-collector.miner-configs)
        -> Redpanda Connect (consumes, transforms, writes)
          -> ClickHouse (miner table)
```

#### Flow 4: Event Collection (Miner -> ClickHouse)

```
Miner
  <- Agent (minerEventsCollector tracks state changes)
    -> Data Collector (gRPC: SubmitMinerEvent)
      -> Kafka (commander.data-collector.miner-events)
        -> Redpanda Connect (consumes, transforms, writes)
          -> ClickHouse (miner_event table)
```

#### Flow 5: Agent Config Update (Client -> Agent)

```
Client
  -> Client Gateway (gRPC: UpdateAgent)
    -> PostgreSQL (update agent row)
    -> Kafka (commander.agent-gateway.agent-update)
      -> Agent Gateway (consumes event)
        -> Agent (via stream: AgentSetAgentConfigRequest with config JSON, networks JSON, enabled flag)
```

#### Flow 6: Agent Lifecycle

```
Agent starts
  -> Agent Gateway (gRPC stream: AgentLogin with version, OS)
  <- Agent Gateway (AgentLoginResponse with config, optional new version)
  -> Agent Gateway (periodic AgentHeartbeat)
  <- Agent Gateway (AgentGatewayHeartbeat)
  -> Agent Gateway (AgentHeartbeat timeout = 130s -> marks offline)
```

### Execution Plan Step Types

Config steps (executed in order specified):

- `powerConfig` - Power mode (ACTIVE/SLEEP), target watts, profile
- `poolGroupConfig` - Pool groups with failover (3 groups x 3 pools each)
- `networkConfig` - DHCP, IP, netmask, gateway, DNS
- `atmConfig` - Automatic thermal management
- `tempControlConfig` - Temperature thresholds

Action steps (executed after config steps):

- `restartMiner` - Restart miner process
- `restartMiningService` - Restart mining service only
- `installFirmware` - Install custom firmware
- `updateLuxOSFirmware` - Update LuxOS firmware
- `genericCommand` - Custom command execution

### Execution Plan Status Transitions

```
Plan:   WAITING -> WORKING -> SUCCESS / FAILED / CANCELED
Step:   WAITING -> WORKING -> SUCCESS / FAILED / CANCELED
Detail: WAITING -> SUCCESS / FAILED / CANCELED
```

### Failure Codes (per miner, per step)

- `NETWORK_FAILURE` - Connection timeout or network error
- `AUTH_FAILURE` - Authentication failed
- `CONFIG_FAILURE` - Validation or apply failed
- `FIRMWARE_INCOMPATIBLE` - Firmware mismatch
- `DEVICE_BUSY` - Miner occupied
- `EXEC_FAILED` - General execution error

### PostgreSQL Schema (Key Tables)

**agents**: id (UUID), site_id (UUID), version, name, token, config (JSONB), networks (JSONB), auto_update, update_channel, enabled, created_by, updated_by, created_at, updated_at

**agent_history**: Immutable audit log triggered on every agent change. Tracks changed_columns[], created_by, updated_by, deleted_by.

**agent_status**: Agent connection snapshot. last_heartbeat_at.

**execution_plans**: id (UUID), agent_id (FK), description, status (ENUM), expires_at, canceled_by, cancellation_reason, batch_size, batch_delay_sec, external_group_id, created_by, created_at, updated_at.

**execution_plan_steps**: id (UUID), execution_plan_id (FK), step_type, action_type, step_params (JSONB), order (INT), status (ENUM), total_miners, completed_miners, created_at, updated_at.

**execution_plan_step_details**: execution_plan_step_id (FK), miner_id (CHAR(20)), miner_ip, status (ENUM), execution_log, failure_code, created_at, updated_at.

### ClickHouse Schema (Key Tables)

**miner_metric**: Real-time metrics. miner_id, site_id, agent_id, mac_address, timestamp, temperatures (board1/2/3), fans (rpm), hashrates (5s/5m/30m), power, pool_status, health_status, hashing_status. Partitioned by month, TTL 6 months.

**miner_metric_5min_agg**: Pre-aggregated 5-minute rollup with averages.

**miner**: Config snapshot (latest state). Network config, pool configs (9 pools), OS type, reachability.

**miner_event**: Event log with event_type, event_data (JSON), severity (info/warning/error).

### Agent Config Structure (JSONB in PostgreSQL)

```json
{
  "miner_credentials": [
    { "os_type": "luxos", "username": "root", "password": "admin" }
  ],
  "agent_config": {
    "config_collector": {
      "collection_interval_seconds": 300,
      "max_workers": 10
    },
    "stats_collector": { "collection_interval_seconds": 60 },
    "events_collector": { "collection_interval_seconds": 60 },
    "execution_plan_handler": {
      "max_config_workers": 5,
      "max_install_firmware_workers": 2
    },
    "scanner": { "scan_interval_seconds": 60, "max_workers": 256 }
  }
}
```

### Agent Networks Structure (JSONB in PostgreSQL)

```json
{
  "networks": [
    { "cidr": "192.168.1.0/24", "name": "Main" },
    { "start_ip": "10.0.0.1", "end_ip": "10.0.0.254", "name": "Range" },
    { "ip_list": ["172.16.0.10", "172.16.0.11"], "name": "Fixed" }
  ]
}
```

---

## Workflow

### STEP 0: Assess the Feature

Before producing any testing steps, analyze what the feature involves:

- **Which services does it touch?** (Client Gateway, Agent Gateway, Agent, Data Collector)
- **What gRPC methods does it add or modify?** (check proto files)
- **What database tables does it read from or write to?** (PostgreSQL and/or ClickHouse)
- **Does it publish or consume Kafka events?** (which topics)
- **Does it involve the agent stream?** (bidirectional streaming messages)
- **Does it require a connected agent?** (real agent via `make dev-agent`)
- **Does it require miners?** (real miners not available locally, but agent-side logic can still be tested)
- **What prerequisite data must exist?** (agents registered, miners discovered)
- **What is the happy-path scenario end to end?**

If there isn't enough information to answer these questions, read the relevant code changes first:

1. Check `git diff` or `git log` to understand what changed
2. Read the proto files for any new/modified gRPC methods
3. Read the service implementation to understand the flow
4. Read database migrations for schema changes

If you still cannot determine the full flow, ask the developer before proceeding. Do not guess at table names, gRPC methods, or Kafka topics.

Based on this assessment, determine which components need to be running:

| Component                     | Start if...                                                                                                                                |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Docker infrastructure         | Always (PostgreSQL, ClickHouse, Redpanda)                                                                                                  |
| Client Gateway                | Feature involves client-facing API                                                                                                         |
| Agent Gateway                 | Feature involves agent communication or execution plans                                                                                    |
| Data Collector                | Feature involves metrics, configs, or events collection                                                                                    |
| Real Agent (`make dev-agent`) | Feature involves agent-side logic (collectors, config manager, scanner, execution plan handler) or needs a connected agent for e2e testing |
| Redpanda Connect              | Feature involves data flowing to ClickHouse via Kafka                                                                                      |

**IMPORTANT: Always use the real agent** (`cmd/agent` via `make dev-agent`) for QA testing. It runs the full agent codebase including ConfigurationManager, collectors, scanner, and execution plan handler. Requires editing `dev-resources/agent/agent.config.toml` to match a seeded agent (see Step 1). Do NOT use the agent mock (`dev-resources/commander-server/agent-mock/`) as it is a standalone program that fakes the gRPC stream with canned responses and does not run any real agent code.

---

### STEP 1: Start the Stack

Recommend the appropriate startup based on the feature:

**Full stack (most common for e2e tests):**

```bash
# Start all infrastructure and services
docker compose up -d

# Verify services are running
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}" | grep commander
```

**Infrastructure only (when running services locally with hot-reload):**

```bash
# Start infrastructure
docker compose up -d commander-postgres commander-clickhouse commander-redpanda-0

# Wait for initialization
docker compose up -d commander-postgres-init commander-clickhouse-init commander-redpanda-init commander-schema-registry-init

# Start services with hot-reload (separate terminals)
make dev-clientgateway    # Terminal 1
make dev-agentgateway     # Terminal 2
make dev-data-collector   # Terminal 3
```

**If schema changes are involved:**

```bash
# Apply PostgreSQL migrations
make pg-deploy

# Apply ClickHouse migrations
make ch-deploy
```

Always include the gRPC health check:

```bash
grpcurl -plaintext localhost:10001 list
grpcurl -plaintext localhost:10000 list
```

**If the real agent is needed** (testing agent-side logic):

First, update `dev-resources/agent/agent.config.toml` to use a seeded agent's credentials:

```toml
AgentId = "10000000-0000-0000-0000-000000000001"
AgentSecret = "quetal"
```

Then start the agent:

```bash
make dev-agent  # Hot-reload mode
```

The agent will connect to Agent Gateway, authenticate via Basic Auth (bcrypt-verified against PostgreSQL), and start all components (collectors, scanner, heartbeat, etc.). The `.air.agent.toml` injects a dev version (`0.0.0-dev`) via ldflags so the Agent Gateway accepts the login. Collectors will attempt to scan IPs in `dev-resources/agent/ips.json` — they won't find real miners, but that's fine for testing config propagation, interval changes, and other agent-side logic.

Verify the agent connected:

```bash
grpcurl -plaintext \
  -H "Authorization: Basic $(echo -n '10000000-0000-0000-0000-000000000001:quetal' | base64)" \
  localhost:10000 agentgateway.v1.AgentGatewayService/GetConnectedAgents
```

**Seeded agent credentials** (all 31 agents use the same token):

- Plaintext token: `quetal`
- Bcrypt hash in DB: `$2a$12$yEz275yskCgUGT64IDrr8exu3FN0U3pJGMWZ829QHJgJmOHafmQV.`
- Available agent IDs: `10000000-0000-0000-0000-00000000000{1-10}`, `20000000-...{1-10}`, `30000000-...{1-10}`, `63054fa2-0000-0000-0000-000000000001`

---

### STEP 2: Authenticate with Client Gateway

The Client Gateway requires authentication on every gRPC call. Locally, it attempts JWT validation first (which fails with a dummy token), then falls back to Basic Auth via the `x-basic-auth` header.

**Required headers for all Client Gateway calls:**

```bash
-H "Authorization: Bearer eyJhbGciOiJFUzI1NiIsImtpZCI6IjI4OTZjNDdiLTAxOTUtNDNmNi05NzVjLTU2ZmZjNzY4YzgyOSIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6Im5pY2tAbHV4b3IudGVjaCIsImV4cCI6MTc2NTI5MTcxNSwiaWF0IjoxNzY1MjkwODE1LCJpc3MiOiJodHRwOi8vbG9jYWxob3N0OjQ0NTUvIiwianRpIjoiMjE2YTFkMDAtNTNjZi00NTI0LWIxYjItNjc2MDc0MzM4MWUyIiwibmJmIjoxNzY1MjkwODE1LCJzaWQiOiI2ZWI3OGZjNC0wMGZlLTRlZGUtOWRlNC1mZjc2ZWY0ZGQ4MjAiLCJzdWIiOiI1NzUzNzBmZC1lY2I3LTQwYmItYTNkYS04YWY3OGUwZGFiNmIiLCJ0cmFpdHMiOiJtYXBbZW1haWw6bmlja0BsdXhvci50ZWNoIGZpcnN0X25hbWU6TmljayBsYXN0X25hbWU6QmVubmV0dCBsb2NhbGU6ZW5dIn0.APvjj1aKItjQpuOt71gZJd416KndXq-A6CHeqrs58vqrNr8zEzdlVwwphicU-jPVeW3kgKU9ef206C-4f2nhpA" \
-H "x-basic-auth: YWRtaW46c2VjcmV0MTIz"
```

The `x-basic-auth` value is base64 of `admin:secret123` (matching `AUTH_USERNAME`/`AUTH_PASSWORD` in `agent-gateway.env`).

**For convenience, define a shell alias or variable at the start of the session:**

```bash
CG_AUTH='-H "Authorization: Bearer eyJhbGciOiJFUzI1NiIsImtpZCI6IjI4OTZjNDdiLTAxOTUtNDNmNi05NzVjLTU2ZmZjNzY4YzgyOSIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6Im5pY2tAbHV4b3IudGVjaCIsImV4cCI6MTc2NTI5MTcxNSwiaWF0IjoxNzY1MjkwODE1LCJpc3MiOiJodHRwOi8vbG9jYWxob3N0OjQ0NTUvIiwianRpIjoiMjE2YTFkMDAtNTNjZi00NTI0LWIxYjItNjc2MDc0MzM4MWUyIiwibmJmIjoxNzY1MjkwODE1LCJzaWQiOiI2ZWI3OGZjNC0wMGZlLTRlZGUtOWRlNC1mZjc2ZWY0ZGQ4MjAiLCJzdWIiOiI1NzUzNzBmZC1lY2I3LTQwYmItYTNkYS04YWY3OGUwZGFiNmIiLCJ0cmFpdHMiOiJtYXBbZW1haWw6bmlja0BsdXhvci50ZWNoIGZpcnN0X25hbWU6TmljayBsYXN0X25hbWU6QmVubmV0dCBsb2NhbGU6ZW5dIn0.APvjj1aKItjQpuOt71gZJd416KndXq-A6CHeqrs58vqrNr8zEzdlVwwphicU-jPVeW3kgKU9ef206C-4f2nhpA" -H "x-basic-auth: YWRtaW46c2VjcmV0MTIz"'
```

Then use in grpcurl calls:

```bash
eval grpcurl -plaintext $CG_AUTH -d '{"site_ids": []}' \
  localhost:10001 clientgateway.v1.ClientGatewayService/GetAgents
```

**Note:** Agent Gateway (port 10000) uses agent-based authentication (not admin auth). To call dev methods like `GetConnectedAgents`, use any seeded agent's credentials:

```bash
grpcurl -plaintext \
  -H "Authorization: Basic $(echo -n '10000000-0000-0000-0000-000000000001:quetal' | base64)" \
  localhost:10000 agentgateway.v1.AgentGatewayService/GetConnectedAgents
```

---

### STEP 3: Verify Prerequisites

Before testing, verify the system state:

#### 3.1 Check registered agents

```bash
eval grpcurl -plaintext $CG_AUTH -d '{"site_ids": []}' \
  localhost:10001 clientgateway.v1.ClientGatewayService/GetAgents
```

#### 3.2 Check agent is connected (if real agent is running)

```bash
grpcurl -plaintext \
  -H "Authorization: Basic $(echo -n '10000000-0000-0000-0000-000000000001:quetal' | base64)" \
  localhost:10000 agentgateway.v1.AgentGatewayService/GetConnectedAgents
```

#### 3.3 Check PostgreSQL state

```bash
docker exec commander-postgres psql -U admin -d commander -c "SELECT id, name, enabled FROM agent LIMIT 10;"
```

#### 3.4 Check ClickHouse state

```bash
docker exec commander-clickhouse clickhouse-client -q "SELECT count() FROM miner_metric;"
docker exec commander-clickhouse clickhouse-client -q "SELECT count() FROM miner;"
```

If the feature requires a specific agent to exist and it doesn't, either:

- Use an existing seeded agent from `create_mock_registries.sql`
- Create one via `CreateAgent` gRPC call (provide the command with auth headers)

---

### STEP 4: Seed Test Data

This is the most feature-specific step. Based on the assessment in Step 0:

- Identify which tables need data and in which database (PostgreSQL or ClickHouse).
- Respect foreign key constraints: `agent` must exist before `execution_plan`, etc.
- Use UUIDs in a recognizable test range for easy identification and cleanup (e.g., `99999999-0000-0000-0000-00000000XXXX`).
- 31 agents are seeded automatically by `create_mock_registries.sql` on stack startup (all with token `quetal`).

**Check existing seed data first:**

```bash
docker exec commander-postgres psql -U admin -d commander \
  -c "SELECT id, name, site_id, enabled FROM agent ORDER BY name LIMIT 10;"
```

If the feature requires creating an agent:

```bash
eval grpcurl -plaintext $CG_AUTH -d '{
  "site_id": "11111111-1111-1111-1111-111111111111",
  "token": "test-token-123",
  "name": "QA Test Agent",
  "networks": {
    "networks": [
      {"cidr": "192.168.1.0/24", "name": "Test Network"}
    ]
  }
}' localhost:10001 clientgateway.v1.ClientGatewayService/CreateAgent
```

If the feature requires miner data in ClickHouse and the data producer is not running:

```bash
# Start data producer (generates synthetic miner data)
docker compose up -d data-producer
```

If the feature requires no seed data (e.g., it operates on data created by the gRPC call itself), explicitly say so and explain why.

---

### STEP 5: Execute the Feature

Provide the exact gRPC call(s) that exercise the feature using `grpcurl`. All Client Gateway calls must include the auth headers (use the `$CG_AUTH` variable defined in Step 2).

**Format for unary calls:**

```bash
eval grpcurl -plaintext $CG_AUTH -d '{
  "field": "value"
}' localhost:10001 clientgateway.v1.ClientGatewayService/MethodName
```

**Save request for repeatability:**

```bash
cat > /tmp/<feature>-request.json << 'EOF'
{
  "field": "value"
}
EOF

eval grpcurl -plaintext $CG_AUTH -d @ \
  localhost:10001 clientgateway.v1.ClientGatewayService/MethodName < /tmp/<feature>-request.json
```

**Common execution plan example:**

```bash
eval grpcurl -plaintext $CG_AUTH -d '{
  "agent_id": "10000000-0000-0000-0000-000000000001",
  "description": "QA smoke test - pool config change",
  "miner_ids": ["MINER_ID_1", "MINER_ID_2"],
  "config": {
    "pool_group_config": {
      "pool_groups": [{
        "group_id": 1,
        "pools": [{
          "url": "stratum+tcp://pool.test:3333",
          "username": "testuser",
          "password": "x"
        }]
      }]
    }
  }
}' localhost:10001 clientgateway.v1.ClientGatewayService/CreateExecutionPlan
```

If the feature involves multiple calls in sequence (e.g., create then query, or create then cancel), provide each one in order with a brief explanation of what it's doing and what to expect.

If the feature involves the agent stream (non-gRPC interaction), explain what the real agent will do when it receives the command and what to watch for in agent logs.

#### 5b: Validate Rejection Cases

After the happy-path execution, include negative test cases that verify the feature's validation rules. These protect data integrity and ensure the API rejects invalid input with clear errors.

**When to include this section:** If the feature adds or modifies validation (required fields, value constraints, authorization checks, state preconditions), include rejection cases. Skip this section only if the feature has no validation logic.

**What to test:**

- **Missing required fields**: Send requests with required fields omitted. Verify the server returns an error with a message identifying the missing field.
- **Invalid values**: Send fields with out-of-range or invalid values (zero, negative, wrong type). Verify the server rejects with a specific error.
- **State preconditions**: If the feature depends on prior state (e.g., agent must exist, plan must be in WAITING status), send requests that violate those preconditions.

**Format:**

For each rejection case, provide:

1. The gRPC call with the invalid payload
2. The expected error message or error code
3. A brief explanation of why it should be rejected

```bash
# Example: Missing required field
eval grpcurl -plaintext $CG_AUTH -d '{
  "agent_id": "10000000-0000-0000-0000-000000000001",
  "config": {}
}' localhost:10001 clientgateway.v1.ClientGatewayService/UpdateAgent

# Expected: Error - "operational_config is required when config is provided"
```

Keep this focused on the 3-5 most critical rejection cases, not an exhaustive matrix. Prioritize cases that would cause data corruption or silent failures if validation were missing.

---

### STEP 6: Validate Results

Provide specific validation steps for each layer the feature touches:

#### 6.1 gRPC Response

Tell the developer exactly what fields to expect in the response and what values indicate success:

```bash
# Example: Verify execution plan was created
# Expected: response contains execution_plan_id (UUID), status should be "WAITING"
```

#### 6.2 PostgreSQL State

Provide SQL queries that verify the feature persisted data correctly:

```bash
# Check execution plan was stored
docker exec commander-postgres psql -U admin -d commander -c "
SELECT id, agent_id, status, description, created_at
FROM execution_plans
WHERE description LIKE '%QA smoke test%'
ORDER BY created_at DESC LIMIT 5;
"

# Check step details
docker exec commander-postgres psql -U admin -d commander -c "
SELECT eps.step_type, eps.status, eps.total_miners, eps.completed_miners
FROM execution_plan_steps eps
JOIN execution_plans ep ON ep.id = eps.execution_plan_id
WHERE ep.description LIKE '%QA smoke test%';
"

# Check per-miner detail status
docker exec commander-postgres psql -U admin -d commander -c "
SELECT epsd.miner_id, epsd.miner_ip, epsd.status, epsd.failure_code
FROM execution_plan_step_details epsd
JOIN execution_plan_steps eps ON eps.id = epsd.execution_plan_step_id
JOIN execution_plans ep ON ep.id = eps.execution_plan_id
WHERE ep.description LIKE '%QA smoke test%';
"
```

Tell the developer what the expected results should look like (e.g., "all detail rows should transition from WAITING to SUCCESS within ~5 seconds").

#### 6.3 ClickHouse State (if applicable)

```bash
# Check metrics were stored
docker exec commander-clickhouse clickhouse-client -q "
SELECT agent_id, miner_id, timestamp, hashrate_5s, power
FROM miner_metric
WHERE agent_id = '63054fa2-0000-0000-0000-000000000001'
ORDER BY timestamp DESC LIMIT 5;
"

# Check miner configs
docker exec commander-clickhouse clickhouse-client -q "
SELECT miner_id, os_type, pool_1_1_url, ip_address
FROM miner
WHERE agent_id = '63054fa2-0000-0000-0000-000000000001'
LIMIT 5;
"

# Check events
docker exec commander-clickhouse clickhouse-client -q "
SELECT miner_id, event_type, severity, timestamp
FROM miner_event
WHERE agent_id = '63054fa2-0000-0000-0000-000000000001'
ORDER BY timestamp DESC LIMIT 10;
"
```

#### 6.4 Kafka Events (if applicable)

```bash
# Check events were published to a topic
docker exec commander-redpanda-0 rpk topic consume \
  commander.agent-gateway.agent-update \
  --num 1 --offset end

# Or use Redpanda Console UI at http://localhost:8088
# Navigate to Topics -> select topic -> Messages tab
```

Tell the developer what the event payload should contain.

#### 6.5 Application Logs

Tell the developer what to look for in the service logs:

```bash
# Agent Gateway logs
docker logs commander-agent-gateway --tail 50

# Client Gateway logs
docker logs commander-client-gateway --tail 50

# Data Collector logs
docker logs commander-data-collector --tail 50
```

Specify:

- Log messages that confirm the feature worked
- Warnings that are expected vs. unexpected
- Errors that would indicate a problem

#### 6.6 Real Agent Behavior (if applicable)

If the real agent is running (`make dev-agent`), explain:

- What the agent should do when it receives the command (e.g., config update, execution plan)
- What log output to expect from the agent terminal
- What internal state changes to look for (e.g., ConfigurationManager updated, collection interval reset, scanner restarted)

---

### STEP 7: Cleanup

Provide cleanup commands in reverse dependency order:

```bash
# Delete test execution plans and related data
docker exec commander-postgres psql -U admin -d commander -c "
DELETE FROM execution_plan_step_details
WHERE execution_plan_step_id IN (
  SELECT eps.id FROM execution_plan_steps eps
  JOIN execution_plans ep ON ep.id = eps.execution_plan_id
  WHERE ep.description LIKE '%QA smoke test%'
);

DELETE FROM execution_plan_steps
WHERE execution_plan_id IN (
  SELECT id FROM execution_plans WHERE description LIKE '%QA smoke test%'
);

DELETE FROM execution_plans WHERE description LIKE '%QA smoke test%';
"

# Delete test agents (if created)
docker exec commander-postgres psql -U admin -d commander -c "
DELETE FROM agents WHERE name LIKE '%QA Test%';
"
```

Or recommend a full reset:

```bash
docker compose down -v && docker compose up -d
```

---

## gRPC Method Reference

All gRPC reflection is enabled locally. To discover available methods and message types:

```bash
# List all Client Gateway methods
grpcurl -plaintext localhost:10001 list clientgateway.v1.ClientGatewayService

# Describe a specific method (shows request/response types)
grpcurl -plaintext localhost:10001 describe clientgateway.v1.ClientGatewayService/CreateExecutionPlan

# Describe a message type
grpcurl -plaintext localhost:10001 describe clientgateway.v1.CreateExecutionPlanRequest
```

### Client Gateway (port 10001) - Key Methods

**Agent Management:**

- `CreateAgent` - Register new agent
- `UpdateAgent` - Modify agent config, networks, enabled status
- `DeleteAgent` - Soft-delete agent
- `GetAgent` / `GetAgents` - Query agents
- `GetAgentHistory` - Audit trail
- `RestartAgent` - Force restart
- `UploadAgentLogs` - Request log upload

**Execution Plans:**

- `CreateExecutionPlan` - Create plan with config/action steps
- `GetExecutionPlans` - Query with filtering, pagination, depth levels
- `GetExecutionPlanStepsInfo` - Step-level details
- `GetExecutionPlanStepInfo` - Single step with per-miner status
- `GetExecutionPlansGrouped` - Multi-agent grouped view
- `CancelExecutionPlan` - Cancel with reason

**Agent Versions:**

- `CreateAgentVersion` / `UpdateAgentVersion` / `DeleteAgentVersion`
- `GetAgentVersion` / `GetAgentVersions` / `GetAgentVersionsByOS`
- `AssignAgentVersionToChannel` / `GetAgentVersionForChannel`

**Other:**

- `HealthCheck` - Service health
- `CheckExecutionActivity` - Check running executions on site/agent

### Agent Gateway (port 10000) - Dev Methods

- `GetConnectedAgents` - List all connected agents
- `GetAgentInfo` - Info for specific agent
- `DisconnectAgent` - Force disconnect

---

## Output Location

The smoke test plan MUST be saved as a markdown file at `./spike/SYS_[NUMBER]_SMOKE.md`, where `[NUMBER]` is the ticket number extracted from the Linear ticket ID or branch name (e.g., `./spike/SYS_884_SMOKE.md`). If the `./spike/` directory does not exist, create it first. If the ticket number cannot be determined, ask the user for it before saving.

---

## Output Format

Present the guide as a numbered walkthrough with clear headers. Every command must be copy-pasteable. The structure should be:

```
## SmokeCheck: [Feature Name]

### Feature Assessment
- Services involved: [list]
- gRPC methods: [new or modified]
- PostgreSQL tables: [list]
- ClickHouse tables: [list, if any]
- Kafka topics: [list, if any]
- Agent stream messages: [list, if any]
- Real agent needed: [yes/no]

### Step 1: Start the Stack
[commands + explanation of which services are needed and why]

### Step 2: Authenticate with Client Gateway
[CG_AUTH variable setup]

### Step 3: Verify Prerequisites
[grpcurl and SQL commands to confirm system state]

### Step 4: Seed Test Data
[SQL inserts or gRPC calls to create prerequisite data]

### Step 5: Execute
[grpcurl commands with full payloads and auth headers]

### Step 5b: Validate Rejection Cases (if feature has validation)
[gRPC calls with invalid payloads, expected error messages]

### Step 6: Validate
#### 6.1 gRPC Response
[expected response fields and values]

#### 6.2 PostgreSQL State
[SQL queries with expected results]

#### 6.3 ClickHouse State (if applicable)
[SQL queries with expected results]

#### 6.4 Kafka Events (if applicable)
[rpk consume commands with expected payloads]

#### 6.5 Application Logs
[what to look for in which service's logs]

#### 6.6 Real Agent Behavior (if applicable)
[expected agent-side behavior and logs]

### Step 7: Cleanup
[DELETE statements or docker compose down]

### Checklist
Feature: _______________
Branch: _______________
Ticket: _______________

[ ] Infrastructure running (PostgreSQL, ClickHouse, Redpanda)
[ ] Services running (Client Gateway, Agent Gateway, Data Collector)
[ ] Real agent connected (if needed)
[ ] Client Gateway auth configured ($CG_AUTH)
[ ] Prerequisite data verified
[ ] Feature executed via gRPC
[ ] Rejection cases validated (if applicable)
[ ] gRPC response validated
[ ] PostgreSQL state verified
[ ] ClickHouse state verified (if applicable)
[ ] Kafka events verified (if applicable)
[ ] Logs checked for errors
[ ] Agent behavior confirmed (if applicable)
[ ] Test data cleaned up
```

---

## Behavioral Guidelines

- **Ask before assuming.** If the gRPC method, tables, or Kafka topics aren't clear from the code, ask. Do not invent method names or table structures.
- **Read the code first.** Before generating the plan, read the relevant proto files, service implementations, and migrations to understand exactly what the feature does.
- **Be explicit about what success looks like.** Don't just say "verify the response." Say "the response should contain `status: EXECUTION_PLAN_STATUS_SUCCESS` and all step details should show `status: EXECUTION_PLAN_STEP_DETAIL_STATUS_SUCCESS`."
- **Respect distributed timing.** Some validations require waiting for async processing (Kafka consumption, agent execution). Indicate expected delays: "wait ~5 seconds for the agent to process the plan" or "allow ~10 seconds for Redpanda Connect to flush to ClickHouse."
- **Keep it focused.** This is primarily a smoke test for the happy path. Include the 3-5 most critical rejection cases (missing required fields, invalid values, state preconditions) when the feature has validation logic, but don't try to cover every edge case.
- **One feature at a time.** Each invocation covers one feature. If the developer wants to test multiple features, run the skill separately for each.
- **Use reflection.** When unsure about exact field names or types, use `grpcurl describe` to get the correct proto definitions rather than guessing.
