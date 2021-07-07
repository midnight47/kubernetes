Для начала создаём namespase elk
kubectl create namespace elk

Выкачиваем Helm chartぎt clone https://github.com/elastic/helm-charts.git
cd helm-charts/ереключаемся на стабильную ветку:
git checkout 7.9

Далее нам надо сгенерировать сертификаты на основе которых будет всё работать.

[root@prod-vsrv-kubemaster1 helm-charts]# mkdir mkdir /certs
[root@prod-vsrv-kubemaster1 helm-charts]# cd /certs/

Создаём файл в котором укажем наши доменные имена (logstash требователен к наличию домена в сертификате)
cat cnf
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
C = KG
ST = Bishkek
L = Bishkek
O = Test
CN = elasticsearch-master

[v3_req]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
basicConstraints = CA:TRUE
subjectAltName = @alt_names

[alt_names]
DNS.1 = elasticsearch-master
DNS.2 = kibana-kibana
DNS.3 = elasticsearch-master-headless
DNS.4 = logstash-logstash-headless
DNS.5 = elk.prod.test.ru
DNS.6 = elasticsearch-master-0
DNS.7 = elasticsearch-master-1
DNS.8 = elasticsearch-master-2

openssl genpkey -aes-256-cbc -pass pass:123456789 -algorithm RSA -out mysite.key -pkeyopt rsa_keygen_bits:3072
Тут надо будет ввести пароль при генерации а дальше заполнить данные сертификата.
Я везде задал пароль 123456789
openssl req -new -x509 -key mysite.key -sha256 -config cnf -out mysite.crt -days 7300
Enter pass phrase for mysite.key:

создаём сертификат p12 который нужен elastic
openssl pkcs12 -export -in mysite.crt -inkey mysite.key -out identity.p12 -name "mykey"
Enter pass phrase for mysite.key:   вот тут вводим наш пароль 123456789
Enter Export Password:   ТУТ ОСТАВЛЯЕМ БЕЗ ПАРОЛЯ
Verifying - Enter Export Password:  ТУТ ОСТАВЛЯЕМ БЕЗ ПАРОЛЯ


[root@prod-vsrv-kubemaster1 certs]# docker run --rm  -v /certs:/certs -it openjdk:oracle  keytool -import -file /certs/cacert.pem -keystore /certs/trust.jks  -storepass 123456789

В появившемся сообщении я соглашаюсь что доверяю этому сертификату:

Вытаскиваем приватный ключ чтоб он у нас был без пароля
openssl rsa -in mysite.key -out mysite-without-pass.key
Enter pass phrase for mysite.key:
writing RSA key


Всё готово, все нужные сертификаты для elasticsearch сгенерированы:

[root@prod-vsrv-kubemaster1 certs]# ll
total 32
-rw-r--r-- 1 root root  575 Feb 10 10:15 cnf
-rw-r--r-- 1 root root 3624 Feb 10 10:15 identity.p12
-rw-r--r-- 1 root root 1935 Feb 10 10:15 mysite.crt
-rw-r--r-- 1 root root 2638 Feb 10 10:15 mysite.key
-rw-r--r-- 1 root root 2459 Feb 10 10:39 mysite-without-pass.key
-rw-r--r-- 1 root root 1682 Feb 10 10:15 trust.jks


Создаём секрет с сертификатами
root@prod-vsrv-kubemaster1 certs]# kubectl create secret generic elastic-certificates -n elk --from-file=identity.p12 --from-file=mysite.crt --from-file=mysite.key --from-file=mysite-without-pass.key --from-file=trust.jks

Создаём секрет с логином и паролем для eastic:
kubectl create secret generic secret-basic-auth -n elk --from-literal=password=elastic --from-literal=username=elastic

Перейдём к настройке переменных у elastic elasticsearch/values.yaml

запускаем elastic в 1 instance 
vim elasticsearch/values.yaml
replicas: 1 
minimumMasterNodes: 1


Тут включаем xpack добавляем сертификаты и указываем директорию для снапшотов 
esConfig:
  elasticsearch.yml: |
    path.repo: /snapshot
    xpack.security.enabled: true
    xpack.security.transport.ssl.enabled: true
    xpack.security.transport.ssl.verification_mode: certificate
    xpack.security.transport.ssl.keystore.path: /usr/share/elasticsearch/config/certs/identity.p12
    xpack.security.transport.ssl.truststore.path: /usr/share/elasticsearch/config/certs/identity.p12
    xpack.security.http.ssl.enabled: true
    xpack.security.http.ssl.truststore.path: /usr/share/elasticsearch/config/certs/identity.p12
    xpack.security.http.ssl.keystore.path: /usr/share/elasticsearch/config/certs/identity.p12


