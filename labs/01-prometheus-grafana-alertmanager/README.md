#  Lab 01: Prometheus + Grafana + Alertmanager on Kubernetes using Helm

---

##  Objective

- Install `kube-prometheus-stack` (Prometheus, Grafana, Alertmanager)
- Access dashboards and alerting UIs
- Configure custom alerts
- Simulate alerts for learning
- Send alert emails using Gmail SMTP

---

##  Prerequisites

- Kubernetes cluster (Minikube, Kind, MicroK8s, GKE, etc.)
- `kubectl` and `helm` installed
- Internet access from nodes

---

##  Step 1: Add Helm Repo and Update

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
````

---

##  Step 2: Deploy `kube-prometheus-stack`

```bash
kubectl create namespace monitoring

helm upgrade --install prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring
```

---

##  Step 3: Access UIs (Port Forwarding)

```bash
# Prometheus
kubectl port-forward svc/prometheus-stack-kube-prometheus-prometheus -n monitoring 9090

# Grafana
kubectl port-forward svc/prometheus-stack-grafana -n monitoring 3000:80

# Alertmanager
kubectl port-forward svc/prometheus-stack-kube-prometheus-alertmanager -n monitoring 9093
```

* Prometheus: [http://localhost:9090](http://localhost:9090)
* Grafana: [http://localhost:3000](http://localhost:3000)
* Alertmanager: [http://localhost:9093](http://localhost:9093)

---

##  Step 4: Login to Grafana

* **Username:** `admin`
* **Password:** `prom-operator`

Change the password on first login.

---

##  Step 5: Import Dashboards in Grafana

Go to **+ → Import Dashboard** in Grafana:

| Dashboard Name     | ID     |
| ------------------ | ------ |
| Node Exporter Full | `1860` |
| Kubernetes Cluster | `6417` |

---

##  Step 6: Add Custom Alert Rules

Use the file `custom-rules.yaml` to define:

* High CPU
* High Memory
* Pod CrashLoop
* Node Down
* Pod Not Ready

### Apply the file:

```bash
kubectl apply -f custom-rules.yaml
```

Important:

* Ensure label `release: prometheus-stack` matches Helm release name

---

##  Step 7: Simulate Alerts

| Alert Name      | How to Simulate                                                                        |
| --------------- | -------------------------------------------------------------------------------------- |
| HighNodeCPU     | `kubectl run cpu-hog --image=busybox -- /bin/sh -c "while true; do :; done"`           |
| HighNodeMemory  | `kubectl run mem-hog --image=polinux/stress -- /usr/bin/stress --vm 1 --vm-bytes 512M` |
| PodCrashLooping | Apply `crashloop-pod.yaml` (container exits with error)                                |
| NodeDown        | `kubectl delete pod -l app.kubernetes.io/name=node-exporter -n monitoring`             |
| KubePodNotReady | Apply `slow-ready-pod.yaml` (fails readiness probe)                                    |

### We can achieve this using below as well

```
kubectl apply -f custom-rules.yaml
kubectl apply -f cpu-hog.yaml
kubectl apply -f mem-hog.yaml
kubectl apply -f crashloop-pod.yaml
kubectl apply -f slow-ready-pod.yaml

```

⏱ Wait 1–2 minutes and check:

* Prometheus UI → Alerts
* Alertmanager → [http://localhost:9093](http://localhost:9093)
* Grafana → Alerting → Manage

---

##  Step 8: Enable Gmail SMTP Alert Notifications

1. Generate a Gmail **App Password** at [https://myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)
2. Use `alertmanager-smtp-config.yaml`:

```yaml
alertmanager:
  config:
    global:
      smtp_smarthost: 'smtp.gmail.com:587'
      smtp_from: 'your-email@gmail.com'
      smtp_auth_username: 'your-email@gmail.com'
      smtp_auth_password: 'your-app-password'
      smtp_require_tls: true

    route:
      receiver: 'gmail'

    receivers:
      - name: 'gmail'
        email_configs:
          - to: 'your-email@gmail.com'
            send_resolved: true
```

3. Upgrade the Helm release:

```bash
helm upgrade prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring -f alertmanager-smtp-config.yaml
```

---

##  Cleanup

```bash
kubectl delete pod cpu-hog mem-hog crashloop slow-pod -n monitoring
helm uninstall prometheus-stack -n monitoring
kubectl delete ns monitoring
```

---

## 📚 References

* [Prometheus Helm Chart](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack)
* [Grafana Dashboards](https://grafana.com/grafana/dashboards)
* [PromQL Docs](https://prometheus.io/docs/prometheus/latest/querying/basics/)
* [Alertmanager Config](https://prometheus.io/docs/alerting/latest/configuration/)


---

## 🗂️ Folder Structure

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
└── alertmanager-smtp-config.yaml
```

---

