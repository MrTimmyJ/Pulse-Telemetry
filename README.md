# Pulse

**Telemetry & Incident Investigation Platform**

Pulse ingests application telemetry, queues it durably in PostgreSQL, detects anomalies with deterministic rules, and runs a three-stage investigation pipeline (analysis, investigation, reporting agents) that produces evidence-linked incident reports. Every service emits OpenTelemetry to Azure Monitor / Application Insights.

> All demo data is **synthetic**, produced by `tools/generate_events.py`. Pulse's data model is generic (service, level, message, latency); only the generator uses e-commerce-style service names.

<!-- ![Dashboard](docs/dashboard.png) -->
<!-- ![Application Insights](docs/appinsights.png) -->

## Architecture

```
generator ─▶ ASP.NET Core API ─▶ PostgreSQL (events table = durable queue)
                                      ▲  SELECT ... FOR UPDATE SKIP LOCKED
                        .NET Worker (BackgroundService)
                        process events + deterministic detection ─▶ incidents
React dashboard ─▶ API ─▶ Python agent runtime (FastAPI, read-only audited tools)
All .NET services ── OpenTelemetry ──▶ Azure Monitor / Application Insights
```

| Component | Tech | Responsibility |
|---|---|---|
| `services/api` | ASP.NET Core 10, EF Core | Validated, idempotent ingestion; query and incident endpoints |
| `services/worker` | .NET BackgroundService | Transactional queue consumer and detection |
| `services/Pulse.Core` | C# | Models, DbContext, detection rules, telemetry setup |
| `services/Pulse.Tests` | xUnit | Detection rule tests |
| `services/agents` | Python, FastAPI | Analysis, investigation, and reporting agents |
| `frontend` | React, TypeScript, Vite | Operations dashboard |
| `tools` | Python (stdlib) | Synthetic telemetry and fault injection |

## The agents

Three specialized agents share one incident context and use **read-only, audited tools** (`get_incident`, `query_events`, `service_stats`, `list_services`). The database session is read-only and queries are row-limited.

| Agent | Job | Uses LLM? |
|---|---|---|
| Analysis | Compares the incident window against a baseline (error ratio, latency) | No |
| Investigation | Ranks evidence-linked hypotheses; finds correlated failing services | No |
| Reporting | Builds facts, timeline, next steps, and a summary | Optional (Ollama) |

Every hypothesis cites the event IDs that support it. Agents do not choose their own tools; the pipeline is fixed. The LLM only phrases the summary from collected evidence, and if it is unavailable the report is generated deterministically. The report shows whether the LLM was actually used.

## Quick start (Docker)

```bash
cp .env.example .env        # optionally add Application Insights connection string
docker compose up --build
python tools/generate_events.py --incident error
```

Dashboard: http://localhost:3000. The generator sends 90 s of healthy traffic, then makes `checkout` fail; an incident appears shortly after. Click **Investigate**. Use `--incident latency` for a latency spike. Run `python tools/generate_events.py --help` for options (`--url`, `--rate`, `--duration`, `--baseline`).

The generator needs only Python and network access to the API, so it can run from any machine: `python tools/generate_events.py --url https://your-host/api`.

## Run without Docker

```bash
docker compose up db -d                  # or any local PostgreSQL (user/db/password: pulse)
dotnet run --project services/api
dotnet run --project services/worker
cd services/agents && pip install -r requirements.txt && uvicorn main:app --port 8000
cd frontend && npm install && npm run dev
dotnet test services/Pulse.Tests
```

Check the API port printed by `dotnet run` and update the proxy target in `frontend/vite.config.ts` if it differs from 5080.

## Azure Monitor

1. Create a workspace-based **Application Insights** resource (leave "Enable OTLP support" off; Pulse uses the Azure Monitor exporter).
2. Set `APPLICATIONINSIGHTS_CONNECTION_STRING` in `.env`, then `docker compose up -d`.
3. Explore Transaction search, Performance, Application map, and custom metrics `pulse.events.ingested`, `pulse.events.processed`, `pulse.incidents.created`.
4. Set a daily cap on the Log Analytics workspace.

```kusto
customMetrics
| where name startswith "pulse"
| summarize sum(value) by name, bin(timestamp, 1m)
| render timechart
```

## Deploying to a VPS (e.g. DigitalOcean)

Use at least 2 GB RAM (or add swap) for Docker builds.

1. In `docker-compose.yml`, bind published ports to localhost so Docker does not expose them publicly:
   `"127.0.0.1:5080:8080"` for `api` and `"127.0.0.1:3000:80"` for `frontend`.
2. Install Docker, NGINX, certbot, and htpasswd: `sudo apt install nginx certbot python3-certbot-nginx apache2-utils`
3. Create a login: `sudo htpasswd -c /etc/nginx/.pulse_htpasswd demo`
4. NGINX site (`/etc/nginx/sites-available/pulse`, then symlink into `sites-enabled`):
```nginx
server {
  server_name pulse.example.com;
  location / {
    auth_basic "Pulse";
    auth_basic_user_file /etc/nginx/.pulse_htpasswd;
    proxy_pass http://127.0.0.1:3000;
    proxy_set_header Host $host;
  }
}
```
5. `sudo nginx -t && sudo systemctl reload nginx`, then `sudo certbot --nginx -d pulse.example.com`
6. `git clone ... && cd pulse && cp .env.example .env && docker compose up --build -d`
7. Firewall: allow only 22, 80, 443 (`sudo ufw allow OpenSSH && sudo ufw allow 'Nginx Full' && sudo ufw enable`).

**Data retention:** continuous synthetic traffic grows the database quickly. Add an hourly cron job:
```bash
docker compose exec -T db psql -U pulse -c "DELETE FROM events WHERE created_at < now() - interval '24 hours'"
```

## Security notes

- The API has **no built-in authentication yet**. Do not expose it directly to the internet; put it behind the NGINX basic auth above.
- Agents are read-only and cannot execute commands or modify data.
- Never commit `.env` or the Application Insights connection string.

## Design notes

- **Idempotent ingestion:** unique `event_id`; duplicates are counted, not stored.
- **Durable queue:** claim, process, and commit happen in one transaction (`FOR UPDATE SKIP LOCKED`). A crash rolls back and events remain pending (at-least-once).
- **Deterministic detection:** rules are pure functions with unit tests; AI is never required.
- **Known limits:** detection assumes a single worker (two could open duplicate incidents); schema is created with `EnsureCreated`, not migrations.

## Verification checklist

- [ ] `dotnet test services/Pulse.Tests` passes (or CI is green)
- [ ] `--incident error` and `--incident latency` each open an incident
- [ ] Duplicate `eventId` returns `"duplicates":1`
- [ ] Worker stop/start loses no events
- [ ] Report contains hypotheses with evidence IDs
- [ ] Telemetry visible in Application Insights

## Roadmap

- [ ] Tool-calling agent loop (LLM selects tools with permission checks)
- [ ] Embedding-based retrieval of similar past incidents
- [ ] EF Core migrations
- [ ] API keys, projects, and multi-tenancy
- [ ] Configurable services and thresholds
- [ ] Azure Container Apps + Key Vault + GitHub Actions deploy
- [ ] Benchmarks: throughput, detection latency, recovery time

## License
© 2026 Timothy Johnson. All rights reserved.
