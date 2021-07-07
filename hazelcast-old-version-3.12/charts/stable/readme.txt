это старая версия hazelcast 
git clone https://github.com/hazelcast/charts/tree/3.12.z
потом
git checkout 3.12.z
cd charts/stable/
vim hazelcast/values.yaml

  ingress:
    enabled: true
    annotations: {}
    hosts:
     - hazelcastmgr.prod.test.ru

  persistence:
    enabled: true
    accessModes:
    - ReadWriteOnce
    size: 8Gi
    storageClass: "nfs-storageclass"


запускаем
[root@prod-vsrv-kubemaster1 stable]# helm upgrade --install hazelcast hazelcast/ --values hazelcast/values.yaml -n hazelcast

