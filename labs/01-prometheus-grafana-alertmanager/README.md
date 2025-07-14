# Lab 01: Prometheus + Grafana + Alertmanager on Kubernetes using Helm

---
##  Objective

* Install `kube-prometheus-stack` (Prometheus, Grafana, Alertmanager)
* Access dashboards and alerting UIs
* Configure custom alerts and simulate them
* Integrate email, Slack, and OpsGenie alert notifications

---
##  Prerequisites

* Kubernetes cluster (e.g., MicroK8s with MetalLB, GKE, Kind, Minikube)
* `kubectl` and `helm` installed
* Internet access from nodes
* Slack webhook, Gmail app password, OpsGenie API key (for notifications)

---
## Pre-requisite - Install an k8s, in this case microk8s

### Install MicroK8s
```
sudo snap install microk8s --classic
```

### Add your user to the microk8s group
```
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
```

### Apply new group without logout
```
newgrp microk8s
```

### Check MicroK8s status
```
microk8s status --wait-ready
```

### Enable common addons (DNS, Storage, Ingress, Helm, MetalLB)
```
microk8s enable dns storage ingress helm3
microk8s enable metallb:10.128.0.51-10.128.0.55   # Replace with your subnet range
```

### Set aliases for kubectl and helm (optional, for ease)
```
alias kubectl='microk8s kubectl'
alias helm='microk8s helm3'
```

### To make aliases permanent
```
echo "alias kubectl='microk8s kubectl'" >> ~/.bashrc
echo "alias helm='microk8s helm3'" >> ~/.bashrc
source ~/.bashrc
```

### Test Kubernetes is working
```
kubectl get nodes
kubectl get pods -A
```

### Install helm, kubectl
```
sudo snap install helm --classic
microk8s config > ~/.kube/config
```
---
###  Configure Gmail SMTP (App Password)

