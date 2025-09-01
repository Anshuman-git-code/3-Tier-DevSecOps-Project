# Monitoring a 3-Tier DevSecOps Project

This documentation provides a detailed explanation of **monitoring in DevOps**, focusing on the **3-Tier Application Project**. It explains the importance of monitoring, the tools used (Prometheus, Grafana, Exporters, kube-state-metrics, and Helm), and step-by-step configuration with practical use cases.

---

## 1. Introduction to Monitoring

Monitoring is a critical part of any **DevOps pipeline**. It ensures that applications, infrastructure, and deployments are running as expected.

**Definition**: Monitoring is the process of **collecting, analyzing, and visualizing** data from systems, applications, and infrastructure.

- **Collecting Data** – Gathering metrics such as CPU usage, memory, disk space, and application-specific logs.  
- **Analyzing Data** – Interpreting collected data to detect issues or performance bottlenecks.  
- **Visualizing Data** – Representing data in user-friendly formats such as graphs and dashboards.  

**Why Monitoring?**
- Ensures application health (uptime, errors, response time).  
- Tracks infrastructure usage (CPU, memory, disk).  
- Identifies failures (pod restarts, worker node crashes).  
- Helps in debugging issues with real-time visibility.  

Without monitoring, teams operate **blindly** and risk system failures without detection.

---

## 2. Key Monitoring Tools

### 2.1 Prometheus
- **Open-source monitoring and alerting toolkit**.  
- **Pull-based system** – Prometheus pulls metrics from endpoints (`/metrics`).  
- **Time-series database** – Stores metrics over time for analysis.  

If the application does not expose `/metrics`, **Exporters** are required.

### 2.2 Exporters
- Lightweight agents that **collect and expose metrics** for Prometheus.  
- Example: **Node Exporter**  
  - Provides **node-level metrics**: CPU, RAM, disk usage, etc.  
  - Runs as a service and exposes data at `/metrics` endpoint.  

### 2.3 kube-state-metrics
- Focuses on **Kubernetes resources** such as:  
  - Pods  
  - Namespaces  
  - Worker nodes  
- Exposes Kubernetes metrics in a Prometheus-compatible format.  

### 2.4 Grafana
- **Visualization tool** – converts raw metrics into **dashboards and charts**.  
- Provides pre-configured dashboards for Kubernetes, nodes, and pods.  
- Allows custom dashboards for specific monitoring needs.  

### 2.5 Helm
- **Package manager for Kubernetes**.  
- Simplifies installation of complex tools like Prometheus and Grafana.  
- Provides **Helm Charts** – pre-configured YAML templates.  
- Supports overriding defaults with **values.yaml** for customization.  

---

## 3. Setting Up Monitoring with Helm
### At Installer-VM instance terminal where the EKS Cluster Setup
### 3.1 Install Helm
Helm installation is simple via CLI commands. It acts as a Kubernetes package manager.  

```bash
mkdir YAML-Manifest && cd YAML-Manifest
sudo apt update && sudo apt upgrade -y
```

```bash
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
helm version
```

### 3.2 Add Helm Repo for Monitoring Tools
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 3.3 Create `values.yaml`
This file defines **custom configurations** for monitoring tools.

Example `values.yaml`:

```yaml
---
alertmanager:
  enabled: false
prometheus:
  prometheusSpec:
    service:
      type: LoadBalancer
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: ebs-sc
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 5Gi
grafana:
  enabled: true
  service:
    type: LoadBalancer
  adminUser: admin
  adminPassword: admin123
nodeExporter:
  service:
    type: LoadBalancer
kubeStateMetrics:
  enabled: true
  service:
    type: LoadBalancer
additionalScrapeConfigs:
  - job_name: node-exporter
    static_configs:
      - targets:
          - node-exporter:9100
  - job_name: kube-state-metrics
    static_configs:
      - targets:
          - kube-state-metrics:8080
```

### 3.4 Deploy Monitoring Stack with Helm
```bash
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack   -f values.yaml --namespace monitoring --create-namespace
```

