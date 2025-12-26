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

Продолжение следует!
