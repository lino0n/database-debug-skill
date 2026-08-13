# Database Debug Skill

A database debugging skill for AI coding agents.

It allows coding agents to use MySQL and PostgreSQL as evidence sources while debugging application issues — discovering datasource configuration from the project, inspecting relevant schema, querying targeted records, and correlating persisted state with code and logs.

## Why

When debugging an application, the answer is often not in the code alone.

For example:

```text
User reports an issue
        ↓
Agent inspects application code
        ↓
Agent discovers the project datasource
        ↓
Agent queries the relevant database records
        ↓
Agent correlates DB state with logs and code
        ↓
Agent identifies the likely root cause
```

Instead of asking you to manually run SQL and paste the results back, the agent can inspect the relevant data directly using local database CLI tools.

## Features

- MySQL support
- PostgreSQL support
- Spring Boot datasource auto-discovery
- `application-dev.yml` / `application.yml` support
- Environment variable resolution
- Docker Compose / `.env` datasource discovery
- Multiple datasource awareness
- Read-only by default
- Targeted queries instead of broad database exploration
- Schema and index inspection
- Large-table query safeguards
- Credential safety
- Sensitive-data minimization
- Code + logs + database evidence correlation
- Evidence-driven root cause investigation

## Requirements

The corresponding database CLI must be available when needed:

```text
MySQL       mysql
PostgreSQL  psql
```

The agent must also have terminal/shell access.

Database credentials must be available through the project environment, development configuration, existing client configuration, or provided by the user when necessary.

A read-only database account is strongly recommended.

## Install

Using the Skills CLI:

```bash
npx skills add https://github.com/lino0n/database-debug-skill \
  --skill database-debug
```

Or:

```bash
npx skills add lino0n/database-debug-skill
```

Then select:

```text
database-debug
```

when prompted.

## Example

You can ask your coding agent:

> Debug why order ORD-123 is still pending even though the payment succeeded.

The skill guides the agent to:

```text
Inspect relevant code
        ↓
Discover datasource configuration
        ↓
Identify the relevant database
        ↓
Inspect relevant schema
        ↓
Query ORD-123
        ↓
Follow related payment/callback records
        ↓
Correlate timestamps with application logs
        ↓
Report the evidence and likely root cause
```

## Spring Boot

The skill understands common Spring Boot datasource configuration:

```yaml
spring:
  datasource:
    url: jdbc:mysql://${DB_HOST}:3306/order_service
    username: ${DB_USER}
    password: ${DB_PASSWORD}
```

It attempts to resolve the datasource from the project before asking the user for connection information.

PostgreSQL JDBC URLs are supported as well:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:5432/order_service
```

## Read-Only by Default

Database access is intended primarily for investigation.

Normal operations include:

```sql
SELECT ...
SHOW ...
DESCRIBE ...
EXPLAIN ...
```

The skill does not treat requests such as:

> fix it

as permission to modify database state.

Before performing a write, the agent must explain the proposed change, show the exact operation, and obtain explicit authorization.

## Query Philosophy

The skill favors targeted investigation:

```sql
SELECT
    id,
    user_id,
    status,
    created_at,
    updated_at
FROM orders
WHERE order_no = 'ORD-123'
LIMIT 20;
```

instead of broad exploration:

```sql
SELECT *
FROM orders;
```

Every database query should answer a debugging question.

## Evidence-Driven Debugging

Database state alone does not necessarily establish root cause.

The skill encourages correlation between:

```text
Database records
      +
Application code
      +
Application logs
      +
Timestamps / trace IDs
      ↓
Evidence chain
      ↓
Root cause
```

It distinguishes between:

- **Observed facts** — directly supported by data, logs, schema, or code.
- **Inferences** — conclusions supported by multiple observations.
- **Hypotheses** — explanations that still require verification.

## Security

The skill is designed to minimize unnecessary access.

It instructs agents to:

- Keep database access read-only by default.
- Avoid exposing credentials.
- Avoid unnecessary sensitive fields.
- Avoid broad queries.
- Prefer strong business identifiers.
- Add limits to exploratory queries.
- Avoid expensive full-table scans.
- Request explicit authorization before database writes.

For development and production debugging, use a dedicated read-only database account whenever possible.

## Supported Databases

| Database | CLI | Status |
| --- | --- | --- |
| MySQL | `mysql` | Supported |
| PostgreSQL | `psql` | Supported |

Additional databases may be considered in the future.

## Skill Structure

```text
database-debug-skill/
├── README.md
├── LICENSE
└── skills/
    └── database-debug/
        └── SKILL.md
```

## Contributing

Issues and pull requests are welcome.

If you use a framework or datasource configuration that the skill does not discover correctly, please open an issue with a sanitized example.

Do not include real database credentials in issues.

## License

MIT
