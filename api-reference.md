# API Reference

Базовый путь — `/api`. Все эндпоинты отдают JSON. Авторизация — заголовок `Authorization: Bearer <access_token>`, если не указано обратное.

## Общие правила

### Формат ошибок

```json
{ "error": "описание ошибки" }
```

HTTP-коды:

| Код | Значение             | Когда                                            |
|-----|----------------------|--------------------------------------------------|
| 400 | Bad Request          | Невалидный JSON или нарушение бизнес-правил      |
| 401 | Unauthorized         | Нет/просрочен/невалиден токен                    |
| 403 | Forbidden            | Нет прав (не владелец/не участник/не админ)     |
| 404 | Not Found            | Сущность не найдена                              |
| 409 | Conflict             | Нарушение уникальности (email, username)         |
| 429 | Too Many Requests    | Rate limit на `/api/auth/*` и `/api/invites/*`   |
| 500 | Internal Server Error| Непредвиденная ошибка                            |

### Денежные суммы

Все суммы приходят как `number` (float64) в валюте счёта. Курсы — `float64`, 4 знака после запятой.

### Даты

Даты транзакций — строка `YYYY-MM-DD`. Timestamp-ы (`createdAt`, `updatedAt`) — ISO 8601.

---

## Public (без авторизации)

### `GET /api/health`

Health-check. Ответ: `200 OK`, тело `{ "status": "ok" }`.

### `GET /api/currencies`

Активные валюты со свежими курсами относительно USD.

```json
[
  { "code": "USD", "name": "US Dollar", "rate": 1.0,    "isActive": true },
  { "code": "EUR", "name": "Euro",      "rate": 0.9231, "isActive": true },
  { "code": "RUB", "name": "Ruble",     "rate": 92.11,  "isActive": true }
]
```

### `GET /api/invites/{token}`

Валидация инвайт-токена. Ответ:

```json
{ "email": "user@example.com", "expiresAt": "2026-04-14T12:00:00Z" }
```

### `POST /api/invites/{token}/accept`

Принятие инвайта: создаёт пользователя и возвращает пару токенов.

Запрос:

```json
{
  "username": "alice",
  "password": "secret123",
  "defaultCurrency": "USD"
}
```

Ответ `200`:

```json
{
  "accessToken": "eyJ...",
  "refreshToken": "eyJ...",
  "user": { "id": "...", "username": "alice", "email": "user@example.com", "defaultCurrency": "USD" }
}
```

Ошибки: `400` слабый пароль, `409` занятый username, `404`/`400` невалидный токен.

---

## Auth

### `POST /api/auth/login`

Rate limit: 5 req/s, burst 10.

```json
{ "emailOrUsername": "alice", "password": "secret123" }
```

Ответ `200`:

```json
{
  "accessToken": "eyJ...",
  "refreshToken": "eyJ...",
  "user": { "id": "...", "username": "alice", "email": "...", "defaultCurrency": "USD", "isAdmin": false }
}
```

Ошибки: `401` неверный логин/пароль, `403` пользователь заблокирован.

### `POST /api/auth/refresh`

```json
{ "refreshToken": "eyJ..." }
```

Ответ `200`: такая же пара токенов. `401` если refresh просрочен/отозван.

---

## Users

### `GET /api/users`

Список активных пользователей (для добавления в совместные счета).

```json
[
  { "id": "...", "username": "alice", "email": "...", "defaultCurrency": "USD" }
]
```

### `GET /api/users/me`

Текущий пользователь по JWT.

### `PATCH /api/users/me`

Обновить профиль. Поля опциональны.

```json
{ "defaultCurrency": "EUR" }
```

---

## Accounts

### `GET /api/accounts`

Счета, в которых участвует пользователь.

