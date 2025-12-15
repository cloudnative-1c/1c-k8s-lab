# Установка кластера kind

## Очистка окружения

Выполнять только если кластер с наименованием `hello-world` создавался ранее и надо вернуться в исходное состояние.

```bash
kind delete cluster -n hello-world
```

## Создание кластера

```bash
kind create cluster --config kind.yaml -n hello-world
```

Команда должна выполниться без ошибок, в конце должно быть сообщение:

## Проверка доступа к кластеру

```bash
kubectl cluster-info
```

Вывод команды должен быть таким:

```console
Kubernetes control plane is running at https://127.0.0.1:36895
CoreDNS is running at https://127.0.0.1:36895/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

## Проверка контейнера kind

```bash
docker ps | grep kind
```

В выводе должна быть одна строка с контейнером `hello-world-control-plane`.

## Деплой приложения

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

## Проверка приложения

```bash
kubectl get deploy
kubectl get svc
```

Команда `kubectl get deploy` должна вывести что-то вроде этого:

```plain
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
hello-world   2/2     2            2           13s
```

Обратите внимание на READY 2/2 и AVAILABLE 2

## Форвард порта

Открыть **новое окно** терминала и выполнить команду:

```bash
kubectl port-forward service/hello-world 8080:80
```

Теперь можно открыть браузер по адресу `http://localhost:8080` или выполнить команду `curl http://localhost:8080`.

Поздравляю! 🎉 Вы разместили приложение в Kubernetes, а заодно проверили, что kind работает как надо!