Также
extraEnvs:
  - name: ELASTIC_PASSWORD
    valueFrom:
      secretKeyRef:
        name: secret-basic-auth
        key: password
  - name: ELASTIC_USERNAME
    valueFrom:
      secretKeyRef:
        name: secret-basic-auth
        key: username

Также
secretMounts:
  - name: elastic-certificates
    secretName: elastic-certificates
    path: /usr/share/elasticsearch/config/certs

Также
volumeClaimTemplate:
  accessModes: [ "ReadWriteOnce" ]
  storageClassName: nfs-storageclass
  resources:
    requests:
      storage: 3Gi

Также
protocol: https

Также 
antiAffinity: "soft"


Перейдём к настройке kibana

kibana/values.yaml

Правим http на https
elasticsearchHosts: "https://elasticsearch-master:9200"

Также 
extraEnvs:
  - name: 'ELASTICSEARCH_USERNAME'
    valueFrom:
      secretKeyRef:
        name: secret-basic-auth
        key: username
  - name: 'ELASTICSEARCH_PASSWORD'
    valueFrom:
      secretKeyRef:
        name: secret-basic-auth
        key: password

Также
secretMounts:
  - name: elastic-certificates
    secretName: elastic-certificates
    path: /usr/share/kibana/config/certs

Также
kibanaConfig:
   kibana.yml: |
    server.ssl:
      enabled: true
      key: /usr/share/kibana/config/certs/mysite-without-pass.key
      certificate: /usr/share/kibana/config/certs/mysite.crt
    xpack.security.encryptionKey: "something_at_least_32_characters"
    elasticsearch.ssl:
      certificateAuthorities: /usr/share/kibana/config/certs/mysite.crt
      verificationMode: certificate

Также
ingress:
  enabled: true
  annotations:
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
     nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
  path: /
  hosts:
    - elk.prod.test.ru


Перейдём к настройке logstash
logstash/values.yaml

Включаем xpack
logstashConfig:
  logstash.yml: |
    http.host: 0.0.0.0
    xpack.monitoring.enabled: true
    xpack.monitoring.elasticsearch.username: '${ELASTICSEARCH_USERNAME}'
    xpack.monitoring.elasticsearch.password: '${ELASTICSEARCH_PASSWORD}'
    xpack.monitoring.elasticsearch.hosts: [ "https://elasticsearch-master:9200" ]
    xpack.monitoring.elasticsearch.ssl.certificate_authority: /usr/share/logstash/config/certs/mysite.crt

Также
logstashPipeline:
  logstash.conf: |
    input {
       exec { command => "uptime" interval => 30 }
       beats {
       port => 5045
       }
    }

    filter {
        if [kubernetes][namespace] == "terminal-soft" {
         mutate {
          add_tag => "tag-terminal-soft"
          remove_field => ["[agent][name]","[agent][version]","[host][mac]","[host][ip]"] }
        }
       }

    output {
      if "tag-terminal-soft" in [tags] {
        elasticsearch {
            hosts => [ "https://elasticsearch-master:9200" ]
            cacert => "/usr/share/logstash/config/certs/mysite.crt"
            manage_template => false
            index => "terminal-soft-%{+YYYY.MM.dd}"
            ilm_rollover_alias => "terminal-soft"
            ilm_policy => "terminal-soft"
            user => '${ELASTICSEARCH_USERNAME}'
            password => '${ELASTICSEARCH_PASSWORD}'
        }
      }
    }

Также
extraEnvs:
  - name: 'ELASTICSEARCH_USERNAME'
    valueFrom:
      secretKeyRef:
        name: secret-basic-auth
        key: username
  - name: 'ELASTICSEARCH_PASSWORD'
    valueFrom:
      secretKeyRef:
        name: secret-basic-auth
        key: password


Также
secretMounts:
  - name: elastic-certificates
    secretName: elastic-certificates
    path: /usr/share/logstash/config/certs

Также
antiAffinity: "soft"

Также раскоментим сервис и укажем в нём порт куда всё пересылать 5045 (его мы задали выше в input)
service:
  annotations: {}
  type: ClusterIP
  ports:
    - name: beats
      port: 5044
      protocol: TCP
      targetPort: 5045


Перейдём к настройке filebeat
filebeat/values.yaml

