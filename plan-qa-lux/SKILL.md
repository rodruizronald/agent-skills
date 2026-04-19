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

| Service        | Port  | Protocol | Description                                                                                                                             |
| -------------- | ----- | -------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| Client Gateway | 10001 | gRPC     | Client-facing API for agents, execution plans                                                                                           |
| Agent Gateway  | 10000 | gRPC     | Agent connections, bidirectional streaming                                                                                              |
| Data Collector | 9090  | gRPC     | Metrics, configs, events ingestion from agents                                                                                          |
| Agent          | N/A   | gRPC     | Runs either in docker-compose (fake-miners workflow) or on host via `make dev-agent` (real-miners workflow). Connects to Agent Gateway. |

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

### Agent Runtime Options

Commander has two supported local-dev workflows for running the Agent. The choice depends on **what you need the miners to be**.

#### Workflow A: Docker-compose + fake miners (default for local QA)

The Agent runs in docker-compose under the `fake-miners` profile alongside TWO fake-miner containers — one per firmware (`fake-miners-luxos`, `fake-miners-microbt`). They impersonate LuxOS and MicroBT miners at the wire-protocol level with deterministic fixture responses.

- **Start**: `make fake-miners-up` (= `docker compose --profile fake-miners up -d`)
- **Stop**: `make fake-miners-down`
- **Logs**: `make fake-miners-logs` (or `docker logs agent -f`, `docker logs fake-miners-luxos -f`, `docker logs fake-miners-microbt -f`)
- **Assert SYS-911-shape wiring**: `make fake-miners-verify`
- **Config paths**: `dev-resources/agent/agent.config.toml` (mounted as `/app/agent.config.toml` in the container; edited for docker networking — `AgentGateway.GRPC.Host = "agent-gateway"`, `DataCollector.GRPC.Host = "data-collector"`, short intervals like `10s` for fast feedback). There is NO `agent.config.dev.toml` variant; the single production config file gets edited in place.
- **Agent credentials**: `AgentId = "10000000-0000-0000-0000-000000000001"`, `AgentSecret = "quetal"` (same seeded creds as host workflow)
- **Miners**: each fake-miners container has a static bridge IP (`172.28.100.10` for LuxOS, `172.28.100.11` for MicroBT). LuxOS container serves port 4028 only; MicroBT container serves port 4028 (cgminer-style for OS detection) AND port 4433 (native MicroBT protocol for config collection). Fixtures are hand-authored in `dev-resources/fake-miners/`.
- **IP list / networks — how they get there**: the agent's IP list is driven by the Agent Gateway, which reads `commander.agent.networks` from PostgreSQL and streams it to the agent on login. `./ips.json` at the repo root is a LOCAL CACHE populated by the gateway stream — **do NOT edit it directly**, the gateway overwrites it on every agent login. To configure the fake-miners network for the test agent, the seed file `dev-resources/data-collector/postgres/create_mock_registries.sql` already contains: `networks = '{"networks":[{"name":"fake-miners","ip_list":["172.28.100.10","172.28.100.11"]}]}'` for `test-agent-001`. This persists across `docker compose down -v`.
- **Safety**: `172.28.0.0/16` is the compose bridge subnet — those IPs have no external route, so even with VPN enabled the packets stay inside the bridge. Non-static services on the bridge get auto-assigned IPs starting around `172.28.0.x`; reserve `172.28.100.x` for fake miners to avoid collisions.

Use this workflow when:

- Testing config/stats/events collection paths with predictable results
- Verifying ClickHouse ingestion for miner data
- Testing agent-side logic that reacts to specific miner responses (serial numbers, pool config, temperature thresholds, etc.)
- You want a fast, offline, deterministic dev loop

Reference: `spike/FAKE_MINERS_LOCAL_TESTING.md`, `dev-resources/fake-miners/README.md`.

#### Workflow B: Host air + real miners (VPN)

