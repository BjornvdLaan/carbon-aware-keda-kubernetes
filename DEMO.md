

## Create the queue 
Note that the producer and consumer create the queue on startup if it does not exist yet.

Create queue manually:
```bash
kubectl port-forward svc/localstack 4566:4566
aws sqs --endpoint-url=http://localhost:4566 create-queue --queue-name task-queue
```

## Metrics Server

```bash
kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/default/pods/taskconsumer-<POD ID> | jq .
```

```bash
kubectl top pod taskconsumer-<POD ID>
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

