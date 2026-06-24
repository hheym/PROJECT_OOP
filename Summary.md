# Этап 1: Проектирование доменной области

**Проект:** Платформа для покупки и продажи скинов CS2 (по аналогии с Lis-Skins)
**Трек:** Альтернативный 

---

## 1. Бизнес-контекст

Предметная область — онлайн-платформа для покупки и продажи внутриигровых предметов CS2, работающая по модели «платформа как контрагент»: платформа не сводит покупателя с продавцом напрямую (как csgo.tm), а сама является стороной каждой сделки.

**Принцип работы:**
- Когда пользователь **продаёт** скин — платформа сразу выкупает его по текущей цене выкупа. Деньги мгновенно зачисляются на внутренний баланс пользователя. Предмет передаётся боту платформы и попадает в инвентарь платформы.
- Когда пользователь **покупает** скин — он покупает его из пула платформы по цене продажи. Деньги списываются с внутреннего баланса. Бот платформы отправляет предмет в Steam-инвентарь покупателя.

**Особенности предметной области:**
- Предмет физически находится в Steam, а не только в базе данных платформы — требуется синхронизация инвентаря и трейд-боты.
- Передача предмета и движение денег разнесены во времени (бот может быть занят, трейд-оффер висит несколько минут, Steam API может быть нестабильно) — необходимы чёткие статусы и устойчивость к сбоям.
- Цена скина зависит от характеристик конкретного экземпляра (float, паттерн, стикеры, фаза) — нужна доменная модель, различающая «тип скина» и «конкретный экземпляр».
- Есть значимые корнер-кейсы: предмет под трейд-холдом, бот офлайн или перегружен, пользователь не принял трейд вовремя, цена изменилась между показом и подтверждением.

**Деньги в системе.** У каждого пользователя есть внутренний баланс. Пополнение баланса и вывод средств происходят через внешнего платёжного провайдера. Покупка и продажа скинов изменяют только внутренний баланс, а не внешние платёжные реквизиты напрямую.

---

## 2. Основные сценарии 

**Регистрация и аккаунт**
1. Пользователь авторизуется через Steam. 
2. Платформа проверяет доступность Steam-инвентаря и трейд-ссылки (инвентарь не приватный, аккаунт не под VAC-баном).

**Продажа** 

3. Пользователь выбирает предмет из своего Steam-инвентаря для продажи.

4. Платформа показывает текущую цену выкупа предмета.

5. Пользователь подтверждает продажу — платформа инициирует трейд через бота.

6. Бот принимает предмет от пользователя — статус трейда обновляется, баланс пользователя пополняется.

**Покупка**

7. Пользователь просматривает каталог предметов, доступных в пуле платформы.

8. Пользователь выбирает предмет и оплачивает его с внутреннего баланса.

9. Платформа инициирует трейд от бота к пользователю.

10. Пользователь принимает трейд в Steam — заказ закрывается.

**Баланс и платежи**

11. Пользователь пополняет баланс через внешнего платёжного провайдера.

12. Пользователь выводит средства на внешние реквизиты.

**Обработка сбоев**

13. Трейд-бот не может отправить или принять предмет (трейд-холд у пользователя, бот офлайн, превышен лимит трейдов бота).

14. Пользователь не принял входящий трейд вовремя — автоматическая отмена, возврат предмета или возврат денег.

15. Цена предмета изменилась между показом котировки и подтверждением сделки.

---

## 3. Доменные модели (сущности)

### User
Пользователь платформы.
- `id`, `steamId`, `displayName`, `avatarUrl`, `tradeUrl`
- `balance` — внутренний баланс
- `status` — active / banned / trade_restricted

### SkinTemplate
Каталожная модель скина («тип» предмета, например «AK-47 | Redline»), не привязана к конкретному физическому экземпляру.
- `id`, `weapon`, `skinName`, `rarity`, `collection`
- `basePrice` — справочная рыночная цена

### Item
Конкретный экземпляр скина — одна «жизнь» предмета в системе. При каждом новом выкупе платформой создаётся новый Item, даже если физически это ранее уже проходивший через платформу предмет, предыдущий экземпляр закрывается финальным статусом.
- `id`, `skinTemplateId`, `assetId` (Steam asset id)
- `float`, `exterior`, `stickers[]`, `pattern`
- `status` — in_steam_user / reserved / in_transfer / in_platform_pool / sold

