### Step-by-Step Observability Lab Using OpenTelemetry + Jaeger + Prometheus

---

### Create project directory

```bash
mkdir -p ~/obsevability-labs/otel-jaeger-lab && cd ~/obsevability-labs/otel-jaeger-lab
```

---

### Create Python Virtual Environment

```bash
python3 -m venv venv
source venv/bin/activate
```

---

### Create `requirements.txt`

```bash
cat > requirements.txt <<EOF
flask
opentelemetry-api
opentelemetry-sdk
opentelemetry-instrumentation
opentelemetry-exporter-otlp
opentelemetry-instrumentation-flask
EOF
```

Install dependencies:

```bash
pip install -r requirements.txt
```

---

### Create `app.py`

```bash
cat > app.py <<EOF
from flask import Flask
import time
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter

# Setup tracer provider
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

# Configure OTLP exporter to send to Jaeger
span_processor = BatchSpanProcessor(OTLPSpanExporter(endpoint="http://jaeger:4318/v1/traces"))
trace.get_tracer_provider().add_span_processor(span_processor)

# Flask App
app = Flask(__name__)
FlaskInstrumentor().instrument_app(app)

@app.route('/')
def index():
    return "Hello, World!"

@app.route('/slow')
def slow():
    time.sleep(2)
    return "This was slow!"

@app.route('/error')
def error():
    raise Exception("Simulated error!")

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
```

---

### Create `Dockerfile`

```bash
cat > Dockerfile <<EOF
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
EOF
```

---

### Create `prometheus.yml`

```bash
cat > prometheus.yml <<EOF
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
EOF
```

---

### Create `docker-compose.yml`

```bash
cat > docker-compose.yml <<EOF
version: "3.9"
services:
  app:
    build: .
    ports:
      - "5000:5000"
    depends_on:
      - jaeger

  jaeger:
    image: jaegertracing/all-in-one:1.52
    ports:
      - "16686:16686"  # Jaeger UI
      - "4318:4318"    # OTLP HTTP receiver

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
EOF
```

---

### Start all services

```bash
docker-compose up --build
```

---

### Test the App (in another terminal)

```bash
curl http://localhost:5000/
curl http://localhost:5000/slow
curl http://localhost:5000/error
```

---

### Access Observability Tools

| Tool       | URL                                              |
| ---------- | ------------------------------------------------ |
| Flask App  | [http://localhost:5000](http://localhost:5000)   |
| Jaeger UI  | [http://localhost:16686](http://localhost:16686) |
| Prometheus | [http://localhost:9090](http://localhost:9090)   |


