# Observability stack (Grafana, Loki, Tempo, Promtail)

Self-contained observability stack meant to run alongside projects and later move to its own repo. It gives you centralized logs (Loki), traces (Tempo), and dashboards/alerts (Grafana). Promtail ships container logs to Loki.

## What's inside
- Grafana (port 3000) with pre-provisioned data sources for Loki and Tempo
- Loki (port 3100) for log storage
- Tempo (ports 3200, 4317 gRPC, 4318 HTTP) for traces via OTLP
- Promtail to scrape Docker container logs

## Run it
```bash
cd observability
docker compose up -d
```
Visit http://localhost:3000 (admin / admin). Change the password on first login.

Data volumes: `loki-data`, `tempo-data`, `grafana-storage` (defined in the compose file).

## Shipping logs
- By default, Promtail scrapes Docker container logs from `/var/lib/docker/containers/*/*-json.log` and forwards them to Loki with `job=docker` and `host=$HOSTNAME` labels.
- If you want to ship a specific app log file, mount it into the Promtail container (see commented block in `promtail-config.yml`) and point `__path__` at the mount.
- Prefer JSON logs so fields are queryable in Grafana. Example:

```python
import json, logging, sys, traceback

logger = logging.getLogger("app")
handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(logging.Formatter('%(message)s'))
logger.addHandler(handler)
logger.setLevel(logging.INFO)

try:
    raise RuntimeError("boom")
except Exception as exc:
    logger.error(json.dumps({
        "event": "error",
        "message": str(exc),
        "exc_type": type(exc).__name__,
        "stack": traceback.format_exc(),
    }))
```

## Shipping traces (FastAPI example)
Install instrumentation:
```bash
pip install opentelemetry-sdk opentelemetry-exporter-otlp opentelemetry-instrumentation-fastapi opentelemetry-instrumentation-requests
```
Wire it up in your FastAPI app (replace `service.name` with your service):

```python
from opentelemetry import trace
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

resource = Resource.create({
    "service.name": "backend",
    "deployment.environment": "dev",
})

trace.set_tracer_provider(TracerProvider(resource=resource))
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(
        OTLPSpanExporter(endpoint="http://localhost:4318/v1/traces", timeout=5)
    )
)

app = FastAPI()
FastAPIInstrumentor().instrument_app(app)
```

Once your app sends traces, Grafana's Tempo data source can show them and correlate to logs ("View logs" from a span).

## Multi-project usage
- Keep this stack running once; point each project's OTLP exporter to `http://<host>:4318`.
- For logs, ensure containers use the default json-file driver (Docker) so Promtail can scrape them, or mount app log files into Promtail with labels per service/environment.

## Next steps
- Add alerting in Grafana based on Loki/Tempo queries.
- Tune retention in `loki-config.yml` and `tempo-config.yml`.
- When moving to its own repo, keep the same directory layout so the compose file and configs remain portable.
