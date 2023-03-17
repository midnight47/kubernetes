настраиваем NFS provision

подготоавливаем новый сервер в моём случае это 
192,168,1,180

правим hosts 
root@master1:/etc/ansible# cat hosts

[all_servers]
192.168.1.171
192.168.1.172
192.168.1.173
192.168.1.174
192.168.1.175
192.168.1.180

root@master1:/etc/ansible# ansible-playbook -u root  playbooks/roles_play/new_server.yml --ask-pass

теперь установим NFS сервер и клиенты:
root@master1:/etc/ansible# cat hosts
[nfs:children]
nfsmaster
nfsclient
[nfsmaster]
192.168.1.180
[nfsclient]
192.168.1.171
192.168.1.172
192.168.1.173
192.168.1.174
192.168.1.175

ставим nfs 
root@master1:/etc/ansible# ansible-playbook -u root  playbooks/roles_play/nfs.yml --ask-pass

переходим в 
root@master1:/etc/ansible# cd kubespray-official/nfs-provision/
и правим файл nfs_provision.yaml
там меняем IP адресс 192.168.1.180 на тот который нам нужен и применяем данные файлы:
root@master1:/etc/ansible/kubespray-official/nfs-provision# kubectl apply -f rbac.yaml -f nfs_class.yaml -f nfs_provision.yaml

всё теперь можно проверить работает ли PVC или нет, запускаем файл:
root@master1:/etc/ansible/kubespray-official/nfs-provision# kubectl apply -f test-claim.yaml

проверяем:

root@master1:/etc/ansible/kubespray-official/nfs-provision# kubectl get pvc
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
test-claim   Bound    pvc-081f8e52-831b-4b08-a33d-299c4d7071fa   50Mi       RWX            nfs-client     7m58s

root@master1:/etc/ansible/kubespray-official/nfs-provision# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                STORAGECLASS   REASON   AGE
pvc-081f8e52-831b-4b08-a33d-299c4d7071fa   50Mi       RWX            Delete           Bound    default/test-claim   nfs-client              8m1s

смотрим на nfs сервере:
root@nfs:~# ls -lah /nfs/
total 12K
drwxr-xr-x  3 root root 4.0K Mar 13 13:09 .
drwxr-xr-x 19 root root 4.0K Mar 13 12:19 ..
drwxrwxrwx  2 root root 4.0K Mar 13 13:09 default-test-claim-pvc-081f8e52-831b-4b08-a33d-299c4d7071fa

всё ок пашет.

