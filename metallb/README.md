

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.9/config/manifests/metallb-native.yaml

вносим правки то что ниже копипастим 

kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system

В файл ip-pool.yaml добавляем или диапазон IP адресов, или 1 IP указав его подсеть /32 
я добавил 192.168.1.191/32

cd /etc/ansible/kubespray-official/metallb 
kubectl apply -f ip-pool.yaml

Добавляем сервис для ingres-controller типа LoadBalancer

kubectl apply -f lb-ingress-controller-svc.yaml

Проверяем:

kubectl get pod -n metallb-system
NAME                         READY   STATUS    RESTARTS   AGE
controller-c6c466d64-5n9df   1/1     Running   0          108m
speaker-77sfq                1/1     Running   0          108m
speaker-8d6kd                1/1     Running   0          108m
speaker-czv52                1/1     Running   0          108m
speaker-h9tf4                1/1     Running   0          108m
speaker-jnvlf                1/1     Running   0          108m


kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.233.21.197   192.168.1.191   80:32276/TCP,443:30354/TCP   36m
ingress-nginx-controller-admission   ClusterIP      10.233.0.95     <none>          443/TCP                      36m


как видим IP 192.168.1.191 является EXTERNAL-IP

проверим работу

helm create test
меняем ingress False на true
vim test/values.yaml


включаем ingress  и ставим classname равным nginx:
ingress:
  enabled: true
  className: "nginx"

ставим:
helm upgrade --install test ./test/ 

добавим в /etc/hosts
192.168.1.191 chart-example.local

проверяем:

kubectl get ingress
NAME   CLASS   HOSTS                 ADDRESS         PORTS   AGE
test   nginx   chart-example.local   192.168.1.191   80      6h12m

curl -I chart-example.local
HTTP/1.1 200 OK
Date: Sun, 12 Mar 2023 01:35:37 GMT
Content-Type: text/html
Content-Length: 612
Connection: keep-alive
Last-Modified: Tue, 23 Apr 2019 10:18:21 GMT
ETag: "5cbee66d-264"
Accept-Ranges: bytes


как видим всё ок
