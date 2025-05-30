# Документация по микросервисной архитектуре Inventory Management API

## Обзор

Система Inventory Management API разделена на три микросервиса для обеспечения независимости, масштабируемости и удобства командной разработки. Микросервисы взаимодействуют синхронно через REST API с JWT-аутентификацией. Асинхронное взаимодействие исключено. Падение Supplier Service не влияет на работу Auth Service и Core Service.

### Микросервисы

1. **Auth Service**: Управление аутентификацией и авторизацией.
2. **Core Service**: Объединяет управление товарами, заказами и клиентами.
3. **Supplier Service**: Управление поставщиками и их товарами.

---

## 1. Auth Service

### Описание

Отвечает за регистрацию, вход, выдачу и обновление JWT-токенов. Обеспечивает безопасность доступа к другим микросервисам.

### Ответственность

- Регистрация новых пользователей.
- Аутентификация (вход).
- Выдача и обновление JWT-токенов.
- Управление ролями (admin, manager, user).

### Эндпоинты

- `POST /auth/register` — Регистрация пользователя.
- `POST /auth/login` — Вход пользователя.
- `POST /auth/refresh` — Обновление токена.

### Схемы данных

- `User`: `{ id, email, name, passwordHash, role }`
- `RegisterRequest`: `{ email, password, name }`
- `LoginRequest`: `{ email, password }`
- `TokenResponse`: `{ accessToken, refreshToken }`

### База данных

- **Тип**: PostgreSQL.
- **Таблицы**:
  - `users`: Хранит данные пользователей (id, email, name, passwordHash, role).
- **Особенности**: Хэширование паролей (bcrypt), индексы на `email`.

### Зависимости

- Нет внешних зависимостей.
- Core Service и Supplier Service зависят от Auth Service для проверки JWT.

---

## 2. Core Service (Inventory + Order + Customer)

### Описание

Объединяет функциональность управления товарами, заказами и клиентами в единый микросервис для упрощения взаимодействия и минимизации сетевых вызовов. Является центральным компонентом системы.

### Ответственность

- **Inventory**: Создание, обновление, удаление, просмотр товаров.
- **Order**: Создание, просмотр, обновление статуса, отмена заказов, генерация счетов.
- **Customer**: Управление клиентами, просмотр заказов, отправка email.

### Эндпоинты

#### Inventory

- `GET /items` — Получить все товары (с пагинацией).
- `POST /items` — Добавить товар.
- `GET /items/{id}` — Просмотр деталей товара.
- `PUT /items/{id}` — Обновить товар.
- `DELETE /items/{id}` — Удалить товар.

#### Order

- `GET /orders` — Получить все заказы (с пагинацией).
- `POST /orders` — Создать заказ.
- `GET /orders/{id}` — Просмотр деталей заказа.
- `POST /orders/{id}/invoice` — Сгенерировать счёт.
- `PUT /orders/{id}/status` — Обновить статус заказа.
- `POST /orders/{id}/cancel` — Отменить заказ.

#### Customer

- `GET /customers` — Получить всех клиентов (с пагинацией).
- `POST /customers` — Добавить клиента.
- `GET /customers/{id}` — Просмотр деталей клиента.
- `PUT /customers/{id}` — Обновить клиента.
- `DELETE /customers/{id}` — Удалить клиента.
- `GET /customers/{id}/orders` — Просмотр заказов клиента.
- `POST /customers/{id}/email` — Отправить email клиенту.

### Схемы данных

- `Item`: `{ id, name, article, category, quantity, location, status }`
- `Order`: `{ id, orderNumber, customer, date, items, totalAmount, status }`
- `Customer`: `{ id, name, email, phone, orders, totalPurchases, lastOrder, status }`
- `Invoice`: `{ id, orderId, amount, issuedDate }`
- `EmailRequest`: `{ subject, body }`

### База данных