### PriceQuote
Котировка цены выкупа и продажи для конкретного типа скина на определённый момент времени.
- `id`, `skinTemplateId`
- `buyPrice`, `sellPrice`
- `validUntil`, `source` 

### Trade
Сделка между платформой и пользователем — продажа платформе или покупка у платформы.
- `id`, `type` — SELL_TO_PLATFORM / BUY_FROM_PLATFORM
- `userId`, `itemId`, `botId`
- `agreedPrice` — зафиксированная цена на момент создания сделки
- `status` — pending / bot_processing / awaiting_user_action / completed / failed / cancelled / expired
- `createdAt`, `completedAt`

### Bot
Трейд-бот — Steam-аккаунт, исполняющий приём и отправку предметов.
- `id`, `steamId`
- `status` — online / offline / busy / trade_banned
- `currentLoad`, `maxConcurrentTrades`

### BalanceTransaction
Движение средств по внутреннему балансу пользователя.
- `id`, `userId`
- `type` — TOPUP / WITHDRAWAL / SALE_CREDIT / PURCHASE_DEBIT / REFUND
- `amount`, `status`
- `relatedTradeId` 
- `createdAt`

### PaymentTransaction
Внешняя операция пополнения или вывода средств через платёжного провайдера.
- `id`, `userId`
- `direction` — in / out
- `amount`, `provider`, `externalTxId`
- `status` — pending / success / failed

---

## 4. UML: классы и связи

Ниже — текстовое описание классов в UML-нотации (атрибуты, типы связей и кардинальность).

```
Класс User
- id, steamId, displayName, avatarUrl, tradeUrl, balance, status

Класс SkinTemplate
- id, weapon, skinName, rarity, collection, basePrice

Класс Item
- id, skinTemplateId, assetId, float, exterior, stickers[], pattern, status

Класс PriceQuote
- id, skinTemplateId, buyPrice, sellPrice, validUntil, source

Класс Trade
- id, type, userId, itemId, botId, agreedPrice, status, createdAt, completedAt

Класс Bot
- id, steamId, status, currentLoad, maxConcurrentTrades

Класс BalanceTransaction
- id, userId, type, amount, status, relatedTradeId, createdAt

Класс PaymentTransaction
- id, userId, direction, amount, provider, externalTxId, status
```

**Связи (ассоциации) между классами:**

| № | Связь | Кардинальность | Тип | Описание |
|---|---|---|---|---|
| 1 | SkinTemplate → Item | 1 : N | Ассоциация | Один шаблон скина имеет множество конкретных экземпляров с разным float |
| 2 | SkinTemplate → PriceQuote | 1 : N | Ассоциация | У шаблона со временем накапливается история котировок |
| 3 | Trade → PriceQuote | N : 1 | Ассоциация | Сделка фиксирует котировку, по которой была согласована цена |
| 4 | User → Item | 1 : N | Ассоциация | Пользователь может владеть (в Steam) несколькими предметами, потенциально доступными к выкупу |
| 5 | Item → Trade | 1 : N (1 активная) | Ассоциация | Предмет участвует в сделках; в один момент времени — не более одной активной сделки |
| 6 | User → Trade | 1 : N | Ассоциация | Пользователь совершает множество сделок (продаж и покупок) |
| 7 | Bot → Trade | 1 : N | Ассоциация | Один бот обрабатывает несколько сделок параллельно |
| 8 | User → BalanceTransaction | 1 : N | Композиция | История движений баланса пользователя; не существует без пользователя |
| 9 | BalanceTransaction → Trade | 1 : 0..1 | Ассоциация (опциональная) | Не каждая транзакция баланса связана со сделкой (пополнение — не связано) |
| 10 | User → PaymentTransaction | 1 : N | Композиция | История внешних платежей пользователя |
| 11 | PaymentTransaction → BalanceTransaction | 1 : 0..1 | Ассоциация (опциональная) | Успешное внешнее пополнение порождает запись в балансе |

