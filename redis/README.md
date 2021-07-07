kubectl create ns redis

helm repo add bitnami https://charts.bitnami.com/bitnami
helm pull bitnami/redis
tar -xvf redis-12.8.3.tgz

cd redis/

 echo -n "password-for-redis-cluster-Asergsdg2345KHJ" | base64
cGFzc3dvcmQtZm9yLXJlZGlzLWNsdXN0ZXItQXNlcmdzZGcyMzQ1S0hK

cat  secret-password.yaml
apiVersion: v1
kind: Secret
metadata:
  name: redis-password
  namespace: redis
type: Opaque
data:
  redis-password: cGFzc3dvcmQtZm9yLXJlZGlzLWNsdXN0ZXItQXNlcmdzZGcyMzQ1S0hK

kubectl apply -f secret-password.yaml


vim values.yaml

## Use password authentication
usePassword: true
password: ""
## Use existing secret (ignores previous password)
existingSecret: redis-password

## Prometheus Exporter / Metrics
##
metrics:
  enabled: true

  # Enable this if you're using https://github.com/coreos/prometheus-operator
  serviceMonitor:
    enabled: true
    ## Specify a namespace if needed
    namespace: monitoring

master:
  persistence:
    enabled: false

slave:
  persistence:
    enabled: false

sentinel:
  enabled: true
  downAfterMilliseconds: 600

clusterDomain: megaprod.local

cluster:
  enabled: true
  slaveCount: 3


В следующем файле в строке  tcp-check send AUTH\ myredispassword\r\n  нам надо указать наш пароль от redis


[root@prod-vsrv-kubemaster1 redis]# cat haproxy.cfg
frontend redis_frontend
mode tcp
bind 0.0.0.0:6379
default_backend redis-backend

backend redis-backend
    mode tcp
    option tcplog
    option tcp-check
    tcp-check send AUTH\ password-for-redis-cluster-Asergsdg2345KHJ\r\n
    tcp-check expect string +OK
    tcp-check send PING\r\n
    tcp-check expect string +PONG
    tcp-check send info\ replication\r\n
    tcp-check expect string role:master
    tcp-check send QUIT\r\n
    tcp-check expect string +OK
    server redis_node1_app01 redis-cluster-node-0.redis-cluster-headless:6379 maxconn 4096 check inter 1s
    server redis_node2_app02 redis-cluster-node-1.redis-cluster-headless:6379 maxconn 4096 check inter 1s
    server redis_node3_app03 redis-cluster-node-2.redis-cluster-headless:6379 maxconn 4096 check inter 1s



[root@prod-vsrv-kubemaster1 redis]# cat deployment-haproxy-redis.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-haproxy-deployment
  namespace: redis
  labels:
    app: redis-haproxy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: redis-haproxy
  template:
    metadata:
      labels:
        app: redis-haproxy
    spec:
      containers:
      - name: redis-haproxy
        image: haproxy:lts-alpine
        volumeMounts:
        - name: redis-haproxy-config-volume
          mountPath: /usr/local/etc/haproxy/haproxy.cfg
          subPath: haproxy.cfg
        ports:
        - containerPort: 6379
      volumes:
      - name: redis-haproxy-config-volume
        configMap:
          name: redis-haproxy-config
          items:
          - key: haproxy.cfg
            path: haproxy.cfg


[root@prod-vsrv-kubemaster1 redis]# cat service-haproxy-redis.yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-haproxy-balancer
  namespace: redis
spec:
  type: ClusterIP
  selector:
    app: redis-haproxy
  ports:
  - port: 6379
    targetPort: 6379





Всё, можно запускать

cd ../
helm install redis-cluster redis/ -f redis/values.yaml -n redis

[root@prod-vsrv-kubemaster1 redis]# kubectl create configmap redis-haproxy-config --from-file=redis/haproxy.cfg -n redis
[root@prod-vsrv-kubemaster1 redis]# kubectl apply -f redis/deployment-haproxy-redis.yaml -f redis/service-haproxy-redis.yaml

Посмотреть пароль можно командой:
kubectl get secret --namespace redis redis-password -o jsonpath="{.data.redis-password}" | base64 --decode

Подключаться к сервису можно командой:  redis-cli -h redis-cluster -p 6379 -a $REDIS_PASSWORD # Read only operations
   redis-cli -h redis-cluster -p 26379 -a $REDIS_PASSWORD # Sentinel access

Подключаться из вне можно командой:   redis-cli -h 127.0.0.1 -p 6379 -a $REDIS_PASSWORD

====================================================
Прокидываем порт наружу:
[root@prod-vsrv-kubemaster1 redis]# cat  /root/ConfigMap.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tcp-services
  namespace: ingress-nginx
data:
  5050: "m-logstash-megabuilder/logstash-logstash:5044"
  5672: "rabbitmq/rabbitmq:5672"
  6379: "redis/redis-haproxy-balancer:6379"


[root@prod-vsrv-kubemaster1 redis]# kubectl apply -f /root/ConfigMap.yml

[root@prod-vsrv-kubemaster1 redis]# kubectl edit service -n ingress-nginx ingress-nginx
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
  - name: redis-port-6379
    port: 6379
    protocol: TCP
    targetPort: 6379


[root@prod-vsrv-kubemaster1 redis]# kubectl edit daemonsets.apps -n ingress-nginx ingress-nginx-controller
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
        - containerPort: 6379
          hostPort: 6379
          name: redis-6379
          protocol: TCP

