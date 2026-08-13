---
name: database-debug
description: Debug application issues by safely inspecting MySQL and PostgreSQL data using local CLI clients, project datasource configuration, application code, and logs. Use when persisted database state can provide evidence for debugging, tracing business entities, validating application behavior, or investigating data inconsistencies.
---

# Database Debug

Use MySQL and PostgreSQL databases as evidence sources during application debugging and data tracing.

The goal is not to explore the database broadly. The goal is to answer a specific debugging question using the smallest useful amount of database access.

Typical investigation:

```text
Reported issue
    ↓
Understand relevant code and business flow
    ↓
Discover the project's datasource
    ↓
Inspect relevant schema when needed
    ↓
Query targeted records
    ↓
Correlate database state with code and logs
    ↓
Identify the likely root cause
```

Prefer useful evidence over procedural ceremony.

## Local Database Clients

Use an available local database CLI:

- MySQL: `mysql`
- PostgreSQL: `psql`

Do not run availability or version checks before normal database access.

Attempt the required database operation directly. If the client is unavailable, diagnose that failure when it occurs.

## When to Use

Use this skill when persisted database state can help:

- Debug unexpected application behavior.
- Trace business entities across related tables.
- Verify whether persisted state matches expected application behavior.
- Investigate unexpected state transitions.
- Inspect relevant tables, columns, indexes, constraints, or relationships.
- Validate assumptions while reading or modifying code.
- Correlate database records with application logs.
- Understand data shape before implementing or reviewing code.
- Investigate records using identifiers such as:
  - `user_id`
  - `order_id`
  - `order_no`
  - `payment_id`
  - `transaction_id`
  - `trace_id`
  - `request_id`
  - other business identifiers

Do not access the database merely because it is available.

Every database query should answer a specific debugging question.

# Connection Discovery

## Discover Before Asking

Do not ask the user for connection details until you have first attempted to discover them from the current project.

Inspect configuration that is relevant to the project and active environment.

Common sources include:

```text
application-dev.yml
application-dev.yaml
application-local.yml
application-local.yaml
application.yml
application.yaml
.env
.env.local
docker-compose.yml
docker-compose.yaml
compose.yml
compose.yaml
```

Also inspect environment variables referenced by these files.

Do not blindly scan unrelated directories or dump entire configuration files.

Read only what is necessary to determine the relevant datasource.

## Spring Boot

For Spring Boot projects, inspect datasource configuration such as:

```yaml
spring:
  datasource:
    url:
    username:
    password:
    driver-class-name:
```

Recognize JDBC URLs including:

```text
jdbc:mysql://host:3306/database
jdbc:mysql://host:3306/database?options

jdbc:postgresql://host:5432/database
jdbc:postgresql://host:5432/database?options
```

Extract only what is needed:

```text
database type
host
port
database
username
credential or credential reference
```

Do not treat JDBC query parameters as part of the database name.

Prefer the active development profile when the task concerns a development environment.

## Other Projects

For non-Spring projects, look for clearly relevant datasource configuration in:

- environment variables
- `.env` files
- framework configuration
- Docker Compose
- ORM configuration
- application configuration

Recognize common variables such as:

```text
DATABASE_URL
DB_HOST
DB_PORT
DB_NAME
DB_USER
DB_PASSWORD
MYSQL_HOST
MYSQL_DATABASE
POSTGRES_HOST
POSTGRES_DB
```

Do not assume a variable is active merely because it exists. Use project context to determine which datasource is actually relevant.

## Resolve Placeholders

Configuration may contain placeholders:

```yaml
spring:
  datasource:
    url: jdbc:mysql://${DB_HOST}:${DB_PORT}/${DB_NAME}
    username: ${DB_USER}
    password: ${DB_PASSWORD}
```

Resolve them from relevant sources when available:

1. Current process environment.
2. Project-local development configuration.
3. `.env` or `.env.local`.
4. Docker Compose environment configuration.
5. Other clearly related local configuration.

Never print resolved credentials.

If a required value remains unavailable after reasonable discovery, ask the user only for the missing information.

