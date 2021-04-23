git clone https://github.com/prometheus-community/helm-charts.git
cd helm-charts/charts/kube-prometheus-stack/
докачиваем чарты:
helm dep update

создаём namescpase в котором будет всё крутиться:
kubectl create ns monitoring

[root@prod-vsrv-kubemaster1 charts]# vim kube-prometheus-stack/values.yaml

namespaceOverride: "monitoring"

  ## Alertmanager configuration directives
  ## ref: https://prometheus.io/docs/alerting/configuration/#configuration-file
  ##      https://prometheus.io/webtools/alerting/routing-tree-editor/
  ##
  config:
    global:
      resolve_timeout: 5m
    route:
        receiver: 'telegram'
        routes:
        - match:
            severity: critical
          repeat_interval: 48h
          continue: true
          receiver: 'telegram'
        - match:
            alertname: Watchdog
          repeat_interval: 48h
          continue: true
          receiver: 'telegram'
    receivers:
        - name: 'telegram'
          webhook_configs:
              - send_resolved: true
                url: 'http://alertmanager-bot:8080'


настраиваем ingress у alertmanager:

  ingress:                                                    
    enabled: true                                             
    hosts:                                                    
      - alertmanager.prod.test.local                       
    paths:                                                    
     - /

настраиваем volume для Alertmanager отмечу что в кластере настроен nfs-provisioner - nfs-storageclass

    ## Storage is the definition of how storage will be used by the Alertmanager instances.
    ## ref: https://github.com/prometheus-operator/prometheus-operator/blob/master/Documentation/user-guides/storage.md
    ##
    storage:
     volumeClaimTemplate:
       spec:
         storageClassName: nfs-storageclass
         accessModes: ["ReadWriteMany"]
         resources:
           requests:
             storage: 10Gi

теперь настроим grafana

grafana:
  enabled: true
  namespaceOverride: "monitoring"
  ## Deploy default dashboards.
  ##
  defaultDashboardsEnabled: true
  adminPassword: prom-operator
  ingress:
    ## If true, Grafana Ingress will be created
    ##
    enabled: true
     labels: {}
    ## Hostnames.
    ## Must be provided if Ingress is enable.
    ##
    hosts:
      - grafana.prod.test.local
    #hosts: []
    ## Path for grafana ingress
    path: /


  ## If using kubeControllerManager.endpoints only the port and targetPort are used
  ##
  service:
    port: 10252
    targetPort: 10252
    selector:
      k8s-app: kube-controller-manager
    #   component: kube-controller-manager


  ## If using kubeScheduler.endpoints only the port and targetPort are used
  ##
  service:
    port: 10251
    targetPort: 10251
    selector:
      k8s-app: kube-scheduler
    #   component: kube-scheduler

  ## If using kubeScheduler.endpoints only the port and targetPort are used
  ##
  service:
    port: 10251
    targetPort: 10251
    selector:
      k8s-app: kube-scheduler
    #   component: kube-scheduler


## Configuration for prometheus-node-exporter subchart
##
prometheus-node-exporter:
  namespaceOverride: "monitoring"


теперь настраиваем ingress для prometheus


  ingress:                                         
    enabled: true                                  
    annotations: {}                                
    labels: {}                                     
    ## Hostnames.                                  
    ## Must be provided if Ingress is enabled.     
    ##                                             
    hosts:                                         
      - prometheus.prod.test.local             
                                         
    ## Paths to use for ingress rules -            
    ##                                             
    paths:                                         
     - /

а так же volume:


    ## Prometheus StorageSpec for persistent data
    ## ref: https://github.com/prometheus-operator/prometheus-operator/blob/master/Documentation/user-guides/storage.md
    ##
    storageSpec:                                              
      volumeClaimTemplate:                                    
        spec:                                                 
          storageClassName: nfs-storageclass                  
          accessModes: ["ReadWriteMany"]                      
          resources:                                          
            requests:                                         
              storage: 10Gi


и теперь важная фишка, добавление label который надо будет добавить на все неймспейсы:


    ## Namespaces to be selected for ServiceMonitor discovery.
    ##
    serviceMonitorNamespaceSelector:
       matchLabels:
         prometheus: enabled



    ## Log level for Alertmanager to be configured with.
    ##
    logLevel: info

    ## Size is the expected size of the alertmanager cluster. The controller will eventually make the size of the
    ## running cluster equal to the expected size.
    replicas: 3



запускаем теперь helm chart

[root@prod-vsrv-kubemaster1 charts]# helm upgrade --install -name prometheus kube-prometheus-stack/ -f kube-prometheus-stack/values.yaml --namespace monitoring


при запуске добавился label release=prometheus

смотрим label на всех неймсмейсах:
kubectl get ns --show-labels

проставим на них label release=prometheus
kubectl label namespace --all "prometheus=enabled"

================================================================================================

теперь настроим сбор метрик с ingress controller,
создаём сервис для ingress. Указываем namespace в котором работает ingress, так же необходим label app.kubernetes.io/name: ingress-nginx данный лейб смотрим так:
kubectl describe pod -n ingress-nginx ingress-nginx-controller-vqjkl | grep -A3 Labels

