# Lab 01: Prometheus + Grafana + Alertmanager on Kubernetes without Helm

---

###  Objective

* Deploy Prometheus, Alertmanager, Grafana using **kube-prometheus-stack CRDs**
* Access dashboards and alerting UIs
* Configure alert rules
* Simulate alerts
* Integrate notifications (SMTP, Slack, OpsGenie)
* No Helm used — fully GitOps-friendly

---

### Prerequisites

* Kubernetes cluster (e.g., MicroK8s with MetalLB, GKE, etc.)
* `kubectl` configured
* Slack Webhook, Gmail App Password, OpsGenie API Key

---

## Setup MicroK8s (if needed)

```bash
sudo snap install microk8s --classic
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
newgrp microk8s
microk8s status --wait-ready

microk8s enable dns storage ingress helm3
microk8s enable metallb:10.128.0.51-10.128.0.55

alias kubectl='microk8s kubectl'
alias helm='microk8s helm3'
echo "alias kubectl='microk8s kubectl'" >> ~/.bashrc
source ~/.bashrc

microk8s config > ~/.kube/config
kubectl get nodes
```

---

## Install kube-prometheus-stack CRDs (No Helm)

```bash
git clone https://github.com/prometheus-operator/kube-prometheus
cd kube-prometheus

kubectl create -f manifests/setup
kubectl wait --for=condition=Established --all CustomResourceDefinition --timeout=60s
kubectl create -f manifests/
```

---

##  Access Dashboards (NodePort)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: grafana-nodeport
  namespace: monitoring
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: grafana
  ports:
    - port: 80
      targetPort: 3000
      nodePort: 32080
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus-nodeport
  namespace: monitoring
spec:
  type: NodePort
  selector:
    prometheus: k8s
  ports:
    - port: 9090
      targetPort: 9090
      nodePort: 32090
---
apiVersion: v1
kind: Service
metadata:
  name: alertmanager-nodeport
  namespace: monitoring
spec:
  type: NodePort
  selector:
    alertmanager: main
  ports:
    - port: 9093
      targetPort: 9093
      nodePort: 32093
EOF
```

* Grafana: `http://<node-ip>:32080`
* Prometheus: `http://<node-ip>:32090`
* Alertmanager: `http://<node-ip>:32093`

---

##  Grafana Login

```bash
kubectl get secret grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d && echo
```

Username: `admin`
Password: `<decoded value>`

---

## Import Dashboards

Use these IDs in **Grafana → + → Import**:

| Dashboard Name     | ID     |
| ------------------ | ------ |
| Node Exporter Full | `1860` |
| K8s Cluster        | `6417` |

---

## Create Alert Rules

```bash
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: demo-alerts
  namespace: monitoring
spec:
  groups:
  - name: demo.rules
    rules:
    - alert: HighCPUUsage
      expr: rate(container_cpu_usage_seconds_total{container!="",namespace="default"}[1m]) > 0.5
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "High CPU usage on {{ \$labels.pod }}"
        description: "Pod {{ \$labels.pod }} is using high CPU."
EOF
```

---

## Simulate Alerts

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: cpu-hog
  namespace: monitoring
spec:
  containers:
  - name: hog
    image: busybox
    args: ["sh", "-c", "while true; do :; done"]
---
apiVersion: v1
kind: Pod
metadata:
  name: crashloop
  namespace: monitoring
spec:
  containers:
  - name: fail
    image: busybox
    command: ["/bin/false"]
---
apiVersion: v1
kind: Pod
metadata:
  name: mem-hog
  namespace: monitoring
spec:
  containers:
  - name: mem
    image: polinux/stress
    command: ["/usr/bin/stress"]
    args: ["--vm", "1", "--vm-bytes", "512M"]
EOF
```

---

##  Configure Alertmanager Notifications

###  Gmail App Password Setup

1. Enable 2FA → [https://myaccount.google.com/security](https://myaccount.google.com/security)
2. Create App Password → [https://myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)

Use `Mail → Custom Name (e.g. Prometheus)`
Copy 16-character password.

---

###  Slack Webhook Setup

1. Create at → [https://api.slack.com/apps](https://api.slack.com/apps)
2. Enable **Incoming Webhooks** → Add to channel (e.g., `#alerts`)
3. Copy webhook URL

---

###  Configure Full Alertmanager

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-main
  namespace: monitoring
  labels:
    alertmanager: main
type: Opaque
stringData:
  alertmanager.yaml: |
    global:
      smtp_smarthost: smtp.gmail.com:587
      smtp_from: your_email@gmail.com
      smtp_auth_username: your_email@gmail.com
      smtp_auth_password: your_gmail_app_password
      smtp_require_tls: true
    route:
      group_by: ['alertname']
      receiver: email-notifier
      group_wait: 10s
      group_interval: 1m
      repeat_interval: 1h
      routes:
      - receiver: slack-notifier
        matchers:
        - severity="critical"
      - receiver: opsgenie-notifier
        matchers:
        - team="ops"
    receivers:
    - name: email-notifier
      email_configs:
      - to: your_email@gmail.com
        send_resolved: true
    - name: slack-notifier
      slack_configs:
      - api_url: https://hooks.slack.com/services/XXX/XXX/XXX
        channel: "#alerts"
        title: "{{ .CommonLabels.alertname }} - {{ .Status }}"
        text: |
          *Severity:* {{ .CommonLabels.severity }}
          *Summary:* {{ .Annotations.summary }}
    - name: opsgenie-notifier
      opsgenie_configs:
      - api_key: your_opsgenie_api_key
        responders:
        - name: DevOps Team
          type: team
EOF
```

---

##  Step 8: Cleanup

```bash
kubectl delete ns monitoring
kubectl delete -f kube-prometheus/manifests/
kubectl delete -f kube-prometheus/manifests/setup
```

---

##  Folder Structure

```
observability/
└── labs/
    └── 01-prometheus-grafana-alertmanager-nohelm/
        ├── README.md
        ├── alertmanager-secret.yaml
        ├── demo-alerts.yaml
        ├── simulate-pods.yaml
        └── nodeport-services.yaml
```

---

##  References

* [kube-prometheus GitHub](https://github.com/prometheus-operator/kube-prometheus)
* [Prometheus Alerting](https://prometheus.io/docs/alerting/latest/overview/)
* [Alertmanager Config](https://prometheus.io/docs/alerting/latest/configuration/)
* [Grafana Dashboards](https://grafana.com/grafana/dashboards)

