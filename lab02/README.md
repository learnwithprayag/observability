# Lab 02: Custom Monitoring with ServiceMonitor, PodMonitor, PrometheusRule & Blackbox Exporter (No Helm)

---

##  Objectives

- Monitor sample app via:
  - `ServiceMonitor`
  - `PodMonitor`
  - `PrometheusRule`
  - `Probe` using Blackbox Exporter
- Visualize metrics in Prometheus & Grafana
- Trigger alerts on failures

---

##  Prerequisites

- `Lab 01` completed with kube-prometheus-stack running (No Helm)
- `kubectl` configured
- NodePort access enabled (Grafana/Prometheus)

---

##  Step 1: Deploy Sample HTTP Echo App

```bash
kubectl apply -f deploy-echo.yaml
````

---

##  Step 2: Add ServiceMonitor

```bash
kubectl apply -f servicemonitor.yaml
```

---

##  Step 3: Add PodMonitor

```bash
kubectl apply -f podmonitor.yaml
```

---

##  Step 4: Add Prometheus Alert Rule

```bash
kubectl apply -f prometheusrule.yaml
```

---

## Step 5: Deploy Blackbox Exporter

```bash
kubectl apply -f blackbox-exporter.yaml
```

Wait for pod to be running:

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=blackbox-exporter
```

---

##  Step 6: Add Probe for External Target

```bash
kubectl apply -f probe.yaml
```

---

##  Step 7: Validate in Prometheus UI

Open:

```
http://<node-ip>:32090
```

Run queries:

* `up{job=~"http-echo.*"}`
* `probe_success{instance="https://www.google.com"}`

---

##  Cleanup

```bash
kubectl delete ns demo
kubectl delete servicemonitor,podmonitor,probe,prometheusrule -n monitoring --all
kubectl delete -f blackbox-exporter.yaml
```

---

##  References

* [ServiceMonitor Spec](https://prometheus-operator.dev/docs/operator/api/#servicemonitor)
* [Blackbox Exporter](https://github.com/prometheus/blackbox_exporter)

````

---

##  `deploy-echo.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: http-echo
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: http-echo
  template:
    metadata:
      labels:
        app: http-echo
    spec:
      containers:
      - name: http-echo
        image: hashicorp/http-echo
        args: ["-text=Hello Prometheus!"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: http-echo
  namespace: demo
  labels:
    app: http-echo
spec:
  ports:
  - name: http
    port: 80
    targetPort: 5678
  selector:
    app: http-echo
````

---

##  `servicemonitor.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: http-echo-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: http-echo
  namespaceSelector:
    matchNames:
    - demo
  endpoints:
  - port: http
    interval: 15s
```

---

##  `podmonitor.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: podmon-echo
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: http-echo
  namespaceSelector:
    matchNames:
    - demo
  podMetricsEndpoints:
  - port: http
    interval: 15s
```

---

##  `prometheusrule.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: http-echo-alert
  namespace: monitoring
spec:
  groups:
  - name: echo.rules
    rules:
    - alert: EchoAppDown
      expr: up{job=~"http-echo.*"} == 0
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Echo App is Down"
        description: "No scrape data received from echo app"
```

---

##  `probe.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Probe
metadata:
  name: google-http-probe
  namespace: monitoring
spec:
  jobName: google-probe
  prober:
    url: http://blackbox-exporter:9115
  module: http_2xx
  targets:
    staticConfig:
      static:
      - https://www.google.com
```

---

##  `blackbox-exporter.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blackbox-exporter
  namespace: monitoring
  labels:
    app.kubernetes.io/name: blackbox-exporter
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: blackbox-exporter
  template:
    metadata:
      labels:
        app.kubernetes.io/name: blackbox-exporter
    spec:
      containers:
      - name: blackbox-exporter
        image: prom/blackbox-exporter:latest
        ports:
        - containerPort: 9115
---
apiVersion: v1
kind: Service
metadata:
  name: blackbox-exporter
  namespace: monitoring
  labels:
    app.kubernetes.io/name: blackbox-exporter
spec:
  ports:
  - name: http
    port: 9115
    targetPort: 9115
  selector:
    app.kubernetes.io/name: blackbox-exporter
```
---

## 📁 `observability/lab02/` Folder Structure

```
lab02/
├── README.md
├── deploy-echo.yaml
├── servicemonitor.yaml
├── podmonitor.yaml
├── prometheusrule.yaml
├── probe.yaml
└── blackbox-exporter.yaml
```

---