**Примечание по типам связей:** BalanceTransaction и PaymentTransaction связаны с User композицией — они не имеют смысла и не должны существовать в отсутствие пользователя, которому принадлежат (при удалении User удаляется вся его история). Остальные связи — обычные ассоциации, так как сущности на обеих сторонах могут существовать независимо друг от друга (например, Item и Trade имеют собственный жизненный цикл).

---

## 5. Ограничения и бизнес-правила

### Ограничения целостности

- Item может участвовать в новом Trade только если его текущий статус допускает это: `in_platform_pool` — для покупки пользователем, либо предмет пользователя не находится под Steam trade-hold и не зарезервирован другой активной сделкой — для продажи платформе.
- У одного Item не может быть двух одновременно активных Trade.
- `Trade.agreedPrice` фиксируется из актуального PriceQuote на момент создания сделки и не изменяется впоследствии, даже если котировка обновится до завершения сделки.
- BalanceTransaction типа PURCHASE_DEBIT или WITHDRAWAL не может привести к отрицательному балансу пользователя.
- Баланс пользователя и статусы предметов не должны расходиться с реальным состоянием системы даже при сбоях (отказ бота, недоступность Steam API).
- Исключена ситуация, при которой два пользователя одновременно покупают один и тот же предмет.
- Аутентификация через Steam, защита платежей и внутреннего баланса от мошеннических операций.
- Витрина доступных предметов отражает реальный пул платформы без устаревших предложений.

### Бизнес-правила

1. Цена выкупа всегда ниже цены продажи (`buyPrice < sellPrice`) — разница составляет маржу платформы.
2. Платформа не принимает к выкупу предмет, находящийся под Steam trade-hold (например, после смены пароля или email — холд действует 7 дней).
3. Купить можно только Item со статусом `in_platform_pool` (выкупленный и не зарезервированный под другую сделку).
4. При создании Trade соответствующий Item резервируется (переходит в статус `in_transfer`/`reserved`), чтобы исключить одновременную покупку одного предмета двумя пользователями.
5. Trade имеет ограничение по времени на подтверждение (например, 10 минут на принятие трейд-оффера в Steam). Если пользователь не успевает — Trade переходит в статус `expired`, Item возвращается в пул (при покупке) или сделка отменяется (при продаже).
6. Bot не принимает новый Trade, если `currentLoad >= maxConcurrentTrades` или `status != online`.
7. Вывод средств возможен только если сумма вывода не превышает текущий баланс пользователя и аккаунт не находится в статусе `banned`.
8. Просроченная котировка (`validUntil` в прошлом) не может быть использована для создания нового Trade — требуется запросить актуальную.
9. Пользователь без указанного `tradeUrl` не может ни продавать, ни покупать предметы.

---

# Этап 2: Проектирование API и контрактов

**Проект:** Платформа для покупки и продажи скинов CS2 (по аналогии с Lis-Skins)
**Трек:** Альтернативный 
**Подход:** REST / JSON, без gRPC и sequence diagrams 

---

## 1. Декомпозиция на сервисы

| Сервис | Зона ответственности | Сущности из Этапа 1 |
|---|---|---|
| **User Service** | Аутентификация через Steam, профиль и статус пользователя | User |
| **Catalog Service** | Каталог типов скинов, формирование и история котировок | SkinTemplate, PriceQuote |
| **Inventory Service** | Учёт конкретных предметов и их статусов | Item |
| **Trade Service** | Оркестрация сделок, управление трейд-ботами | Trade, Bot |
| **Wallet Service** | Внутренний баланс, внешние платежи | BalanceTransaction, PaymentTransaction |

**Схема взаимодействия:** Trade Service взаимодействует с остальными — при создании сделки обращается к User (проверка статуса), Catalog (получение котировки), Inventory (резервирование предмета) и Wallet (списание/зачисление средств). Остальные сервисы между собой напрямую не взаимодействуют.

---

## 2. API сервисов

### 2.1 User Service

| Метод | Путь | Описание |
|---|---|---|
| POST | `/auth/steam/login` | Аутентификация через Steam OpenID |
| GET | `/users/{id}` | Профиль пользователя |
| PATCH | `/users/{id}/trade-url` | Обновление трейд-ссылки |
| GET | `/users/{id}/status` | Проверка статуса (используется другими сервисами) |

