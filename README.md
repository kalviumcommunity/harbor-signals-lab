# harbor-signals-lab

Node.js checkout service plus Prometheus. Local Docker Compose only. No Grafana. No cloud. No CI.

There are three faults. `docker compose up -d --build` starts the stack. Fix them so the checks below pass.

## Expected behaviour

- `GET /` returns `Harbor checkout is running` with HTTP 200.
- `GET /checkout` returns HTTP 200 `{"result":"paid"}`.
- `GET /checkout?fail=1` returns HTTP 500 `{"result":"failed"}` and increments `checkout_errors_total`.
- Increment `checkout_requests_total` on every checkout.
- `GET /metrics` uses `prom-client`.
- `GET /health` returns 200 `{"status":"ok"}`. The image `HEALTHCHECK` must call `http://localhost:8080/health`.
- A failed checkout logs one JSON line: `timestamp`, `level`, `request_id`, `message`. `level` is `error`.
- Cause one failure with `curl "http://localhost:8080/checkout?fail=1"`.
- `HighErrorRate` uses `rate(checkout_errors_total[1m]) > 0.1`, `for: 1m`, label `severity: warning`.
- Keep failing checkouts coming for 120 seconds, and check `http://localhost:9090/alerts` while traffic is still running.

## Checks

```bash
docker compose up -d --build

curl -i http://localhost:8080/health

docker compose ps

curl -s -o /dev/null -w "%{http_code}\n" "http://localhost:8080/checkout?fail=1"

docker compose logs app | grep '"level":"error"'

end=$((SECONDS+120))
while [ $SECONDS -lt $end ]; do curl -s "http://localhost:8080/checkout?fail=1" >/dev/null; done
```

While sustained failing traffic continues, `http://localhost:9090/alerts` shows `HighErrorRate` firing with `severity="warning"`.

## Prerequisites and boundaries

Git, running Docker Engine with Compose v2, curl and internet access for images/npm dependencies. Reserve host ports 8080 and 9090. Commands run from repository root; Bash is needed for supplied traffic loops/scripts. No host Node.js installation, cloud account, billing or real credentials needed. Local monitoring interfaces are exercise-only; do not expose them on a public network. Repair only documented health, logging and monitoring faults; preserve existing service routes, controlled failures and metric semantics. Do not disable checks or fabricate log/metric output.

Wait for listeners, health probes and initial scrapes before interpreting results. After application/image changes run `docker compose up -d --build`; after mounted Prometheus/rule changes run `docker compose restart prometheus`. Local stack status does not prove notification delivery or production readiness.

Cleanup only exercise resources: `docker compose down`.
