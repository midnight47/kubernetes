# k8s-ad-auth

Authorisation in k8s based on AD accounts. Used stack:  gangway+dex

Настройка аутентификации пользователей из AD
Все работы проводим с первого мастера под root'ом

Генерация сертификатов
1. Создаем корневой сертификат

openssl genrsa -out ca.key 4096
openssl req -x509 -sha256 -new -key ca.key -days 10000 -out ca.crt
2. Во второй команде отвечаем на вопросы

Можно просто все оставить по default'у

openssl genrsa -out dex.key 4096
openssl req -new -key dex.key -out dex.csr
Во второй команде отвечаем на вопросы.

Главное — ответить на вопрос Common Name (eg, YOUR name) []:

Остальное можно оставить по default'у

Тут нужно вписать имя хоста, для которого генерим сертификат. 
Например:  dex.prod.test.ru
4. И подписываем запрос корневым сертификатом

openssl x509 -sha256 -req -in dex.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out dex.crt -days 5000
5. После этого создаем секрет для ingress'a с полученным сертификатом

kubectl create secret tls dex-tls --key dex.key --cert dex.crt --namespace kube-system

Повторяем для Gangway
1. Создаем запрос на сертификат

openssl genrsa -out gangway.key 4096
openssl req -new -key gangway.key -out gangway.csr
2. Во второй команде отвечаем на вопросы.

Главное — ответить на вопрос Common Name (eg, YOUR name) []:

Остальное можно оставить по default'у

Тут нужно вписать имя хоста, для которого генерим сертификат. 
Например:  gangway.prod.test.ru
3. И подписываем запрос корневым сертификатом

openssl x509 -sha256 -req -in gangway.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out gangway.crt -days 5000
4. После этого создаем секрет для ingress'a с полученным сертификатом

kubectl create secret tls gangway-tls --key gangway.key --cert gangway.crt --namespace kube-system
5. Далее создаем секрет с корневым сертификатом для Gangway

kubectl create secret generic ca --from-file ca.crt --namespace kube-system
И складываем этот же сертификат в /etc/ssl/certs/ca.crt на всех трех мастерах

Запуск Dex и Gangway
1. Переходим в директорию с yaml манифестами

cd  auth_in_ad/

2. Создаем секрет для Gangway

kubectl -n kube-system create secret generic gangway-key --from-literal=sesssionkey=$(openssl rand -base64 32)
3. И применяем все манифесты

kubectl apply -f . -n kube-system
4. Проверяем, что все pod'ы запустились

kubectl get po -n kube-system
5. Обновляем конфигурацию kube-api
Если кластер ставился с помощью kubeadm:

kubectl edit configmap --namespace=kube-system kubeadm-config
6. И добавляем

apiServer:
  extraArgs:
    authorization-mode: Node,RBAC
# вот отсюда
    oidc-ca-file: /etc/ssl/certs/ca.crt
    oidc-client-id: oidc-auth-client
    oidc-groups-claim: groups
    oidc-issuer-url: https://dex.prod.test.ru/
    oidc-username-claim: email
7. Далее на каждом из трех мастеров выполняем

kubeadm upgrade apply <k8s version> -y

Если кластер ставился через kuberspray от SB или любым другим способом отличным от kubeadm:
правим /etc/kubernetes/manifests/kube-apiserver.manifest  \
Добавляем после    - --authorization-mode=Node,RBAC:
    - --oidc-ca-file=/etc/ssl/certs/ca.crt
    - --oidc-client-id=oidc-auth-client
    - --oidc-groups-claim=groups
    - --oidc-issuer-url=https://dex.prod.test.ru/
    - --oidc-username-claim=email


8. Проверяем, как все работает.

Открываем в браузере gangway.prod.test.ru

Открывайте приложение в режиме "инкогнито", т.к. в обычном режиме браузер блокирует доступ по http. В Google Chrome к примеру, можно запустить режим с помощью комбинации "Ctrl+Shift+N".

Авторизуемся, получаем инструкции для настройки доступа к кластеру.

Единственное ограничение — так как у нас все сертификаты самоподписанные, то в предоставленной инструкции по адресу kubectl.s<Ваш номер логина>.edu.slurm.io нужно убрать сертификат сервера и вместо него вписать insecure-skip-tls-verify: true

Эту строку

kubectl config set-cluster Slurm.io --server=https://<api_server_address>:6443 --certificate-authority=ca-Slurm.io.pem --embed-certs
Заменить на эту

kubectl config set-cluster Slurm.io --server=https://<api_server_address>:6443 --insecure-skip-tls-verify=true
