<!--more-->
# Spark application k8s deployment series

## TLDR;
1. Где взять все ресурсы, чтобы ничего не читать?
2. Рассказать какой способ выбран, дать ссылки на все шаги.
3. Где получить необходимые ссылки и дополнительные ссылки?

## Part1: Simple spark application k8s deployment

### Introduction
1. Зачем деплоить spark на k8s? С какой версии спарк может работать на k8s (стабильно/экспериментально)?
2. Какие способы деплоймента есть (spark-submit, gcp-spark-on-k8s-operator, bitnami helm-chart)? 

### Requirements
1. Какие требования нужно удовлетворить, для того чтобы деплоймент считать успешным?

### Prerequisites
1. Какой k8s выбран для локальных тестов? Как его стартануть? Как стартануть с несколькими нодами?
2. Как проверить, что все работает как требуется?

### Simple spark Application deployment
1. Какие способы деплоймента есть?
2. Какой способ деплоймента в каких ситуациях выбирать?
3. Какой способ деплоймента выбран?
4. Как запустить простое приложение на spark один раз?
5. Как запустить простое приложение на spark периодически?
6. Показать как запустить, как посмотреть что выполнилось, как настроить время выполнения изменив параметры.

### Spark History Server deployment
1. Как задеплоить history-server используя s3 в качестве backend для логов?
2. Промежуточный вывод и подводка к следующей статье.

### References

## Part2: Spark application advanced scheduling and custom job deployment

### Advanced spark application scheduling
1. Какие проблемы планирование не решены и как их решить?
2. Какие планировщики есть? Какой выбран и почему?
3. Фичи volcano scheduler.
4. Как настроить volcano scheduler для spark-джобы и проверить, что все работает как надо?

### Executing custom spark application
1. Как можно запустить кастомное spark-приложение?
2. Какие image-ы для спарка бывают (native, datamechanics, gcp-spark-image, bitnami)?
3. Как расширить image для того, чтобы запустить свое приложение?
4. Промежуточный вывод и подводка к следующей статье.

### References

## Part3: Spark application monitoring and logging

### Spark application metric collection
1. Как можно собирать метрики используя встроеные возможности оператора?
2. Какие возможности для сбора метрик предоставлены в spark3?
3. Как собрать метрики используя prometheus + spark3?

### Spark application log aggregation
1. Какие подходы используются в k8s для сбора логов (kubectl logs, sidecar, daemon set)?
2. Как собрать логи driver и executor используя sidecar pattern?

### References