**POST /auth/steam/login**
- Запрос: `{ "steamOpenIdToken": "string" }`
- Ответ `200`: `{ "userId": "uuid", "accessToken": "string", "isNewUser": "boolean" }`
- Ошибки: `401 INVALID_STEAM_TOKEN`

**GET /users/{id}**
- Ответ `200`: `User`
- Ошибки: `404 USER_NOT_FOUND`

**PATCH /users/{id}/trade-url**
- Запрос: `{ "tradeUrl": "string" }`
- Ответ `200`: `User`
- Ошибки: `400 INVALID_TRADE_URL`, `404 USER_NOT_FOUND`

**GET /users/{id}/status**
- Ответ `200`: `{ "status": "active|banned|trade_restricted", "hasValidTradeUrl": "boolean" }`
- Ошибки: `404 USER_NOT_FOUND`

---

### 2.2 Catalog Service

| Метод | Путь | Описание |
|---|---|---|
| GET | `/skin-templates` | Список типов скинов (каталог) |
| GET | `/skin-templates/{id}` | Детали типа скина |
| GET | `/skin-templates/{id}/quote` | Текущая котировка |
| GET | `/skin-templates/{id}/quotes/history` | История котировок |
| GET | `/quotes/{quoteId}` | Котировка по id |

**GET /skin-templates**
- Query: `weapon`, `rarity`, `page`, `limit`
- Ответ `200`: `{ "items": [SkinTemplate], "page": "number", "total": "number" }`

**GET /skin-templates/{id}**
- Ответ `200`: `SkinTemplate`
- Ошибки: `404 SKIN_TEMPLATE_NOT_FOUND`

**GET /skin-templates/{id}/quote**
- Ответ `200`: `PriceQuote`
- Ошибки: `404 SKIN_TEMPLATE_NOT_FOUND`, `503 QUOTE_CALCULATION_FAILED`

**GET /skin-templates/{id}/quotes/history**
- Query: `from`, `to`
- Ответ `200`: `{ "items": [PriceQuote] }`

**GET /quotes/{quoteId}**
- Ответ `200`: `PriceQuote`
- Ошибки: `404 QUOTE_NOT_FOUND`, `410 QUOTE_EXPIRED`

---

### 2.3 Inventory Service

| Метод | Путь | Описание |
|---|---|---|
| GET | `/users/{userId}/steam-inventory` | Steam-инвентарь пользователя |
| GET | `/items/pool` | Витрина предметов платформы |
| GET | `/items/{id}` | Детали предмета |
| POST | `/items` | Создание записи предмета (при выкупе) |
| PATCH | `/items/{id}/status` | Изменение статуса предмета |

**GET /users/{userId}/steam-inventory**
- Ответ `200`: `{ "items": [{ "assetId", "skinTemplateId", "float", "exterior", "stickers": ["string"], "tradable": "boolean" }] }`
- Ошибки: `404 STEAM_INVENTORY_UNAVAILABLE`

**GET /items/pool**
- Query: `skinTemplateId`, `minFloat`, `maxFloat`, `page`, `limit`
- Ответ `200`: `{ "items": [Item], "page": "number", "total": "number" }`

**GET /items/{id}**
- Ответ `200`: `Item`
- Ошибки: `404 ITEM_NOT_FOUND`

**POST /items**
- Запрос: `{ "skinTemplateId", "assetId", "float", "exterior", "stickers": ["string"], "pattern", "ownerUserId" }`
- Ответ `201`: `Item` (статус `in_steam_user`)
- Ошибки: `400 INVALID_ITEM_DATA`

**PATCH /items/{id}/status**
- Запрос: `{ "status": "reserved|in_transfer|in_platform_pool|sold", "expectedCurrentStatus": "string" }`
- Ответ `200`: `Item`
- Ошибки: `404 ITEM_NOT_FOUND`, `409 STATUS_CONFLICT` (если `expectedCurrentStatus` не совпал с реальным)

---

### 2.4 Trade Service

| Метод | Путь | Описание |
|---|---|---|
| POST | `/trades/sell` | Инициировать продажу платформе |
| POST | `/trades/buy` | Инициировать покупку у платформы |
| GET | `/trades/{id}` | Статус и детали сделки |
| GET | `/users/{userId}/trades` | История сделок пользователя |
| POST | `/trades/{id}/cancel` | Отмена сделки |
| PATCH | `/trades/{id}/status` | Обновление статуса (вызывается ботом) |
| GET | `/bots` | Список ботов |
| GET | `/bots/available` | Найти свободного бота |

