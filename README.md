# Postgres-docker-compose


Данный репозиторий создан, для автоматической сборки образа Postgres, с автоматизацией через Docker-compose.

Все работы будут производится на Ubuntu 22.04 LTS

1. Для начала требуется установить Docker.
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
2. Дальше создаем нашу папку для проекта, в нее войдет docker-compose.yml, а также скрипты для первичной подготовки базы.

- Первое с чего мы начнем, это напишем наш docker-compose.yml
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

- Для облегчения задачи, я воспользуюсь docker-entrypoint-initdb.d, для того что бы через нее использовать запуск скриптов, с последющим наполнением базы.

### Внимание: данным способом передачи скриптов, они будут исполняться только при первичной инициализации контейнера. 
- Для того что бы использовать при каждом запуске можно использовать COMMAND, но для этого требуется переписать скрипт.

3. Переходим к написанию, нашего скрипта.
В нем мы укажем создание тестовой БД, создадим пользователей, таблицы, а также наполним их данными.
Для удобства, я сложу весь скрипт, в отдельную директорию.

```
#!/bin/bash
psql -f docker-entrypoint-initdb.d/startup 
```

- Мы указываем в скрипте, что после того, как он запускается использовать скрипт и путь до файла с командами, его приложу отдельным файлом, так как получился многострочным.
После запуска нашего контейнера у нас создается два Volume, и контейнер с самой Postgres.

# Приведу промежуточный результат результат.
```
[+] Running 3/3
 ✔ Volume "postgres_postgressql_data"         Created                                                                    0.0s 
 ✔ Volume "postgres_backup_postgressql_data"  Created                                                                    0.0s 
 ✔ Container postgressql                      Started                                                                    0.0s 
root@psql:/docker/postgres# docker compose up -d --buil
```
- Мы видим что были созданы Volumes, а также наш контейнер. 
```
docker exec -it postgressql psql -U postgres
```
- После данной команды мы попадаем внутрь контейнера в обалочку Postgres. 

```
root@psql:/docker/postgres# docker exec -it postgressql psql -U postgres
psql (12.17 (Debian 12.17-1.pgdg110+1))
Type "help" for help.

postgres=# 
```
- Выполним проверку, как отработал наш скрипт:

```
1. postgres=# \l
                                 List of databases
    Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
------------+----------+----------+------------+------------+-----------------------
 library_db | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =Tc/postgres         +
            |          |          |            |            | postgres=CTc/postgres+
            |          |          |            |            | admin=CTc/postgres   +
            |          |          |            |            | backup=c/postgres    +
            |          |          |            |            | librarian=c/postgres
2. postgres=# \dt
          List of relations
 Schema |  Name   | Type  |  Owner   
--------+---------+-------+----------
 public | authors | table | postgres
 public | books   | table | postgres
 public | clients | table | postgres
 public | orders  | table | postgres

3. postgres=# SELECT * FROM authors;
 id |      name      |  country  
----+----------------+-----------
  1 | Джоан Роулинг  | UK
  2 | Джордж Оруэлл  | UK
  3 | Фредрик Бакман | Swede
  4 | Оливер Сакс    | USA
  5 | Донна Тартт    | USA
  6 | Колин Маккалоу | Australia
  7 | Джордж Клейсон | USA

4.postgres=# SELECT * FROM books;
 id |                 title                 
----+---------------------------------------
  1 | Гарри Поттер и философский камень
  2 | 1984
  3 | Самый богатый человек в Вавилоне
  4 | Щегол
  5 | Человек, который принял жену за шляпу
  6 | Тревожные Люди
  7 | Поющие в терновнике
postgres=# SELECT * FROM orders;
 id | order_name | Rentend book
----+------------+-------------
  1 | Книга      |       1
  2 | Книга      |       6
  3 | Книга      |       5
  4 | Книга      |       2
  5 | Книга      |       4
  6 | Книга      |       3
  7 | Книга      |       7

postgres=# SELECT * FROM clients;
 id |         last_name          |  
----+----------------------------+
  1 | Иванов Иван Иванович       |             
  2 | Петров Петр Петрович       |             
  3 | Иоганн Себастьян Бах       |             
  4 | Ронни Джеймс Дио           |             
  5 | Ritchie Blackmore          |             
  6 | Крюков Андрей Владимирович |             
  7 | Попов Владислав Андреевич  |  

```
- Вообще мы можем не использовать скрипт, поднять наш контейнер с PostgresSQL, и вбить это все руками, но для наглядности, а также для демонстрации, возможности использовать скрипты. Можно сразу вложить .sql файл и обойтись без использования .sh и сразу использовать его для первичного запуска, операция пройдет точно также.

4. Давайте сделаем наш docker-compose файл правильным и приведем его к правильному виду. 
- Добавим проверку состояния работы контейнера, вообще это являеется устоявшимся шаблоном, и считаю, что должно использоваться без исключения.
- HEALTHCHECK - будем использовать его, для проверки состояния нашего контейнера. При отсувствии отклика от контейнера, будет выполняться автоматический. 

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

    healthcheck: 
      test: ["CMD-SHELL", "pg_isready -U postgres -d library_db"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    restart: unless-stopped
```
- interval - указывам интервал переодичности
- timeout - время ожидания до перезагрузки 

5. Давайте для удобства добавим графический интерфейс. Для этого я воспользуюсь PgAdmin4 - добавим сервис, в наш docker-compose. 
- Давайте перепишем наш docker-compose файл

```

```

