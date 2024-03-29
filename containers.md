# Памятка по работе с контейнерами на примере Podman
Пример команды запуска контейнера:

```sh 
$ podman run --interactive --tty \
    --name mycontainer \
    --volume /tmp/.X11-unix:/tmp/.X11-unix \
    --env DISPLAY --device /dev/dri --device /dev/snd --device /dev/input \
    --volume /etc/localtime:/etc/localtime:ro \
    --volume /sysroot/home/$(whoami)/sharing:/mnt ubuntu:latest
```

Пояснение параметров:
* **run** - Создать контейнер и запустить его немедленно
* **–interactive** - Открыть терминал в только что созданном контейнере
* **–tty** - Создать новый терминал внутри контейнера, т.е открыть потоки stdin/stdout
* **–name mycontainer** - Дать контейнеру требуемое имя
* **–volume /tmp/.X11-unix:/tmp/.X11-unix** - Отмапить X11 сокет для доступа к иксам из контейнера
* **–env DISPLAY** - Задать переменную DISPLAY для сессии
* **–device /dev/dri** - Даёт доступ к 3Д ускорителям
* **–device /dev/snd** - Даёт доступ к звуковым устройствам
* **–device /dev/input** - Даёт доступ к устройствам ввода, например, к геймпадам
* **–volume /etc/localtime:/etc/localtime:ro** Отмапить /etc/localtime внутрь контейнера, чтобы получить время как на хосте
* **–volume /home/$(whoami)/sharing:/mnt** - Отмапить папку ~/share в /mnt внутри контейнера
* **ubuntu:latest** - Используемый образ

Дополнительные часто используемые параметры:
* **-privileged** - Запуск контейнера без дополнительных блокировок ("security" lockdown). Позволяет, например, использовать ping из контейнера.
* **-P** - Проброс всех открытых портов в хост
* **-p ip:host_port:cont_port** - проброс определенного порта в хост

**Примечание**: _В случае с Podman, запуск контейнера в этом режиме не означает, что контейнер получит больше привилегий, чем пользователь, запустивший процесс._

**Примечание**: _"The bottom line is that using the --privileged flag does not tell the container engines to add additional security constraints. The --privileged flag does not add any privilege over what the processes launching the containers have. Tools like Podman and Buildah do NOT give any additional access beyond the processes launched by the user."_
 
**Примечание**: _Docker запускается как демон, поэтому для него всё может быть иначе_




## Часто используемые команды движка Podman
### top
Позволяет посмотреть список запущенных контейнерами процессов. Требует ID контейнера.

```sh
$ podman top --latest
USER   PID   PPID   %CPU    ELAPSED           TTY     TIME   COMMAND
root   1     0      0.000   20m6.220534216s   pts/0   0s     /bin/bash 
root   305   1      0.000   3m35.220609764s   pts/0   0s     htop 
```

Описание параметров:
* **-latest** - последний запущенный контейнер. 


Списком параметров можно управлять
```sh
$ podman top --latest pid seccomp args %C
PID   SECCOMP    COMMAND      %CPU
1     disabled   /bin/bash    0.000
305   disabled   htop         0.000
```


### container
Управление контейнерами

```sh
$ podman container list -a
CONTAINER ID  IMAGE          COMMAND    CREATED         STATUS                       PORTS   NAMES
c73947e94d52  ubuntu:latest  /bin/bash  7 minutes ago   Exited (126) 21 seconds ago          mycontainer
```


Получить список запущенных контейнеров для пользователя можно с помощью команды:
```sh
$ podman ps --all
CONTAINER ID  IMAGE          COMMAND    CREATED         STATUS                       PORTS   NAMES
c75811b4d26e  ubuntu         /bin/bash  29 minutes ago  Exited (0) 29 minutes ago            mystifying_dewdney
c5cc51791620  ubuntu:latest  /bin/bash  9 minutes ago   Up 9 minutes ago                     mycontainer
```

### export/import
Экспортирование/импортирование образа файловой системы контейнера. Из получнного образа можно создать контейнер на другой машине. 

**Примечание**: _Podman не умеет делать чекпоинты "non root"-контейнеров. export/import частичная альтернатива_

```sh
podman export -o myubi.tar a6a6d4896142
```

```sh
$ podman import myubi.tar myubi-imported
Getting image source signatures
Copying blob 277cab30fe96 done
Copying config c296689a17 done
Writing manifest to image destination
Storing signatures
c296689a17da2f33bf9d16071911636d7ce4d63f329741db679c3f41537e7cbf
```

```sh
$ podman images
REPOSITORY                              TAG     IMAGE ID      CREATED         SIZE
docker.io/library/myubi-imported       latest  c296689a17da  51 seconds ago  211 MB
```

### exec
Выполняет команду в запущенном контейнере.

Подключиться к контейнеру как root:
```sh
$ podman exec --interactive --tty mycontainer /bin/bash
```

Подключиться к контейнеру как user:
```sh
$ podman exec --interactive --tty --user $(whoami) --workdir /home/$(whoami) mycontainer /bin/bash
```

### pull
Загрузка образов. Например, загрузка с docker hub.
```sh
podman pull docker.io/library/debian:buster-slim
```

## Troubleshooting
### Запуск GUI-приложений в контейнере
Попытка запуска приложений с графическим интерфейсомв контейнере может привести к следующей проблеме:
```sh
root@c5cc51791620:/# galculator 
No protocol specified
Unable to init server: Could not connect: Connection refused

(galculator:15290): Gtk-WARNING **: 21:46:06.256: cannot open display: :0
```

Одна из причин, по которой это может произойти состоит в том, что X-сервер хоста запрещает внешние подключение. Самый простой способ их разрешить:
```sh
$ xhost +
access control disabled, clients can connect from any host
```

### Проброс портов из контейнера
Контейнеры могут получаться к внешнему миру без какой-либо настройки, но получить сетевой доступ к контейнеру снаружи без настройки невозможно. Для этого необходимо пробросить порты контейнера в хост, далее два основных способа это сделать:

Пробросить все порты, предоставленные в Dockerfile или добавленные при сборке:
```sh
$ podman container run -P nginx
```
Каждый открытый порт будет напрямую привязан к случайному порту хоста. Список проброшенных портов можно посмотреть:
```sh
$ podman container port nginx
80/tcp -> 127.0.0.1:8000
```

Пробрасывать все порты может быть не очень хорошей идеей. Альтернативный вариант - проброс определенного порта:
```sh
$ podman container run -p 127.0.0.1:8000:80 nginx
```
По-умолчанию контейнерные порты будут привязаны к ip 0.0.0.0


