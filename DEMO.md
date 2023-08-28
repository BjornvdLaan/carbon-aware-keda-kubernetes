## Preparation
Set up the cluster:
```bash 
kind create cluster --config deployment/kind-nodeport.yml
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

Create the queue manually:
```bash
kubectl port-forward svc/localstack 4566:4566
aws sqs --endpoint-url=http://localhost:4566 create-queue --queue-name task-queue
```
(Note that the producer and consumer create the queue on startup if it does not exist yet.)

## Demo steps
Add producer to fill the queue
```bash
kubectl apply -f deployment/taskproducer.yaml
```

Add consumer to consume from queue with 1 replica
```bash
kubectl apply -f deployment/taskconsumer.yaml
```

Add KEDA to make num of replicas depend on the queue
```bash
kubectl apply -f deployment/kedascaler.yaml
```

See number of messages in queue
```bash
awslocal sqs get-queue-attributes --queue-url http://localhost:4566/000000000000/task-queue --attribute-names All
```

Remove producer again to see KEDA scale down
```bash
kubectl delete deploy taskproducer
```

## Demo: Metrics Server and External Metrics Server
```bash
// Get metrics for cpu and memory
kubectl top pod taskconsumer-<POD ID>
```

```bash
// Get metrics as raw json data
kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/default/pods/taskconsumer-<POD ID> | jq .
```

## KEDA Metrics Server

```bash
// Get name of the queue metric
kubectl get so taskconsumer-scaler -o jsonpath={.status.externalMetricNames}
```

```bash
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1" | jq .
```

```bash
// Get value of metric from external server
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1/namespaces/default/s0-aws-sqs-task-queue?labelSelector=scaledobject.keda.sh%2Fname%3Dtaskconsumer-scaler" | jq .
```

