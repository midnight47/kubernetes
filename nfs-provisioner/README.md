Перед запуском надо подготовить nfs сервер.
После чего можно запускать:

kubectl apply -f rbac.yaml
kubectl apply -f nfs_class.yaml

в файле правим IP 10.242.146.21 (это наш nfs сервер) директория /nfs/prod_vsrv_kubernetes добавлена на nfs сервере и к ней разрешён доступ кластера kubernetes,  
как только всё поправили можно запускать:
kubectl apply -f nfs_provision.yaml

для примера есть ещё один файл: pvc.yaml  в нём указано как создавать запросы pvc 
