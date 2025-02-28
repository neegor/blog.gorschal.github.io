---
layout: post
title: Как надо. Django Url
excerpt_separator: <!--more-->
categories:
  - Разработка
tags:
  - python
  - django
  - how
---

Казалось бы, что может быть проще URL dispatcher в Django. И хотя я здесь напишу абсолютно очевидные вещи, но почему-то мне кажется, что некоторых я удивлю.

<!--more-->

Давайте посмотрим пример:

```python
# Хорошо
path('article/<int:id>/', views.article_detail, name='article_detail'),
# Плохо
url(r'^article/(?P<id>\d+)/$', views.article_detail, name='article_detail'),
```

## Почему path() в Django хорошо

Потому что это базовый синтаксис и это `DRY`. Функция была введена в Django 2.0, чтобы обеспечить более простой и понятный человеку способ определения шаблонов URL. А также это читаемо. Надеюсь, никто не будет спорить, что регулярки нечитаемы.

```python
re_path(r'^articles/(?P<year>[0-9]{4})/$', views.archive, name='archive'),
```

## Почему url() в Django плохо

Потому что это устарело и поддерживается только лишь для обратной совместимости. Но на смену ему пришел `re_path()`. И это тоже про регулярки. Вы будете смеяться, но `re_path()` хоть и поддерживаемый официально, но в то же время официально не рекомендуется. Что нам делать, если нужен сложный URL?

## Path converters

Чистый и интуитивно понятный URL — это то, что нам надо. Django предлагает эффективное решение этой задачи, позволяя создавать красивые и читабельные URL-адреса с помощью преобразователей путей.

Пример URL, отображающий данные пользователя по его имени, но при этом устанавливает определенные ограничения для имени, такие как обязательная буква в начале, разрешенные символы — от a до z, цифры от 0 до 9, а также подчеркивания, точки и дефисы, и длина имени должна составлять от 8 до 20 символов:


```python
# converters.py

class UsernamePathConverter:
    regex = '^[a-zA-Z0-9_.-]+$'

    def to_python(self, value):
        # преобразовать значение в соответствующий ему тип данных python
        return value

    def to_url(self, value):
        # преобразуйте значение в str 
        return value
```
```python
# urls.py

from django.urls import register_converter, path
from .views import views
register_converter(UsernamePathConverter, 'username')

urlpatterns = [
    path('articles/<username:uname>/', views.user_detail, name="user_detail")
]
```



