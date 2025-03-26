---
layout: post
title: Как надо. Django Test
excerpt_separator: <!--more-->
categories:
  - Разработка
tags:
  - python
  - django
  - testing
  - how
---

В прошлом году я уже высказывал свою точку зрения на [процесс тестирования](https://blog.gorschal.com/testing.html). Сейчас я постараюсь спроецировать её на `Django`.

<!--more-->

## Итак

1.  Тестируем только то, что может сломаться.
2.  Каждый тест проверяет только одну функцию.
3.  Всегда это говорю, но повторю еще раз. Будьте проще.
4.  Очень рекомендую [factory_boy](https://github.com/FactoryBoy/factory_boy). 

## Структура

Организуйте свои тесты в соответствии с логикой вашего проекта. Я предпочитаю собирать все тесты для каждого приложения в одном файле `tests.py`. Внутри этого файла я разделяю тесты на группы, ориентируясь на то, что именно я проверяю: модели, представления, формы и так далее.

```
└── app_name
    └── tests
        ├── __init__.py
        ├── test_forms.py
        ├── test_models.py
        └── test_views.py
```

## Тестирование Models

```python
from django.test import TestCase
from whatever.models import Whatever
from django.utils import timezone
from django.core.urlresolvers import reverse
from whatever.forms import WhateverForm


class WhateverTest(TestCase):

    def create_whatever(self, title="only a test", body="yes, this is only a test"):
        return Whatever.objects.create(title=title, body=body, created_at=timezone.now())

    def test_whatever_creation(self):
        w = self.create_whatever()
        self.assertTrue(isinstance(w, Whatever))
        self.assertEqual(w.__unicode__(), w.title)
```

Что мы сделали? Мы создали `объект` и убедились, что он соответствует нашим ожиданиям.

## Тестирование Views

```python
    def test_whatever_list_view(self):
        w = self.create_whatever()
        url = reverse("whatever.views.whatever")
        resp = self.client.get(url)

        self.assertEqual(resp.status_code, 200)
        self.assertIn(w.title, resp.content)
```
Что мы сделали? Сначала получаем код ответа, а затем проверяем фактический ответ.

__ВНИМАНИЕ!__  
Тестируя "вьюхи", не надо уходить в функциональное тестирование. Функциональные тесты это тема для отдельного разговора.

## Тестирование Forms

```python
def test_valid_form(self):
    w = Whatever.objects.create(title='Foo', body='Bar')
    data = {'title': w.title, 'body': w.body,}
    form = WhateverForm(data=data)
    self.assertTrue(form.is_valid())

def test_invalid_form(self):
    w = Whatever.objects.create(title='Foo', body='')
    data = {'title': w.title, 'body': w.body,}
    form = WhateverForm(data=data)
    self.assertFalse(form.is_valid())
```

Как то так в первом приближении.
