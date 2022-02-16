# Tekton-with-s3-logs

Based on next materials:
* [Tekton Pipeline and Dashboard](https://github.com/tektoncd/dashboard/blob/main/docs/walkthrough/walkthrough-kind.md#installing-tekton-pipelines)
* [Tekton Dashboard S3](https://github.com/tektoncd/dashboard/blob/main/docs/walkthrough/walkthrough-logs.md#before-you-begin)
* [BanzaiCloud logging params](https://banzaicloud.com/docs/one-eye/logging-operator/configuration/plugins/outputs/s3/)
* [BanzaiCloud to S3](https://banzaicloud.com/docs/one-eye/logging-operator/quickstarts/example-s3/)

## Before you begin

Tested and work with next versions:
* Kubernetes 1.21.5
* Tekton pipelines v0.32.1
* Tekton Dashboard v0.24.1
* BanzaiCloud Logging 3.17.1

Before you begin, make sure the following tools are installed:

1. [`kind`](https://kind.sigs.k8s.io/): For creating a local cluster running on top of docker.
1. [`kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl/): For interacting with your kubernetes cluster.
1. [`helm`](https://helm.sh/docs/intro/install/): For installing helm charts in your kubernetes cluster.

## Installing a working Tekton Pipeline and Dashboard

Install Tekton Pipeline to `tekton-pipelines` namespace

```bash
kubectl wait -n tekton-pipelines \
--for=condition=ready pod \
--selector=app.kubernetes.io/part-of=tekton-pipelines,app.kubernetes.io/component=controller \
--timeout=90s
```

Install Tekton Dashboard to `tekton-pipelines` namespace

```bash
curl -sL https://raw.githubusercontent.com/tektoncd/dashboard/main/scripts/release-installer | \
   bash -s -- install latest

kubectl wait -n tekton-pipelines \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/part-of=tekton-dashboard,app.kubernetes.io/component=dashboard \
  --timeout=90s
```

To access the Dashboard through the ingress controller it is necessary to set up an ingress rule. The ingress rule maps a host name to the Tekton Dashboard service running inside the cluster with $DOMAIN domain.
```bash
  kubectl apply -n tekton-pipelines -f ./yaml/tekton-db-ingress.yaml
```

## Collecting TaskRuns pod logs

Now you have a running storage, you can start collecting logs from the pods baking your `TaskRun`s and store those logs in S3.

To collect logs, you will install [banzaicloud logging operator](https://github.com/banzaicloud/logging-operator). The `logging operator` makes it easy to deploy a fluentd/fluentbit combo for streaming logs to a destination of your choice.

First, deploy the logging operator by running the following commands:

```bash
helm repo add banzaicloud-stable https://kubernetes-charts.banzaicloud.com

helm repo update

helm upgrade --install --wait --create-namespace --namespace tools logging-operator banzaicloud-stable/logging-operator --set createCustomResource=false
```

To start collecting logs you will need to create the logs pipeline using the available CRDs:

- `Logging` will deploy the necessary fluentd/fluentbit workloads:

```bash
kubectl -n tools apply -f ./yaml/fluentbit.yaml
```

This is a very simple deployment, please note that the position database and buffer volumes are ephemeral, this will stream logs again if pods restart.

Create next AWS :
* [S3 bucket](https://s3.console.aws.amazon.com/s3/bucket/create?region=us-east-1)
* [IAM Access Key user with read/write S3 access](https://console.aws.amazon.com/iam/home#/users$new?step=details) 

Create AWS secret

If you have your $AWS_ACCESS_KEY_ID and $AWS_SECRET_ACCESS_KEY set you can use the following snippet.

```bash
kubectl -n tools create secret generic logging-s3 --from-literal "awsAccessKeyId=$AWS_ACCESS_KEY_ID" --from-literal "awsSecretAccessKey=$AWS_SECRET_ACCESS_KEY"
```

- `ClusterOutput` defines the output of the logs pipeline. In our case AWS S3 into S3 bucket $S3_BUCKET:

```bash
kubectl -n tools apply -f ./yaml/clusterOutput.yaml
```

The `ClusterOutput` above will stream logs to our S3 Bucket in a `tekton-logs` bucket. It will buffer logs and will store one file per minute in the `<namespace_name>/<pod_name>/<container_name>/` path. All metadata will be omitted and only the log message will be stored, it will be the raw pod logs.

- `ClusterFlow` defines how the collected logs are dispatched to the outputs:

```bash
kubectl -n tools apply -f ./yaml/clusterFlow.yaml
```

The `ClusterFlow` above takes all logs from pods that have the `app.kubernetes.io/managed-by: tekton-pipelines` label (those are the pods baking `TaskRun`s) and dispatches them to the `ClusterOutput` created in the previous step.

Running the `PipelineRun` below produces logs and you will see corresponding objects being added in S3 bucket as logs are collected and stored by the logs pipeline.

```bash
kubectl -n tekton-pipelines create -f ./yaml/testPipelineRun.yaml
```
## Creating a service to serve collected logs

Now pod logs are collected and stored in your object storage, you will create a service to serve those logs.

Given a `namespace`, `pod` and `container`, the service will list files from s3, then stream the content of those files to serve logs to the caller.

Run the command below to create the kubernetes `Deployment` to serve your logs:

```bash
kubectl apply -n tools -f ./yaml/logServer.yaml
```

This deployment will start a container running `nodejs`. It will install `express` web server and `aws-sdk` to interact with S3.

Then it will run a web server exposing the `'/logs/:namespace/:pod/:container'` route to serve logs fetched from s3.

To make this available you will need to deploy a `Service` and an `Ingress`(optional) rule to expose the `Deployment`:

```bash
kubectl apply -n tools -f ./yaml/logServerService.yaml
kubectl apply -n tools -f ./yaml/logServerIngress.yaml
```

The logs server is available at `http://logs.$DOMAIN`.

## Setting up the Dashboard logs fallback

The last step in this walk-through is to setup the Tekton Dashboard to use the logs server you created above. The logs server will act as a fallback when the logs are not available anymore because the underlying pods baking `TaskRun`s are gone away.

First, delete the pods for your `TaskRun`s so that the Dashboard backend can't find the pod logs:

```bash
kubectl delete pod -l=app.kubernetes.io/managed-by=tekton-pipelines -n tekton-pipelines
```

The Dashboard displays the `Unable to fetch logs` message when browsing tasks.

Second, patch the Dashboard deployment to add the `--external-logs=http://logs-server.tools.svc.cluster.local:3000/logs` option:

```bash
kubectl patch deployment tekton-dashboard -n tekton-pipelines --type='json' \
  --patch='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--external-logs=http://logs-server.tools.svc.cluster.local:3000/logs"}]'
```

The logs are now displayed again, fetched from the logs server configured in the previous steps.

**NOTE:** Alternatively you can use the `--external-logs` argument when invoking the `installer` script:

```bash
curl -sL https://raw.githubusercontent.com/tektoncd/dashboard/main/scripts/release-installer | \
   bash -s -- install latest --external-logs http://logs-server.tools.svc.cluster.local:3000/logs

kubectl wait -n tekton-pipelines \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/part-of=tekton-dashboard,app.kubernetes.io/component=dashboard \
  --timeout=90s
```