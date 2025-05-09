---
layout: post
title: Универсальный язык (Ubiquitous Language) в DDD
excerpt_separator: <!--more-->
categories:
  - Разработка
  - Управление
tags:
  - design
  - architecture
  - ddd
---

[Начало и оглавление здесь](https://blog.gorschal.com/domain-driven-design.html)

__Универсальный язык (Ubiquitous Language)__ — это один из ключевых концептов Domain-Driven Design, который подразумевает создание общего, точного и однозначного языка между разработчиками и экспертами предметной области (domain experts). Этот язык используется на всех этапах работы: от обсуждения требований до проектирования и реализации кода.

<!--more-->

## Зачем нужен Ubiquitous Language?

В сложных доменах (например, банковская сфера, логистика, медицинские системы) часто возникают недопонимания между разработчиками и бизнес-экспертами. Разные группы людей могут использовать разные термины для одних и тех же понятий или, наоборот, называть одним словом разные сущности.

### Пример проблемы без Ubiquitous Language:

- Бизнес говорит: "Клиент может оформить заказ".
- Разработчик понимает: "Пользователь может создать запись в таблице Orders".
- Но на самом деле "оформить заказ" включает: проверку доступности товара, резервирование, создание заявки в CRM, отправку уведомления и т. д.

### Решение: создать единый словарь терминов, который будет:
- Точно описывать бизнес-процессы.
- Использоваться в коде (названия классов, методов, переменных).
- Быть понятным и разработчикам, и бизнесу.

### Как формируется Ubiquitous Language?
- __Выявление ключевых понятий__ – совместная работа разработчиков и экспертов домена.
- __Уточнение терминов__ – избегание двусмысленностей, устранение синонимов.
- __Фиксация в документации__ – глоссарий, диаграммы, пользовательские истории.
- __Отражение в коде__ – классы, методы, база данных называют вещи так же, как и в бизнес-логике.

### Преимущества Ubiquitous Language

- __Уменьшает недопонимание__ – все говорят на одном языке.
- __Упрощает рефакторинг__ – код отражает бизнес-логику.
- __Помогает в проектировании__ – четкие границы моделей (Bounded Context).
- __Делает документацию полезной__ – она остается актуальной, так как термины используются в коде.

### Проблемы и антипаттерны

- __Разные термины в коде и документации__ – если разработчики используют свои названия, язык перестает быть универсальным.
- __Слишком технические термины__ – бизнес не должен разбираться в "DTO", "репозиториях" и "фасадах".
- __Игнорирование изменений__ – если домен эволюционирует, язык тоже должен обновляться.

### Главное правило

Если бизнес называет это "Заказом", то в коде это должно быть `Order`, а не `PurchaseRequest` или `UserTransaction`.
