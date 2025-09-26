## Demo Setup

### Prerequisites

- Helm installed on your system
- A Kubernetes cluster (minikube, GKE, EKS, etc.)

### Installation using Kubernetes Prometheus Stack

The recommended way to install Prometheus is using the official Kubernetes Prometheus Stack (kube-prometheus-stack) Helm chart. This chart provides a complete monitoring stack with all necessary components pre-configured and integrated.

#### Components Installed

1. **Prometheus Server**
   - Main time-series database for metrics collection
   - Scrapes metrics from configured targets
   - Provides powerful query language (PromQL)
   - Stores metrics in its own time-series database
   - Exposes a powerful HTTP API for querying

2. **Alertmanager**
   - Handles alerts sent by Prometheus
   - Groups, deduplicates, and routes alerts to the right people
   - Supports multiple notification methods (email, Slack, PagerDuty, etc.)
   - Provides a web UI for managing alerts

3. **Prometheus Node Exporter**
   - Exports hardware and OS metrics from nodes
   - Provides metrics about CPU, memory, disk, network, and more
   - Runs on each node in the cluster
   - Essential for monitoring node-level metrics

4. **Grafana**
   - Open-source analytics and monitoring solution
   - Provides rich visualization capabilities
   - Comes with pre-configured dashboards for Kubernetes
   - Supports custom dashboard creation
   - Integrates with Prometheus data source

5. **kube-state-metrics**
   - Exposes metrics about Kubernetes objects
   - Provides information about pods, nodes, deployments, etc.
   - Helps monitor Kubernetes resource state
   - Essential for cluster-level monitoring

6. **Prometheus Operator**
   - Manages Prometheus and Alertmanager instances
   - Provides custom resource definitions (CRDs) for Prometheus
   - Enables declarative configuration of monitoring
   - Simplifies management of monitoring resources

#### Install Helm Repository
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

#### Install the Stack
```bash
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
```

#### Accessing Services
1. **Prometheus UI**
```bash
kubectl port-forward svc/prometheus-operated 9090:9090
```
Access at: http://localhost:9090
- Query metrics using PromQL
- View target health
- Explore metrics
- Configure alert rules

2. **Grafana**
```bash
kubectl port-forward svc/prometheus-grafana 3000:80 --namespace monitoring
```
Access at: http://localhost:3000
Default admin password: 
```bash
kubectl get secret --namespace monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```
- View pre-configured dashboards
- Create custom dashboards
- Configure data sources
- Manage user permissions

3. **Alertmanager**
```bash
kubectl port-forward svc/prometheus-alertmanager-operated 9093:9093 --namespace monitoring
```
Access at: http://localhost:9093
- View and manage alerts
- Configure alert routing
- Monitor alert silencing
- View alert history

### Install sample application

```bash

# Deploy the sample application
kubectl apply -f ./sample-service.yaml

# Wait for the pod to be ready
kubectl wait --namespace demo --for=condition=ready pod --selector=app=prometheus-example-app --timeout=300s

# Get the NodePort
NODE_PORT=$(kubectl get svc prometheus-example-app -n demo -o jsonpath='{.spec.ports[0].nodePort}')

# Enable port-forward to access the application
kubectl port-forward svc/prometheus-example-app -n demo 8080:$NODE_PORT

# Access the application
curl http://localhost:$NODE_PORT

# Access the metrics endpoint
curl http://localhost:$NODE_PORT/actuator/prometheus
```
