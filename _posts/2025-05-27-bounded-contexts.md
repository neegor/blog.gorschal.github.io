---
layout: post
title: Ограниченные контексты (Bounded Contexts) в DDD
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

Ограниченные контексты (Bounded Contexts) - это ключевая концепция Domain-Driven Design (DDD), которая помогает организовать сложные доменные модели, разделяя их на логические модули с четкими границами.

<!--more-->

Bounded Context - это четко определенная граница, внутри которой конкретная доменная модель применяется и остается согласованной. В разных контекстах одни и те же термины могут иметь разное значение.

## Примеры на Python

### Пример 1: Разные значения в разных контекстах

```python
# Контекст электронной коммерции (E-commerce)
class Product:
    def __init__(self, id: str, name: str, price: float, inventory: int):
        self.id = id
        self.name = name
        self.price = price
        self.inventory = inventory  # Количество на складе

    def is_available(self) -> bool:
        return self.inventory > 0


# Контекст логистики (Logistics)
class Product:
    def __init__(self, id: str, name: str, weight: float, dimensions: tuple):
        self.id = id
        self.name = name
        self.weight = weight  # Вес в кг
        self.dimensions = dimensions  # Габариты (длина, ширина, высота)

    def calculate_shipping_cost(self) -> float:
        # Логика расчета стоимости доставки
        return self.weight * 100  # Упрощенный расчет
```

Здесь класс `Product` имеет разное значение в разных контекстах:
- В e-commerce важны цена и наличие
- В logistics важны вес и габариты

### Пример 2: Разделение контекстов с помощью модулей

```python
# Файл: ecommerce/product.py
class Product:
    """Контекст электронной коммерции"""
    def __init__(self, id: str, name: str, price: float):
        self.id = id
        self.name = name
        self.price = price


# Файл: logistics/product.py
class Product:
    """Контекст логистики"""
    def __init__(self, id: str, weight: float):
        self.id = id
        self.weight = weight


# Файл: main.py
from ecommerce.product import Product as EcommerceProduct
from logistics.product import Product as LogisticsProduct

# Использование разных контекстов
ecommerce_prod = EcommerceProduct("123", "Laptop", 999.99)
logistics_prod = LogisticsProduct("123", 2.5)
```

### Пример 3: Контекстный маппинг

```python
class ProductMapper:
    @staticmethod
    def to_ecommerce(logistics_product, price: float) -> "EcommerceProduct":
        return EcommerceProduct(
            id=logistics_product.id,
            name=f"Product {logistics_product.id}",  # В реальности это могло бы быть из БД
            price=price
        )

    @staticmethod
    def to_logistics(ecommerce_product, weight: float) -> "LogisticsProduct":
        return LogisticsProduct(
            id=ecommerce_product.id,
            weight=weight
        )
```

## Ключевые принципы Bounded Context

1. **Явные границы**: Каждый контекст должен иметь четко определенные границы
2. **Собственный язык (Ubiquitous Language)**: В каждом контексте свой согласованный язык
3. **Автономность**: Контексты должны быть максимально независимы
4. **Интеграция**: Контексты могут взаимодействовать через антикоррупционные слои (Anti-Corruption Layer)

## Когда использовать Bounded Contexts

1. Когда модель становится слишком сложной
2. Когда разные команды работают над разными частями системы
3. Когда одни и те же термины имеют разное значение в разных частях системы

Bounded Contexts помогают управлять сложностью, изолируя разные части системы и позволяя им развиваться независимо.