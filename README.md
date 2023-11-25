# Postgres-docker-compose


Данный репозиторий создан, для автоматической сборки образа Postgres, с автоматизацией через Docker-compose.

Все работы будут производится на Ubuntu 22.04 LTS

Для начала требуется установить Docker.
Порядок установки не буду вкладывать командами, для актуальности лучше всегда пользоваться оффициальной страницей, [docker](https://docs.docker.com/engine/install/ubuntu/)

Можем проверить корректную установку Docker, запустив стандартный контейнер hello-world

```
root@psql:~# docker run hello-world
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/

```
А версию Docker-Compose

```
root@psql:~# docker compose version
Docker Compose version v2.21.0
```
Дальше создаем нашу папку для проекта, в нее войдет docker-compose.yml, а также скрипты для первичной подготовки базы.

Первое с чего мы начнем, это напишем наш docker-compose.yml
Я буду создавать один контенер, с двумя подключенными к нему volume.

```
version: "3.9" # Указываем версию

volumes: 
  postgressql_data: # Основной Volume
  backup_postgressql_data: # Volume для бэкапа баз.

services:

  postgressql: # Название нашего сервиса
    image: postgres:12-bullseye # Образ из которого мы разворачиваем
    container_name: postgressql # Имя контейнера
    environment: 
      - PGDATA=/var/lib/postgresql/data/
      - POSTGRES_PASSWORD=passwd # Указываем пароль для пользователя. 
    volumes: 
      - postgressql_data:/var/lib/postgresql/data
      - backup_postgressql_data:/backup
      - ./config:/docker-entrypoint-initdb.d # Потребуется для первоночальной установке и выполнении скриптов
    network_mode: "host"    
```

Для облегчения задачи, я воспользуюсь docker-entrypoint-initdb.d, для того что бы через нее использовать запуск скриптов, с последющим наполнением базы.

### Внимание: данным способом передачи скриптов, они будут исполняться только при первичной инициализации контейнера. 

Переходим к написанию, нашего скрипта.
В нем мы укажем создание тестовой БД, создадим пользователей, таблицы, а также наполним их данными.
Для удобства, я сложу весь скрипт, в отдельную директорию.

```
#!/bin/bash
psql -f docker-entrypoint-initdb.d/startup 
```

Мы указываем в скрипте, что после того, как он запускается использовать скрипт и путь до файла с командами, его приложу отдельным файлом, так как получился многострочным.




