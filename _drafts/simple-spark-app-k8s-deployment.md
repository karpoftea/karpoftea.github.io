In this article I'll give a short overview of different ways to deploy spark application to k8s cluster and use one of them run a trivial spark application.

<!--more-->

Kubernetes gives a list of opportunities for executing spark applications:
- cost effectiveness via
    - elastic scaling
    - separated scaling of compute and storage resources
- fault tolerance (job can be redeployed to another availability zone if original zone is down)
- increased performance via smart resource planning (e.g. pod/node affinity, binpacking, gang scheduling)

In order to use this opportunities one should know how to deploy spark application to k8s.
Spark supports running on k8s natively from version [2.3](https://spark.apache.org/releases/spark-release-2-3-0.html) (GA came in [3.1.1](https://spark.apache.org/releases/spark-release-3-1-1.html)), and from that early times community used different ways to deploy spark applications to k8s.

# Requirements
If you want to use spark applications on k8s in production then application deployment should should be:
1. scalable
2. observable
3. fault tolerant

I'll use these characteristics to choose the way to deploy spark application.

# Prerequisites
For testing purposes we will use local [minikube](https://minikube.sigs.k8s.io/docs/) k8s cluster. Minikube provides excellent [Getting Started Guide](https://minikube.sigs.k8s.io/docs/start/) to setup local k8s cluster. When you are done with it just start minikube:
```shell
minikube start
```

If you want to start 2 node minikube cluster (to test multi-executor job deployment) use this command (mind `-p` option which stands for minikube profile, default is "minikube"):
```shell
minikube start --cpus 2 --memory 2g --nodes 2 -p multikube
```

# Spark-native VS k8s-native deployment types
There are two ways to deploy spark application to k8s: [spark-native](https://spark.apache.org/docs/latest/running-on-kubernetes.html) and k8s-native. Spark-native way is developed in apache-spark project, it uses spark-submit and set of kubernetes-related parameters to deploy spark application to k8s and in many ways looks similar for those who previously deployed applications to YARN (k8s is treated as another resource manager). K8s-native way uses kubernetes primitives Pod, Deployment, CRD etc to deploy spark application. Spark-native way is more imperative style while k8s-native is declarative. Because declarative way is native to k8s, supported by modern CD tools (ArgoCD, Flux) and allows to unify deployment for all applications (including non-spark) I'll use this way.

# Using k8s-operator for k8s-native deployment
Because spark application is quite complex (consists of 1 driver and N executors) the most promising way to handle its deployment is to use [k8s-operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) and delegate managing job lifecycle to it. [GCP spark-on-k8s-operator](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator) is advanced k8s-operator implementation for spark applications. It has rich set of features and community support:
- implements [large amount of parameters](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/user-guide.md#table-of-contents) to execute spark job
- has [pluggable](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/api-docs.md#sparkoperator.k8s.io/v1beta2.BatchSchedulerConfiguration) scheduler (e.g. [volcano scheduler](https://volcano.sh/en/) [integration](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/volcano-integration.md))
- allows to colocate pod via [pod (anti-)affinity](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/user-guide.md#using-pod-affinity)
- allows to setup [job](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/user-guide.md#monitoring) and [operator](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/quick-start-guide.md#enable-metric-exporting-to-prometheus) monitoring
- allows to use [sidecar container](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/user-guide.md#using-sidecar-containers) for driver and executor pods (useful for implementing log collection)
- is used by [many companies](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/who-is-using.md) and has gained [lots of stars](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/stargazers) on github

# Installing gcp-spark-on-k8s-operator
To install gcp-spark-on-k8s-operator lets use [helm chart](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/tree/master/charts/spark-operator-chart). If you new to [Helm](https://helm.sh) think of it as a package manager for k8s: helm chart contains all k8s resources required for application(see [intallation guide](https://helm.sh/docs/intro/install/) for setup instructions).

First of all lets add helm repository of spark-operator:
```shell
helm repo add spark-operator https://googlecloudplatform.github.io/spark-on-k8s-operator
```

Then create namespace where all spark-jobs are going to be executed:
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
- `--set webhook.enable=true` - enable webhook which is responsible for ????
- `--set metrics.enable=true` - enable exposure of spark-operator metrics ???
- `--set batchScheduler.enable=true` - enable custom batch scheduler (more on this later)
See [operator docs](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/tree/master/charts/spark-operator-chart#values) for a full list of parameters.

Check that chart was successfully installed:
```shell
helm status -n spark-operator spark-operator-release
```
, operator pods are running:
```shell
kubectl get pods -n spark-operator
```
, service account for spark jobs was created:
```shell
kubectl get sa -n -n spark-jobs
```

# Deploying one-off spark job
Successfuly deployed operator waits for SparkApplication/ScheduledSparkApplication (CRD) to be deployed.
For demo purposes we will use [SparkPi](https://github.com/apache/spark/blob/master/examples/src/main/scala/org/apache/spark/examples/SparkPi.scala) application bundled with spark disribution in `spark-examples_<scala-ver>-<spark-ver>.jar`.
To deploy one-off spark job we will use SparkApplication manifest:
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

Lets apply it:
```shell
kubectl apply -f manifests/spark-pi-minimal.yaml
```

List job pods to find driver pod and check logs: 
```shell
kubectl get pods -n spark-jobs
kubectl logs <spark-pi-minimal-pod> -n spark-jobs
```
![spark-pi-minimal-driver-startup-logs](/assets/images/spark-pi-minimal-driver-startup-logs.png)

Also list SparkApplication instances and check job events:
```shell
kubectl get sparkapplications -n spark-jobs
kubectl describe sparkapplications spark-pi-minimal -n spark-jobs
```
![spark-pi-minimal-crd-events](/assets/images/spark-pi-minimal-crd-events.png)

Hopefully job has started, now lets find job service and use port-forwarding to show driver UI:
```shell
kubectl get svc -n spark-jobs
kubectl port-forward service/spark-pi-minimal-1625753252972366200-ui-svc 4040:4040 -n spark-jobs
```
You can see job progress in UI and after job is done check logs to see the result:
![spark-pi-minimal-result-logs](/assets/images/spark-pi-minimal-result-logs.png)


4. Как запустить простое приложение на spark один раз?
5. Как запустить простое приложение на spark периодически?
6. Показать как запустить, как посмотреть что выполнилось, как настроить время выполнения изменив параметры.

# Spark History Server deployment
1. Как задеплоить history-server используя s3 в качестве backend для логов?
2. Промежуточный вывод и подводка к следующей статье.

# References