---
layout: post
title: Использование концепции Ядра (Core Domain) в (DDD)
excerpt_separator: <!--more-->
categories:
  - Разработка
  - Управление
tags:
  - design
  - architecture
  - ddd
---

В Domain-Driven Design (DDD) концепция **Core Domain** (Ядро домена) относится к центральной, наиболее важной части бизнес-логики, которая обеспечивает конкурентное преимущество компании. Это та часть системы, которая требует наибольшего внимания и инвестиций.

<!--more-->

### Ключевые концепции:

- **Core Domain** - уникальная, критически важная бизнес-логика
- **Supporting Subdomains** - вспомогательные поддомены, важные но не уникальные
- **Generic Subdomains** - общие поддомены, которые можно купить или использовать готовые решения

## Пример реализации Core Domain на Python

Рассмотрим пример системы для банковского приложения, где Core Domain - это обработка транзакций и управление счетами.

### 1. Определение bounded contexts (ограниченных контекстов)

```python
class Account:
    def __init__(self, account_id: str, balance: float = 0.0):
        self.account_id = account_id
        self._balance = balance
        self._transactions = []

    @property
    def balance(self) -> float:
        return self._balance

    def deposit(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Amount must be positive")
        self._balance += amount
        self._transactions.append(("deposit", amount))

    def withdraw(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Amount must be positive")
        if self._balance < amount:
            raise ValueError("Insufficient funds")
        self._balance -= amount
        self._transactions.append(("withdrawal", amount))

    def get_transactions(self) -> list:
        return self._transactions.copy()
```

### 2. Сервис Core Domain (ядро системы)

```python
class TransactionService:
    """Сервис для обработки транзакций - часть Core Domain"""

    def __init__(self, account_repository):
        self.account_repository = account_repository

    def transfer(self, from_account_id: str, to_account_id: str, amount: float) -> None:
        from_account = self.account_repository.find_by_id(from_account_id)
        to_account = self.account_repository.find_by_id(to_account_id)

        if not from_account or not to_account:
            raise ValueError("One or both accounts not found")

        from_account.withdraw(amount)
        to_account.deposit(amount)

        self.account_repository.save(from_account)
        self.account_repository.save(to_account)
```

### 3. Реализация репозитория (инфраструктурный слой)

```python
class AccountRepository:
    """Инфраструктурный слой - не часть Core Domain, но необходим для работы"""

    def __init__(self):
        self._accounts = {}

    def find_by_id(self, account_id: str):
        return self._accounts.get(account_id)

    def save(self, account: Account) -> None:
        self._accounts[account.account_id] = account

    def delete(self, account_id: str) -> None:
        if account_id in self._accounts:
            del self._accounts[account_id]
```

### 4. Пример использования Core Domain

```python
# Инициализация
repo = AccountRepository()
transaction_service = TransactionService(repo)

# Создание счетов
account1 = Account("acc1", 1000.0)
account2 = Account("acc2", 500.0)
repo.save(account1)
repo.save(account2)

# Выполнение транзакции (Core Domain в действии)
try:
    transaction_service.transfer("acc1", "acc2", 200.0)
    print(f"Account 1 balance: {repo.find_by_id('acc1').balance}")
    print(f"Account 2 balance: {repo.find_by_id('acc2').balance}")
except ValueError as e:
    print(f"Transaction failed: {str(e)}")
```

## Разделение на Core Domain и Supporting/Generic Subdomains

В реальном приложении мы бы разделили функциональность:

### Core Domain:

- Бизнес-правила перевода денег
- Управление лимитами и комиссиями
- Обработка сложных транзакций

### Supporting Subdomains:

- Генерация отчетов
- Уведомления пользователей
- История транзакций

### Generic Subdomains:

- Аутентификация пользователей
- Логирование операций
- Интеграция с платежными системами

## Пример с более сложной бизнес-логикой (Core Domain)

```python
class ComplexTransactionService:
    """Сервис для сложных транзакций - часть Core Domain"""

    def __init__(self, account_repo, limit_service, fee_calculator):
        self.account_repo = account_repo
        self.limit_service = limit_service
        self.fee_calculator = fee_calculator

    def execute_complex_transaction(self, transaction_data: dict) -> dict:
        # Проверка лимитов (часть Core Domain)
        if not self.limit_service.check_transaction_limit(
            transaction_data['user_id'],
            transaction_data['amount']
        ):
            raise ValueError("Transaction limit exceeded")

        # Расчет комиссии (часть Core Domain)
        fee = self.fee_calculator.calculate_fee(
            transaction_data['amount'],
            transaction_data['transaction_type']
        )

        # Выполнение транзакции
        total_amount = transaction_data['amount'] + fee

        from_account = self.account_repo.find_by_id(transaction_data['from_account'])
        from_account.withdraw(total_amount)

        to_account = self.account_repo.find_by_id(transaction_data['to_account'])
        to_account.deposit(transaction_data['amount'])

        self.account_repo.save(from_account)
        self.account_repo.save(to_account)

        return {
            'status': 'completed',
            'fee': fee,
            'final_amount': transaction_data['amount']
        }
```

## Советы по реализации Core Domain в Python

1. **Четкое разделение ответственностей**: Core Domain должен быть изолирован от инфраструктурного кода

2. **Использование value objects**:

```python
class Money:
    def __init__(self, amount: float, currency: str):
        self.amount = amount
        self.currency = currency

    def __add__(self, other):
        if self.currency != other.currency:
            raise ValueError("Currencies don't match")
        return Money(self.amount + other.amount, self.currency)

    # Другие операции...
```

3. **Агрегаты как границы транзакций**:

```python
class CustomerAggregate:
    """Агрегат, управляющий клиентом и его счетами"""
    def __init__(self, customer_id: str):
        self.customer_id = customer_id
        self._accounts = []
        self._credit_limit = 0

    def open_account(self, initial_balance: float = 0) -> Account:
        account = Account(
            account_id=f"{self.customer_id}-{len(self._accounts)}",
            balance=initial_balance
        )
        self._accounts.append(account)
        return account

    def get_total_balance(self) -> float:
        return sum(acc.balance for acc in self._accounts)

    # Другие методы управления агрегатом...
```

4. **Domain Events для отслеживания изменений в Core Domain**:

```python
class DomainEvent:
    pass

class AccountCreated(DomainEvent):
    def __init__(self, account_id: str, customer_id: str):
        self.account_id = account_id
        self.customer_id = customer_id

class MoneyTransferred(DomainEvent):
    def __init__(self, from_account: str, to_account: str, amount: float):
        self.from_account = from_account
        self.to_account = to_account
        self.amount = amount
```

## Заключение

Реализация Core Domain в DDD требует:

- Четкого выделения наиболее важной бизнес-логики
- Изоляции ядра от инфраструктурного кода
- Использования богатых доменных моделей
- Правильного разделения на bounded contexts

Главное - сосредоточить усилия на Core Domain, так как именно он обеспечивает ценность вашего приложения и конкурентное преимущество.