**POST /trades/sell**
- Запрос: `{ "userId", "assetId", "skinTemplateId", "quoteId" }`
- Ответ `201`: `{ "tradeId", "status": "pending", "agreedPrice": "number", "botSteamId": "string" }`
- Ошибки: `400 INVALID_REQUEST`, `403 USER_NOT_ELIGIBLE`, `409 QUOTE_EXPIRED`, `404 NO_AVAILABLE_BOTS`

**POST /trades/buy**
- Запрос: `{ "userId", "itemId" }`
- Ответ `201`: `{ "tradeId", "status": "pending", "agreedPrice": "number", "botSteamId": "string" }`
- Ошибки: `403 USER_NOT_ELIGIBLE`, `404 ITEM_NOT_FOUND`, `409 ITEM_ALREADY_RESERVED`, `409 INSUFFICIENT_BALANCE`, `404 NO_AVAILABLE_BOTS`

**GET /trades/{id}**
- Ответ `200`: `Trade`
- Ошибки: `404 TRADE_NOT_FOUND`

**GET /users/{userId}/trades**
- Query: `type`, `status`, `page`, `limit`
- Ответ `200`: `{ "items": [Trade], "page": "number", "total": "number" }`

**POST /trades/{id}/cancel**
- Ответ `200`: `{ "id", "status": "cancelled" }`
- Ошибки: `409 TRADE_CANNOT_BE_CANCELLED` (если сделка уже в финальном статусе)

**PATCH /trades/{id}/status**
- Запрос: `{ "status": "bot_processing|awaiting_user_action|completed|failed|expired", "botId" }`
- Ответ `200`: `Trade`
- Ошибки: `409 INVALID_STATUS_TRANSITION` 

**GET /bots**
- Ответ `200`: `{ "items": [Bot] }`

**GET /bots/available**
- Ответ `200`: `{ "botId", "steamId" }`
- Ошибки: `404 NO_AVAILABLE_BOTS`

---

### 2.5 Wallet Service

| Метод | Путь | Описание |
|---|---|---|
| GET | `/wallets/{userId}` | Текущий баланс |
| GET | `/wallets/{userId}/transactions` | История транзакций баланса |
| POST | `/wallets/{userId}/topup` | Пополнение баланса |
| POST | `/wallets/{userId}/withdraw` | Вывод средств |
| POST | `/wallets/internal/debit` | Внутреннее списание (вызывается Trade Service) |
| POST | `/wallets/internal/credit` | Внутреннее зачисление (вызывается Trade Service) |

**GET /wallets/{userId}**
- Ответ `200`: `{ "userId", "balance": "number", "currency": "string" }`
- Ошибки: `404 WALLET_NOT_FOUND`

**GET /wallets/{userId}/transactions**
- Query: `type`, `page`, `limit`
- Ответ `200`: `{ "items": [BalanceTransaction], "page": "number", "total": "number" }`

**POST /wallets/{userId}/topup**
- Запрос: `{ "amount": "number", "provider": "string" }`
- Ответ `200`: `{ "paymentTransactionId", "status": "success", "newBalance": "number" }`
- Ошибки: `400 INVALID_AMOUNT`, `502 PAYMENT_PROVIDER_ERROR`

**POST /wallets/{userId}/withdraw**
- Запрос: `{ "amount": "number", "provider": "string", "destination": "string" }`
- Ответ `200`: `{ "paymentTransactionId", "status": "success", "newBalance": "number" }`
- Ошибки: `400 INVALID_AMOUNT`, `409 INSUFFICIENT_BALANCE`

**POST /wallets/internal/debit**
- Запрос: `{ "userId", "amount": "number", "relatedTradeId" }`
- Ответ `200`: `{ "balanceTransactionId", "newBalance": "number" }`
- Ошибки: `409 INSUFFICIENT_BALANCE`

**POST /wallets/internal/credit**
- Запрос: `{ "userId", "amount": "number", "relatedTradeId" }`
- Ответ `200`: `{ "balanceTransactionId", "newBalance": "number" }`

