# Monitoring (lightweight)

This folder contains the lightweight Promtail + Loki + Grafana monitoring configs used for local LMStudio and Cognee-MCP log collection.

Quick notes
- Promtail config: `monitoring/promtail/config.yml` — scrapes `/var/log/lmstudio/**/*.log` and `/var/log/cognee/*.log`.
- Grafana dashboard: `monitoring/grafana/cognee_lmstudio_dashboard.json` (LMStudio + Cognee panels).

Known issue
- If Loki returns HTTP 429 (ingestion rate limit) Promtail will retry. To mitigate locally we reduced Promtail `batchsize` and increased `batchwait` in `promtail/config.yml`.

How to verify ingestion
1. Append a test log line to an LMStudio file:

```bash
echo "TEST_LOG $(date -u +%Y-%m-%dT%H:%M:%SZ) promtail test" >> ~/.lmstudio/server-logs/2026-03/2026-03-23.1.log
```

2. Restart promtail and check logs/positions:

```bash
cd /path/to/cognee
docker compose -f docker-compose.light.yml restart promtail
docker compose -f docker-compose.light.yml logs --tail 200 promtail
docker compose -f docker-compose.light.yml exec -T promtail cat /tmp/positions.yaml || true
```

3. Query Loki for LMStudio job (last hour):

```bash
start=$(python3 -c "import time; print(int(time.time()*1000)-3600000)")
end=$(python3 -c "import time; print(int(time.time()*1000))")
curl -sS -G 'http://localhost:3101/loki/api/v1/query_range' --data-urlencode "query={job=\"lmstudio\"} |= \"TEST_LOG\"" --data-urlencode "start=$start" --data-urlencode "end=$end" --data-urlencode 'limit=50' | jq .
```

If you still see many 429 errors, either further reduce `batchsize` or increase Loki ingestion limits in its config.
