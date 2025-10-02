# WebLab#1 — Анализ и проектирование (Биржа коллекционных вещей для СНГ/Азии)

**Проект**: Биржа коллекционной одежды и электроники для рынков СНГ и Азии.  
**Ключевое отличие**: выставлять лоты можно **только** из предзаполненного системного каталога (никаких произвольных товаров). Это обеспечивает быстрый поиск и контроль предметной области на физическом уровне сервиса. Биржевая модель позволяет формализовать взаимодействие между пользователями на платформе. 
__Замечание:__ В теории биржевая модель может позволить подключать третьи страны.
---

## Цель и решаемая проблема / предоставляемая возможность

**Цель** — создать торговую площадку для коллекционных вещей по принципу биржи, где:
- исключена энтропия наименований товаров за счёт **строгого каталога товаров**;
- снижена вероятность получить неподлинный товар за счет **верификации** товара;
- сделки проходят через **биржевую модель** (ставки/офферы, клиринг, комиссия);
- есть **верификация** подлинности и **логистическая цепочка** (селлер → проверка → покупатель).

**Проблема рынка**: на классических досках объявлений (Avito, OLX, eBay, Wallapop, FacebookMarket) высокая энтропия ассортимента, а также проверка товара происходит посредством **видео, фото** верификации, не учитывая физические материалы товара (актуально для одежды). Взаимодействие между пользователями происходит посредством общения в чатах, что также добавляет риски быть обманутым. Биржевая модель несёт ответственность за подлинность товара и анонимность участников сделки.

**Возможность**: «биржевой» UX, физическая проверка подлинности, все взаимодействия посредством биржи.

## Сравнение
- ![сравнение](img/1.png)


## Краткий перечень функциональных требований

1. **Каталог**
   - CRUD для сущностей каталога: Бренд, Модель, Артикул (SKU), Выпуск/Релиз, Цвет, Размер/Конфигурация, Наименование.
   - Импорт/версионирование справочника, модерация карточек (роль: Редактор/Админ).
   - Поиск и фильтры: бренд, модель, размер, состояние (**только** new/ **дополнительно**used).

2. **Биржевые операции**
   - **Bid** (покупательская заявка) и **Ask** (продающая заявка) на конкретный `CatalogItem`.
   - Автосовпадение по правилам книги заявок (order book): лучшая цена/время.
   - Завершение сделки → создание **Trade** (с фиксацией комиссии и налогов).

3. **Уведомления и подписки**
    - **Подписка** на товар для отслеживания изменения.
    - **Уведомление** о достижении целовой цены. 

4. **Аудит и мониторинг**
   - Логи аутентификации, бизнеса (ордера, трейды), доставки, ошибок.
   - Сводки в `/monitoring`.

5. **Платежи/выплаты**
   - Резервирование средств покупателя (escrow), списание комиссии, **Payout** селлеру.
   - Интеграция с провайдером платежей; моки/эмуляторы.

6. **Профили и роли**
   - Пользователи: Покупатель, Продавец, Модератор/Аутентификатор, Редактор, Админ.
   - KYC/верификация аккаунта, 2FA. Сессии, JWT (для последующих ЛР).

7. **Нефункциональные требования**
   - Производительность каталога и поисковых фильтров.
   - Надёжность workflow сделки (идемпотентность вебхуков/ретраев).
   - Масштабируемость (деление на сервисы), отказоустойчивость.

8. **Сделка и логистика __Доп__**
   - Отправка товара селлером в верификационный центр → **VerificationReport** (подлинность/состояние).
   - Если OK — пересылка покупателю (**Shipment**), если нет — возврат/эскалация.
   - Отслеживание статусов, SLA, трекинг отправлений.


## Usecase диаграмма 
![usecase](img/usecases.png)


## BPMN #1 — Модерация каталога (нелинейный процесс)

```mermaid
flowchart TD
  A[Запрос на новую карточку CatalogItem] --> B{Проверка дубликатов?}
  B -- Да --> C[Отклонить / Мерж]
  B -- Нет --> D[Проверка атрибутов: бренд/модель/SKU/фото]
  D --> E{Достаточно метаданных?}
  E -- Нет --> F[Запросить уточнение автору]
  F --> D
  E -- Да --> G[Публикация CatalogItem vX.Y]
  G --> H[Индексировать поиск + кэш]
  H --> I[Готово]
```
![bpmn1](img/bpmn1.png)