Do not ask the user to repeat information already available in the project.

## Multiple Datasources

A project may define multiple datasources.

Do not automatically use the first datasource found.

Determine the relevant datasource using evidence from:

- active application profile
- repository or mapper configuration
- entity/model packages
- datasource routing
- service code
- table mappings
- database names
- business domain

If the correct datasource cannot be determined reliably and choosing the wrong database could produce misleading results, ask the user.

# Credential Safety

Treat database credentials as secrets.

Never:

- include passwords in final responses
- echo passwords unnecessarily
- copy passwords into source code
- write passwords into documentation
- commit passwords
- expose passwords in examples
- include credentials in logs when avoidable

Avoid passing passwords as visible command-line arguments when a safer mechanism is available.

Prefer existing client credential configuration or process-scoped credentials.

For PostgreSQL, for example:

```bash
PGPASSWORD="$DB_PASSWORD" psql ...
```

For MySQL, prefer existing MySQL credential configuration, option files, or appropriately scoped environment-based credentials.

Do not create persistent credential files unless explicitly requested.

# Read-Only Policy

Database access through this skill is read-only by default.

Normal allowed operations include:

```text
SELECT
SHOW
DESCRIBE
DESC
EXPLAIN
```

PostgreSQL inspection may also use:

```text
\d
\d+
\dt
\dn
```

Read-only information schema and system catalog queries are allowed.

## Write Operations

Never execute database writes merely as part of debugging.

This includes:

```text
INSERT
UPDATE
DELETE
REPLACE
TRUNCATE
DROP
ALTER
CREATE
RENAME
GRANT
REVOKE
```

Also avoid:

- migrations
- write-capable stored procedures
- functions with side effects
- administrative operations that modify state

unless the user explicitly authorizes the specific operation.

Statements such as:

```text
fix it
make it work
handle it
solve the issue
```

do not by themselves authorize database modification.

If a write appears necessary:

1. Identify the incorrect or missing state.
2. Explain why modification may be appropriate.
3. Show the exact proposed SQL or operation.
4. Wait for explicit authorization.
5. Execute only what was authorized.

Prefer fixing the underlying application problem instead of manually repairing database symptoms.

# Query Strategy

## Start From the Debugging Question

Do not mechanically run generic discovery commands such as:

```text
SHOW DATABASES
SHOW TABLES
SELECT DATABASE()
```

unless they are actually needed.

Start from the issue being investigated.

For example:

```text
Why is order ORD-123 still pending?
```

should lead toward the code and tables responsible for that order, not broad database exploration.

## Inspect Code Before Guessing

Use application code to identify likely:

- entities
- models
- repositories
- mappers
- table names
- column mappings
- relationships
- state transitions
- asynchronous jobs
- event handlers

Do not guess table or column names when they can be determined from code or schema.

## Inspect Schema When Needed

Before querying unfamiliar tables, inspect the relevant schema.

MySQL examples:

```sql
DESCRIBE orders;
SHOW CREATE TABLE orders;
SHOW INDEX FROM orders;
```

PostgreSQL examples:

```text
\d orders
\d+ orders
```

Use structured information schema queries when more useful.

Do not inspect every table when only one or two are relevant.

## Prefer Strong Identifiers

Start from the strongest available identifier.

Examples:

```text
primary key
user_id
order_id
order_no
payment_id
transaction_id
trace_id
request_id
external_id
```

Prefer:

```sql
SELECT id, user_id, status, created_at, updated_at
FROM orders
WHERE order_no = 'ORD-123'
LIMIT 20;
```

over broad searches such as:

```sql
SELECT *
FROM orders
WHERE status = 'PENDING'
LIMIT 500;
```

Follow relationships outward only when necessary.

## Select Only Relevant Columns

Avoid `SELECT *` by default.

Select the smallest useful set of columns.

Prefer:

```sql
SELECT
    id,
    user_id,
    status,
    payment_status,
    created_at,
    updated_at
FROM orders
WHERE order_no = 'ORD-123'
LIMIT 20;
```

instead of:

