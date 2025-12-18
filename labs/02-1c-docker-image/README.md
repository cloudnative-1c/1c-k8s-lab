# Сборка образа с сервером 1С:Предприятие

## Начальные требования

- Linux
- установленный Docker Engine, см. [00-environment/](../00-environment/README.md#установка-docker)
- логин и пароль от сайта ИТС с доступом к releases.1c.ru

## Сборка образа

- клонировать репозиторий [onec-docker](https://github.com/firstBitMarksistskaya/onec-docker)
- прочитать раздел `Использование` и выполнить все, что там указано с учетом пунктов ниже:
    - версию платформы можно указать произвольную, но желательно взять какую-то сборку 8.3.27, например, `8.3.27.1936`
    - удалить из `.onec.env` переменные `DOCKER_LOGIN`, `DOCKER_PASSWORD` и `DOCKER_REGISTRY_URL` (или оставить все значения пустыми)
- запустить `./build-server.sh`
- выполнить команду `docker image ls` и убедиться, что образ `onec-server` с тегом `8.3.27.1936` есть в списке

## Проверка (запуск образа в Docker)

Запустить контейнер

```bash
docker run -d --rm --name "onec-server" onec-server:8.3.27.1936
```

Выполнить `docker ps` и убедиться, что контейнер запущен. В списке должна быть такая запись:

```
CONTAINER ID   IMAGE                                        COMMAND                  CREATED         STATUS         PORTS                                                                                                                                           NAMES
442a8c70c7f3   onec-server:8.3.27.1936                      "/usr/local/bin/dock…"   3 seconds ago   Up 2 seconds   1540-1541/tcp, 1545/tcp, 1560/tcp                                                                                                               onec-server
```

Залогиниться в контейнер командой `docker exec -it onec-server /bin/bash`

Вывести запущенные процессы кластера 1С:
- `ps aux | grep -E "ragent|rmngr|rphost"`

Обратить внимание, что процесс ragent запущен с PID 1.

Вывести тех. журнал:
- `cat /var/log/1C/**/*.log`

> На самом деле заходить для этого внутрь контейнера не обязательно, но иногда это просто удобнее.

Похоже, что сервер 1С запущен, все системы работают в норме. Образ с сервером 1С готов!