This command:  
- Installs monitoring stack (Prometheus + Grafana + Exporters).  
- Uses `values.yaml` for custom configuration.  
- Creates namespace `monitoring` if not present.  

### 3.5 Expose Services

Check which services are not exposed via LoadBalancer ,i.e Not given the EXTERNAL-IP:

```bash
kubectl get all -n monitoring
```
<img width="1311" height="464" alt="Screenshot 2025-09-01 at 4 15 13 PM" src="https://github.com/user-attachments/assets/c5a4ee4f-6a63-4346-8bb2-ed96d06e4581" />

If services are not exposed via LoadBalancer, patch them:

```bash
# For Prometheus (use the main service, not prometheus-operated)
kubectl patch svc monitoring-kube-prometheus-prometheus -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'

# For kube-state-metrics (correct service name)
kubectl patch svc monitoring-kube-state-metrics -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'

# For node-exporter (correct service name)
kubectl patch svc monitoring-prometheus-node-exporter -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
```
<img width="1311" height="287" alt="Screenshot 2025-09-01 at 4 15 39 PM" src="https://github.com/user-attachments/assets/9d43b9e3-3b34-4466-bb85-8e1caffa7822" />

---

## 4. Accessing Grafana Dashboards

1. Get Grafana LoadBalancer URL:  
   ```bash
   kubectl get svc -n monitoring
   ```

2. Login with default credentials (set in `values.yaml`):  
   - Username: `admin`  
   - Password: `admin123`  

3. Navigate to **Dashboards**:  
   - Pre-built dashboards for:  
     - Pod metrics (CPU, memory usage).  
     - Node metrics.  
     - Network usage.  
   - Example metrics:  
     - Pod restarts.  
     - Network bytes received/sent.  
     - Worker node health.  

---

## 5. Benefits of Monitoring Setup

- **Real-time Visibility** – Tracks system performance continuously.  
- **Early Alerts** – Detects failures before they affect end-users.  
- **Scalability** – Supports monitoring across multi-cluster and multi-namespace.  
- **Custom Metrics** – Allows defining app-specific metrics.  
- **Centralized Dashboards** – Grafana unifies metrics for teams.  
<img width="1918" height="990" alt="Screenshot 2025-09-01 at 4 27 36 PM" src="https://github.com/user-attachments/assets/3a199401-1d30-4ce5-8887-88ab65a8f2f6" />
<img width="1918" height="990" alt="Screenshot 2025-09-01 at 4 27 00 PM" src="https://github.com/user-attachments/assets/04502c32-87c8-489e-ad20-e498e92a4ae9" />
<img width="1918" height="990" alt="Screenshot 2025-09-01 at 4 24 20 PM" src="https://github.com/user-attachments/assets/923072af-3cad-44e8-b904-92e07bf70d1b" />
<img width="1918" height="990" alt="Screenshot 2025-09-01 at 4 25 45 PM" src="https://github.com/user-attachments/assets/61c66b6e-e254-4257-a2df-61feed7e72c3" />
<img width="1918" height="990" alt="Screenshot 2025-09-01 at 4 24 47 PM" src="https://github.com/user-attachments/assets/fbc1b6a3-0c6f-4eab-a08e-5e78ab358c7e" />
<img width="1918" height="990" alt="Screenshot 2025-09-01 at 4 24 00 PM" src="https://github.com/user-attachments/assets/620c77f5-3b89-44a3-95a0-a08538ee620f" />
<img width="1918" height="990" alt="Screenshot 2025-09-01 at 4 25 08 PM" src="https://github.com/user-attachments/assets/a9b8f405-00e5-44e9-83ab-2ccf86eccaf6" />
---

## 6. Summary

In this monitoring setup, we achieved:

✅ Installed **Prometheus + Grafana** using Helm.  
✅ Configured **Node Exporter** and **kube-state-metrics** for metrics collection.  
✅ Patched services to expose monitoring tools externally.  
✅ Accessed pre-built **Grafana dashboards** for pods, nodes, and networking.  

This approach makes monitoring **efficient, scalable, and user-friendly** for DevOps environments.

---
