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

Наблюдать за состоянием объектов до тех пор, пока под `cnpg-1` не перейдет в состояние Running.

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

Скорректировать IP адрес Применить манифест для публикации CoreDNS командой `kubectl apply -f coredns-svc.yaml`

> ВАЖНО! При необходимости в файле `coredns-svc.yaml` надо скорректировать IP-адрес.

Проверить, что появился сервис LoadBalancer с EXTERNAL-IP равным 172.23.240.5 командой `kubectl get svc -o wide -n kube-system`

Вывод должен быть примерно таким:

```plain
NAME       TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                     AGE   SELECTOR
ext-dns    LoadBalancer   10.96.155.151   172.23.240.5   53:32496/UDP,53:32496/TCP   82s   k8s-app=kube-dns
kube-dns   ClusterIP      10.96.0.10      <none>         53/UDP,53/TCP,9153/TCP      46h   k8s-app=kube-dns
```

## resolvectl

Найти имя сетевого интефейса, на котором работает kind. Это можно сделать командой `ip a | grep 172.23`.
В моем случае имя `br-4c5879dd94b7`, но у вас оно будет другим.

Установить для этого интерфейса настройки DNS на хосте:

```bash
sudo resolvectl dns <interface> 172.23.240.5
sudo resolvectl domain <interface> '~cluster.local'
```

Проверить работоспобность можно командой `nslookup server1c-01-lb.default.cluster.local`.
Если все корректно, то в ответе должен быть указан **внешний** IP-адрес LoadBalancer-а:

```plain
Non-authoritative answer:
Name:   server1c-01-lb.default.cluster.local
Address: 172.23.240.100
```

Продолжение следует!