**Enable 2-Step Verification** on your Gmail account:
    [https://myaccount.google.com/security](https://myaccount.google.com/security)

**Create an App Password**:
    [https://myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)

   * Choose `Mail` and `Other` (enter e.g., `K8s Alertmanager`)
   * Copy the **16-character password**

**Use in Alertmanager config**:

```yaml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'your_email@gmail.com'
  smtp_auth_username: 'your_email@gmail.com'
  smtp_auth_password: 'your_app_password'  # paste 16-char app password here
  smtp_require_tls: true
```

---

###  Configure Slack Alerting

**Create a Slack App / Webhook**:
    [https://api.slack.com/messaging/webhooks](https://api.slack.com/messaging/webhooks)

**Choose a channel** (e.g., `#alerts`) and **install** the webhook

**Copy the Webhook URL**, it looks like:

```
https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX
```

**Add to Alertmanager config**:

```yaml
receivers:
  - name: slack-notifier
    slack_configs:
      - send_resolved: true
        api_url: https://hooks.slack.com/services/...
        channel: "#alerts"
        title: '{{ .CommonLabels.alertname }} - {{ .Status }}'
        text: |
          *Severity:* {{ .CommonLabels.severity }}
          *Instance:* {{ .CommonLabels.instance }}
          *Summary:* {{ .Annotations.summary }}
```

---


##  Step 1: Add Helm Repo and Update

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

---

##  Step 2: Install kube-prometheus-stack

```bash
kubectl create namespace monitoring

helm upgrade --install prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring
```

---

##  Step 3: Access Dashboards (NodePort Method)

```bash
cat <<EOF > nodeport-monitoring.yaml
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
    prometheus: prometheus-stack-kube-prom-prometheus
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
    alertmanager: prometheus-stack-kube-prom-alertmanager
  ports:
    - port: 9093
      targetPort: 9093
      nodePort: 32093
EOF

kubectl apply -f nodeport-monitoring.yaml
```

Now access via:

* **Grafana:** `http://<node-ip>:32080`
* **Prometheus:** `http://<node-ip>:32090`
* **Alertmanager:** `http://<node-ip>:32093`

---

##  Step 4: Grafana Login

* **Username:** `admin`
* **Password:** auto-generated:

  ```bash
  kubectl get secret prometheus-stack-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d && echo
  ```

---

##  Step 5: Import Grafana Dashboards

| Dashboard Name     | ID     |
| ------------------ | ------ |
| Node Exporter Full | `1860` |
| Kubernetes Cluster | `6417` |

Navigate to: **+ → Import** in Grafana and enter the ID.

---

##  Step 6: Create Alert Rules

```bash
cat <<EOF > custom-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: demo-alerts
  namespace: monitoring
spec:
  groups:
  - name: example.rules
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

kubectl apply -f custom-rules.yaml
```

---

##  Step 7: Simulate Alerts

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
    args: ["/bin/sh", "-c", "while true; do :; done"]
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
    args: ["--vm", "1", "--vm-bytes", "512M", "--vm-hang", "0"]
---
apiVersion: v1
kind: Pod
metadata:
  name: crashloop
  namespace: monitoring
spec:
  containers:
  - name: bad
    image: busybox
    command: ["/bin/false"]
---
apiVersion: v1
kind: Pod
metadata:
  name: slow-pod
  namespace: monitoring
spec:
  containers:
  - name: wait
    image: busybox
    args: ["sleep", "3600"]
    readinessProbe:
      exec:
        command: ["false"]
      initialDelaySeconds: 5
      periodSeconds: 5
EOF
```

---

##  Step 8: Configure Notifications (SMTP + Slack + OpsGenie)

```bash
cat <<EOF > alertmanager-values.yaml
alertmanager:
  config:
    global:
      resolve_timeout: 5m
      smtp_from: prayag.rhce@gmail.com
      smtp_smarthost: smtp.gmail.com:587
      smtp_auth_username: prayag.rhce@gmail.com
      smtp_auth_password: <replace-me>
      smtp_require_tls: true
    route:
      receiver: email-notifier
      group_by: ['alertname']
      group_wait: 10s
      group_interval: 1m
      repeat_interval: 1h
      routes:
      - receiver: slack-notifier
        matchers:
        - severity="critical"
      - receiver: email-notifier
        matchers:
        - severity="warning"
      - receiver: opsgenie-notifier
        matchers:
        - team="ops"
    receivers:
    - name: email-notifier
      email_configs:
      - to: prayag.rhce@gmail.com
        send_resolved: true
    - name: slack-notifier
      slack_configs:
      - api_url: https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX
        channel: '#alerts'
        send_resolved: true
        title: '{{ .CommonLabels.alertname }} - {{ .Status }}'
        text: |
          *Severity:* {{ .CommonLabels.severity }} *Instance:* {{ .CommonLabels.instance }} *Summary:* {{ .Annotations.summary }}
    - name: opsgenie-notifier
      opsgenie_configs:
      - api_key: your_opsgenie_api_key
        send_resolved: true
        responders:
        - name: DevOps Team
          type: team
EOF

helm upgrade prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring -f alertmanager-values.yaml
```

---

##  Cleanup

```bash
kubectl delete pod cpu-hog mem-hog crashloop slow-pod -n monitoring
helm uninstall prometheus-stack -n monitoring
kubectl delete ns monitoring
```

---

##  Folder Structure

```
observability/
└── labs/
    └── 01-prometheus-grafana-alertmanager/
        ├── README.md
        ├── custom-rules.yaml
        ├── cpu-hog.yaml
        ├── mem-hog.yaml
        ├── crashloop-pod.yaml
        ├── slow-ready-pod.yaml
        ├── nodeport-monitoring.yaml
        └── alertmanager-values.yaml
```

---

##  References

* [kube-prometheus-stack Chart](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack)
* [Prometheus Alerting](https://prometheus.io/docs/alerting/latest/overview/)
* [Alertmanager Config](https://prometheus.io/docs/alerting/latest/configuration/)
* [Grafana Dashboards](https://grafana.com/grafana/dashboards)

---

