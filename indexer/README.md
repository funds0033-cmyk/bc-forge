# Indexer microservice

## API authentication

The indexer read API (`/api/v1/mints`, `/api/v1/transfers`, `/api/v1/burns`,
`/api/v1/stats`) is protected by a shared secret. Set it in the environment
(dotenv loads `.env` automatically) and send it on every request as a bearer
token:

```bash
# .env  — never commit this file
INDEXER_API_TOKEN=replace-with-a-long-random-secret
```

```bash
curl -H "Authorization: Bearer $INDEXER_API_TOKEN" http://localhost:3000/api/v1/stats
```

Requests with a missing or incorrect token receive HTTP `401` with
`{ "error": "Unauthorized" }`. The token value is never logged. `GET /health`
is registered outside the authenticated router and stays public so uptime
probes keep working.

## Health/readiness probe

`GET /health` is a database readiness check. It runs a `SELECT 1` through the
shared Prisma client on every request:

- `200 { "status": "ok" }` — the database answered the ping.
- `503 { "status": "error" }` — the ping failed (wrong `DATABASE_URL`,
  database down, etc.). Hosting probes should treat this as unhealthy.

The route needs no bearer token. The response body is always a fixed literal,
so no driver error text, database URL, or credential can leak to the client;
failures are logged server-side with credentials scrubbed.

**Interpreting probe results in production:**

| Response | Meaning | Action |
|----------|---------|--------|
| `200 { "status": "ok" }` | DB reachable, service healthy | None |
| `503 { "status": "error" }` | DB unreachable or `DATABASE_URL` wrong | Check DB connectivity; inspect server logs (credentials are scrubbed) |
| Connection refused / timeout | Process not running or wrong port | Restart the container; verify `PORT` env var |

## Docker

### Building the Image

Build the production Docker container from the root directory:

```bash
docker build -f indexer/Dockerfile -t bc-forge-indexer indexer
```

See the [Dockerfile](./Dockerfile) for the full build steps. For security
guidance on the image supply chain (e.g. pinning base images), see
[SECURITY.md](../SECURITY.md).

### Running the Container

Run the image, passing the required environment variables:

```bash
docker run -p 3000:3000 \
  -e DATABASE_URL="postgresql://user:password@host:5432/bc_forge?schema=public" \
  -e CONTRACT_ID="CABC...XYZ" \
  -e RPC_URL="https://soroban-testnet.stellar.org" \
  -e INDEXER_API_TOKEN="your-secret-token" \
  -e PORT=3000 \
  bc-forge-indexer
```

> **Never hard-code real credentials in `docker run` commands that appear in
> scripts, shell history, or CI logs.** Pass secrets via your orchestrator's
> secret store (e.g. Docker secrets, Kubernetes secrets, AWS Secrets Manager)
> or a `.env` file that is excluded from version control.

### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `DATABASE_URL` | **Yes** | — | PostgreSQL connection URL used by Prisma. |
| `CONTRACT_ID` | **Yes** | — | Soroban smart contract ID to index. |
| `INDEXER_API_TOKEN` | **Yes** | — | Bearer token for `/api/v1/*` endpoints. |
| `RPC_URL` | No | `https://soroban-testnet.stellar.org` | Soroban RPC endpoint URL. |
| `PORT` | No | `3000` | Port the Express HTTP server listens on. |
| `INDEXER_RATE_LIMIT_WINDOW_MS` | No | `60000` | Rate-limit window in milliseconds. |
| `INDEXER_RATE_LIMIT_MAX` | No | `60` | Maximum requests per IP per window. |

### Database Migrations

