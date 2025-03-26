---
layout: post
title: Что стоит тестировать в Django
excerpt_separator: <!--more-->
categories:
  - Разработка
tags:
  - python
  - django
  - testing
  - how
---

Я уже неоднократно обсуждал тему тестирования. Мы говорили о том, стоит ли превращать тестирование в культ, и о том, как следует тестировать Django. Теперь осталось только рассмотреть, что именно нужно тестировать в Django, если мы всё же решили это делать.

<!--more-->

Дисклеймер: тестируйте всё, что содержит логику.

## Модели (Models)
- Проверка создания, обновления и удаления объектов.  
- Проверка валидации полей (например, `blank`, `null`, `unique`).
- Проверка методов моделей (если есть кастомная логика).
- Проверка связей (`ForeignKey`, `ManyToManyField`).

```py
from django.test import TestCase
from myapp.models import MyModel

class MyModelTest(TestCase):
    def test_model_creation(self):
        obj = MyModel.objects.create(name="Test")
        self.assertEqual(obj.name, "Test")
```

## Представления (Views)
- Проверка HTTP-ответов (`status_code`, `template_used`, `redirects`).
- Проверка контекста (передаваемых в шаблон данных).
- Проверка работы с аутентификацией и правами (`@login_required`, `@permission_required`).
- Проверка обработки форм и валидации.

```py
from django.test import TestCase, Client

class MyViewTest(TestCase):
    def setUp(self):
        self.client = Client()

    def test_home_page_status(self):
        response = self.client.get('/')
        self.assertEqual(response.status_code, 200)
```

## Формы (Forms)
- Проверка валидации данных.
- Проверка кастомных методов форм.
- Проверка ошибок при неверных данных.

```py
from django.test import TestCase
from myapp.forms import MyForm

class MyFormTest(TestCase):
    def test_valid_form(self):
        form = MyForm(data={"name": "Test", "email": "test@example.com"})
        self.assertTrue(form.is_valid())
```

## API (DRF, если используется)
- Проверка сериализаторов.
- Проверка эндпоинтов (`GET`, `POST`, `PUT`, `DELETE`).
- Проверка прав доступа (`permissions`).

```py
from rest_framework.test import APITestCase
from rest_framework import status

class MyApiTest(APITestCase):
    def test_get_request(self):
        response = self.client.get('/api/my-endpoint/')
        self.assertEqual(response.status_code, status.HTTP_200_OK)
```

## Шаблоны (Templates)
- Проверка корректного отображения данных.
- Проверка условий (`if`, `for` в шаблонах).

```py
from django.test import TestCase

class TemplateTest(TestCase):
    def test_template_used(self):
        response = self.client.get('/')
        self.assertTemplateUsed(response, 'index.html')
```

## Сигналы (Signals)
- Проверка выполнения логики при срабатывании сигналов (`post_save`, `pre_delete` и т. д.).

```py
from django.test import TestCase
from django.db.models.signals import post_save
from myapp.models import MyModel
from myapp.signals import my_signal_handler

class SignalTest(TestCase):
    def test_signal(self):
        post_save.connect(my_signal_handler, sender=MyModel)
        obj = MyModel.objects.create(name="Test")
        # Проверка, что обработчик выполнил нужные действия
```

## Утилиты и хелперы (Utils/Helpers)
- Тестирование вспомогательных функций, не связанных напрямую с Django.

```py
from django.test import TestCase
from myapp.utils import my_helper

class UtilsTest(TestCase):
    def test_helper(self):
        result = my_helper(2, 3)
        self.assertEqual(result, 5)
```

## Команды управления (Management Commands)
- Проверка кастомных команд (`python manage.py mycommand`).

```py
from django.core.management import call_command
from django.test import TestCase
from io import StringIO

class CommandTest(TestCase):
    def test_my_command(self):
        out = StringIO()
        call_command('mycommand', stdout=out)
        self.assertIn("Success", out.getvalue())
```

## Middleware (если есть кастомная логика)
- Проверка изменения запросов/ответов.

```py
from django.test import TestCase, RequestFactory
from myapp.middleware import MyMiddleware

class MiddlewareTest(TestCase):
    def test_middleware(self):
        request = RequestFactory().get('/')
        middleware = MyMiddleware(lambda r: None)
        processed_request = middleware(request)
        self.assertIn("custom_header", processed_request.META)
```

## Интеграционные тесты (API, внешние сервисы, Celery)
- Проверка взаимодействия с внешними API (можно мокать `requests`).
- Проверка асинхронных задач (Celery).

```py
from unittest.mock import patch
from django.test import TestCase

class ExternalAPITest(TestCase):
    @patch('requests.get')
    def test_api_call(self, mock_get):
        mock_get.return_value.status_code = 200
        response = my_api_call()
        self.assertEqual(response.status_code, 200)
```