filebeatConfig:
  filebeat.yml: |
    filebeat.inputs:
    - type: container
      paths:
        - /var/log/containers/*.log
      processors:
      - add_kubernetes_metadata:
          host: ${NODE_NAME}
          matchers:
          - logs_path:
              logs_path: "/var/log/containers/"

    output.logstash:
      enabled: true
      hosts: ["logstash-logstash:5044"]


Дополнение для работы snapshot
Создаём PV который дальше будем подкидывать в template elasticsearch
cat pvc-snapshot.yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: elasticsearch-snapshot-dir
  namespace: elk
  labels:
   app: elasticsearch-snapshot
spec:
  storageClassName: nfs-storageclass
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi

kubectl apply -f pvc-snapshot.yml
смотрим имя созданного PV
kubectl get pv -n elk | grep elk
pvc-30e262ad-770c-45ad-8e3c-28d70a6400ef   5Gi        RWX            Delete           Bound    elk/elasticsearch-snapshot-dir                                                                                              nfs-storageclass              21h

вот наше имя
pvc-30e262ad-770c-45ad-8e3c-28d70a6400ef
теперь правим temaplate
vim elasticsearch/templates/statefulset.yaml

      volumes:
        - name: "pvc-30e262ad-770c-45ad-8e3c-28d70a6400ef"
          persistentVolumeClaim:
            claimName: elasticsearch-snapshot-dir

И в этом же файле в ещё одном месте:
        volumeMounts:
          - name: pvc-30e262ad-770c-45ad-8e3c-28d70a6400ef
            mountPath: "/snapshot"


Всё можно запускать:

[root@prod-vsrv-kubemaster1 helm-charts]# helm install elasticsearch -n elk --values elasticsearch/values.yaml elasticsearch/
[root@prod-vsrv-kubemaster1 helm-charts]# helm install kibana -n elk --values kibana/values.yaml kibana/

ждём когда запустится кибана и переходим по домену указанному в ingress у кибана в переменных
https://elk.prod.test.ru/login


Периодически мы заводим новые индексы в эластик для этого правим logstash чтобы изменения применились выполняем следующую команду:
helm upgrade --install logstash -n elk --values logstash/values.yaml logstash/







# Elastic Stack Kubernetes Helm Charts

[![Build Status](https://img.shields.io/jenkins/s/https/devops-ci.elastic.co/job/elastic+helm-charts+7.9.svg)](https://devops-ci.elastic.co/job/elastic+helm-charts+7.9/)

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Charts](#charts)
- [Supported Configurations](#supported-configurations)
  - [Support Matrix](#support-matrix)
  - [Kubernetes Versions](#kubernetes-versions)
  - [Helm versions](#helm-versions)
    - [Helm 3 beta](#helm-3-beta)
- [ECK](#eck)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


## Charts

These Helm charts are designed to be a lightweight way to configure our official
Docker images. Links to the relevant Docker image documentation has also been
added below.

We recommend that the Helm chart version is aligned to the version of the product
you want to deploy. This will ensure that you using a chart version that has been
tested against the corresponding production version.
This will also ensure that the documentation and examples for the chart will work
with the version of the product you are installing.

For example if you want to deploy an Elasticsearch `7.7.1` cluster, use the
corresponding `7.7.1` [tag][elasticsearch-771].

The `master` version of these charts are intended to support the latest pre-release
versions of our products, and therefore may or may not work with current released
versions.

| Chart                                      | Docker documentation                                                        | Latest 7 Version           | Latest 6 Version            |
|--------------------------------------------|-----------------------------------------------------------------------------|----------------------------|-----------------------------|
| [APM-Server](./apm-server/README.md)       | https://www.elastic.co/guide/en/apm/server/7.9/running-on-docker.html       | [`7.9.3`][apm-7]           | [`6.8.12`][apm-6]           |
| [Elasticsearch](./elasticsearch/README.md) | https://www.elastic.co/guide/en/elasticsearch/reference/7.9/docker.html     | [`7.9.3`][elasticsearch-7] | [`6.8.12`][elasticsearch-6] |
| [Filebeat](./filebeat/README.md)           | https://www.elastic.co/guide/en/beats/filebeat/7.9/running-on-docker.html   | [`7.9.3`][filebeat-7]      | [`6.8.12`][filebeat-6]      |
| [Kibana](./kibana/README.md)               | https://www.elastic.co/guide/en/kibana/7.9/docker.html                      | [`7.9.3`][kibana-7]        | [`6.8.12`][kibana-6]        |
| [Logstash](./logstash/README.md)           | https://www.elastic.co/guide/en/logstash/7.9/docker.html                    | [`7.9.3`][logstash-7]      | [`6.8.12`][logstash-6]      |
| [Metricbeat](./metricbeat/README.md)       | https://www.elastic.co/guide/en/beats/metricbeat/7.9/running-on-docker.html | [`7.9.3`][metricbeat-7]    | [`6.8.12`][metricbeat-6]    |

## Supported Configurations

Starting with the `7.7.0` release, some charts are reaching GA.

Note that only the released charts coming from [Elastic Helm repo][] or
[GitHub releases][] are supported.

### Support Matrix

|     | Elasticsearch | Kibana | Logstash | Filebeat | Metricbeat | APM Server |
|-----|---------------|--------|----------|----------|------------|------------|
| 6.8 | Beta          | Beta   | Beta     | Beta     | Beta       | Alpha      |
| 7.0 | Alpha         | Alpha  | /        | /        | /          | /          |
| 7.1 | Beta          | Beta   | /        | Beta     | /          | /          |
| 7.2 | Beta          | Beta   | /        | Beta     | Beta       | /          |
| 7.3 | Beta          | Beta   | /        | Beta     | Beta       | /          |
| 7.4 | Beta          | Beta   | /        | Beta     | Beta       | /          |
| 7.5 | Beta          | Beta   | Beta     | Beta     | Beta       | Alpha      |
| 7.6 | Beta          | Beta   | Beta     | Beta     | Beta       | Alpha      |
| 7.7 | GA            | GA     | Beta     | GA       | GA         | Beta       |
| 7.8 | GA            | GA     | Beta     | GA       | GA         | Beta       |
| 7.9 | GA            | GA     | Beta     | GA       | GA         | Beta       |

### Kubernetes Versions

The charts are [currently tested][] against all GKE versions available. The
exact versions are defined under `KUBERNETES_VERSIONS` in
[helpers/matrix.yml][].

### Helm versions

While we are checking backward compatibility, the charts are only tested with
Helm version mentioned in [helm-tester Dockerfile][] (currently 2.17.0).

#### Helm 3 beta

While we don't have automated tests for [Helm 3][] yet, we fixed the main
blockers to use it. We now have enough feedbacks from internal and external
users to add support in beta.

## ECK

In addition of these Helm charts, Elastic also provides
[Elastic Cloud on Kubernetes][] which is based on [Operator pattern][] and is
Elastic recommended way to deploy Elasticsearch, Kibana and APM Server on
Kubernetes.


[currently tested]: https://devops-ci.elastic.co/job/elastic+helm-charts+7.9/
[elastic cloud on kubernetes]: https://github.com/elastic/cloud-on-k8s
[elastic helm repo]: https://helm.elastic.co
[github releases]: https://github.com/elastic/helm-charts/releases
[helm 3]: https://v3.helm.sh
[helm-tester Dockerfile]: https://github.com/elastic/helm-charts/blob/7.9/helpers/helm-tester/Dockerfile
[helpers/matrix.yml]: https://github.com/elastic/helm-charts/blob/7.9/helpers/matrix.yml
[operator pattern]: https://kubernetes.io/docs/concepts/extend-kubernetes/operator/
[elasticsearch-771]: https://github.com/elastic/helm-charts/tree/7.7.1/elasticsearch/

[apm-7]: https://github.com/elastic/helm-charts/tree/7.9.3/apm-server/README.md
[apm-6]: https://github.com/elastic/helm-charts/tree/6.8.12/apm-server/README.md
[elasticsearch-7]: https://github.com/elastic/helm-charts/tree/7.9.3/elasticsearch/README.md
[elasticsearch-6]: https://github.com/elastic/helm-charts/tree/6.8.12/elasticsearch/README.md
[filebeat-7]: https://github.com/elastic/helm-charts/tree/7.9.3/filebeat/README.md
[filebeat-6]: https://github.com/elastic/helm-charts/tree/6.8.12/filebeat/README.md
[kibana-7]: https://github.com/elastic/helm-charts/tree/7.9.3/kibana/README.md
[kibana-6]: https://github.com/elastic/helm-charts/tree/6.8.12/kibana/README.md
[logstash-7]: https://github.com/elastic/helm-charts/tree/7.9.3/logstash/README.md
[logstash-6]: https://github.com/elastic/helm-charts/tree/6.8.12/logstash/README.md
[metricbeat-7]: https://github.com/elastic/helm-charts/tree/7.9.3/metricbeat/README.md
[metricbeat-6]: https://github.com/elastic/helm-charts/tree/6.8.12/metricbeat/README.md
