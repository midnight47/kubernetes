После установки через kubespray настроим dashboard


[root@prod-vsrv-kubemaster1 ~]# kubectl get deployments.apps -A | grep -i dashboard
kube-system     kubernetes-dashboard                  1/1     1            1           177d

Правим:root@prod-vsrv-kubemaster1 ~]# kubectl -n kube-system edit deployments.apps kubernetes-dashboard
В перечислении agrs добавляем ­ --authentication-mode=basic

Находим 

  template:
    metadata:
      creationTimestamp: null
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
      - args:
        - --namespace=kube-system
        - --auto-generate-certificates
        - --authentication-mode=basic
        - --authentication-mode=token
        - --token-ttl=900


Можем поправить сразу тут:
/etc/kubernetes/dashboard.yml



Создаём ingress (он есть в этом git)
[root@prod-vsrv-kubemaster1 ~]# cat ingress-dashboard.yml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: dashboard
  annotations:
    nginx.ingress.kubernetes.io/secure-backends: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
spec:
  rules:
  - host: dashboard.prod.test.ru
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443


применяем его:
kubectl apply -f  ingress-dashboard.yml


На всех мастерах создаём файл для аутентификации
mkdir /etc/kubernetes/basic-user-auth/
cat /etc/kubernetes/basic-user-auth/users.csv
UeALRowpoh12Dpassword123,kube,admin,"system:masters"

На всех мастерах правим файл:im /etc/kubernetes/manifests/kube-apiserver.manifest а именно:

Добавляем - --basic-auth-file=spec:
  containers:
  - name: kube-apiserver
    resources:
    command:
    - /hyperkube
    - kube-apiserver
    - --basic-auth-file=/etc/kubernetes/basic-user-auth/users.csv


Добавляем:
    volumeMounts:
    - mountPath: /etc/kubernetes/basic-user-auth
      name: basic-auth-config
  volumes:
  - hostPath:
      path: /etc/kubernetes/basic-user-auth
      type: ""
    name: basic-auth-config


Всё готово, ничего перезапускать не нужно.

Панель будет доступна по адресу:
https://dashboard.prod.test.ru/



файлы которые правим и создаём есть в этом репозитории.