## BPMN #2 — Биржевая сделка (Bid/Ask → Trade)

```mermaid
flowchart TD
  A[Создать Bid/Ask] --> B{Есть контр‑заявка?}
  B -- Нет --> C[Поместить в order book]
  B -- Да --> D[Сверка цен и атрибутов]
  D --> E{Совпадение?}
  E -- Нет --> C
  E -- Да --> F[Резерв средств escrow]
  F --> G{Успех платежа?}
  G -- Нет --> C
  G -- Да --> H[Создать Trade + ShipmentToVerifier]
  H --> I[Уведомить стороны]
```
![bpmn2](img/bpmn2.png)

## BPMN #3 — Покупка товара

```mermaid
flowchart TD
  A[Создание Bid заявка на покупку] --> B[Матчинг с Ask]
  B --> C[Блокировка средств Escrow]
  C --> D[Ожидание верификации товара]
  D --> E{Результат проверки}
  E -- PASS --> F[Доставка товара покупателю]
  F --> G[Получение товара]
  G --> H[Закрытие сделки]
  E -- FAIL --> I[Возврат покупателю]
  I --> H
```
![bpmn3](img/bpmn3.png)

## BPMN #4 — Продажа товара

```mermaid
flowchart TD
  A[Создание Ask заявка на продажу] --> B[Матчинг с Bid]
  B --> C[Продавец получает этикетку от платформы]
  C --> D[Отправка товара в центр проверки]
  D --> E[Верификация товара]
  E --> F{Результат проверки}
  F -- PASS --> G[Выплата продавцу ]
  G --> H[Закрытие сделки]
  F -- FAIL --> I[Возврат товара продавцу]
  I --> H
```
![bpmn4](img/bpmn4.png)

## Примеры пользовательских сценариев (3+)

1. **Покупка «в один клик по лучшей цене»**
   - Пользователь находит `CatalogItem`, жмёт «Купить по лучшей цене» → система создаёт Bid, мэчит с лучшим Ask, блокирует средства, запускает логистику и верификацию.

2. **Продажа «в один клик по лучшей цене»**
- Пользователь находит `CatalogItem`, жмёт «Продать по лучшей цене» → система создаёт Ask, мэчит с лучшим Bid, создает сделку, запускает логистику и верификацию.

3. **Выставление Ask c авто‑продлением**
   - Продавец выбирает `CatalogItem`, указывает данные, включает авто‑обновление цены при отсутствии спроса, система управляет Ask в книге заявок.

4. **Выставление Bid c авто‑продлением**
    - Покупатель выбирает `CatalogItem`, указывает таргетную цены, система добавляет Bid книгу предожений.

5. **Подписка на изменение цены**
    - Пользователь заходит в приложение. Выбирает интересующий товар и подписывается на изменение цены.

6. **Уведомление о достижении цены**
    - Пользователь заходит в приложение. Выбирает интересующий товар и создает уведомление для ценового уровня товара.

**__Доп__**
1. **Эскалация при провале проверки**
    - Верификатор фиксирует несоответствие → система останавливает сделку, уведомляет стороны, инициирует возврат и черный список для селлера при повторных нарушениях.

2. **Импорт релизов по бренду**
    - Редактор импортирует CSV от бренда → система валидирует, ищет дубликаты, публикует новые SKU с версионированием.


## ER‑диаграмма (Mermaid ER)