The Agent runs on the host via air hot-reload. Infrastructure + other services remain in docker-compose. Agent talks to real miners over the corporate VPN.

- **Start**: infrastructure via `docker compose up -d` (no `fake-miners` profile), then `make dev-agent` on host
- **Stop**: `Ctrl+C` the air process, `docker compose down` for infra
- **Logs**: the air process stdout in the terminal
- **Config paths**: `dev-resources/agent/agent.config.toml` (edited in place on host — `Host = "localhost"` or `"0.0.0.0"` so the agent binary reaches the gateway via Docker Desktop's port-forward)
- **IP list / networks**: same rule as Workflow A — the gateway (via PG `commander.agent.networks`) is authoritative. Editing `./ips.json` at the host is futile; the gateway overwrites it on login. To set real miner IPs, use **Client Gateway `UpdateAgent` gRPC** (production-correct; publishes Kafka event the gateway consumes) OR a direct `UPDATE commander.agent SET networks = ...` SQL (transient: lost on `docker compose down -v` and immediately overwritten if the agent is already running).
- **Miners**: whatever real IPs are set in PG `networks` for the test agent, reachable via VPN.

Use this workflow when:

- Testing against real firmware responses (known-bad payloads, new model types)
- Validating miner-side side effects of execution plans (pool changes, power changes, reboots)
- Reproducing a bug reported in staging/prod that requires real hardware

**Rule**: `commander.agent.networks` is the single source of truth for what the agent scans. Never treat `ips.json` as authoritative — it's a runtime cache the gateway overwrites on login. Configure via `create_mock_registries.sql` (seed), `UpdateAgent` gRPC (runtime, production-correct), or a direct `UPDATE` against PG (transient, test-only).

### Key Commands

| Action                                 | Command                    |
| -------------------------------------- | -------------------------- |
| Start infra + server services          | `docker compose up -d`     |
| Stop the stack                         | `docker compose down`      |
| Start fake-miners + agent (Workflow A) | `make fake-miners-up`      |
| Stop fake-miners + agent               | `make fake-miners-down`    |
| Tail fake-miners + agent logs          | `make fake-miners-logs`    |
| Assert SYS-911-shape wiring            | `make fake-miners-verify`  |
| Hot-reload Agent (Workflow B, host)    | `make dev-agent`           |
| Hot-reload Agent Gateway               | `make dev-agentgateway`    |
| Hot-reload Client Gateway              | `make dev-clientgateway`   |
| Hot-reload Data Collector              | `make dev-data-collector`  |
| PostgreSQL migrations                  | `make pg-deploy`           |
| ClickHouse migrations                  | `make ch-deploy`           |
| List Kafka topics                      | `make redpanda-topic-list` |
| ClickHouse CLI                         | `make clickhouse`          |
| Unit tests                             | `make test`                |
| Integration tests                      | `make test-integration`    |
| All checks                             | `make check`               |

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

**miner**: Config snapshot (latest state). Network config, pool configs (9 pools), OS type, reachability, `serial_number` (nullable, populated by LuxOS + MicroBT readers).

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

## Known Gotchas (from real smoke runs)

These are traps that have bitten us in actual smoke testing. Reference them when drafting plans and when someone's test fails in a way that matches the symptom.

1. **Don't use hostnames in `ip_list` — use static IPs.** The Agent's `mineriprepository.ipToUint32` sort comparator calls `net.ParseIP(entry).To4()` and dereferences the result. Hostnames parse to `nil` and panic the Agent once there are ≥2 entries. Always use static bridge IPs (e.g. `172.28.100.x`) for fake-miners networks.

2. **OS detection races 3 workers on port 4028.** `getCGMinerOS`, `getMicroBTOS`, `getVnishOS` run in parallel; first recognized firmware wins. This means ONE IP cannot simultaneously simulate two firmwares. Each firmware needs its own container with its own IP. The `fake-miners-luxos` + `fake-miners-microbt` split in docker-compose is specifically for this reason.

3. **cgminer protocol has two response shapes, single vs multi-command.** Single command (`"version"`): flat response `{"STATUS":[...],"VERSION":[...]}`. Multi command (`"version+config"`): wrapped response `{"version":[{...}],"config":[{...}]}`. The Agent's OS-detection uses single; `GetConfig` uses multi. Any fake miner simulator must handle BOTH shapes.

4. **Agent login requires an injected version.** If the Agent binary is built without `-ldflags "${GOLDFLAGS}"` (which injects `buildinfo.version`), the gateway rejects login with `"missing agent version"`. `.air.agent.toml` invokes `make dev-build-app-agent` for this reason. Plain `go build` in a dev air config will break authentication.

5. **The agent writes its local state back to PG on login.** If the agent's local `ips.json` is empty and the gateway sends down whatever PG has, the agent will overwrite PG with its own (possibly empty) view. Net effect: if you `UPDATE commander.agent SET networks = ...` while the agent is running, the update is gone by the next heartbeat. Always update PG networks with the agent stopped, OR use Client Gateway `UpdateAgent` gRPC (which publishes a Kafka event the gateway consumes and distributes correctly).

6. **`create_mock_registries.sql` only runs on PG volume init.** `docker compose up -d` on an existing volume won't re-seed. To get seed changes picked up: `docker compose down -v` + `up -d`.

7. **Default `docker compose up -d` does NOT start fake-miners or data-producer.** Both are under opt-in profiles (`fake-miners` and `synthetic` respectively). This is intentional so default compose runs stay clean.

8. **`data-consumer` is dead code.** It references `./dev-resources/data-collector/consumer/main.go` which doesn't exist. Gated under `profiles: [synthetic]` so it only runs if you explicitly enable that profile, and it'll crash-loop if you do. Ignore the `Restarting (1)` status unless you intentionally enabled `synthetic`.

9. **`data-producer` pollutes miner-data tables under the same test agent id.** If left running, it injects synthetic config rows for `test-agent-001` (and other seeded agents) in `miner`, `miner_metric`, `miner_event` — your fake-miner validation sees mixed rows. Keep it disabled (default) unless a feature specifically needs synthetic volume.

10. **Bridge IP auto-assignment order matters.** Services without a static `ipv4_address` claim IPs sequentially from the start of the subnet. If 172.28.0.10 is taken by the time fake-miners tries to claim it, you'll get `Address already in use`. Mitigation: put static fake-miner IPs in a reserved range like `172.28.100.x`, not the low end of the subnet.

---

## Workflow

### STEP 0: Assess the Feature

Before producing any testing steps, analyze what the feature involves:

- **Which services does it touch?** (Client Gateway, Agent Gateway, Agent, Data Collector)
- **What gRPC methods does it add or modify?** (check proto files)
- **What database tables does it read from or write to?** (PostgreSQL and/or ClickHouse)
- **Does it publish or consume Kafka events?** (which topics)
- **Does it involve the agent stream?** (bidirectional streaming messages)
- **Does it require a connected agent?**
- **Does it require miners to actually respond?** (if yes, use Workflow A with fake-miners unless specific hardware behavior is needed)
- **What prerequisite data must exist?** (agents registered, miners discovered)
- **What is the happy-path scenario end to end?**

If there isn't enough information to answer these questions, read the relevant code changes first:

1. Check `git diff` or `git log` to understand what changed
2. Read the proto files for any new/modified gRPC methods
3. Read the service implementation to understand the flow
4. Read database migrations for schema changes

If you still cannot determine the full flow, ask the developer before proceeding. Do not guess at table names, gRPC methods, or Kafka topics.

Based on this assessment, determine which components need to be running:

| Component                        | Start if...                                                                                                                                                                     |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Docker infrastructure            | Always (PostgreSQL, ClickHouse, Redpanda)                                                                                                                                       |
| Client Gateway                   | Feature involves client-facing API                                                                                                                                              |
| Agent Gateway                    | Feature involves agent communication or execution plans                                                                                                                         |
| Data Collector                   | Feature involves metrics, configs, or events collection                                                                                                                         |
| Agent + fake-miners (Workflow A) | Feature involves agent-side logic AND wants deterministic miner responses (config/stats/events collection, serial_number wiring, pool changes, etc.) — **default for local QA** |
| Agent on host (Workflow B)       | Feature specifically needs real firmware responses (new model parsing, real miner side effects) — requires VPN + real IPs in `ips.json`                                         |
| Redpanda Connect                 | Feature involves data flowing to ClickHouse via Kafka                                                                                                                           |

**IMPORTANT**: Always use the real Agent binary (`cmd/agent`) — whether via Workflow A or B. Do NOT use the agent mock (`dev-resources/commander-server/agent-mock/`) for feature QA; that mock is a standalone program that fakes the gRPC stream with canned responses and does not exercise the real agent codebase.

Decide between Workflow A and B based on **what the miners need to be**:

- **Workflow A (fake-miners)** — miner responses are deterministic fixtures. Covers config/stats/events collection end-to-end. No VPN, no real hardware. Best for ~90% of local QA.
- **Workflow B (host + real miners)** — requires corporate VPN and `ips.json` with real miner IPs. Use only when the feature specifically depends on real firmware responses.

---

### STEP 1: Start the Stack

Recommend the appropriate startup based on the feature and chosen workflow.

#### 1a. Reset to a clean state (recommended for first-time smoke of a feature)

A smoke test gives the cleanest signal from a known-clean starting point. Stale state from prior dev sessions poisons results: old `miner`/`miner_event`/etc. rows with wrong values, cached `miners.dev.json` inside the agent container, stale Kafka offsets, pre-existing DLQ retry noise, un-applied migrations.

Offer the developer two paths and tell them to pick one:

**Strict fresh-start (default; add ~2 min; recommended for first-time validation):**

```bash
# Tear down everything including volumes (wipes Postgres, ClickHouse, Redpanda, agent/fake-miners state)
docker compose down -v

# Bring infra + server services back up fresh (*-init services re-apply migrations from zero)
docker compose up -d
```

Guarantees: no stale rows in any DB, no leftover Kafka offsets, no cached agent state, migrations re-applied from scratch, Redpanda Connect consumes from a clean position.

**Targeted reset (faster; for subsequent runs when you know the rest of the stack is healthy):**

Include only the targeted commands relevant to the feature under test. Examples:

```bash
# Drop fake-miners + agent containers (also drops in-container caches like miners.dev.json)
make fake-miners-down

# Clear test-agent rows from ClickHouse tables the feature touches
docker exec commander-clickhouse clickhouse-client -q "
ALTER TABLE <table> DELETE WHERE agent_id = '<test-agent-uuid>' SETTINGS mutations_sync = 1;
"

# If the feature writes to PostgreSQL tables, add scoped DELETE here.
```

Tell the developer: "If in doubt, go strict." Then continue with 1b.

#### 1b. Start infrastructure + server services (both workflows)

```bash
# Start infrastructure and server-side services
docker compose up -d

# Verify services are running
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}" | grep commander
```

**If schema changes are involved:**

```bash
make pg-deploy      # PostgreSQL migrations
make ch-deploy      # ClickHouse migrations
```

Health-check the gateways:

```bash
grpcurl -plaintext localhost:10001 list
grpcurl -plaintext localhost:10000 list
```

#### 1c-A. Workflow A: start the Agent + fake-miners in docker-compose

```bash
make fake-miners-up
```

This starts three services (all gated by `profiles: [fake-miners]`): `agent`, `fake-miners-luxos` (172.28.100.10), `fake-miners-microbt` (172.28.100.11). The Agent is configured via `dev-resources/agent/agent.config.toml` (mounted as `/app/agent.config.toml`) with seeded credentials (`10000000-0000-0000-0000-000000000001` / `quetal`). Its IP list comes from `commander.agent.networks` in PostgreSQL, which is seeded by `create_mock_registries.sql` to `{"networks":[{"name":"fake-miners","ip_list":["172.28.100.10","172.28.100.11"]}]}` for `test-agent-001`. On agent login, the gateway streams that list down, the agent writes it to `./ips.json` (cache), the scanner probes both IPs, OS-detection identifies each, and the appropriate reader collects config from each.

Verify the agent started cleanly:

```bash
docker logs agent --tail 30 | grep -iE "ip-repository|connecting to agent gateway|created bidirectional stream"
```

Expect:

- `loaded IPs from file system ... count:1` (the fake-miners service name)
- `connecting to agent gateway address:agent-gateway:10000`
- `created bidirectional stream`

Verify the fake-miners listeners:

```bash
docker logs fake-miners-luxos --tail 10 | grep "listener ready"
docker logs fake-miners-microbt --tail 10 | grep "listener ready"
```

Expect on `fake-miners-luxos`: one line `firmware:luxos port:4028`.
Expect on `fake-miners-microbt`: two lines — `firmware:microbt port:4028` (cgminer detection only) and `firmware:microbt port:4433` (native protocol).

Verify the agent is registered as connected at the gateway:

```bash
grpcurl -plaintext \
  -H "Authorization: Basic $(echo -n '10000000-0000-0000-0000-000000000001:quetal' | base64)" \
  localhost:10000 agentgateway.v1.AgentGatewayService/GetConnectedAgents
```

#### 1c-B. Workflow B: start the Agent on host with real miners

Requires corporate VPN to be up.

Edit `dev-resources/agent/agent.config.toml` to use seeded credentials:

```toml
AgentId = "10000000-0000-0000-0000-000000000001"
AgentSecret = "quetal"
```

Configure the test agent's networks via Client Gateway `UpdateAgent` gRPC with the real miner IPs (production-correct, publishes a Kafka event the gateway consumes). For quick-and-dirty local runs, you can also do a direct PG `UPDATE commander.agent SET networks = '{"networks":[{"name":"prod","ip_list":["10.206.0.31","10.206.0.34"]}]}'::jsonb WHERE id = '...'` — but only while the agent is stopped, and know it will be wiped on `docker compose down -v`. Editing `./ips.json` directly does nothing, because the gateway overwrites it on login. **Never commit real IPs anywhere.**

Start the agent:

```bash
make dev-agent
```

The `.air.agent.toml` injects a dev version (`0.0.0-dev`) via ldflags so the Agent Gateway accepts the login. Collectors will poll the real miners and populate the backend as they do in production.

Verify connection via the same `GetConnectedAgents` call above.

---

**Seeded agent credentials** (all 31 agents use the same token — applies to both workflows):

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

#### 3.2 Check agent is connected

Gateway-side (both workflows):

```bash
grpcurl -plaintext \
  -H "Authorization: Basic $(echo -n '10000000-0000-0000-0000-000000000001:quetal' | base64)" \
  localhost:10000 agentgateway.v1.AgentGatewayService/GetConnectedAgents
```

Agent-side:

- Workflow A: `docker logs agent -f --tail 50` — look for `created bidirectional stream` and no `authentication failed` errors.
- Workflow B: read the air terminal output — same signal, in the terminal where you ran `make dev-agent`.

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

**Miner data in ClickHouse**:

- Workflow A: real miner-shaped data will land in ClickHouse automatically once the agent starts polling the fake-miners service. Wait one scan interval (default in `agent.config.toml` is 300s; the fake-miners dev config typically reduces it to 10-30s) and the `miner` table will have rows for the fake LuxOS and MicroBT miners.
- Workflow B: real miner data lands naturally when collecting from real miners over VPN.
- If you need synthetic data unrelated to miner polling, enable the data producer via its opt-in profile: `docker compose --profile synthetic up -d data-producer`. Default `docker compose up` does not start it, so it cannot contaminate fake-miners tests with synthetic rows.

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

If the feature involves the agent stream (non-gRPC interaction), explain what the agent will do when it receives the command and what to watch for in agent logs (docker-agent logs via `docker logs agent`, host-agent logs in the air terminal).

**Workflow-A-specific note on execution plans**: fake miners do not side-effect on write commands (pool changes, reboots, firmware installs) — they respond with benign acknowledgements. Execution plan STEPS that only exercise the config-write proto contract will still SUCCESS at the detail level, but the fake miner's responses on subsequent read commands will not reflect those changes (fixtures are static). If the feature tests actual write-effect correctness on the miner, use Workflow B instead.

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
WHERE agent_id = '10000000-0000-0000-0000-000000000001'
ORDER BY timestamp DESC LIMIT 5;
"

# Check miner configs (includes serial_number as of SYS-911)
docker exec commander-clickhouse clickhouse-client -q "
SELECT miner_id, os_type, serial_number, pool_1_1_url, ip_address
FROM miner
WHERE agent_id = '10000000-0000-0000-0000-000000000001'
LIMIT 5;
"

# Check events
docker exec commander-clickhouse clickhouse-client -q "
SELECT miner_id, event_type, severity, timestamp
FROM miner_event
WHERE agent_id = '10000000-0000-0000-0000-000000000001'
ORDER BY timestamp DESC LIMIT 10;
"
```

**Workflow A shortcut**: `make fake-miners-verify` asserts the baseline (non-NULL serial_number rows in `miner`). If your feature adds a new miner-level column, extend that target or run a bespoke SELECT.

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
docker logs agent-gateway --tail 50

# Client Gateway logs
docker logs client-gateway --tail 50

# Data Collector logs
docker logs data-collector --tail 50

# Agent logs (Workflow A)
docker logs agent --tail 50

# Fake-miners logs (Workflow A, for wire-level sanity checks)
docker logs fake-miners-luxos --tail 50
docker logs fake-miners-microbt --tail 50

# Redpanda Connect for miner configs (useful if serial/config fields look wrong)
docker logs commander-redpanda-connect-miner --tail 50
```

Specify:

- Log messages that confirm the feature worked
- Warnings that are expected vs. unexpected
- Errors that would indicate a problem

#### 6.6 Agent Behavior (if applicable)

Explain:

- What the agent should do when it receives the command (e.g., config update, execution plan)
- What log output to expect and where to find it:
  - **Workflow A**: `docker logs agent -f`
  - **Workflow B**: the air terminal where `make dev-agent` is running
- What internal state changes to look for (e.g., ConfigurationManager updated, collection interval reset, scanner restarted)

For features that exercise the miner-read path (config/stats/events), also tell the developer what fixture response the fake miner is returning (Workflow A) or what the real miner should report (Workflow B).

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

**Tear down the agent + fake-miners** (Workflow A):

```bash
make fake-miners-down
```

**Or recommend a full reset:**

```bash
docker compose down -v && docker compose up -d
```

If you were on Workflow B, also `Ctrl+C` the air terminal and revert any changes to `dev-resources/agent/agent.config.toml` (the docker `Host` overrides ↔ host `localhost`/`0.0.0.0`) before committing. There is no need to revert `./ips.json` — it's a runtime cache, not committed.

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
- Agent workflow: [A (fake-miners) | B (host + real miners) | not needed]
- Rationale for workflow choice: [why A or B fits this feature]

### Step 1: Start the Stack
[1a. Reset (strict `docker compose down -v` vs. targeted reset — pick one based on first-run vs. repeat)]
[1b. Infrastructure + server services]
[1c-A or 1c-B. Agent startup per chosen workflow]
[explanation of which services are needed, which workflow, and why]

### Step 2: Authenticate with Client Gateway
[CG_AUTH variable setup]

### Step 3: Verify Prerequisites
[grpcurl and SQL commands to confirm system state, with docker-logs checks for Workflow A]

### Step 4: Seed Test Data
[SQL inserts or gRPC calls to create prerequisite data; note fake-miners auto-populates miner tables on Workflow A]

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
[what to look for in which service's logs, including docker logs agent / fake-miners for Workflow A]

#### 6.6 Agent Behavior (if applicable)
[expected agent-side behavior and logs; where to look per workflow]

### Step 7: Cleanup
[DELETE statements, make fake-miners-down if Workflow A, or docker compose down]

### Checklist
Feature: _______________
Branch: _______________
Ticket: _______________
Workflow: [ ] A (fake-miners)  [ ] B (host + real miners)

[ ] Clean state established (strict `docker compose down -v` or targeted reset)
[ ] Infrastructure running (PostgreSQL, ClickHouse, Redpanda)
[ ] Server services running (Client Gateway, Agent Gateway, Data Collector)
[ ] Agent connected (Workflow A: docker logs agent | Workflow B: air terminal)
[ ] Fake-miners listeners up (Workflow A only)
[ ] Client Gateway auth configured ($CG_AUTH)
[ ] Prerequisite data verified
[ ] Feature executed via gRPC
[ ] Rejection cases validated (if applicable)
[ ] gRPC response validated
[ ] PostgreSQL state verified
[ ] ClickHouse state verified (if applicable)
[ ] Kafka events verified (if applicable)
[ ] Logs checked for errors (all services + agent + fake-miners where relevant)
[ ] Agent behavior confirmed (if applicable)
[ ] Test data cleaned up
[ ] fake-miners-down (Workflow A) or air terminal Ctrl+C (Workflow B)
```

---

## Behavioral Guidelines

- **Ask before assuming.** If the gRPC method, tables, or Kafka topics aren't clear from the code, ask. Do not invent method names or table structures.
- **Read the code first.** Before generating the plan, read the relevant proto files, service implementations, and migrations to understand exactly what the feature does.
- **Pick the workflow deliberately.** Default to Workflow A (fake-miners). Choose Workflow B only when the feature genuinely needs real firmware behavior; justify the choice in the "Rationale for workflow choice" field.
- **Be explicit about what success looks like.** Don't just say "verify the response." Say "the response should contain `status: EXECUTION_PLAN_STATUS_SUCCESS` and all step details should show `status: EXECUTION_PLAN_STEP_DETAIL_STATUS_SUCCESS`."
- **Start from a known-clean state.** Always include Step 1a (reset). Default to recommending the strict `docker compose down -v` path for first-time validation of a feature; offer the targeted-reset alternative for iteration. Stale state is a common source of confusing smoke-test failures — the extra 2 minutes of cold start is worth the diagnostic clarity.
- **Respect distributed timing.** Some validations require waiting for async processing (Kafka consumption, agent execution, collector cycles). Indicate expected delays: "wait ~5 seconds for the agent to process the plan", "allow ~10 seconds for Redpanda Connect to flush to ClickHouse", "wait one scan interval (30s on Workflow A) before asserting miner rows exist".
- **Respect the safety invariant.** PG `commander.agent.networks` is the source of truth for the agent's IP list. `./ips.json` is a runtime cache — never rely on editing it. Workflow A: seed fake-miners service names in PG (via `create_mock_registries.sql`). Workflow B: set real IPs in PG via `UpdateAgent` gRPC; never commit real IPs to seed files. Never mix the two.
- **Keep it focused.** This is primarily a smoke test for the happy path. Include the 3-5 most critical rejection cases (missing required fields, invalid values, state preconditions) when the feature has validation logic, but don't try to cover every edge case.
- **One feature at a time.** Each invocation covers one feature. If the developer wants to test multiple features, run the skill separately for each.
- **Use reflection.** When unsure about exact field names or types, use `grpcurl describe` to get the correct proto definitions rather than guessing.