---

## 3. Схемы данных 

```
User
{
  "id": "uuid",
  "steamId": "string",
  "displayName": "string",
  "avatarUrl": "string",
  "tradeUrl": "string | null",
  "balance": "number",
  "status": "active | banned | trade_restricted"
}

SkinTemplate
{
  "id": "uuid",
  "weapon": "string",
  "skinName": "string",
  "rarity": "string",
  "collection": "string",
  "basePrice": "number"
}

PriceQuote
{
  "quoteId": "uuid",
  "skinTemplateId": "uuid",
  "buyPrice": "number",
  "sellPrice": "number",
  "validUntil": "ISO-8601 datetime",
  "source": "string"
}

Item
{
  "id": "uuid",
  "skinTemplateId": "uuid",
  "assetId": "string",
  "float": "number",
  "exterior": "string",
  "stickers": ["string"],
  "pattern": "number | null",
  "status": "in_steam_user | reserved | in_transfer | in_platform_pool | sold"
}

Trade
{
  "id": "uuid",
  "type": "SELL_TO_PLATFORM | BUY_FROM_PLATFORM",
  "userId": "uuid",
  "itemId": "uuid",
  "botId": "uuid",
  "agreedPrice": "number",
  "status": "pending | bot_processing | awaiting_user_action | completed | failed | cancelled | expired",
  "createdAt": "ISO-8601 datetime",
  "completedAt": "ISO-8601 datetime | null"
}

Bot
{
  "id": "uuid",
  "steamId": "string",
  "status": "online | offline | busy | trade_banned",
  "currentLoad": "number",
  "maxConcurrentTrades": "number"
}

BalanceTransaction
{
  "id": "uuid",
  "userId": "uuid",
  "type": "TOPUP | WITHDRAWAL | SALE_CREDIT | PURCHASE_DEBIT | REFUND",
  "amount": "number",
  "status": "string",
  "relatedTradeId": "uuid | null",
  "createdAt": "ISO-8601 datetime"
}

PaymentTransaction
{
  "id": "uuid",
  "userId": "uuid",
  "direction": "in | out",
  "amount": "number",
  "provider": "string",
  "externalTxId": "string",
  "status": "pending | success | failed"
}
```

---

## 4. Переходы статусов Trade

Допустимые переходы статуса сделки (любой переход вне этой таблицы запрещён и возвращает `409 INVALID_STATUS_TRANSITION`):

| Из статуса | В статус | Когда происходит |
|---|---|---|
| — | `pending` | Сделка создана (POST /trades/sell или /trades/buy) |
| `pending` | `bot_processing` | Бот взял сделку в работу |
| `pending` | `cancelled` | Пользователь отменил сделку до начала обработки |
| `bot_processing` | `awaiting_user_action` | Бот отправил трейд-оффер, ждёт принятия в Steam |
| `bot_processing` | `failed` | Бот не смог создать трейд-оффер (например, Steam API недоступен) |
| `awaiting_user_action` | `completed` | Пользователь принял трейд-оффер |
| `awaiting_user_action` | `expired` | Истёк таймаут на принятие трейда |
| `awaiting_user_action` | `cancelled` | Пользователь отменил сделку до принятия трейда |
| `failed`, `expired`, `cancelled` | — | Финальные состояния, дальнейшие переходы недопустимы |
| `completed` | — | Финальное состояние |

---

## 5. Общие коды ошибок

Единый формат тела ошибки для всех сервисов:

```
{
  "error": "ERROR_CODE",
  "message": "Человекочитаемое описание"
}
```

| HTTP-код | Когда используется |
|---|---|
| `400` | Невалидные входные данные запроса |
| `401` | Не пройдена аутентификация |
| `403` | Пользователь аутентифицирован, но не имеет права на действие (бан, ограничение) |
| `404` | Сущность не найдена (включая случай отсутствия доступных ботов) |
| `409` | Конфликт состояния (недостаточно средств, предмет уже зарезервирован, недопустимый переход статуса) |
| `410` | Запрошенный ресурс больше не действителен (просроченная котировка) |
| `502` | Сбой при обращении к внешнему провайдеру (платёжная система) |
| `503` | Сервис временно не может обработать запрос (не удалось рассчитать котировку) |