```json
[
  {
    "id": "...",
    "name": "Семья",
    "type": "shared",
    "currency": "RUB",
    "icon": "👨‍👩‍👧",
    "initialBalance": 0,
    "includeInBalance": true,
    "ownerId": "...",
    "createdAt": "2026-03-10T08:00:00Z",
    "members": [
      { "userId": "...", "username": "alice", "defaultShare": 0.5 },
      { "userId": "...", "username": "bob",   "defaultShare": 0.5 }
    ]
  }
]
```

### `POST /api/accounts`

```json
{
  "name": "Карта Сбер",
  "type": "personal",
  "currency": "RUB",
  "icon": "💳",
  "initialBalance": 12500.00,
  "includeInBalance": true
}
```

`type`: `"personal"` | `"shared"`. Валюта — иммутабельна после создания.

### `GET /api/accounts/{accountID}`

Один счёт. Требует членства — middleware `AccountMember`. `403` если не участник.

### `PATCH /api/accounts/{accountID}`

Обновление полей (кроме `currency` и `type`).

```json
{ "name": "Новое имя", "icon": "🏦", "includeInBalance": false }
```

### `DELETE /api/accounts/{accountID}`

Soft-delete. Только владелец (`403` иначе).

### `GET /api/accounts/{accountID}/members`

Участники совместного счёта с долями по умолчанию.

### `POST /api/accounts/{accountID}/members`

Добавить участника. Только владелец.

```json
{ "userId": "...", "defaultShare": 0.3 }
```

### `PATCH /api/accounts/{accountID}/members/{userID}`

Изменить долю по умолчанию. Только владелец.

```json
{ "defaultShare": 0.4 }
```

### `DELETE /api/accounts/{accountID}/members/{userID}`

Удалить участника. Только владелец.

---

## Categories

### `GET /api/categories`

Плоский список категорий пользователя, отсортированный по названию.

```json
[
  {
    "id": "...",
    "name": "Еда",
    "type": "expense",
    "icon": "🍔"
  }
]
```

`type`: `"expense"` | `"income"`. Названия активных категорий уникальны в пределах пользователя и типа без учёта регистра.

### `POST /api/categories`

```json
{ "name": "Транспорт", "type": "expense", "icon": "🚗" }
```

### `PATCH /api/categories/{categoryID}`

Любые поля опционально (`name`, `icon`).

### `DELETE /api/categories/{categoryID}`

При наличии транзакций категория удаляется мягко и сохраняется в истории. Без транзакций удаляется полностью.

---

## Transactions

### `GET /api/transactions`

Фильтры (все опциональны):

| Параметр       | Формат                        | Описание                        |
|----------------|-------------------------------|---------------------------------|
| `account_ids`  | `uuid,uuid,...`               | Один или несколько счетов       |
| `category_ids` | `uuid,uuid,...`               |                                 |
| `tag_ids`      | `uuid,uuid,...`               |                                 |
| `tag_mode`     | `or` \| `and`                 | По умолчанию `or`               |
| `date_from`    | `YYYY-MM-DD`                  |                                 |
| `date_to`      | `YYYY-MM-DD`                  |                                 |
| `type`         | `expense` \| `income` \| `transfer` |                           |
| `page`         | int, default 1                |                                 |
| `limit`        | int, default 50, max 200      |                                 |

Ответ:

```json
{
  "items": [
    {
      "id": "...",
      "type": "expense",
      "accountId": "...",
      "amount": 1200.50,
      "currency": "RUB",
      "displayCurrency": "RUB",
      "rate": 1.0,
      "categoryId": "...",
      "description": "Обед",
      "date": "2026-04-10",
      "createdAt": "2026-04-10T12:03:00Z",
      "shares": [
        { "userId": "...", "amount": 600.25 },
        { "userId": "...", "amount": 600.25 }
      ],
      "tags": [ { "id": "...", "name": "обед" } ],
      "toAccountId": null,
      "toAmount": null
    }
  ],
  "total": 123,
  "page": 1,
  "limit": 50
}
```

### `POST /api/transactions`

