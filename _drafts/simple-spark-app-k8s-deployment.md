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


4. Как запустить простое приложение на spark один раз?
5. Как запустить простое приложение на spark периодически?
6. Показать как запустить, как посмотреть что выполнилось, как настроить время выполнения изменив параметры.

# Spark History Server deployment
1. Как задеплоить history-server используя s3 в качестве backend для логов?
2. Промежуточный вывод и подводка к следующей статье.

# References