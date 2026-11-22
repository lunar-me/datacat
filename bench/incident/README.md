# Incident demo: injected faulty query → alert → affected service → trace → faulty query and logs

Repeatable on the Acme Shop running on the DataCat stack (`examples/acme-shop`, started with `run.sh`).

## Run it

```
node bench/incident/webhook-receiver.js &        # records notifications to /var/log/datacat-webhooks.jsonl
node bench/incident/demo.js                      # creates channel + monitor, injects the fault, waits, collects evidence
#   ...walk the UI (below) while the fault is on...
node bench/incident/demo.js --recover            # removes the fault, waits for OK and the recovery notification
bash bench/incident/inject.sh on|off|status      # the fault switch on its own
```

**The fault:** with `/tmp/acme-fault/cart-query` present, cart-service runs `select sku, sum(quantity)::int ...`, but the column is `qty`. That's the kind of bug a bad deploy ships. Every cart lookup fails with Postgres error 42703, and checkout returns 502 at the gateway.

**The monitor:** `spans("env:prod team:acme status:error").by("service").last("2m") > 60`, notifying `@webhook-incident-demo`. Normal error spans peak around 40 per 2 minutes per service (shop-api); the incident produces about 250. The scheduler evaluates every 30 s and nothing forces it.

## Measured run (Nov 12, 2026; evidence in `bench/evidence/incident/`)

| Step | Result |
| --- | --- |
| Fault injected | 04:36:48 UTC |
| Webhook "cart-service is failing" delivered | **+38 s** (HTTP 200) |
| Monitor in Alert | **+40 s**; alerting groups exactly `service:web-gateway` (128) and `service:cart-service` (98) |
| Related telemetry on the monitor page | cart-service: error spans, error logs, failure box with `select sku, sum(quantity)::int as qty from carts ...` and an example trace |
| Example trace | `POST /checkout` (web-gateway, error) → `GET 127.0.0.1` client span → `GET /cart/:user` (cart-service, error) → `postgresql.query` (error, statement `select sku, sum(quantity)...`, code 42703) |
| Logs attached to that trace | `checkout failed: cart-service returned 500` (web-gateway), `cart lookup failed: column "quantity" does not exist` (cart-service) |
| Fault removed | recovery webhooks for both groups at **+108 s**, monitor OK at **+140 s** (2-minute window clearing plus the 30 s schedule) |

The browser walkthrough was recorded by an automated browser tester: monitor (Alert, both groups) → Related telemetry → Example trace → `postgresql.query` span with the faulty statement → trace logs → cart-service APM page with an Errors spike and `GET /cart/:user` showing 482 errors.

Note: newer OpenTelemetry pg instrumentation redacts database error text on spans, keeping only the SQLSTATE code. The full message ("column 'quantity' does not exist") reaches DataCat through the service's error log in the same trace.

## Evidence-based correlation

When a monitor alerts on a service, its status page shows a **What changed** panel. The endpoint behind it is `GET /api/ui/correlate/:service?from=&to=`, implemented in `artifacts/api-server/src/query/correlate.ts`. It compares the incident window with the window of equal length just before it, and returns only facts it can show evidence for:

| Section | What it lists | Evidence shown |
|---|---|---|
| `changes` | Deploy/change events tagged `service:<name>` (or naming it); versions of the service that appeared in its spans | Event source, tags and text; requests per version in each window. A version is "first seen" when it did not run in the 6 h before, and "returned" when it ran earlier but not in the last 10 min. A restart on the version that was already running is not reported. |
| `errors` | Error spans of the service grouped by resource, message and DB statement, **only where the count increased** | Baseline count, incident count, example trace and span |
| `origins` | The deepest failing span in the service's failing traces (an error span with no failing child), across services | Number of failing traces it is the origin of, and the share of all failing traces |
| `logs` | Warn/error log patterns (numbers and ids masked) **only where the count increased** | Baseline count, incident count, example trace |
| `latency` | Hits, errors and p95 per resource in both windows | The numbers |

DataCat does not guess a cause: no ranking by "likelihood" and no text diagnosis. The user sees which change came first and which span fails in every failing trace, and every row links to an example trace.

Example: deploying the faulty cart-service 2.1.0 (`inject.sh deploy-bad`) produced, in `correlation-cart-deploy-bad.json` and `correlation-gateway-deploy-bad.json`:
- version 2.1.0 first seen, 0 → 764 requests
- the "Deployed cart-service 2.1.0" event with its PR note
- `GET /cart/:user` errors 0 → 210
- origin: cart's `select sku, sum(quantity)…` query in 210/210 failing cart traces and 210/270 failing gateway traces
- log pattern `cart lookup failed: column "quantity" does not exist` 0 → 210

Re-run after the restart fixes: `correlation-*-deploy-bad-8080.json` (see below).
