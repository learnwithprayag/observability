
---

# DevOps Observability Lab: Prometheus SDK vs OpenTelemetry (Python)

## Overview

This lab demonstrates two approaches for instrumenting Python Flask apps:

1. **Prometheus Python SDK** – direct metric exposure
2. **OpenTelemetry + Prometheus Exporter** – scalable observability pipeline

---
## Folder Structure

```
prometheus-python-app/
├── .gitignore
├── prometheus/
│   ├── app.py
│   ├── requirements.txt
│   └── Dockerfile
├── opentelemetry/
│   ├── app.py
│   ├── requirements.txt
│   └── Dockerfile
```

---

## `.gitignore`

Create in the root (`prometheus-python-app/`):

```bash
cat >> .gitignore << 'EOF'
# Python
venv/
__pycache__/
*.pyc

# VSCode or other IDE
.vscode/
.idea/

# Environment
.env
EOF
```
---

## Setup (Both Apps)

### Python Virtual Environment

```bash
mkdir -p prometheus-python-app/prometheus
mkdir -p prometheus-python-app/opentelemetry

prometheus-python-app/<prometheus|opentelemetry>
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python app.py
```

> Repeat the above for both `prometheus` and `opentelemetry` folders.

---

## Prometheus Python Client SDK

### 📄 `app.py`

```python
from flask import Flask
from prometheus_client import Counter, Histogram, generate_latest
import time
import random

app = Flask(__name__)

REQUEST_COUNT = Counter("app_requests_total", "Total number of requests")
REQUEST_LATENCY = Histogram("app_request_latency_seconds", "Request latency")

@app.route("/")
def home():
    REQUEST_COUNT.inc()
    with REQUEST_LATENCY.time():
        time.sleep(random.uniform(0.1, 0.5))
    return "Hello, Prometheus!"

@app.route("/metrics")
def metrics():
    return generate_latest(), 200, {"Content-Type": "text/plain"}

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

### `requirements.txt`

```
flask
prometheus_client
```

### `Dockerfile`

```dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
EXPOSE 5000
CMD ["python", "app.py"]
```

### Docker Commands

```bash
cd prometheus-python-app/prometheus
docker build -t prometheus-python-app .
docker run -d -p 5000:5000 --name prometheus-app prometheus-python-app
```

---

## OpenTelemetry + Prometheus Exporter

###  `app.py`

```python
from flask import Flask
import time, random

from opentelemetry import metrics
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.exporter.prometheus import PrometheusMetricReader
from opentelemetry.sdk.resources import SERVICE_NAME, Resource
from opentelemetry.instrumentation.flask import FlaskInstrumentor

from prometheus_client import make_wsgi_app
from werkzeug.middleware.dispatcher import DispatcherMiddleware

reader = PrometheusMetricReader()
provider = MeterProvider(metric_readers=[reader], resource=Resource.create({SERVICE_NAME: "otel-app"}))
metrics.set_meter_provider(provider)

meter = metrics.get_meter(__name__)
request_counter = meter.create_counter(
    "app_requests_total", description="Total number of requests"
)
request_histogram = meter.create_histogram(
    "app_request_latency_seconds", description="Request latency"
)

app = Flask(__name__)
FlaskInstrumentor().instrument_app(app)

@app.route("/")
def hello():
    request_counter.add(1)
    latency = random.uniform(0.1, 0.5)
    time.sleep(latency)
    request_histogram.record(latency)
    return "Hello from OTEL!"

app.wsgi_app = DispatcherMiddleware(app.wsgi_app, {"/metrics": make_wsgi_app()})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

### `requirements.txt`

```
flask
opentelemetry-api
opentelemetry-sdk
opentelemetry-exporter-prometheus
opentelemetry-instrumentation-flask
prometheus_client
werkzeug
```

### `Dockerfile`

```dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
EXPOSE 5000
CMD ["python", "app.py"]
```

### Docker Commands

```bash
cd prometheus-python-app/opentelemetry
docker build -t otel-python-app .
docker run -d -p 5001:5000 --name otel-app otel-python-app
```

---

## Access URLs

| App                    | URL                                                            |
| ---------------------- | -------------------------------------------------------------- |
| Prometheus SDK app     | [http://localhost:5000/](http://localhost:5000/)               |
| Prometheus SDK metrics | [http://localhost:5000/metrics](http://localhost:5000/metrics) |
| OpenTelemetry app      | [http://localhost:5001/](http://localhost:5001/)               |
| OpenTelemetry metrics  | [http://localhost:5001/metrics](http://localhost:5001/metrics) |

---

## SDK vs OpenTelemetry

| Feature                   | Prometheus SDK                     | OpenTelemetry + Prometheus Exporter     |
| ------------------------- | ---------------------------------- | --------------------------------------- |
| Metric API                | `prometheus_client`                | `opentelemetry.sdk.metrics`             |
| Export format             |  Manual `/metrics` route            |  PrometheusMetricReader                  |
| Counter usage             | `Counter.inc()`                    | `counter.add()`                         |
| Histogram usage           | `Histogram.time()` or `.observe()` | `histogram.record()`                    |
| Auto-instrumentation      |  No                                |  Flask, FastAPI, etc.                  |
| Tracing & Logging Support |  Not supported                     |  Full observability suite              |
| Best for                  |  Simple apps, quick metrics         |  Microservices, enterprise observability |


