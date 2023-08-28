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
cd ../kepler-operator
make cluster-up CLUSTER_PROVIDER='kind' CI_DEPLOY=true GRAFANA_ENABLE=true
make deploy IMG=quay.io/sustainable_computing_io/kepler-operator:latest

kubectl config set-context --current --namespace=monitoring
kubectl apply -k config/samples/
```
`