```sql
SELECT *
FROM orders
WHERE order_no = 'ORD-123';
```

Use `SELECT *` only when inspecting the complete row shape is genuinely useful and the result is known to be small.

## Limit Exploratory Queries

Use `LIMIT` for exploratory row queries unless there is a clear reason not to.

Example:

```sql
SELECT id, status, created_at
FROM orders
ORDER BY created_at DESC
LIMIT 20;
```

Aggregate queries normally do not require `LIMIT`:

```sql
SELECT COUNT(*)
FROM orders
WHERE user_id = 123;
```

## Large Tables

Avoid unbounded scans.

For large tables, prefer filtering by:

- primary key
- indexed business identifier
- timestamp range
- partition key
- known relationship
- relevant indexed fields

Example:

```sql
SELECT
    id,
    order_id,
    status,
    created_at
FROM payment_record
WHERE order_id = 123
  AND created_at >= '2026-08-01'
ORDER BY created_at DESC
LIMIT 50;
```

Use `EXPLAIN` before potentially expensive investigative queries when useful.

Do not run expensive queries merely to gather additional context.

# MySQL

Use `mysql` directly from the terminal.

Useful schema inspection commands include:

```sql
DESCRIBE table_name;

SHOW CREATE TABLE table_name;

SHOW INDEX FROM table_name;
```

Structured column inspection:

```sql
SELECT
    column_name,
    data_type,
    is_nullable,
    column_default
FROM information_schema.columns
WHERE table_schema = DATABASE()
  AND table_name = 'table_name'
ORDER BY ordinal_position;
```

Constraint inspection:

```sql
SELECT
    constraint_name,
    constraint_type
FROM information_schema.table_constraints
WHERE table_schema = DATABASE()
  AND table_name = 'table_name';
```

When multiple databases exist on the same MySQL instance, use the database identified from the current project's datasource configuration.

Fully qualified table names may be used when helpful:

```sql
SELECT
    id,
    status,
    created_at
FROM database_name.orders
WHERE id = 123
LIMIT 20;
```

Do not assume another database on the same instance belongs to the current project.

# PostgreSQL

Use `psql` directly from the terminal.

Useful schema inspection commands include:

```text
\dn
\dt
\d table_name
\d+ table_name
```

Structured column inspection:

```sql
SELECT
    column_name,
    data_type,
    is_nullable,
    column_default
FROM information_schema.columns
WHERE table_schema = 'public'
  AND table_name = 'table_name'
ORDER BY ordinal_position;
```

Constraint inspection:

```sql
SELECT
    constraint_name,
    constraint_type
FROM information_schema.table_constraints
WHERE table_schema = 'public'
  AND table_name = 'table_name';
```

Do not assume the relevant schema is always `public`.

Use application configuration, mappings, search path, or schema inspection to determine the correct schema when necessary.

For exploratory PostgreSQL queries, a read-only transaction may be used when practical:

```sql
BEGIN READ ONLY;

SELECT
    id,
    status,
    created_at
FROM orders
WHERE id = 123
LIMIT 20;

ROLLBACK;
```

Do not add transaction ceremony when a simple read-only query is sufficient.

# Debugging Workflow

Use this workflow as guidance, not as a rigid checklist.

Skip steps when the answer is already known.

## 1. Understand the Failure

Determine:

```text
What was expected?
What actually happened?
Which entity is affected?
What identifier is available?
What time range matters?
```

## 2. Understand the Relevant Code Path

Inspect the smallest relevant portion of the application.

Identify:

```text
entry point
service logic
repository/mapper
expected database changes
state transitions
async/event processing
```

Build an initial hypothesis before issuing broad queries.

## 3. Discover the Datasource

Read the project's relevant configuration.

Determine:

```text
database type
host
port
database/schema
credentials or credential references
```

Do this automatically whenever the information is available.

## 4. Inspect Relevant Schema

Verify unfamiliar tables and columns before querying them.

Use code mappings first when they are clear.

Use database schema inspection when verification is needed.

## 5. Query the Primary Entity

Start with the strongest business identifier.

Example:

```text
order_no = ORD-123
```

