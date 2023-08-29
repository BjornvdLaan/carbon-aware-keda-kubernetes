## Preparation
Set up the cluster:
```bash 
kind create cluster --config deployment/kind-nodeport-config.yml
```

Metrics Server
```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update
helm upgrade --install --set args={--kubelet-insecure-tls} metrics-server metrics-server/metrics-server --namespace kube-system
```

Set up monitoring:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://charts.helm.sh/stable
helm repo update
```
```bash
kubectl create namespace monitoring
helm install kind-prometheus prometheus-community/kube-prometheus-stack --namespace monitoring \
  --set prometheus.service.nodePort=30000 \
  --set prometheus.service.type=NodePort \
  --set grafana.service.nodePort=31000 \
  --set grafana.service.type=NodePort \
  --set prometheus-node-exporter.service.nodePort=32001 \
  --set prometheus-node-exporter.service.type=NodePort
```

Build and load Docker images:
```bash
docker build -t taskconsumer:latest consumer
docker build -t taskproducer:latest producer
```
```bash
kind load docker-image taskproducer:latest
kind load docker-image taskconsumer:latest
```

Set up KEDA and Localstack:
```bash
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda --namespace keda --version 2.11.1 --create-namespace

helm repo add localstack https://localstack.github.io/helm-charts
helm install localstack localstack/localstack
```

[Optional] Create the queue manually:
```bash
kubectl rollout status deployment localstack -n default --timeout=90s
aws sqs --endpoint-url=http://localhost:4566 create-queue --queue-name task-queue
```
(Note that the producer and consumer create the queue on startup if it does not exist yet)

## Demo: Showcase KEDA
Add producer to fill the queue
```bash
kubectl apply -f deployment/taskproducer.yaml
```

See number of messages in queue
```bash
awslocal sqs get-queue-attributes --queue-url http://localhost:4566/000000000000/task-queue --attribute-names All
```

Add consumer to consume from queue with 1 replica
```bash
kubectl apply -f deployment/taskconsumer.yaml
```

Add KEDA to make num of replicas depend on the queue
```bash
kubectl apply -f deployment/kedascaler.yaml
```

Remove consumer again to let queue fill up
```bash
kubectl delete deploy taskconsumer
```

Remove producer again to see KEDA scale down
```bash
kubectl delete deploy taskproducer
```

Scale up producer to fill up more quickly
```bash
kubectl scale --replicas=10 deployment/taskproducer
```

Scale down producer to fill up more slowly
```bash
kubectl scale --replicas=1 deployment/taskproducer
```

## Demo: Metrics Server and External Metrics Server

### 'Regular' Metrics Server
Get metrics for cpu and memory:
```bash
kubectl top pod taskconsumer-<POD ID>
```

Get metrics as raw json data:
```bash
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/namespaces/default/pods/taskconsumer-<POD ID>" | jq .
```

### KEDA External Metrics Server
Get name of the queue metric:
```bash
kubectl get so taskconsumer-scaler -o jsonpath={.status.externalMetricNames}
```

Get value of metric from external server:
```bash
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1/namespaces/default/s0-aws-sqs-task-queue?labelSelector=scaledobject.keda.sh/name=taskconsumer-scaler" | jq .
```

## Demo: CarbonAwareKEDAOperator
Install CarbonAwareKEDAOperator controller:
```bash
kubectl apply -f "https://github.com/Azure/carbon-aware-keda-operator/releases/download/v0.2.0/carbonawarekedascaler-v0.2.0.yaml"
```

```bash
kubectl apply -f deployment/carbonawarescaler.yaml
```


