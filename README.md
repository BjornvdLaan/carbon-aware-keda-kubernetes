# Carbon Aware Consumer Demo using CarbonAwareSDK
CarbonAwareSDK helps us retrieve information about the carbon intensity (gCO2/kwh) in a certain time and place.
This repo contains an example project where we use this API to build two carbon aware consumers.

## Project setup

Our setup contains three components:

1. `localstack`, for providing a local AWS SQS queue.
2. `producer`, which puts a random task description on the queue at a configurable interval (1 sec by default).
3. `consumer`, reading tasks from the queue.

## Instructions

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda --namespace keda --version 2.11.1 --create-namespace

helm repo add localstack https://localstack.github.io/helm-charts
helm install localstack localstack/localstack
```

```bash
docker build -t taskconsumer:latest consumer
docker build -t taskproducer:latest producer

kubectl apply -f deployments/taskconsumer.yaml
kubectl apply -f deployments/taskproducer.yaml
kubectl apply -f deployments/kedascaler.yaml
```