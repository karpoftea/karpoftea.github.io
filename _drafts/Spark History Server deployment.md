Развертывание Spark HS является довольно тривиальной задачей, поскольку это не более чем развертывание обычного веб-приложения. Сам Spark HS идет в официальном образе spark'а, поэтому для нашего примера мы возьмем именно его. Для развертывания можно использовать [heml-chart](https://github.com/helm/charts/tree/master/stable/spark-history-server), но для наглядности я буду использовать обычные манифесты.

Будем устанавливать Spark HS в отдельном от spark-приложений неймспейсе `sparkhs`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: sparkhs
```
и создадим для него сервисный аккаунт:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sparkhs
  namespace: sparkhs
```

Чтобы Spark HS был отказоустойчивым используем Deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sparkhs
  namespace: spark
spec:
  replicas: 1
  template:
    spec:
      serviceAccountName: sparkhs
      containers:
      - name: sparkhs
        image: gcr.io/spark-operator/spark:v3.1.1
        imagePullPolicy: IfNotFound
        env:
          - name: SPARK_NO_DAEMONIZE
            value: "true"
          - name: SPARK_DAEMON_MEMORY
            value: "512m"
          - name: SPARK_LOG_DIR
            value: "/tmp/sparkhs/event-logs"
          - name: SPARK_HISTORY_OPTS
            value: "-Dspark.history.fs.logDirectory=$(SPARK_LOG_DIR)"
        args:
          - "/bin/sh"
          - "-c"
          - "/opt/spark/bin/spark-class org.apache.spark.deploy.history.HistoryServer"
        ports:
          - name: historyport
            containerPort: 18080
            protocol: TCP
        livenessProbe:
          initialDelaySeconds: 90
          periodSeconds: 30
          httpGet:
            path: /
            port: historyport
        readinessProbe:
          initialDelaySeconds: 90
          periodSeconds: 30
          httpGet:
            path: /
            port: historyport
        resources:
          requests:
            cpu: 250m
            memory: 700Mi
          limits:
            cpu: 250m
            memory: 700Mi
```
Для того, чтобы 