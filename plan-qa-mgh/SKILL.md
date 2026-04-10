---
name: plan-qa-mgh
description: Generates feature-specific local smoke test plans with concrete commands, seed data, and validation steps
---

# plan-qa

Guide developers through local smoke testing with the precision of a senior QA engineer. Produce step-by-step validation plans against the locally running Offer Pricing API stack with real data, real API calls, and real event flows.

This is not a replacement for unit or integration tests. The goal is happy-path confidence that the feature works as a whole.

## How to Use This Skill

The user will describe the feature they want to test. This could be:

- A description of what they built
- A ticket or requirement document
- A branch name or PR description
- A plain English explanation

Assess what the feature touches and produce a step-by-step testing guide specific to that feature.

## Core Principles

1. **Feature-specific, not generic** — Every step must be tailored to what the developer is actually testing. No boilerplate filler.
2. **Concrete commands** — Give copy-pasteable commands, SQL, and curl calls. The developer should be able to follow without thinking about infrastructure.
3. **Correct ordering** — Steps must respect dependency chains (seed data before API calls, start consumers before testing events, etc.).
4. **Verify everything** — Every action should have a corresponding validation step. Don't just run the feature — confirm it worked.
5. **Clean up after** — Always include cleanup steps so the local database stays usable.

---

## Stack Reference

This is the fixed stack. All guidance must reference these specifics.

| Component         | Detail                                                                                                        |
| ----------------- | ------------------------------------------------------------------------------------------------------------- |
| Language          | Go 1.24+                                                                                                      |
| Database          | Postgres on port 5432, user `edoms`, password `edoms`, database `offer_pricing_local`, schema `offer_pricing` |
| Cloud emulation   | LocalStack on port 4566 (SQS)                                                                                 |
| API server        | Port 8080, base path `/v1/api/`                                                                               |
| Consumers         | SQS message processors, port 8082                                                                             |
| Jobs              | Scheduled tasks (outbound dispatcher), port 8081                                                              |
| Auth              | Internal tokens bypass JWT validation when `ENVIRONMENT=local` (default in `variables.env`)                   |
| Container runtime | Colima + Docker Compose (not Docker Desktop)                                                                  |
| Migrations        | Liquibase, run automatically on stack start                                                                   |
| Fresh start       | `make fs` (stops containers, restarts, runs all migrations from scratch)                                      |
| Regular start     | `make run-local` (starts Docker Compose + API server)                                                         |
| Unit tests        | `make unit`                                                                                                   |
| Integration tests | `make integration`                                                                                            |
| Stop everything   | `make down`                                                                                                   |

---

## Workflow

### STEP 0: Assess the Feature

Before producing any testing steps, analyze what the feature involves:

- **What API endpoints does it touch?** (new or modified)
- **What database tables does it read from or write to?**
- **Does it publish or consume SQS events?**
- **Does it depend on scheduled jobs?**
- **What prerequisite data must exist for the feature to work?**
- **What is the happy-path scenario?**

If there isn't enough information to answer these questions, ask the developer before proceeding. Do not guess at table names, endpoints, or event queues.

Based on this assessment, determine which components need to be running:

| Component                                  | Start if...                                                        |
| ------------------------------------------ | ------------------------------------------------------------------ |
| API server (`make run-local` or `make fs`) | Always                                                             |
| Consumers (`cd consumers && make run-all`) | Feature publishes or consumes SQS events                           |
| Jobs (`cd jobs && make run-all`)           | Feature depends on the outbound dispatcher or scheduled processing |

---

### STEP 1: Start the Stack

Recommend either a fresh start or regular start based on the feature:

- **Fresh start (`make fs`)** — Recommend when the feature involves schema changes, new migrations, or when the developer wants a clean database.
- **Regular start (`make run-local`)** — Recommend when the database is already seeded and no migration changes are involved.

Always include the health check:

```bash
curl -s http://localhost:8080/v1/api/health | jq .
```

If consumers or jobs are needed, include their start commands with an explanation of why they're needed for this specific feature.

---

### STEP 2: Authenticate

When running locally with `ENVIRONMENT=local` (the default in `variables.env`), the auth middleware accepts internal tokens that bypass JWT validation entirely. No IDM secret or token generation is required.

| Token               | Roles Granted                                 | Use Case                |
| ------------------- | --------------------------------------------- | ----------------------- |
| `internal`          | Admin (full permissions)                      | Most local testing      |
| `internal-nonadmin` | PricingApprover, PricingEditor, PricingViewer | Testing non-admin flows |

Usage in curl:

```bash
# Admin access
curl -s -X POST http://localhost:8080/v1/api/<endpoint> \
  -H "Authorization: Bearer internal" \
  -H "Content-Type: application/json" \
  -d @request.json | jq .

# Non-admin access
curl -s -X POST http://localhost:8080/v1/api/<endpoint> \
  -H "Authorization: Bearer internal-nonadmin" \
  -H "Content-Type: application/json" \
  -d @request.json | jq .
```

Default to `internal` (admin) unless the feature specifically involves role-based access control, in which case guide the developer to test with both tokens.

---

### STEP 3: Seed Test Data

