# Milestone 7: Observability and Troubleshooting

## Components

The `monitoring` Helm release installs Prometheus, Grafana, kube-state-metrics,
node-exporter and the Prometheus Operator. Prometheus data is stored on a 4Gi
Minikube hostpath PVC for three days. Grafana is intentionally stateless for
this learning cluster.

The application chart creates one `ServiceMonitor`. It selects the four Spring
services and scrapes their named `http` Service port at `/actuator/prometheus`
every 30 seconds. The matching NetworkPolicy permits that inbound traffic only
from Prometheus Pods in the `monitoring` namespace.

## Access and verification

```bash
kubectl port-forward -n monitoring service/monitoring-grafana 3000:80
```

Open `http://localhost:3000` and log in as `admin`. Read the password without
printing it in terminal history:

```bash
kubectl get secret grafana-admin -n monitoring \
  -o jsonpath='{.data.admin-password}' | base64 --decode; echo
```

Useful PromQL checks:

```promql
up{namespace="microservices-helm-dev"}
count(kube_pod_info{namespace="microservices-helm-dev"})
jvm_memory_used_bytes{namespace="microservices-helm-dev"}
http_server_requests_seconds_count{namespace="microservices-helm-dev"}
```

The bundled multi-cluster Kubernetes dashboards assume every series has a
`cluster` label. This small Minikube setup does not add that label, so use the
included **Microservices / Spring Boot** dashboard for the application. It is
deliberately written with the labels that this Prometheus actually stores.

## Safe incident exercises

### 1. Pod deletion

```bash
POD=$(kubectl get pod -n microservices-helm-dev -l app=employee \
  -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod -n microservices-helm-dev "$POD"
kubectl get pods -n microservices-helm-dev -w
kubectl describe deployment employee -n microservices-helm-dev
kubectl logs -n microservices-helm-dev deployment/employee --since=5m
```

The Deployment controller creates a replacement Pod. The old Pod is gone, but
the desired replica count remains unchanged.

### 2. MongoDB interruption and recovery

```bash
kubectl scale statefulset mongodb -n microservices-helm-dev --replicas=0
kubectl get pods -n microservices-helm-dev -w
kubectl scale statefulset mongodb -n microservices-helm-dev --replicas=1
kubectl rollout status statefulset/mongodb -n microservices-helm-dev --timeout=180s
kubectl logs -n microservices-helm-dev statefulset/mongodb --since=5m
```

Never delete `mongodb-data-mongodb-0` during this exercise. The StatefulSet
reattaches the same PVC when `mongodb-0` returns.

### 3. GitOps drift and recovery

Make a deliberate, reversible Helm values change in Git, commit and push it.
Argo CD detects the Git revision and reconciles the live cluster. Revert the
commit to recover. Do not use `kubectl edit` for desired configuration: Argo CD
self-heal will overwrite that drift.

## First-response order

1. `kubectl get pods -n <namespace>`: identify unhealthy workload.
2. `kubectl describe pod <pod> -n <namespace>`: inspect scheduling, probe and
   image events.
3. `kubectl logs <pod> -n <namespace> --previous`: inspect the prior crashed
   container if it restarted.
4. `kubectl get events -n <namespace> --sort-by='.lastTimestamp'`: correlate
   cluster events.
5. Use Prometheus/Grafana to determine when the symptom started and whether it
   affects one Pod or the whole service.
