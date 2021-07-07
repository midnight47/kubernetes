развернуть KEDA несколькими способами, они перечислены в документации. Я использую монолитный YAML:

wget https://github.com/kedacore/keda/releases/download/v2.1.0/keda-2.1.0.yaml
kubectl apply -f keda-2.1.0.yaml

ну или можно установить через helm

helm repo add kedacore https://kedacore.github.io/charts
helm repo update
kubectl create namespace keda
helm install keda kedacore/keda --namespace keda

я ставил через монолитный файл.
проверим что всё поднялось:

[root@prod-vsrv-kubemaster1 keda]# kubectl get pod -n keda
NAME                                      READY   STATUS    RESTARTS   AGE
keda-metrics-apiserver-57cbdb849f-cb8tl   1/1     Running   0          60s
keda-operator-58cb545446-29gm5            1/1     Running   0          60s


будем автоскейлить - для примера по метрике nginx nginx_ingress_controller_requests
запрос в prometheus будет следующий:
すm(irate( nginx_ingress_controller_requests{namespace="my-site"}[3m] )) by (ingress)*10

т.е. считаем общее количество запросов в неймспейс my-site за 3 минуты
создаём keda сущность:

apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: prometheus-scaledobject
  namespace: my-site
spec:
  scaleTargetRef:
    name: my-deployment-apache
  minReplicaCount: 1   # Optional. Default: 0
  maxReplicaCount: 8 # Optional. Default: 100
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus-kube-prometheus-prometheus.monitoring.svc.test.ru:9090
      metricName: nginx_ingress_controller_requests
      threshold: '100'
      query: sum(irate(nginx_ingress_controller_requests{namespace="my-site"}[3m])) by (ingress)*10


тут мы указываем в каком namespace нам запускаться:
namespace: my-site

указываем цель, т.е. наш deployment:
name: my-deployment-apache

задаём минимальное и максимальное количество реплик
minReplicaCount: 1 # значение по умолчанию: 0
maxReplicaCount: 8 # значение по умолчанию: 100

есть ещё 2 стандартные переменные отвечающие за то когда поды будут подниматься и убиваться:
pollingInterval: 30 # Optional. Default: 30 seconds
cooldownPeriod: 300 # Optional. Default: 300 seconds

указываем адрес нашего prometheus

serverAddress: http://prometheus-kube-prometheus-prometheus.monitoring.svc.test.ru:9090
адрес идёт в виде сервис.неймспейс.svc.имя_кластера

указываем нашу метрику:
metricName: nginx_ingress_controller_requests

указываем пороговое значение при котором начнётся автоскейлинг:
threshold: '100'

и соответственно наш запрос в prometheus:
query:

всё можно применять:

[root@kub-master-1 ~]# kubectl apply -f hpa-keda.yaml

Теперь если метрика пройдёт пороговое значение 100 запросов за 3 минуты будет создан новый pod
