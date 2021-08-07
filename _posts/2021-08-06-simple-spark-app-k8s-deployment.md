---
layout: post
title: "Simple Spark application k8s deployment"
date: 2021-08-06 20:00:00 +0300
tags: [spark, k8s, deployment]
comments: true
excerpt: In this post I'll give a short overview of different ways to deploy spark application to k8s cluster and use one of them run a trivial spark application.
---

* Table of contents
{:toc}

# Intro
[Kubernetes](https://kubernetes.io/) gives a list of opportunities for executing spark applications:
- cost effectiveness via
    - elastic scaling
    - separated scaling of compute and storage resources
- fault tolerance (job can be redeployed to another availability zone if original zone is down)
- increased performance via smart resource planning (e.g. pod/node affinity, binpacking, gang scheduling)

In order to use these opportunities one should know how to deploy spark application to k8s.
Spark supports running on k8s natively from version [2.3](https://spark.apache.org/releases/spark-release-2-3-0.html) (GA came in [3.1.1](https://spark.apache.org/releases/spark-release-3-1-1.html)), and from that early times community used different ways to deploy spark applications to k8s.

# Spark-native VS k8s-native deployment types
There are two ways to deploy spark application to k8s: [spark-native](https://spark.apache.org/docs/latest/running-on-kubernetes.html) and k8s-native. Spark-native way is developed in apache-spark project, it uses spark-submit and a set of kubernetes-related parameters to deploy spark application to k8s and in many ways looks similar for those who previously deployed applications to YARN (k8s is treated as another resource manager). It looks like this:
```
./bin/spark-submit \
    --master k8s://https://<k8s-apiserver-host>:<k8s-apiserver-port> \
    --deploy-mode cluster \
    --name spark-pi \
    --class org.apache.spark.examples.SparkPi \
    --conf spark.executor.instances=5 \
    --conf spark.kubernetes.container.image=<spark-image> \
    local:///path/to/examples.jar
```

K8s-native way uses kubernetes primitives Pod, Deployment, CRD etc to deploy spark application. Spark-native way is more imperative style while k8s-native is declarative. It looks like this:
```
kubectl apply -f manifests/spark-pi-minimal.yaml
```

Because declarative way is native to k8s, supported by modern CD tools (ArgoCD, Flux) and allows to unify deployment for any type of applications I'll use this way.

# K8s-native way for spark application deployment
Because spark application is quite complex (consists of 1 driver and N executors) the most promising way to handle its deployment is to use [k8s-operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) and delegate managing application lifecycle to it. [GCP spark-on-k8s-operator](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator) is an advanced k8s-operator implementation for spark applications. It has rich set of features and community support:
- implements [large amount of parameters](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/user-guide.md#table-of-contents) to execute spark job
- has [pluggable](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/api-docs.md#sparkoperator.k8s.io/v1beta2.BatchSchedulerConfiguration) scheduler (e.g. [volcano scheduler](https://volcano.sh/en/) [integration](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/volcano-integration.md))
- allows to co-locate pod via [pod (anti-)affinity](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/user-guide.md#using-pod-affinity)
- allows to setup [application](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/user-guide.md#monitoring) and [operator](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/quick-start-guide.md#enable-metric-exporting-to-prometheus) monitoring
- allows to use [sidecar container](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/user-guide.md#using-sidecar-containers) for driver and executor pods
- is used by [many companies](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/who-is-using.md) and has gained [lots of stars](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/stargazers) on github

## Prerequisites: installing minikube
For demo purposes we will use local [minikube](https://minikube.sigs.k8s.io/docs/) k8s cluster. Minikube provides excellent [Getting Started Guide](https://minikube.sigs.k8s.io/docs/start/) to setup local k8s cluster. When you are done with minikube installation start cluster:
```shell
minikube start
```
Check that cluster is ok:
```shell
minikube status
```

## Installing gcp-spark-on-k8s-operator
To install gcp-spark-on-k8s-operator lets use [helm chart](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/tree/master/charts/spark-operator-chart). If you new to [Helm](https://helm.sh) think of it as a package manager for k8s: helm chart contains all k8s resources required for application(see [intallation guide](https://helm.sh/docs/intro/install/) for setup instructions).

First of all lets add helm repository of spark-operator:
```shell
helm repo add spark-operator https://googlecloudplatform.github.io/spark-on-k8s-operator
```

Then create namespace where all spark-jobs are going to be deployed:
```shell
kubectl create ns spark-jobs
```

And install spark-operator:
```shell
helm install spark-operator-release spark-operator/spark-operator \
        --namespace spark-operator \
        --create-namespace \
        --set image.tag=v1beta2-1.2.3-3.1.1 \
        --set sparkJobNamespace=spark-jobs \
        --set serviceAccounts.spark.name=spark-jobs-sa \
        --set webhook.enable=true \
        --set metrics.enable=true \
        --set batchScheduler.enable=true
```
We used several options to install operator, they are:
- `--namespace spark-operator` - namespace called `spark-operator` where all operator resources will reside
- `--set image.tag=v1beta2-1.2.3-3.1.1` - use operator image for spark v3.1.1
- `--set sparkJobNamespace=spark-jobs` - set namespace where operator will launch spark jobs
- `--set serviceAccounts.spark.name=spark-jobs-sa` - set service account for spark jobs
- `--set webhook.enable=true` - enable webhook which is responsible for mounting arbitrary volumes and setting pod affinity
- `--set metrics.enable=true` - enable exposure of spark-operator metrics to Prometheus
- `--set batchScheduler.enable=true` - enable custom batch scheduler

See [operator docs](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/tree/master/charts/spark-operator-chart#values) for a full list of parameters.

Check that chart was successfully installed:
```shell
helm status -n spark-operator spark-operator-release
```
and service account for spark jobs was created:
```shell
kubectl get sa -n spark-jobs
```
and operator pods are running:
```shell
kubectl get pods -n spark-operator
```

It will take about 3-5 minutes to pull operator image and start all needed pods, so be patient.

## One-off spark job deployment
Successfuly deployed operator waits for [SparkApplication or ScheduledSparkApplication CRDs](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/tree/master/charts/spark-operator-chart/crds) to be deployed.
For demo purposes we will use [SparkPi](https://github.com/apache/spark/blob/master/examples/src/main/scala/org/apache/spark/examples/SparkPi.scala) application bundled in spark disribution (`spark-examples_<scala-ver>-<spark-ver>.jar`).
To deploy one-off spark job we will use following manifest:
```yaml
apiVersion: "sparkoperator.k8s.io/v1beta2"
kind: SparkApplication
metadata:
  name: spark-pi-minimal
  namespace: spark-jobs
spec:
  type: Scala
  mode: cluster
  image: "gcr.io/spark-operator/spark:v3.1.1"
  sparkVersion: "3.1.1"
  imagePullPolicy: IfNotPresent
  mainClass: org.apache.spark.examples.SparkPi
  mainApplicationFile: "local:///opt/spark/examples/jars/spark-examples_2.12-3.1.1.jar"
  arguments: ["25000"]
  restartPolicy:
    type: Never
  driver:
    cores: 1
    memory: "512m"
    serviceAccount: spark-jobs-sa
  executor:
    cores: 1
    instances: 1
    memory: "512m"
```
(All resources used in this post can be found in [git repo](https://github.com/karpoftea/spark-on-k8s-example))

Lets apply it:
```shell
kubectl apply -f manifests/spark-pi-minimal.yaml
```

List job pods to find driver pod and check logs: 
```shell
kubectl get pods -n spark-jobs
kubectl logs spark-pi-minimal-driver -n spark-jobs | less
```

You should see that driver has started, created web-app, and registered executor:
```plain
15:44:35 INFO Utils: Successfully started service 'SparkUI' on port 4040.
...
15:44:43 INFO KubernetesClusterSchedulerBackend$KubernetesDriverEndpoint: Registered executor NettyRpcEndpointRef(spark-client://Executor) (172.17.0.5:57074) with ID 1,  ResourceProfileId 0
...
15:44:45 INFO DAGScheduler: Submitting 50000 missing tasks from ResultStage 0 (MapPartitionsRDD[1] at map at SparkPi.scala:34) (first 15 tasks are for partitions Vector(0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14))
```

Also list SparkApplication instances and check job events:
```shell
kubectl get sparkapplications -n spark-jobs
kubectl describe sparkapplications spark-pi-minimal -n spark-jobs
```
You will see that sparkapplication has started driver and executor:
```shell
Events:
Type    Reason                     Age                    From            Message
----    ------                     ----                   ----            -------
Normal  SparkApplicationAdded      4m6s                   spark-operator  SparkApplication spark-pi-minimal was added, enqueuing it for submission
Normal  SparkApplicationSubmitted  4m                     spark-operator  SparkApplication spark-pi-minimal was submitted successfully
Normal  SparkDriverRunning         3m58s                  spark-operator  Driver spark-pi-minimal-driver is running
Normal  SparkExecutorPending       3m52s (x2 over 3m52s)  spark-operator  Executor spark-pi-eb46b77b1c233988-exec-1 is pending
Normal  SparkExecutorRunning       3m51s                  spark-operator  Executor spark-pi-eb46b77b1c233988-exec-1 is running
```

So job has started and doing well. Now lets find job service and use port-forwarding to view driver UI:
```shell
kubectl get svc -n spark-jobs
kubectl port-forward service/spark-pi-minimal-1625753252972366200-ui-svc 4040:4040 -n spark-jobs
```
Watch for job progress in UI (at [http://localhost:4040](http://localhost:4040))
![spark-pi-minimal-ui](/assets/images/spark-pi-minimal-ui.png)

After it's done check logs for result:
```shell
kubectl logs spark-pi-minimal-driver -n spark-jobs | grep 'Pi is roughly'
```

To cleanup delete SparkApplication resource:
```shell
kubectl delete -f manifests/spark-pi-minimal.yaml
```

## Periodic spark job deployment
To deploy periodic spark job use `ScheduledSparkApplication` resource:
```yaml
apiVersion: "sparkoperator.k8s.io/v1beta2"
kind: ScheduledSparkApplication
metadata:
  name: spark-pi-scheduled-minimal
  namespace: spark-jobs
spec:
  schedule: "@every 5m"
  template:
    type: Scala
    mode: cluster
    image: "gcr.io/spark-operator/spark:v3.1.1"
    sparkVersion: "3.1.1"
    imagePullPolicy: IfNotPresent  
    mainClass: org.apache.spark.examples.SparkPi
    mainApplicationFile: "local:///opt/spark/examples/jars/spark-examples_2.12-3.1.1.jar"
    arguments: ["50000"]
    restartPolicy:
      type: Never
    driver:
      cores: 1
      memory: "512m"
      serviceAccount: spark-jobs-sa
    executor:
      cores: 1
      instances: 1
      memory: "512m"
```

Apply manifest to start scheduling:
```shell
kubectl apply -f manifests/spark-pi-scheduled-minimal.yaml
```

Find instance of ScheduledSparkApplication and see jobs next execution time:
```shell
kubectl get scheduledsparkapplications -n spark-jobs
kubectl describe scheduledsparkapplications spark-pi-scheduled-minimal -n spark-jobs | grep 'Status' -A 5
```

Wait for a while and you'll see that job has started:
```shell
kubectl get pods -n spark-jobs
NAME                                                    READY   STATUS    RESTARTS   AGE
spark-pi-02d6dc7b1c3c79d4-exec-1                        1/1     Running   0          35s
spark-pi-scheduled-minimal-1628341920040907157-driver   1/1     Running   0          43s
```

To stop job scheduling delete resource:
```shell
kubectl delete -f manifests/spark-pi-scheduled-minimal.yaml
```

In the next post I'll show how to depoy Spark History Server on k8s and use it you spark jobs.

# References
- [Repository with k8s manifests used in this blog post](https://github.com/karpoftea/spark-on-k8s-example)
- [Running Spark on k8s natively](https://spark.apache.org/docs/latest/running-on-kubernetes.html)
- [GoogleCloudPlatform spark-on-k8s-operator](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator)