---
layout: post
title: Как стартануть контейнер X после того как стартанет Y
categories:
  - Разработка
tags:
  - develop
  - devops
  - docker
---

Небольшой лафхак. Стандартная ситуация состоящая из приложения и базы данных где хранятся наши данные. В виде docker-compose файла это выглядит примерно так:

```yaml
version: "3"
services:
 cassandra:
   image: cassandra:3.11
   container_name: cassandra
   restart: always
   volumes:
     - cassandra-metrix-data:/var/lib/cassandra
   ports:
     - 7000:7000
     - 7001:7001
     - 7199:7199
     - 9042:9042
     - 9160:9160
 agent:
   build:
     context: .
     dockerfile: ./Dockerfile
     target: develop
   container_name: agent
   depends_on:
     - cassandra
```

Но есть одна небольшая проблема. Наше приложение *agent* зависит от данных и скорее всего запустится быстрее чем ***cassandra***. Результат предсказуем: не найдя источник данных наш *agent* сфейлится.

Чтобы этого не произошло можно в *агент* добавить нечто подобное:

```yaml
restart: on-failure
```

Что позволяет перезапускать агента в случае падения. И теоретически он так будет запускаться пока cassandra не поднимется.

Но можно поступить красивее. Добавляем в .Dockerfile агента следующую запись:

```docker
ADD https://github.com/ufoscout/docker-compose-wait/releases/download/2.5.1/wait ${PROJECTPATH}/wait
RUN chmod +x ${PROJECTPATH}/wait
```

Теперь наш *docker-compose* файл примет следующий вид:

```yaml
agent:
   build:
     context: .
     dockerfile: ./Dockerfile
     target: develop
   container_name: agent
   environment:
     WAIT_HOSTS: cassandra:9160, cassandra:9042
   command: python agent.py
   volumes:
     - .:/opt/metrix-agent/app
   ports:
     - 8095:8095
     - 8094:8094
   tty: true
   restart: on-failure
   depends_on:
     - cassandra
```

Поясню. Теперь наш *agent* прежде чем выполнить команду ```python agent.py``` будет ждать пока наша ***cassandra*** не будет доступна на соответствующих портах. В нашем случае это:

```yaml
environment:
     WAIT_HOSTS: cassandra:9160, cassandra:9042
```

Данный лайфхак доступен для любого случая кода вам необходимо стартануть контейнер X после того как стартанет Y. Не забывайте только подставлять правильные имена контейнеров и порты которые необходимо контролировать.