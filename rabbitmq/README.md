это helm chart для установки rabbitmq ниже описан процесс установки 
в этом readmе  синтаксис может криво отображаться скачивайте этот файл ну или смотрите в wiki OneNote

Создаём namespace 
kubectl create namespace rabbitmq 
Выкачиваем чарт
helm pull stable/rabbitmq


генерируем пароль админки
echo -n "rlvjD8QZUB" | base64
cmx2akQ4UVpVQg==

создаём секрет 
cat rabbitmq/secret-admin-password.yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-admin-password
  namespace: rabbitmq
type: Opaque
data:
  rabbitmq-password: cmx2akQ4UVpVQg==

Применяем kubectl apply -f rabbitmq/secret-admin-password.yaml

Правим переменные:vim rabbitmq/values.yaml  

rabbitmq:
  username: admin
  password:
  existingPasswordSecret: secret-admin-password
  clustering:
    address_type: hostname
    k8s_domain: test.local
    rebalance: false

  configuration: |-
    ## Clustering
    cluster_formation.peer_discovery_backend  = rabbit_peer_discovery_k8s
    cluster_formation.k8s.host = kubernetes.default.svc.test.local
    cluster_formation.node_cleanup.interval = 10
    cluster_formation.node_cleanup.only_log_warning = true
    cluster_partition_handling = autoheal
    # queue master locator
    queue_master_locator=min-masters
    # enable guest user
    loopback_users.guest = false


persistence:
  enabled: true
  storageClass: "nfs-storageclass"
  accessMode: ReadWriteOnce
  size: 8Gi
  path: /opt/bitnami/rabbitmq/var/lib/rabbitmq
	
	
replicas: 2

ingress:
  enabled: true
  hostName: rabbitmq.prod.test.local
  path: /



helm install rabbitmq rabbitmq/ -n rabbitmq --values rabbitmq/values.yaml
  
  
если забыл пароль то посмотреть его можно командой:
echo "Password      : $(kubectl get secret --namespace rabbitmq secret-admin-password -o jsonpath="{.data.rabbitmq-password}" | base64 --decode)"



надо открыть доступ по порту для tcp 5672 для этого редактируем наш общий configmap

cat /root/ConfigMap.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tcp-services
  namespace: ingress-nginx
data:
  5050: "m-logstash-megabuilder/logstash-logstash:5044"
  5672: "rabbitmq/rabbitmq:5672"


здесь первые 5672 это указатель на порт снаружи.
первый rabbitmq - это наш namespace 
второй rabbitmq - это имя нашего сервиса который висит на порту 5672

применяем:
kubectl apply -f rabbitmq/ConfigMap-for-rabbit-port.yml

далее правим сервис:
kubectl edit service -n ingress-nginx ingress-nginx
spec:
  clusterIP: 13.100.150.207
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
  - name: proxied-tcp-5050
    port: 5050
    protocol: TCP
    targetPort: 5050
  - name: rabbitmq-port-5672
    port: 5672
    protocol: TCP
    targetPort: 5672

kubectl edit daemonsets.apps -n ingress-nginx  ingress-nginx-controller
        ports:
        - containerPort: 80
          hostPort: 80
          name: http
          protocol: TCP
        - containerPort: 443
          hostPort: 443
          name: https
          protocol: TCP
        - containerPort: 5050
          hostPort: 5050
          name: test-5050
          protocol: TCP
        - containerPort: 5672
          hostPort: 5672
          name: rabbitmq-5672
          protocol: TCP

		  
		  
		  
после добавленния на ингресах появится пор 5672 на который можно обращаться из вне.

если необходимо обращаться изнутри кластера то можно использовать адресс:
rabbitmq-headless.rabbitmq.svc.test.local:5672

