---
layout: post
title: Как в Django сделать Multi Host
excerpt_separator: <!--more-->
categories:
  - Разработка
tags:
  - python
  - django
  - how
---

Зачем? Во-первых, существующий `Site Framework` явно вне архитектурной парадигмы `Django`. И, скорее всего, рано или поздно будет удален. Во-вторых, все существующие батарейки так себе.

<!--more-->

Попробуем сделать все по правилам через кастомный `middleware`.

```python
virtual_hosts = {
    "example.local": "example.urls",
    "a.example.local": "a_example.urls",
    "b.example.local": "b_example.urls",
}


class VirtualHostMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        host = request.get_host()
        request.urlconf = virtual_hosts.get(host)
        response = self.get_response(request)
        return response
```

Добавим наш `Middleware` в список пользовательское промежуточное ПО

```python
MIDDLEWARE = [
    # our custom middleware
    "app.middleware.VirtualHostMiddleware",
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]
```

В принципе всё!

## Важное примечание о пространствах имен URL

В Django хорошей идеей является создание пространства имен для URL-адреса каждого приложения:

```python
app_name = "landing"

urlpatterns = [path("login/", index, name="login")]
```

Всякий раз ссылаясь на что либо нам необходимо ставить префикс

```python
view = reverse("landing:login")
```

Но так как мы перезаписываем `reverse`, чтобы перенаправить пользователей на представление с именем `landing:login`, то с нашим виртуальным хостом `middleware` мы фактически делаем пространство имен избыточным.

Делаем так. Просто опускаем префикс

```python
view = reverse("login")
```
