---
layout: post
title: Multi-Stage Docker на все случаи жизни
categories:
  - Develop
  - Devops
tags:
  - devops
  - develop
  - docker
  - multistage
  - docker-compose
---

Докер еще с версии 17.05 поддерживает *Multi-Stage* сборку. Не знаю, возможно это мне так везет, но все что удалось почитать на эту тему сводится к сборке максимально минималистичного **Golang** образа.

Выглядит это примерно так (взял практически из официальной документации):

```docker
FROM golang:latest AS builder
WORKDIR ./src
COPY . .
RUN go build ./src/main.go

FROM alpine:latest  
COPY --from=builder /go/main .
CMD ["./main"]
```
При сборке этого образа происходит дословно следующее. Первый блок **FROM** под названием *builder* собирает (компилирует) наше **Go** приложение. Второй блок **FROM** берет голый *alpine* образ и вставляет в него артефакт из первого блока (под артефактом подразумевается скомпилированный бинарник).

Если мы выполним нижеследующую команду, то мы действительно, и по понятным причинам получим минимальный из возможных образ.

```bash
$ docker build -t ./go-docker:latest .
```

И здесь начинается самое интересное. Уменьшив размер образа мы удалили из него практически все, в том и числе, и все что связано с компиляцией. Но ведь мы уже привыкли использовать докеры не только в продакшене, но и в девелоп процессах и возможность компиляции для нас критически важна. Так как же совместить эти заведомо противоречие требования? Как по мне, так это и есть основная причина использования *Multi-Stage* сборок, а отнюдь не размер образа.

В первую очередь *Multi-Stage* сборка это когда мы можем использовать один и тот же докер файл в массе сценариев. В нашем случае нам просто достаточно добавить в *docker-compose* один параметр и мы решаем проблему.

```
version: "3.4"
services:
  go-builder:
    build:
      context: .
      dockerfile: ./Dockerfile
      target: builder
    container_name: go-builder
```

Параметр ```target: builder``` говорит нам,что нам необходимо использовать в докере именованный блок *builder*. И это простое действие позволяет нам ничего не менять в нашем процессе разработки. Все наши компиляторы на месте, но при этом в продакшене наш образ максимально минимизирован. Одни плюсы.

Для наглядности в наш докер файл можно добавить еще один слой. И в разработке уже использовать блок *develop*.

```docker
FROM golang:latest AS develop
WORKDIR ./src
COPY . .

FROM develop AS builder
RUN go build ./src/main.go

FROM alpine:latest  
COPY --from=builder /go/main .
CMD ["./main"]
```

**Golang** это компилируемый язык, а будет ли польза от использования *Multi-Stage* в интерпретируемых языках. Ведь интерпретатор в них должен присутствовать в любом случае, хоть в девелоп окружении, хоть в продакшене. И ответ, да.

Для примера возьмем **Python** и у нас имеется две среды: *develop* и *production*. А также в наличии три файла requirements. В первом (requirements.txt) содержатся пакеты необходимые для всех используемых сред (например Django). Во втором (requirements.dev.txt) содержатся пакеты необходимые нам в процессе разработки (например coverage). А в третьем (requirements.prod.txt) пакеты необходимые для развертывания приложения (например gunicorn). В данном примере наш docker файл примет примерно такой вид:

```docker
FROM        python:3.7 AS stage
WORKDIR     /opt/app
ADD         requirements.txt /opt/app/
RUN         pip install --no-cache-dir -r /opt/app/requirements.txt

FROM        stage AS develop
ADD         requirements.dev.txt /opt/app/
RUN         pip install --no-cache-dir -r /opt/app/requirements.dev.txt

FROM        stage AS production
ADD         requirements.prod.txt /opt/app/
RUN         pip install --no-cache-dir -r /opt/app/requirements.prod.txt
```

Думаю, из контекста все понятно. Но есть один вопрос. Как нам собрать образ для продакшена, ведь в нашем случае туда попадут все три блока, а нам бы хотелось иметь только один *production*. И в этом нам поможет уже знакомый по *docker-compose* параметр *target*. Только этот раз мы его используем уже непосредственно при сборке docker файла.

```bash
$ docker build — target production -t ./python-docker:latest .
```

Осталось только осветить как *Multi-Stage* дружит с **kubernetes**. На самом деле никак. В **kubernetes** нет ничего на эту тему. Какой докер образ пропишем такой и будет. А как собрать докер образ с необходимыми слоями показано чуть выше.