```mermaid
erDiagram
  USER ||--o{ BID : places
  USER ||--o{ ASK : places
  USER ||--o{ WALLET : owns
  USER }o--o{ ROLE : has

  CATALOG_ITEM ||--o{ ASK : for
  CATALOG_ITEM ||--o{ BID : for
  CATALOG_ITEM ||--o{ VERIFICATION_REPORT : about

  BID ||--o| TRADE : matched_to
  ASK ||--o| TRADE : matched_to

  TRADE ||--o| SHIPMENT : has
  TRADE ||--o| PAYOUT : results_in
  TRADE ||--o| PAYMENT : uses

  VERIFICATION_REPORT ||--o| TRADE : gates

  PAYMENT {
    uuid id PK
    uuid user_id FK
    uuid trade_id FK
    decimal amount
    text status
    timestamptz created_at
  }

  PAYOUT {
    uuid id PK
    uuid seller_id FK
    uuid trade_id FK
    decimal amount_net
    text status
    timestamptz created_at
  }

  SHIPMENT {
    uuid id PK
    uuid trade_id FK
    text direction  "SELLER->VERIFIER or VERIFIER->BUYER"
    text carrier
    text tracking
    text status
    timestamptz created_at
  }

  VERIFICATION_REPORT {
    uuid id PK
    uuid catalog_item_id FK
    uuid trade_id FK
    text result "PASS/FAIL"
    text notes
    jsonb checklist
    timestamptz created_at
  }

  CATALOG_ITEM {
    uuid id PK
    text brand
    text model
    text sku
    text colorway
    text size_conf
    text region
    jsonb meta
    timestamptz published_at
  }

  BID {
    uuid id PK
    uuid user_id FK
    uuid catalog_item_id FK
    decimal price
    text currency
    int quantity
    text status
    timestamptz created_at
  }

  ASK {
    uuid id PK
    uuid user_id FK
    uuid catalog_item_id FK
    decimal price
    text currency
    int quantity
    text condition "NEW/USED/…"
    text status
    timestamptz created_at
  }

  TRADE {
    uuid id PK
    uuid bid_id FK
    uuid ask_id FK
    decimal price_final
    decimal fee_buyer
    decimal fee_seller
    text status
    timestamptz created_at
  }

  USER {
    uuid id PK
    text email
    text phone
    text display_name
    text locale
    text region
    text kyc_status
    timestamptz created_at
  }

  ROLE {
    text code PK "BUYER, SELLER, VERIFIER, CM, ADMIN"
  }

  WALLET {
    uuid id PK
    uuid user_id FK
    text provider
    text account_ref
    text status
    timestamptz created_at
  }
```


## Технологический стек

- **Backend**: Kotlin + Ktor (чистая архитектура, Repository;), DAO, Coroutines.
- **БД**: PostgreSQL 16, схемы/миграции (Flyway), JSONB для гибких атрибутов, индексы GIN.
- **Кэш/поиск**: Redis (кэш фильтров и популярных запросов); по желанию — Meilisearch/Elastic.
- **API**: REST /api/v1 (OpenAPI 3.1), подготовка к /api/v2. JWT.
- **Gateway/Web‑server**: Nginx (маршрутизация, статика, gzip, кэш), подготовка к балансировке .
- **Messaging (доп)**: RabbitMQ асинхронные события сделок/логистики; 
- **Frontend (доп)**: React + TypeScript + Vite (SPA, Remote Facade), Jest/Vitest.
- **CI/CD**: GitHub Actions (линтеры, тесты, сборка docker‑образов).
- **Observability**: Grafana + Loki, Prometheus + exporters (подготовка к /monitoring).
- **Delivery**: Docker Compose (стенд).

## Диаграмма БД (ключевые связи и ограничения)

- Все PK — `uuid`; внешние ключи с каскадами там, где это безопасно (например, удаление пользователя **не** каскадит сделки).
- Уникальные ограничения: `CatalogItem(sku, size_conf, region)`, `Role(code)`, уникальный `email` у `User`.
- Индексы: B‑tree по внешним ключам; `gin` по `CatalogItem.meta` и `VerificationReport.checklist`.
- Идемпотентность: таблица `payment` хранит `provider_request_id` с `UNIQUE` для защиты от дублей.


## Компонентная диаграмма

```mermaid
flowchart LR
  Browser[SPA (React TS)] -->|REST /api/v1| API[Ktor API Gateway]
  API --> Core[Core Service (Domain)]
  Core --> DS[Data Service (PostgreSQL + Redis)]
  Core --> MQ[(RabbitMQ)]
  VerifierApp[Verifier Console/TUI] -->|Events| MQ
  AdminUI[Admin UI] -->|REST| API
  Monitoring[(Grafana+Loki)] <-->|logs/metrics| API
  Monitoring <-->|logs/metrics| Core
  Monitoring <-->|logs/metrics| DS
```


## Черновые эскизы экранов (wireframes)