```json
{
  "type": "expense",
  "accountId": "...",
  "amount": 1200.50,
  "date": "2026-04-10",
  "categoryId": "...",
  "description": "Обед",
  "tags": ["обед", "рестораны"],
  "shares": [
    { "userId": "...", "amount": 600.25 },
    { "userId": "...", "amount": 600.25 }
  ]
}
```

Правила:

- `expense`/`income`: сумма долей = `amount`. Для личных счетов `shares` можно опустить — подставится одна доля на владельца.
- `transfer`: нужны `toAccountId`, `toAmount` (сумма в валюте получателя); `categoryId` и `shares` игнорируются.
- Тег создаётся автоматически, если не существует. Максимум 10 тегов.
- Валюта подставляется из счёта; `rate` можно передать вручную для перевода в валюту отображения.

### `GET /api/transactions/{transactionID}`

Одна транзакция. `403` если пользователь не участник счёта.

### `PATCH /api/transactions/{transactionID}`

Полная замена: все поля как в `POST`. Доли пересчитываются.

### `DELETE /api/transactions/{transactionID}`

Жёсткое удаление с каскадом на `transaction_shares` и `transaction_tags`.

---

## Tags

### `GET /api/tags`

Все теги пользователя с числом использований.

```json
[ { "id": "...", "name": "отпуск", "usageCount": 12 } ]
```

### `PATCH /api/tags/{tagID}`

```json
{ "name": "новое-имя" }
```

### `DELETE /api/tags/{tagID}`

Жёсткое удаление, связи с транзакциями удаляются каскадно.

---

## Analytics

Все эндпоинты принимают единые query-параметры:

| Параметр      | Формат        | Описание                            |
|---------------|---------------|-------------------------------------|
| `date_from`   | `YYYY-MM-DD`  | обязательный                        |
| `date_to`     | `YYYY-MM-DD`  | обязательный                        |
| `account_ids` | `uuid,...`    | опционально; если нет — все счета   |
| `currency`    | ISO 4217      | валюта отображения, по умолчанию профиля |
| `type`        | `expense`\|`income` | для `by-category` и `by-tag`   |

### `GET /api/analytics/summary`

```json
{
  "currency": "USD",
  "balance": 15000.00,
  "income":  3200.00,
  "expense": 2100.00
}
```

Баланс — только доли текущего пользователя в включённых в баланс счетах.

### `GET /api/analytics/by-category`

```json
[
  { "categoryId": "...", "categoryName": "Еда", "amount": 850.00, "share": 0.40 }
]
```

### `GET /api/analytics/by-tag`

Аналогично `by-category`, но группировка по тегам.

---

## Admin (требуется `isAdmin=true`)

### `GET /api/admin/users`

Все пользователи, включая заблокированных.

```json
[
  { "id": "...", "username": "alice", "email": "...", "isActive": true, "isAdmin": false, "createdAt": "..." }
]
```

### `GET /api/admin/users/{userID}`

### `PATCH /api/admin/users/{userID}`

```json
{ "isActive": false, "isAdmin": false }
```

### `GET /api/admin/currencies`

Все валюты, включая отключённые.

### `POST /api/admin/currencies`

```json
{ "code": "BYN", "name": "Belarusian Ruble" }
```

`409` если код уже существует. Код должен присутствовать в последнем ответе внешнего провайдера курсов.

### `PATCH /api/admin/currencies/{code}`

```json
{ "name": "...", "isActive": false }
```

### `POST /api/admin/currencies/rates/refresh`

Принудительное обновление курсов с open.er-api.com.

### `GET /api/admin/invites`

Список всех инвайтов (активных, использованных, просроченных).

### `POST /api/admin/invites`

```json
{ "email": "new-user@example.com" }
```

Ответ:

```json
{
  "id": "...",
  "email": "new-user@example.com",
  "token": "...",
  "link": "https://app.co-wallet.local/invite/...",
  "expiresAt": "2026-04-14T12:00:00Z"
}
```

Если в `.env` заданы `SMTP_*`, ссылка автоматически отправляется на email.