Labels:               app.kubernetes.io/name=ingress-nginx
                      app.kubernetes.io/part-of=ingress-nginx
                      controller-revision-hash=bd6d56f49
                      pod-template-generation=1

mkdir exporter-ingres

cat exporter-ingres/service.yaml

apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: ingress-nginx
    release: prometheus
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  - name: https
    port: 443
    protocol: TCP
    targetPort: 443
  - name: prometheus
    port: 10254
    protocol: TCP
    targetPort: 10254
  selector:
    app.kubernetes.io/name: ingress-nginx

В данном файле так же обращаем внимание на:
name: prometheus
на это имя будет натравлен port у ServiceMonitor

теперь создаём ServiceMonitor, он будет создавать в prometheus target с метриками ingress controller:

cat exporter-ingres/service-monitor.yaml


apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app: ingress-nginx
    release: prometheus
  name: ingress-nginx
  namespace: monitoring
spec:
  endpoints:
  - honorLabels: true
    interval: 10s
    path: /metrics
    port: prometheus
    scheme: http
    scrapeTimeout: 10s
  namespaceSelector:
     any: true
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      release: prometheus

также правим:


    ## Enable scraping /metrics/resource from kubelet's service
    ## This is disabled by default because container metrics are already exposed by cAdvisor
    ##
    resource: true

применяем:

[root@prod-vsrv-kubemaster1 charts]# kubectl apply -f exporter-ingres/service.yaml -f exporter-ingres/service-monitor.yaml

=========================================================================================


Настроим Elasticsearch-exporter
он есть в том же репозитории:
https://github.com/prometheus-community/helm-charts.git с которого мы запускали сам prometheus, лежит он тут:

helm-charts/charts/prometheus-elasticsearch-exporter

!!!!!!!!!!!!сам elasticsearch уже должен быть установлен.

vim helm-charts/charts/prometheus-elasticsearch-exporter/values.yaml

ранее мы добавляли лейбл командой:
проставим на них label release=prometheus
kubectl label namespace --all "prometheus=enabled"

в конфиге будем указывать elasticsearch-master

смотрим какие есть лейблы на этом сервисе:
kubectl describe service -n elk elasticsearch-master | grep -A4 Labels
Labels:            app=elasticsearch-master
                   app.kubernetes.io/managed-by=Helm
                   chart=elasticsearch
                   heritage=Helm
                   release=elasticsearch

нас интересует app=elasticsearch-master

правим конфиг:

vim prometheus-elasticsearch-exporter/values.yaml

es:
  uri: http://elasticsearch-master:9200

serviceMonitor:
  ## If true, a ServiceMonitor CRD is created for a prometheus operator
  ## https://github.com/coreos/prometheus-operator
  ##
  enabled: true
  namespace: monitoring
  labels:
    app: elasticsearch-master
    release: prometheus
  interval: 10s
  scrapeTimeout: 10s
  scheme: http
  relabelings: []
  targetLabels:
    app: elasticsearch-master
    release: prometheus
  metricRelabelings: []
  sampleLimit: 0


можем устанавливать:

helm install elasticsearch-exporter --values prometheus-elasticsearch-exporter/values.yaml prometheus-elasticsearch-exporter/ -n elk

========================================================================


exporter rabbitmq

прогоняем label по всем namespace
kubectl label namespace --all "prometheus=enabled"

у меня уже установлен rabbitmq в кластере в namespace rabbitmq, прометеус в namespace monitoring

пароль от rabbitmq у меня закрыт в секрете:

vim prometheus-rabbitmq-exporter/values.yaml

loglevel: info
rabbitmq:
  url: http://rabbitmq-headless.rabbitmq.svc.test.local:15672
  user: admin
  password: secret-admin-password
  # If existingPasswordSecret is set then password is ignored
  existingPasswordSecret: ~



prometheus:
  monitor:
    enabled: true
    additionalLabels:
      release: prometheus
    interval: 15s
    namespace: []

helm install rabbitmq-exporter prometheus-rabbitmq-exporter/ -n monitoring --values prometheus-rabbitmq-exporter/values.yaml

===================================================================
exporter redis

прогоняем label по всем namespace
kubectl label namespace --all "prometheus=enabled"

у меня уже установлен redis в кластере в namespace redis, прометеус в namespace monitoring

пароль от redis у меня закрыт в секрете:

vim prometheus-redis-exporter/values.yaml

redisAddress: redis://redis-cluster-headless.redis.svc.test.local:6379

serviceMonitor:
  enabled: true
  namespace: monitoring
  # Set labels for the ServiceMonitor, use this to define your scrape label for Prometheus Operator
  labels:
    release: prometheus

auth:
  # Use password authentication
  enabled: true
  # Use existing secret (ignores redisPassword)
  secret:
    name: redis-password
    key: redis-password


helm install redis-exporter prometheus-redis-exporter/ -n redis --values prometheus-redis-exporter/values.yaml

=====================================================================================

телеграм бот валяется тут:

helm-charts/charts/telegrambot/
