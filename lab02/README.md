---

# Lab 02: Custom Monitoring with ServiceMonitor, PodMonitor, PrometheusRule & Blackbox Exporter (No Helm)

---

##  Objectives

* Monitor sample application using:

  * `ServiceMonitor`
  * `PodMonitor`
  * `PrometheusRule`
  * `Probe` with Blackbox Exporter
* View metrics in Prometheus & Grafana
* Trigger alerts if app or endpoints are unreachable

---

## Prerequisites

*  Lab 01 completed with kube-prometheus-stack deployed (No Helm)
* `kubectl` configured
* NodePort access enabled to Prometheus & Grafana

---

## Deploy Sample HTTP Echo App

```bash
cat <<EOF | tee deploy-echo.yaml
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
EOF

kubectl apply -f deploy-echo.yaml
kubectl -n demo get all

```

---

##  Add ServiceMonitor

```bash
cat <<EOF | tee servicemonitor.yaml
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
EOF

kubectl apply -f servicemonitor.yaml
```

---

##  Add PodMonitor

```bash
cat <<EOF | tee podmonitor.yaml
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
EOF

kubectl apply -f podmonitor.yaml
```

---

##  Add PrometheusRule

```bash
cat <<EOF | tee prometheusrule.yaml
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
EOF

kubectl apply -f prometheusrule.yaml
```

---

##  Deploy Blackbox Exporter

```bash
cat <<EOF | tee blackbox-exporter.yaml
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
EOF

kubectl apply -f blackbox-exporter.yaml
```

Wait for the pod:

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=blackbox-exporter
```

---

## Add Probe for External Target

```bash
cat <<EOF | tee probe.yaml
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
EOF

kubectl apply -f probe.yaml
```

---

##  Validate in Prometheus UI

Visit:

```
http://<node-ip>:32090
```

Try Prometheus queries:

```promql
up{job=~"http-echo.*"}
probe_success{instance="https://www.google.com"}
```

---

## Cleanup

```bash
kubectl delete ns demo
kubectl delete servicemonitor,podmonitor,probe,prometheusrule -n monitoring --all
kubectl delete -f blackbox-exporter.yaml
```

---

## 📂 Folder Structure

```
observability/
└── lab02/
    ├── README.md
    ├── deploy-echo.yaml
    ├── servicemonitor.yaml
    ├── podmonitor.yaml
    ├── prometheusrule.yaml
    ├── probe.yaml
    └── blackbox-exporter.yaml
```

---

## References

* 🔗 [ServiceMonitor API Spec](https://prometheus-operator.dev/docs/operator/api/#servicemonitor)
* 🔗 [Blackbox Exporter GitHub](https://github.com/prometheus/blackbox_exporter)


