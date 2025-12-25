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

Продолжение следует!
