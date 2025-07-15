###  Lab 02: Instrumented Python App with `/metrics` + ServiceMonitor (No Helm)

---

###  Create Folder Structure

```bash
mkdir -p observability/lab02
cd observability/lab02
```

---

### Create `app.py`

```bash
cat >> app.py <<EOF
from flask import Flask
from prometheus_client import Counter, generate_latest, CONTENT_TYPE_LATEST

app = Flask(__name__)

# Define a counter
REQUEST_COUNT = Counter('app_requests_total', 'Total HTTP requests')

@app.route("/")
def home():
    REQUEST_COUNT.inc()
    return "Hello from Instrumented Flask App!"

@app.route("/metrics")
def metrics():
    return generate_latest(), 200, {'Content-Type': CONTENT_TYPE_LATEST}

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
EOF
```

---

### Create `Dockerfile`

```bash
cat >> Dockerfile <<EOF
FROM python:3.10-slim

WORKDIR /app

COPY app.py .

RUN pip install flask prometheus_client

EXPOSE 5000

CMD ["python", "app.py"]
EOF
```

---

###  Build & Push Docker Image

```bash
docker build -t learnwithprayag/flask-metrics-app:v1 .
docker push learnwithprayag/flask-metrics-app:v1
```

---

###  Create Kubernetes Deployment & Service

```bash
cat >> k8s-deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-metrics
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-metrics
  template:
    metadata:
      labels:
        app: flask-metrics
    spec:
      containers:
      - name: flask-metrics
        image: learnwithprayag/flask-metrics-app:v1
        ports:
        - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: flask-metrics
  namespace: demo
  labels:
    app: flask-metrics
spec:
  ports:
  - name: http
    port: 5000
    targetPort: 5000
  selector:
    app: flask-metrics
EOF
```

---

###  Create `ServiceMonitor`

```bash
cat >> servicemonitor.yaml <<EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: flask-metrics-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: flask-metrics
  namespaceSelector:
    matchNames:
    - demo
  endpoints:
  - port: http
    path: /metrics
    interval: 15s
EOF
```

---

###  Apply Everything

```bash
kubectl create ns demo  # skip if already created
kubectl apply -f k8s-deployment.yaml
kubectl apply -f servicemonitor.yaml
```

---

###  Validate in Prometheus

Open:

```
http://<your-node-ip>:32090
```

Try queries:

```
up{job="flask-metrics"}
app_requests_total
```

---

###  Test via Internal Curl Pod

```bash
kubectl -n demo run curlpod --rm -it --image=curlimages/curl --restart=Never -- \
  curl flask-metrics.demo.svc.cluster.local:5000/metrics
```

---

###  Cleanup (Optional)

```bash
kubectl delete ns demo
kubectl -n monitoring delete servicemonitor flask-metrics-monitor
```

---

### 📁 Folder Structure

```
observability/
└── lab02/
    ├── app.py
    ├── Dockerfile
    ├── k8s-deployment.yaml
    ├── servicemonitor.yaml
```

