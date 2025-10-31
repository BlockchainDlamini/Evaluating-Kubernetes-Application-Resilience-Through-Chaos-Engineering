# Chaos Environment Setup and Reproduction Guide

## 1. Minikube Installation

```bash
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
minikube start --cpus=4 --memory=8192
```

## 2. Helm Installation

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

## 3. LitmusChaos Installation

```bash
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
helm repo update
kubectl create ns litmus

helm install chaos litmuschaos/litmus --namespace=litmus \
--set portal.frontend.service.type=NodePort \
--set portal.server.graphqlServer.genericEnv.CHAOS_CENTER_UI_ENDPOINT=http://chaos-litmus-frontend-service.litmus.svc.cluster.local:9091
```

## 4. Mealie Deployment (Nightly Image)

```bash
docker pull hkotel/mealie:nightly
```

Update `mealie-deployment.yaml` to use:

```yaml
image: hkotel/mealie:nightly
```

Apply manifests:

```bash
kubectl apply -f mealie-deployment.yaml
kubectl apply -f mealie-service.yaml
kubectl apply -f mealie-cluster-svc.yaml
kubectl apply -f mealie-env-litmus-chaos-enable.yml
```

Verify deployment:

```bash
kubectl get pods -n default
kubectl logs <mealie-pod-name> -n default
```

## 5. Prometheus and Grafana Setup

```bash
git clone https://github.com/litmuschaos/litmus.git
cd litmus/monitoring

kubectl -n litmus apply -f utils/metrics-exporters/litmus-metrics/chaos-exporter/
kubectl create ns monitoring

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prom prometheus-community/kube-prometheus-stack --namespace monitoring
```

## 6. Chaos Exporter Monitoring Integration

### PodMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: chaos-exporter-monitor
  namespace: monitoring
  labels:
    release: prometheus-stack
spec:
  selector:
    matchLabels:
      app: chaos-exporter
  namespaceSelector:
    matchNames:
      - litmus
  podMetricsEndpoints:
    - port: tcp
      interval: 1s
      metricRelabelings:
        - targetLabel: instance
          replacement: 'chaos-exporter-service'
```

### ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: chaos-exporter
  labels:
    k8s-app: chaos-exporter
  namespace: litmus
spec:
  jobLabel: app
  selector:
    matchLabels:
      app: chaos-exporter
  namespaceSelector:
    matchNames:
      - litmus
  endpoints:
    - port: tcp
      interval: 1s
      metricRelabelings:
        - targetLabel: instance
          replacement: 'chaos-exporter-service'
```

## 7. Application Instrumentation (Latency and Error Rate)

Mealie does not expose Prometheus metrics by default. To enable observability for chaos experiments—particularly latency and error rate—you must instrument the application manually or via a sidecar.

1. **Install Prometheus client library**
   ```bash
   pip install prometheus_client
   ```

2. **Add middleware to track latency and error rate**  
   Insert this into Mealie’s FastAPI app (e.g. in `main.py` or a dedicated metrics module):

   ```python
   from prometheus_client import Summary, Counter, generate_latest, CONTENT_TYPE_LATEST
   from starlette.middleware.base import BaseHTTPMiddleware
   from fastapi import Response

   REQUEST_LATENCY = Summary('http_request_latency_seconds', 'Latency of HTTP requests')
   REQUEST_ERRORS = Counter('http_request_errors_total', 'Total number of failed requests')

   class MetricsMiddleware(BaseHTTPMiddleware):
       async def dispatch(self, request, call_next):
           with REQUEST_LATENCY.time():
               response = await call_next(request)
               if response.status_code >= 400:
                   REQUEST_ERRORS.inc()
               return response

   app.add_middleware(MetricsMiddleware)

   @app.get("/metrics")
   def metrics():
       return Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)
   ```

3. **Expose `/metrics` endpoint**  
   This allows Prometheus to scrape metrics directly from the application pod.

4. **Configure Prometheus scraping**  
   Use a `PodMonitor` or `ServiceMonitor` to target the Mealie pod:

   ```yaml
   apiVersion: monitoring.coreos.com/v1
   kind: PodMonitor
   metadata:
     name: mealie-metrics
     namespace: default
   spec:
     selector:
       matchLabels:
         app: mealie
     podMetricsEndpoints:
       - port: http
         path: /metrics
         interval: 5s
   ```

## 8. Running Chaos Experiments via the LitmusChaos Portal (v3.22+)

### 1. Access the Litmus Portal
- Port-forward the frontend service:
  ```bash
  kubectl port-forward svc/chaos-litmus-frontend-service -n litmus 9091:9091
  ```
- Open your browser and visit: [http://localhost:9091](http://localhost:9091)

### 2. Log in to the Portal
- Use your configured credentials or the default admin login.

### 3. Navigate to Chaos Experiments
- Go to **Experiments** in the sidebar.
- Select **Schedule a Chaos Experiment**.

### 4. Choose Experiment Type
- Browse available experiments from the integrated ChaosHub.
- Select a predefined experiment (e.g. `pod-delete`, `cpu-hog`) or create a custom one.

### 5. Configure the Experiment
- Set the target namespace (e.g. `default`) and application labels.
- Define parameters such as duration, interval, ramp time, and probes.
- Optionally attach observability integrations or steady state checks.

### 6. Launch the Experiment
- Review the configuration summary.
- Click **Run Experiment** to initiate fault injection.

### 7. Monitor and Analyse
- View real-time logs, verdicts, and probe outcomes under the **Experiments** tab.
- Use the visual graph to trace execution stages.
- Correlate with Prometheus and Grafana dashboards for deeper analysis.


Here’s the updated section for your README, now including instructions for importing a FastAPI monitoring dashboard into Grafana:

---

## 9. Grafana Dashboard Setup

```bash
kubectl port-forward svc/prom-grafana -n monitoring 3000:80
```

### Access Grafana

- Open your browser and visit: [http://localhost:3000](http://localhost:3000)
- Login credentials:
  - Username: `admin`
  - Password: `prom-operator` (or check Helm release notes)

### Add Prometheus as a Data Source

1. Go to **Settings** → **Data Sources**
2. Select **Prometheus**
3. Set the URL to: `http://prom-prometheus.monitoring.svc.cluster.local:9090`
4. Click **Save & Test**

Got it. Here's the revised section with **Option 1 omitted**, keeping only the JSON import method:

---

### Import FastAPI Monitoring Dashboard

You can use a dashboard that visualises the metrics exposed by your FastAPI app (`http_request_latency_seconds`, `http_request_errors_total`).

#### Import via JSON File

1. Visit the dashboard repo such [Jafri115/fastapi-monitoring-grafana](https://github.com/Jafri115/fastapi-monitoring-grafana)
2. Locate the dashboard JSON file (usually under `grafana/dashboards/`)
3. In Grafana, go to **Dashboards** → **Import**
4. **Upload the JSON file or paste its contents directly**
5. Select Prometheus as the data source and click **Import**


### What This Dashboard Shows

- Request latency (`http_request_latency_seconds`)
- Error rate (`http_request_errors_total`)
- Request throughput (if `http_requests_total` is added)
- Endpoint-level breakdown (if labels are configured)

This dashboard helps correlate chaos experiment verdicts with application performance degradation, supporting your resilience analysis.

Login:

- Username: `admin`
- Password: `prom-operator` (or check Helm release notes)
- Custom dashboard for latency and error rate (if instrumented)
