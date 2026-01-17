# Запуск 1С в Kubernetes

## Создание кластера kind

```bash
kind create cluster --config kind.yml -n hello-world-1c
```

## Загрузка образа в kind

Как создать образ см. [тут](../02-1c-docker-image).

```bash
kind load docker-image "onec-server:8.3.27.1936" -n hello-world-1c
```

## Установка MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
```

## Настройка MetalLB

Нужно узнать диапазон IP-адресов, который будет использовать MetalLB.

Выясняем, какой INTERNAL-IP получила нода kind:

```bash
kubectl get nodes -o wide
```

В моем случае IP-адрес ноды `172.23.0.4`.
Так как это сеть докера, надо выяснить ее параметры:

```bash
ip a | grep 172.23
```

Видим в консоли что-то вроде такого:

```bash
inet 172.23.0.1/16 brd 172.23.255.255 scope global br-4c5879dd94b7
```

Это значит, что MetalLB должен раздавать адреса в диапазоне `172.23.0.1/16`.

В файле `metallb.yaml` определено два диапазона:

1) `172.23.240.100-172.23.240.200` для серверов 1С, этого более чем достаточно.
2) `172.23.240.5-172.23.240.5` для DNS.

> ВАЖНО! Необходимо скорректировать адреса в `metallb.yaml` согласно вашей сети docker.

```bash
kubectl apply -f metallb.yaml
```

## cnpg

Установить оператор по [этой](https://cloudnative-pg.io/docs/1.28/installation_upgrade/#directly-using-the-operator-manifest) инструкции из официальной документации.

Применить манифест с кластером postgres:

```bash
kubectl apply -f cnpg.yaml
```

Наблюдать за состоянием объектов до тех пор, пока у пода `cnpg-1` в колонке `STATUS` не появится `Running`, а значение в колонке `READY` не станет равным `1/1`.

```bash
kubectl get all
```

Вывод должен быть примерно таким:

```plain
NAME         READY   STATUS    RESTARTS   AGE
pod/cnpg-1   1/1     Running   0          68s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/cnpg-r       ClusterIP   10.96.101.158   <none>        5432/TCP   78s
service/cnpg-ro      ClusterIP   10.96.246.175   <none>        5432/TCP   78s
service/cnpg-rw      ClusterIP   10.96.80.161    <none>        5432/TCP   78s
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP    23h
```

## CoreDNS

Открыть для изменения ConfigMap от CoreDNS командой `kubectl edit cm -n kube-system coredns`

Добавить блок k8s_external, так, чтобы получилось следующее:

```yaml
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           <что-то там>
        }
        k8s_external cluster.local {
          headless
          fallthrough
        }
```

Сохранить изменения и перезапустить CoreDNS командой `kubectl rollout restart deployment coredns -n kube-system`

Применить манифест для публикации CoreDNS командой `kubectl apply -f coredns-svc.yaml`

> ВАЖНО! При необходимости в файле `coredns-svc.yaml` надо скорректировать IP-адрес.

Проверить, что появился сервис LoadBalancer с EXTERNAL-IP равным 172.23.240.5 командой `kubectl get svc -o wide -n kube-system`

Вывод должен быть примерно таким:

```plain
NAME       TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                     AGE   SELECTOR
ext-dns    LoadBalancer   10.96.155.151   172.23.240.5   53:32496/UDP,53:32496/TCP   82s   k8s-app=kube-dns
kube-dns   ClusterIP      10.96.0.10      <none>         53/UDP,53/TCP,9153/TCP      46h   k8s-app=kube-dns
```
## Платформа 1С

Развернуть серверы 1С командой `kubectl apply -f cluster1c.yaml`

> ВАЖНО! Текущий вариант на самом деле разворачивает два разных кластера 1С. Сборка кластера из нескольких серверов будет реализована в следующих лабораторных.

Если все прошло корректно, то команда `kubectl get all` должна иметь такой вывод (список портов сокращен мной вручную для ясности):

```plain
NAME                READY   STATUS    RESTARTS   AGE
pod/cnpg-1          1/1     Running   0          47h
pod/server1c-01-0   1/1     Running   0          24h
pod/server1c-02-0   1/1     Running   0          24h

NAME                     TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                                             AGE
service/cnpg-r           ClusterIP      10.96.101.158   <none>           5432/TCP                                            47h
service/cnpg-ro          ClusterIP      10.96.246.175   <none>           5432/TCP                                            47h
service/cnpg-rw          ClusterIP      10.96.80.161    <none>           5432/TCP                                            47h
service/kubernetes       ClusterIP      10.96.0.1       <none>           443/TCP                                             2d23h
service/server1c-01      ClusterIP      None            <none>           1540/TCP,1541/TCP,1545/TCP,1560-1591/TCP,5900/TCP   24h
service/server1c-01-0   LoadBalancer   10.96.194.182   172.23.240.100   1540/TCP,1541/TCP,1545/TCP,1560-1591/TCP,5900/TCP   24h
service/server1c-02      ClusterIP      None            <none>           1540/TCP,1541/TCP,1545/TCP,1560-1591/TCP            24h
service/server1c-02-0   LoadBalancer   10.96.156.220   172.23.240.101   1540/TCP,1541/TCP,1545/TCP,1560-1591/TCP            24h