- **Тип**: PostgreSQL.
- **Таблицы**:
  - `items`: Данные о товарах.
  - `orders`: Данные о заказах.
  - `order_items`: Элементы заказа (связь между заказами и товарами).
  - `customers`: Данные о клиентах.
  - `invoices`: Счета для заказов.
- **Особенности**:
  - Транзакции для операций с заказами (например, резервирование товаров).
  - Индексы на часто запрашиваемые поля (`orderNumber`, `customer.email`).
  - Кэширование списков товаров и клиентов (Redis).

### Зависимости

- Auth Service для проверки JWT-токенов.
- Нет зависимости от Supplier Service.

---

## 3. Supplier Service

### Описание

Отвечает за управление поставщиками и их товарами. Независим от Core Service, его падение не влияет на работу системы.

### Ответственность

- Создание, обновление, удаление, просмотр поставщиков.
- Просмотр списка товаров, поставляемых конкретным поставщиком.

### Эндпоинты

- `GET /suppliers` — Получить всех поставщиков (с пагинацией).
- `POST /suppliers` — Добавить поставщика.
- `GET /suppliers/{id}` — Просмотр деталей поставщика.
- `PUT /suppliers/{id}` — Обновить поставщика.
- `DELETE /suppliers/{id}` — Удалить поставщика.
- `GET /suppliers/{id}/items` — Просмотр товаров поставщика.

### Схемы данных

- `Supplier`: `{id, name, contactPerson, email, phone, category, items, status }`

### База данных

- **Тип**: PostgreSQL.
- **Таблицы**:
  - `suppliers`: Данные о поставщиках.
  - `supplier_items`: Связь поставщиков с товарами (через `item_id`).
- **Особенности**:
  - Индексы на `email` и `name` для быстрого поиска.
  - Кэширование данных о поставщиках (Redis).

### Зависимости

- Auth Service для проверки JWT-токенов.
- Core Service для получения данных о товарах (через REST API, эндпоинт `/items`).
- Если Core Service недоступен, Supplier Service может возвращать только данные о поставщиках без списка товаров.

## Взаимодействие микросервисов

### Архитектура

- **Синхронное взаимодействие**: REST API с JWT-аутентификацией.
- **Проверка токенов**:
  - Auth Service выдает JWT-токены с ролями.
  - Сервисы проверяют токены через публичный ключ Auth Service.

### Поток данных

1. **Аутентификация**:
   - Клиент отправляет запрос на `/auth/login` → Auth Service возвращает JWT.
   - JWT включается в заголовок `Authorization: Bearer <token>` для всех запросов.
2. **Core Service**:
   - Обрабатывает запросы к товарам, заказам, клиентам.
   - Проверяет JWT через Auth Service (или локально с публичным ключом).
3. **Supplier Service**:
   - Запрашивает данные о товарах у Core Service через `/items`.
   - Может работать без Core Service, возвращая только данные о поставщиках.

### Обработка ошибок

- Все сервисы возвращают стандартизированные ошибки (схема `Error`: `{ code, message }`).
- Логирование ошибок в централизованной системе (ELK Stack).
- HTTP-коды:
  - `400`: Неверный запрос.
  - `401`: Неверный или отсутствующий JWT.
  - `404`: Ресурс не найден.
  - `500`: Внутренняя ошибка сервера.

---

### Масштабируемость

- **Auth Service**: Высокая нагрузка на чтение/запись, использование горизонтального масштабирования.
- **Core Service**: Кэширование частых запросов (Redis), шардирование базы данных при росте данных.
- **Supplier Service**: Минимальная нагрузка, достаточно одного инстанса на начальном этапе.

---

## Итог

Архитектура из трех микросервисов (Auth, Core, Supplier) обеспечивает:

- Независимость разработки и развертывания.
- Устойчивость системы при падении Supplier Service.
- Простоту синхронного взаимодействия через REST API.
- Гибкость для масштабирования Core Service как основного компонента.

Для дальнейшей детализации (например, схемы баз данных, примеры кода) обратитесь с конкретным запросом.
