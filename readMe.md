Chaos Environment Setup & Reproduction Guide
============================================

1. Minikube Installation
------------------------
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64

Start Minikube:
minikube start --cpus=4 --memory=8192

2. Helm Installation
--------------------
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

3. LitmusChaos Installation
---------------------------
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
helm repo update
kubectl create ns litmus

helm install chaos litmuschaos/litmus --namespace=litmus \
--set portal.frontend.service.type=NodePort \
--set portal.server.graphqlServer.genericEnv.CHAOS_CENTER_UI_ENDPOINT=http://chaos-litmus-frontend-service.litmus.svc.cluster.local:9091

4. Mealie Deployment (Nightly Image)
------------------------------------
Pull the latest nightly image:
docker pull hkotel/mealie:nightly

Update `mealie-deployment.yaml` to use:
image: hkotel/mealie:nightly

Apply manifests:
kubectl apply -f mealie-deployment.yaml
kubectl apply -f mealie-service.yaml
kubectl apply -f mealie-cluster-svc.yaml
kubectl apply -f mealie-env-litmus-chaos-enable.yml

Verify:
kubectl get pods -n default
kubectl logs <mealie-pod-name> -n default

5. Prometheus & Grafana Setup
-----------------------------
git clone https://github.com/litmuschaos/litmus.git
cd litmus/monitoring

kubectl -n litmus apply -f utils/metrics-exporters/litmus-metrics/chaos-exporter/
kubectl create ns monitoring

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prom prometheus-community/kube-prometheus-stack --namespace monitoring

6. Chaos Exporter Monitoring Integration
----------------------------------------

Create PodMonitor:
------------------
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

Create ServiceMonitor:
----------------------
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

7. Running Chaos Experiments
----------------------------
kubectl apply -f experiments/<engine-name>.yaml -n litmus

Monitor:
kubectl get wf -n litmus
kubectl describe wf <workflow-name> -n litmus

Fetch Logs:
kubectl logs <pod-name> -n litmus
kubectl logs <pod-name> -c <container-name> -n litmus

8. Grafana Dashboard Setup
--------------------------
Access Grafana:
kubectl port-forward svc/prom-grafana -n monitoring 3000:80

Login:
Username: admin
Password: prom-operator (or check Helm release notes)

Add Prometheus as a data source

Import LitmusChaos dashboard:
Dashboard ID: 11865

9. Cleanup
----------
kubectl delete -f experiments/
kubectl delete -f mealie-deployment.yaml
kubectl delete -f mealie-service.yaml
kubectl delete -f mealie-cluster-svc.yaml
kubectl delete -f mealie-env-litmus-chaos-enable.yml
helm uninstall chaos -n litmus
helm uninstall prom -n monitoring