Database schema migrations are **not** automatically applied when the container
starts. Apply them separately before starting the container — see
[Migration order](#migration-order) below.

---

## Production operations runbook

### Environment validation

Before starting the service, confirm every required variable is set and
reachable:

```bash
# 1. Check required vars are non-empty
: "${DATABASE_URL:?DATABASE_URL is required}"
: "${CONTRACT_ID:?CONTRACT_ID is required}"
: "${INDEXER_API_TOKEN:?INDEXER_API_TOKEN is required}"

# 2. Verify Postgres is reachable
psql "$DATABASE_URL" -c "SELECT 1;" > /dev/null && echo "DB ok"

# 3. Verify the Soroban RPC endpoint responds
curl -sf "${RPC_URL:-https://soroban-testnet.stellar.org}/health" \
  && echo "RPC ok"

# 4. Verify the contract ID looks well-formed (56-char Stellar contract address)
echo "$CONTRACT_ID" | grep -E '^C[A-Z2-7]{55}$' \
  && echo "CONTRACT_ID format ok"
```

Failure modes and remediation:

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `DATABASE_URL is required` | Variable not exported | Export it or add to `.env` |
| `psql: connection refused` | DB not running / wrong host | Check DB host/port; verify network access |
| `psql: password authentication failed` | Wrong credentials | Rotate and update the secret |
| `curl` to RPC returns non-2xx | RPC node down or wrong URL | Switch to a fallback RPC endpoint |
| `CONTRACT_ID` grep fails | Truncated or placeholder value | Set the correct deployed contract ID |

### Migration order

Migrations must be applied **before** the indexer starts. Running the indexer
against a schema that does not match `prisma/schema.prisma` will cause Prisma
query errors.

```bash
# From the indexer/ directory (requires DATABASE_URL in environment or .env)
npx prisma migrate deploy
```

`prisma migrate deploy` applies every pending migration in
`prisma/migrations/` in order and is safe to run in production — it never
prompts interactively and never resets data. Use `prisma migrate dev` only in
local development.

**Recommended startup sequence:**

1. Provision or verify the PostgreSQL database is running.
2. Run `npx prisma migrate deploy` to bring the schema up to date.
3. Start the indexer container (`npm start` / `docker run`).

### Graceful shutdown

The process handles `SIGTERM` and `SIGINT`:

1. The signal handler calls `disconnectPrismaClient()` to drain in-flight
   queries and close the connection pool cleanly.
2. The process exits with code `0`.

To stop the service cleanly:

```bash
# Docker
docker stop <container-id>          # sends SIGTERM, waits 10 s, then SIGKILL

# Kubernetes
kubectl delete pod <pod-name>       # sends SIGTERM to PID 1 inside the container

# Direct process
kill -TERM <pid>                    # graceful
kill -KILL <pid>                    # last resort — may leave DB connections open
```

> **Avoid `SIGKILL` in normal operations.** It bypasses the shutdown handler
> and can leave idle PostgreSQL connections behind until the server-side
> `idle_in_transaction_session_timeout` cleans them up.

### Database backup and restore

**Backup (pg_dump):**

```bash
# Full logical backup — safe to run while the indexer is live (non-blocking)
pg_dump "$DATABASE_URL" \
  --format=custom \
  --no-password \
  --file="bc_forge_indexer_$(date +%Y%m%d_%H%M%S).dump"
```

**Restore:**

```bash
# Stop the indexer before restoring to avoid write conflicts
docker stop <container-id>

# Drop and recreate the target database, then restore
dropdb --if-exists bc_forge
createdb bc_forge
pg_restore --dbname "$DATABASE_URL" --no-password --exit-on-error bc_forge_indexer_<timestamp>.dump

# Re-run migrations in case the dump pre-dates the latest schema
npx prisma migrate deploy

# Restart the indexer
docker start <container-id>
```

**Retention policy:** keep at minimum one daily backup for 7 days and one
weekly backup for 4 weeks. Test restores regularly in a staging environment.

### Cursor recovery

The indexer tracks its position in the ledger stream via the
`LastIndexedLedger` table (a single row with `id = 1`). On restart the indexer
reads this row and resumes from `ledger + 1`. Duplicate events are silently
ignored thanks to the `txHash UNIQUE` constraint on `Mint`, `Transfer`, and
`Burn`.

**Check current cursor:**

```bash
psql "$DATABASE_URL" -c "SELECT ledger FROM \"LastIndexedLedger\" WHERE id = 1;"
```

**Reset cursor to re-index from a specific ledger** (e.g. after a restore or
to recover missed events):

```bash
# Stop the indexer first
docker stop <container-id>

# Set the cursor to one ledger before the desired start
psql "$DATABASE_URL" \
  -c "UPDATE \"LastIndexedLedger\" SET ledger = <target_ledger - 1> WHERE id = 1;"

# Restart — duplicate events are safe due to UNIQUE constraints
docker start <container-id>
```

**Reset to re-index from genesis:**

```bash
psql "$DATABASE_URL" -c "DELETE FROM \"LastIndexedLedger\" WHERE id = 1;"
```

When no row exists the indexer starts from ledger `0`.

> **Note:** re-indexing from an early ledger can take a long time depending on
> network conditions and the number of events. Monitor logs for progress.

### Log triage

The indexer writes structured logs via the `logger` utility. Credential-like
fields are redacted automatically before any log is emitted.

**Common log patterns and their meaning:**

| Log message | Severity | Meaning | Action |
|-------------|----------|---------|--------|
| `Indexer started` | INFO | Service is up and polling | None |
| `Indexing ledgers: X to Y` | INFO | Normal batch processing | None |
| `Indexer error: ...` | ERROR | RPC call failed; will retry in 5 s | Check RPC connectivity; inspect full error |
| `Error processing event topic <t>` | ERROR | Event parsing failed | Inspect `txHash` in log; check contract ABI |
| `health check failed` | ERROR | DB ping failed | Check DB; restart if DB recovered |
| `Shutdown signal received` | INFO | Graceful stop in progress | None |
| `Fatal indexer error` | FATAL | Unrecoverable startup failure | Check `CONTRACT_ID` and `DATABASE_URL`; fix and restart |

**Quick log commands:**

```bash
# Tail live logs
docker logs -f <container-id>

# Show last 100 lines
docker logs --tail 100 <container-id>

# Filter for errors only
docker logs <container-id> 2>&1 | grep -i '"level":"error"'
```

---

## Secret handling

> **Never commit real credentials.** The `.env` file is listed in
> `.gitignore`. Use `.env.example` as a template — it contains only
> placeholder values.

**Generating a strong `INDEXER_API_TOKEN`:**

```bash
# 32 random bytes, base64-encoded — suitable for a bearer token
openssl rand -base64 32
```

**Generating a strong database password:**

```bash
openssl rand -base64 24
```

Store generated secrets in your team's secret manager (e.g. AWS Secrets
Manager, HashiCorp Vault, Doppler) and inject them at runtime via environment
variables or mounted secret files. Do **not** hard-code them in `Dockerfile`,
`docker-compose.yml`, or any file tracked by git.

When rotating `INDEXER_API_TOKEN`:

1. Generate a new token.
2. Update the secret in your secret store.
3. Redeploy or restart the indexer (it reads the token at startup).
4. Update any clients that send the bearer token.

For vulnerability reporting or security concerns, see [SECURITY.md](../SECURITY.md).