Retrieve only fields relevant to the investigation.

## 6. Follow the Evidence

If the primary record suggests another component is involved, follow the relationship.

Example:

```text
orders
   ↓
payment_record
   ↓
payment_callback
   ↓
async_task
```

Do not query related tables without a reason.

Each query should answer a specific debugging question.

## 7. Correlate With Logs and Code

Use timestamps and identifiers to correlate:

```text
database state
+
application code
+
application logs
+
request/trace identifiers
```

For example:

```text
payment_record.status = SUCCESS
             ↓
orders.status = PENDING
             ↓
payment completed at 14:31:18
             ↓
callback log at 14:31:19
             ↓
exception occurred before order update
             ↓
likely failure point identified
```

## 8. Establish the Evidence Chain

Before concluding root cause, distinguish:

### Observed Facts

Directly supported by database records, logs, schema, or code.

### Inferences

Strong conclusions supported by multiple observed facts.

### Hypotheses

Possible explanations that still require verification.

Never present a hypothesis as confirmed fact.

## 9. Stop When the Question Is Answered

Do not continue querying merely because additional data is available.

Stop when enough evidence exists to answer the debugging question or identify the next required piece of evidence.

# Connection Failures

If a connection attempt fails, diagnose the actual error.

Possible causes include:

```text
client unavailable
DNS failure
connection refused
timeout
authentication failure
SSL requirement
database does not exist
insufficient permissions
VPN requirement
SSH tunnel requirement
private network access
firewall restriction
```

Do not perform speculative environment checks before an actual failure occurs.

Report connection errors without exposing credentials.

Do not change network or security configuration unless requested.

# Permission Errors

Read-only accounts are expected and preferred.

Do not treat missing write permissions as a problem during debugging.

If a required read fails:

1. Identify the object or operation that was denied.
2. Determine whether that read is necessary.
3. Explain the minimum additional read permission required.
4. Do not attempt privilege escalation.

# Sensitive Data

Database records may contain sensitive information.

Retrieve only what is relevant.

Avoid selecting unnecessary:

```text
password hashes
access tokens
refresh tokens
API keys
session tokens
secrets
email addresses
phone numbers
addresses
payment details
personal identifiers
```

If sensitive values appear incidentally, do not reproduce them unless essential to the task.

Prefer IDs, statuses, timestamps, and other minimally necessary debugging fields.

# Reporting Findings

Report conclusions rather than dumping database output.

Include when relevant:

- database type
- database/schema name
- tables inspected
- important fields or relationships
- relevant timestamps
- relevant persisted state
- correlation with code or logs
- likely root cause
- remaining uncertainty
- next useful investigation step

Never include database passwords.

Do not include large result sets unless explicitly requested.

A useful report looks like:

```text
The payment succeeded, but the order state transition did not.

Evidence:
- payment_record.status = SUCCESS at 14:31:18
- orders.status remained PENDING
- the callback started at 14:31:19
- application logs show an exception before the order update completed

The database state is consistent with a failure in the post-payment callback
rather than a payment failure.

The next place to inspect is the exception path in PaymentCallbackService.
```

# Efficiency Principles

Minimize unnecessary tool calls.

Do not do this:

```text
check mysql version
→ check connection
→ list databases
→ list every table
→ describe unrelated tables
→ query broad samples
```

when the task already provides enough context for a targeted investigation.

Prefer:

```text
read relevant code/config
→ identify datasource
→ inspect relevant table if needed
→ run targeted query
→ follow evidence
```

Failure of a cheap operation is often more informative than several speculative pre-checks.

# Core Principles

Always follow these principles:

```text
Discover before asking.

Use the project's datasource configuration.

Read code before guessing.

Inspect schema only when needed.

Start from strong identifiers.

Query narrowly.

Select only relevant columns.

Keep database access read-only by default.

Protect credentials.

Minimize sensitive data.

Correlate database state with code and logs.

Distinguish facts from hypotheses.

Stop when enough evidence exists.

Fix root causes instead of manually repairing symptoms.

Prefer useful evidence over procedural ceremony.
```