This is the most feature-specific step. Based on the assessment in Step 0:

- Identify exactly which tables need data.
- Respect the foreign key dependency chain:

```
products → product_offers → pricings → price_config_rulesets → price_config_ruleset_versions
                                         ↓
                                    bundle_offers → bundle_offer_offers
```

- Use IDs in the 9000+ range to avoid collisions with migration seed data.
- Provide ready-to-run SQL `INSERT` statements with realistic test values relevant to the feature.
- Include a note to check existing data first if the feature might work with data already seeded by migrations:

```bash
psql -h localhost -p 5432 -U edoms -d offer_pricing_local
```

```sql
SELECT * FROM offer_pricing.<table> LIMIT 10;
```

If the feature requires no seed data (e.g., it operates on data created by the API call itself), explicitly say so and explain why.

---

### STEP 4: Execute the Feature

Provide the exact API call(s) that exercise the feature:

- Save the request payload to a file for repeatability.
- Use `curl` with the JWT header.
- Capture the response to a file for validation.

Format:

```bash
cat > /tmp/<feature>-request.json << 'EOF'
{
  // feature-specific payload
}
EOF

curl -s -X <METHOD> http://localhost:8080/v1/api/<endpoint> \
  -H "Authorization: Bearer internal" \
  -H "Content-Type: application/json" \
  -d @/tmp/<feature>-request.json | jq . | tee /tmp/<feature>-response.json
```

If the feature involves multiple API calls in sequence (e.g., create then update, or create then retrieve), provide each one in order with a brief explanation of what it's doing and what to expect.

---

### STEP 5: Validate Results

Provide specific validation steps for each layer:

#### 5.1 API Response

Provide `jq` commands that extract and verify the key fields from the response:

```bash
jq '<path-to-key-field>' /tmp/<feature>-response.json
```

Tell the developer exactly what values to expect.

#### 5.2 Database State

Provide SQL queries that verify the feature persisted data correctly. Be specific — query the exact tables and columns the feature modifies:

```sql
SELECT <relevant-columns>
FROM offer_pricing.<table>
WHERE <condition>;
```

Tell the developer what the expected results should look like.

#### 5.3 Application Logs

Tell the developer what to look for in the terminal running `make run-local`:

- Specific log messages the feature should produce.
- Warnings or fallback messages that are expected.
- Errors or stack traces that would indicate a problem.

#### 5.4 Events (if applicable)

If the feature publishes SQS events, provide the command to check the queue:

```bash
aws --endpoint-url=http://localhost:4566 sqs receive-message \
  --queue-url http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/<queue-name> \
  --max-number-of-messages 10 | jq .
```

Tell the developer what the event payload should contain.

---

### STEP 6: Cleanup

Provide SQL `DELETE` statements in reverse dependency order to remove the test data:

```sql
-- Delete in reverse dependency order
DELETE FROM offer_pricing.<child-table> WHERE <condition>;
DELETE FROM offer_pricing.<parent-table> WHERE id >= 9000;
```

Or recommend `make fs` if a full reset is simpler.

---

## Output Location

The smoke test plan MUST be saved as a markdown file at `./spike/SYS_[NUMBER]_SMOKE.md`, where `[NUMBER]` is the ticket number extracted from the Linear ticket ID or branch name (e.g., `./spike/SYS_816_SMOKE.md`). If the `./spike/` directory does not exist, create it first. If the ticket number cannot be determined, ask the user for it before saving.

---

## Output Format

Present the guide as a numbered walkthrough with clear headers. Every command must be copy-pasteable. The structure should be:

```
## SmokeCheck: [Feature Name]

### Feature Assessment
- Endpoints involved: [list]
- Tables involved: [list]
- Events: [yes/no, which queues]
- Jobs/Consumers needed: [yes/no, why]

### Step 1: Start the Stack
[commands + explanation]

### Step 2: Authenticate
[token usage commands]

### Step 3: Seed Test Data
[SQL statements]

### Step 4: Execute
[curl commands]

### Step 5: Validate
[jq commands, SQL queries, log guidance, event checks]

### Step 6: Cleanup
[SQL deletes or make fs]

### Checklist
Feature: _______________
Branch: _______________

[ ] Stack started
[ ] Consumers/jobs started (if needed)
[ ] Prerequisite data seeded
[ ] Authenticated with internal token
[ ] API call executed
[ ] Response verified
[ ] Database state verified
[ ] Logs checked
[ ] Events verified (if applicable)
[ ] Test data cleaned up
```

---

## Behavioral Guidelines

- **Ask before assuming.** If the endpoint, tables, or events aren't known — ask. Do not invent table names or API paths.
- **Be explicit about what success looks like.** Don't just say "verify the response." Say "the response should contain a `requestStatus` of `success` and the `allocatedNetPrice` should be `50.00`."
- **Respect the dependency chain.** Never provide seed SQL that violates foreign key constraints.
- **Keep it focused.** This is a smoke test for the happy path. Don't try to cover every edge case — that's what unit and integration tests are for.
- **One feature at a time.** Each invocation of this skill covers one feature. If the developer wants to test multiple features, run the skill separately for each.
