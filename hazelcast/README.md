данная инструкция описывает установку hazelcast, синтаксис может криво отображаться, поэтому смотрим в wiki


За основу взят  helm chart https://github.com/hazelcast/charts/tree/master/stable/hazelcast

Добавляем helm репозиторий и выкачиваем содержимое чарта:

helm repo add hazelcast https://hazelcast-charts.s3.amazonaws.com/
helm repo update
helm pull hazelcast/hazelcast
tar zxvf hazelcast-3.6.1.tgz
rm -f hazelcast-3.6.1.tgz

Переходим в разархивированную директорию и правим следующие блоки параметров в файле values.yaml:
### Persistence включается только для веб-панели. 
 # Management Center persistence properties
  persistence:
    # enabled is a flag to enable persistence for Management Center
    enabled: true
    # existingClaim is a name of the existing Persistence Volume Claim that will be used for persistence
    # if not defined, a new Persistent Value Claim is created with the default name
    # existingClaim:
    # accessModes defines the access modes for the created Persistent Volume Claim
    accessModes:
    - ReadWriteOnce
    # size is the size of Persistent Volume Claim
    size: 8Gi
    # storageClass defines the storage class used for Management Center. Read more at: https://kubernetes.io/docs/concepts/storage/storage-classes/
    storageClass: "nfs-storageclass"

----
  # ingress configuration for mancenter
  ingress:
    enabled: true
    annotations: {}
    hosts:
     - hazelcastmgr.prod.test.local
    # tls:
    # - secretName: hazelcast-ingress-tls
    #   hosts:
    #   - hazelcast-mancenter.cluster.domain

---
# Configure resource requests and limits for hazelcast
# ref: http://kubernetes.io/docs/user-guide/compute-resources/

resources:
   requests:
     memory: 256Mi
     cpu: 100m
   limits:
     memory: 1024Mi
     cpu: 200m

---
  # Configure resource requests and limits for mancenter. 
  # ref: http://kubernetes.io/docs/user-guide/compute-resources/
  #
resources:
   requests:
     memory: 1024Mi
     cpu: 300m
   limits:
     memory:  2048Mi
     cpu: 500m

Создаем отдельный namespace:

Kubectl create ns hazelcast

Переходим в директорию с распакованным чартом и устанавливаем его:

helm upgrade --install hazelcast ./ -f ./values.yaml -n hazelcast


Переходим в браузере по адресу  http://hazelcastmgr.prod.test.local. 

При первом запуске будет необходимо  создать учетную запись администратора. 

Логинимся под вновь  созданной учёткой и проверяем состояние ксластера. 

Если все ok - закончили упражнение 