NAME                           READY   AGE
statefulset.apps/server1c-01   1/1     24h
statefulset.apps/server1c-02   1/1     24h
```

На что обратить особое внимание:

- все pod-ы должны находиться в состоянии RUNNING
- у сервисов типа LoadBalancer должен быть заполнен EXTERNAL-IP

## resolvectl

Найти имя сетевого интефейса, на котором работает kind. Это можно сделать командой `ip a | grep 172.23`.
В моем случае имя `br-4c5879dd94b7`, но у вас оно будет другим.

Установить для этого интерфейса настройки DNS на хосте:

```bash
sudo resolvectl dns <interface> 172.23.240.5
sudo resolvectl domain <interface> '~cluster.local'
```

Проверить работоспобность CoreDNS можно командой `nslookup server1c-01-0.default.cluster.local`.
Если все корректно, то в ответе должен быть указан **внешний** IP-адрес LoadBalancer-а:

```plain
Non-authoritative answer:
Name:   server1c-01-0.default.cluster.local
Address: 172.23.240.100
```

ВАЖНО! Эти настройки сбросятся после перезагрузки. Для того, чтобы они действовали постоянно, нужно внести их в netplan:

```yaml
network:
  ethernets:
    br-423f48228513:
      dhcp4: true
      nameservers:
        addresses: [172.23.240.5]
        search: [~cluster.local]
      dhcp4-overrides:
        use-dns: true
```

После внесения изменений надо выполнить команду `netplan apply`

## Настройка кластера

Кластер создается автоматически при запуске контейнера, в параметр `agent-host` передается имя хоста. Это же имя хоста будет передано тонкому или толстому клиенту при начале работы с информационной базой. А это соврешенно неправильно, потому что клиент находится снаружи кластера Kubernetes и не может обращаться к узлам кластера 1С по именам хостов.

Имя хоста необходимо изменить. Для этого требуется удалить текущий кластер и создать новый.

> Если на локальной машине установлены серверные компоненты платформы, то можно работать через локальный rac, например так:
> `rac server1c-01-0.default.cluster.local:1545 cluster list`

Узнать uuid текущего кластера:

```bash
kubectl exec -it server1c-01-0 -- /opt/1cv8/current/rac localhost:1545 cluster list
```

Удалить кластер:

```bash
kubectl exec -it server1c-01-0 -- /opt/1cv8/current/rac localhost:1545 cluster remove --cluster=<uuid>
```

Добавить кластер с правильным именем хоста:

```bash
kubectl exec -it server1c-01-0 -- /opt/1cv8/current/rac localhost:1545 cluster insert --host "server1c-01-0.default.cluster.local" --port 1541
```

Запомнить его uuid.

## Информационная база

Создавать информационную базу придется из пода, потому что платформа 1С устроена так, что при интерактивном добавлении новой ИБ хост СУБД должен резолвиться с локального хоста, а развернутый в кластере Kubernetes cloudnative-pg не выставлен наружу (это и не планировалось).

Однако, требуется получить логин и пароль от Postgres. Для этого на локальной машине надо выполнить команды:

```bash
kubectl get secret cnpg-app -o jsonpath='{.data.user}' | base64 -d
kubectl get secret cnpg-app -o jsonpath='{.data.password}' | base64 -d
```

Добавить новую информационную базу с помощью команды ниже.

> ВАЖНО! Необходимо установить параметры `uuid` и `pwd`.

```bash
kubectl exec -it server1c-01-0 -- /opt/1cv8/current/rac infobase create \
  --cluster=<uuid> \
  --create-database \
  --name=cn1c-test \
  --dbms=PostgreSQL \
  --db-server=cnpg-rw \
  --db-name=cn1c-test \
  --locale=ru \
  --db-user=app \
  --db-pwd=<pwd> \
  --license-distribution=allow \
  --scheduled-jobs-deny=on
```

Добавить базу в список баз на локальной машине, параметры ниже:

- Кластер серверов 1С:Предприятия: `server1c-01-0.default.cluster.local`
- Имя информационной базы в кластере: `cn1c-test`

## Активация comminuty-лицензии 1С на сервере 1С

```bash
kubectl exec -it server1c-01-0 -- /bin/bash
```

Затем:

```bash
echo "deb http://deb.debian.org/debian bullseye main" > /etc/apt/sources.list
apt update
apt install -y libcups2 libgtk-3-0 libwebkit2gtk-4.0-37 libglu1-mesa xvfb x11vnc
Xvfb :99 -screen 0 1024x768x24 &
x11vnc -storepasswd "12345678" /etc/vncpasswd
mkdir -p /var/log/x11vnc
/usr/bin/x11vnc -rfbport 5900 -display :99 -forever -o /var/log/x11vnc/x11vnc.log -rfbauth /etc/vncpasswd -xkb -ncache 10 -bg -noxrecord -noxfixes -noxdamage -nomodtweak &
su - usr1cv8
DISPLAY=:99.0 /opt/1cv8/current/1cv8
```

Установить на хост клиент VNC

```bash
apt install -y remmina
```

Открыть Remmina и подключиться по протоколу `VNC` к `server1c-01-0.default.cluster.local`

Далее необходимо активировать комьюнити-лицензию (потребуется логин и пароль от учетной записи ИТС).

## Активация comminuty-лицензии 1С на клиенте

Аналогично необходимо активировать comminuty-лицензию на клиенте.

После активации с созданной в Kubernetes информационной базой можно работать как в режиме Конфигуратора, так и в режиме Предприятия.
