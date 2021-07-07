kubectl create ns kafka

git clone https://github.com/confluentinc/cp-helm-charts.git
cd cp-helm-charts/へlm dependency update charts/cp-kafka/[root@prod-vsrv-kubemaster1 cp-helm-charts]# vim charts/cp-zookeeper/values.yaml
persistence:
  enabled: true
  ## Size for Data dir, where ZooKeeper will store the in-memory database snapshots.
  dataDirSize: 5Gi
  dataDirStorageClass: "nfs-storageclass"

  ## Size for data log dir, which is a dedicated log device to be used, and helps avoid competition between logging and snaphots.
  dataLogDirSize: 5Gi
  dataLogDirStorageClass: "nfs-storageclass"


[root@prod-vsrv-kubemaster1 cp-helm-charts]# vim charts/cp-kafka/values.yaml
persistence:
  enabled: true
  size: 1Gi
  storageClass: "nfs-storageclass"



[root@prod-vsrv-kubemaster1 cp-helm-charts]# vim charts/cp-kafka/values.yaml
configurationOverrides:
  "offsets.topic.replication.factor": "3"
  "default.replication.factor": 3


Ставим:
helm install confluent ./charts/cp-kafka/ --values ./charts/cp-kafka/values.yaml -n kafka


==================================
поставим теперь kafka-manager:

helm repo add stable https://kubernetes-charts.storage.googleapis.com
helm pull stable/kafka-manager
tar -xvf kafka-manager-2.3.5.tgz
rm -rf kafka-manager-2.3.5.tgz
[root@prod-vsrv-kubemaster1 cp-helm-charts]# vim kafka-manager/values.yaml

zkHosts: "confluent-cp-zookeeper.kafka.svc.test.ru:2181"

basicAuth:
  enabled: true
  username: "admin"
  ## Defaults to a random 10-character alphanumeric string if not set
  ##
  password: "admin"

ingress:
  enabled: true
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  path: /
  hosts:
    - kafka.prod.test.local



[root@prod-vsrv-kubemaster1 cp-helm-charts]# helm install kafka-manager kafka-manager/ --values kafka-manager/values.yaml -n kafka

Проверяем по адресу:
http://kafka.prod.test.ru/

далее настраиваем в панельке кластер, в качестве адреса для zookeeper указываем:℅nfluent-cp-zookeeper.kafka.svc.test.ru:2181




cat test-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: kafka-client
  namespace: kafka
spec:
  containers:
  - name: kafka-client
    image: confluentinc/cp-enterprise-kafka:6.0.1
    command:
      - sh
      - -c
      - "exec tail -f /dev/null"

kubectl apply -f test-pod.yaml

kubectl exec -it kafka-client -n kafka /bin/bash


Смотрим список топиков:
[appuser@kafka-client ~]$ kafka-topics --bootstrap-server confluent-cp-kafka-headless:9092 --list
__consumer_offsets
_confluent-metrics
test-ropic


создаём producer:
[appuser@kafka-client ~]$ kafka-console-producer --broker-list confluent-cp-kafka-0.confluent-cp-kafka-headless.kafka:9092 --topic test-ropic
>sdfsf
>sdfsf
>rtert
>hyhy

Читаем эти сообщения с помощью consumer appuser@kafka-client ~]$ kafka-console-consumer --bootstrap-server confluent-cp-kafka-0.confluent-cp-kafka-headless.kafka:9092 --topic test-ropic --from-beginning
sdfsf
sdfsf
rtert
hyhy


Создаём topic:
[appuser@kafka-client ~]$ kafka-topics --bootstrap-server confluent-cp-kafka-headless:9092 --topic NEW-TEST-TOPIC --create --partitions 1 --replication-factor 1 --if-not-exists
Created topic NEW-TEST-TOPIC.
роверяем:
[appuser@kafka-client ~]$ kafka-topics --bootstrap-server confluent-cp-kafka-headless:9092 --list
NEW-TEST-TOPIC
__consumer_offsets
_confluent-metrics
new-test-topic

Удаляем:
[appuser@kafka-client ~]$ kafka-topics --bootstrap-server confluent-cp-kafka-headless:9092 --topic NEW-TEST-TOPIC --delete --if-exists

Проверяем:
[appuser@kafka-client ~]$ kafka-topics --bootstrap-server confluent-cp-kafka-headless:9092 --list
__consumer_offsets
_confluent-metrics
new-test-topic

