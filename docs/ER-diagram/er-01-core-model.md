# ER-01. Ядро модели данных «Car Import Tracker»

Документ описывает диаграмму ER-01 (`ER-diagram_drawio.png`, исходник — `ER-diagram.drawio`): 15 таблиц, которые покрывают основной путь заказа от заявки до передачи автомобиля, платежи, претензии и служебные данные. Остальные таблицы полной модели описаны в [schema-design.md](./schema-design.md) и на этой диаграмме не показаны (раздел 1).

![ER-01. Ядро модели данных](ER-diagram_drawio.png)

Соглашения: имена в `snake_case` на английском, первичные ключи `bigint` с автогенерацией, деньги в `numeric(12,2)` (рубли), время в `timestamptz`. Таблица пользователей называется `app_user`, потому что `user` в PostgreSQL зарезервировано.

---

## 1. Состав диаграммы

| Группа | Таблицы | Назначение |
|---|---|---|
| Пользователи и доступ | `app_user`, `consent`, `user_notification`, `audit_log` | Учётные записи, согласия на обработку данных, каналы уведомлений, журнал действий |
| Заказ и статусы | `customer_order`, `order_status`, `order_status_history`, `tracking_status`, `tracking_event` | Заказ, его этапы и подстатусы логистики и таможни |
| Подбор и осмотр | `car_variant`, `inspection_report`, `media_file` | Варианты автомобилей, отчёт об осмотре, фото и видео |
| Платежи и претензии | `payment_type`, `payment`, `claim` | Платежи и претензии клиента |

**Вне диаграммы ER-01** (описаны в schema-design.md):

| Таблица | Назначение | Требования |
|---|---|---|
| `document`, `document_type` | Документы заказа с типом и этапом | FR-10, FR-13, FR-18 |
| `refund_request` | Запрос на возврат средств | FR-05 |
| `photo_category` | Категории фото отчёта | FR-06 |
| `order_option` | Дополнительные опции заказа | FR-09 |
| `catalog_item` | Каталог доставленных автомобилей | FR-17 |
| `chat_message` | Чат заказа | FR-18 |
| `notification_log` | Журнал отправки уведомлений | FR-15, NFR-05 |

Поля, которые в полной модели появляются в таблицах диаграммы за счёт этих сущностей (`media_file.photo_category_id`, `media_file.catalog_item_id`, `payment.refund_request_id`), на ER-01 тоже не показаны.

---

## 2. Ключевые решения

| Вопрос | Решение | Почему |
|---|---|---|
| Заказ и автомобиль | У заказа несколько вариантов (`car_variant`), принятым может быть только один | Процесс допускает возврат к подбору (FR-04, FR-07) |
| Параметры заявки и вариант | Заявка хранится в заказе (`req_*`), предложенные автомобили — в вариантах | Это разные факты: что хотел клиент и что предложил агент |
| Пользователи | Одна таблица `app_user` с ролью | Клиент, агент и администратор делят вход, e-mail и телефон; специфичных полей у ролей нет |
| Паспорт клиента | Полей нет, только копия как документ | Номер и серия системе не нужны (минимизация данных по 152-ФЗ) |
| Статусы заказа | Справочник `order_status` и неизменяемая история | История не должна меняться задним числом (FR-14) |
| Логистика и таможня | Отдельные справочник и таблица событий | Это подстатусы с собственным набором значений и обязательной причиной (FR-11, FR-12) |
| Статусы платежей, претензий, решений | Ограничения `CHECK` | Значения фиксированы логикой процесса |
| Типы платежей | Справочник `payment_type` | FR-19: администратор управляет типами платежей |
| Файлы | В БД только метаданные, сам файл в хранилище (`storage_key`) | Файлы большие, нужно шифрование (NFR-02) |
| Фото и видео | Одна таблица `media_file`, у записи ровно один владелец | Единые проверки и единое удаление |
| Сроки хранения | Поля `completed_at`, `decided_at`, `purged_at` | Из них считаются сроки таблицы 5.1 требований |

---

## 3. Связи

| Дочерняя таблица.поле | Родительская таблица | Кратность | Правило |
|---|---|---|---|
| `customer_order.client_id` | `app_user` | N : 1, обязательная | Роль `client` проверяется в приложении |
| `customer_order.agent_id` | `app_user` | N : 0..1 | Назначается, когда агент берёт заказ |
| `customer_order.current_status_id` | `order_status` | N : 1, обязательная | Текущее значение; история отдельно |
| `order_status_history.order_id` | `customer_order` | N : 1, обязательная | |
| `order_status_history.status_id` | `order_status` | N : 1, обязательная | |
| `order_status_history.changed_by` | `app_user` | N : 0..1 | Пусто, если изменила система |
| `tracking_event.order_id` | `customer_order` | N : 1, обязательная | |
| `tracking_event.tracking_status_id` | `tracking_status` | N : 1, обязательная | |
| `tracking_event.recorded_by` | `app_user` | N : 1, обязательная | Агент, внёсший данные |
| `car_variant.order_id` | `customer_order` | N : 1, обязательная | Принятый вариант один на заказ |
| `car_variant.created_by` | `app_user` | N : 1, обязательная | |
| `inspection_report.variant_id` | `car_variant` | 0..1 : 1 | Один отчёт на вариант (`UNIQUE`) |
| `inspection_report.created_by` | `app_user` | N : 1, обязательная | |
| `media_file.variant_id` | `car_variant` | N : 0..1 | Ровно одно из трёх полей владельца |
| `media_file.report_id` | `inspection_report` | N : 0..1 | |
| `media_file.claim_id` | `claim` | N : 0..1 | |
| `media_file.uploaded_by` | `app_user` | N : 1, обязательная | |
| `payment.order_id` | `customer_order` | N : 1, обязательная | |
| `payment.payment_type_id` | `payment_type` | N : 1, обязательная | |
| `payment.recorded_by` | `app_user` | N : 1, обязательная | |
| `claim.order_id` | `customer_order` | N : 1, обязательная | У заказа может быть несколько претензий |
| `claim.submitted_by` | `app_user` | N : 1, обязательная | |
| `claim.decided_by` | `app_user` | N : 0..1 | Пусто, пока нет решения |
| `consent.user_id` | `app_user` | N : 1, обязательная | |
| `user_notification.user_id` | `app_user` | N : 1, обязательная | Часть составного ключа |
| `audit_log.user_id` | `app_user` | N : 0..1 | Пусто для системных событий |

Поведение при удалении: по умолчанию `RESTRICT`, данные заказа каскадом не удаляются. Исключения: `user_notification` и `media_file` удаляются вместе с владельцем (пользователем, вариантом, отчётом, претензией), так как при очистке по срокам хранения нужно удалять именно фото и видео.

---

## 4. Таблицы

### 4.1 Пользователи и доступ

**app_user** — пользователь любой роли.

| Поле | Тип | Ограничения | Комментарий |
|---|---|---|---|
| user_id | bigint | PK | |
| role | text | NOT NULL, CHECK in (`client`, `agent`, `admin`) | |
| full_name | text | NOT NULL | |
| email | text | NOT NULL, UNIQUE | Логин и канал уведомлений |
| phone | text | UNIQUE | Нужен для SMS |
| password_hash | text | NOT NULL | Только необратимый хэш (NFR-02) |
| is_active | boolean | NOT NULL, по умолчанию true | Блокировка вместо удаления (FR-19) |
| created_at | timestamptz | NOT NULL | |
| anonymized_at | timestamptz | | Заполняется при обезличивании (NFR-06, NFR-08) |

**consent** — согласие на обработку персональных данных.

| Поле | Тип | Ограничения | Комментарий |
|---|---|---|---|
| consent_id | bigint | PK | |
| user_id | bigint | FK → app_user, NOT NULL | |
| doc_version | text | NOT NULL | Версия текста согласия |
| accepted_at | timestamptz | NOT NULL | |
| revoked_at | timestamptz | | Отзыв фиксируется, запись не удаляется |

**user_notification** — каналы уведомлений пользователя.

| Поле | Тип | Ограничения | Комментарий |
|---|---|---|---|
| user_id | bigint | PK (часть), FK → app_user | |
| channel | text | PK (часть), CHECK in (`email`, `sms`, `telegram`) | Составной ключ: у пользователя до трёх строк |
| is_enabled | boolean | NOT NULL | Хотя бы один канал должен остаться включённым (правило приложения) |
| telegram_id | text | | Обязателен для включённого Telegram |

**audit_log** — журнал действий (NFR-03).

| Поле | Тип | Ограничения | Комментарий |
|---|---|---|---|
| audit_id | bigint | PK | |
| user_id | bigint | FK → app_user | |
| action | text | NOT NULL | Вход, смена статуса, платёж, доступ к документу |
| entity_type | text | | Например `customer_order` |
| entity_id | bigint | | |
| details | jsonb | | Подробности действия |
| created_at | timestamptz | NOT NULL | Срок хранения 1 год |

### 4.2 Заказ и статусы

**order_status** — справочник статусов заказа (13 значений, раздел 3 требований).

| Поле | Тип | Ограничения | Комментарий |
|---|---|---|---|
| status_id | bigint | PK | |
| code | text | NOT NULL, UNIQUE | Например `inspection`, `awaiting_payment` |
| name | text | NOT NULL | Название для интерфейса |
| sort_order | smallint | NOT NULL | Порядок этапов |
| is_final | boolean | NOT NULL | Истина для «Завершён» и «Отменён» |

**customer_order** — заказ клиента.

| Поле | Тип | Ограничения | Комментарий |
|---|---|---|---|
| order_id | bigint | PK | |
| client_id | bigint | FK → app_user, NOT NULL | |
| agent_id | bigint | FK → app_user | |
| current_status_id | bigint | FK → order_status, NOT NULL | Полная история в `order_status_history` |
| req_brand | text | NOT NULL | Параметры заявки (FR-02) |
| req_model | text | NOT NULL | |
| req_budget_rub | numeric(12,2) | NOT NULL, CHECK > 0 | |
| req_year | smallint | | Необязательно |
| req_mileage_max_km | integer | CHECK ≥ 0 | Необязательно |
| created_at | timestamptz | NOT NULL | |
| booked_at | timestamptz | | Бронирование (FR-09) |
| purchased_at | timestamptz | | Выкуп (FR-09) |
| handed_over_at | timestamptz | | Передача клиенту; от этой даты считаются 14 дней (FR-13, FR-16) |
| act_signed_at | timestamptz | | |
| act_signed_auto | boolean | NOT NULL, по умолчанию false | Истина, если акт считается подписанным по истечении срока |
| completed_at | timestamptz | | Начало отсчёта сроков хранения |

**order_status_history** — неизменяемая история статусов (FR-14).

| Поле | Тип | Ограничения | Комментарий |
|---|---|---|---|
| history_id | bigint | PK | |
| order_id | bigint | FK → customer_order, NOT NULL | Индекс (order_id, changed_at) |
| status_id | bigint | FK → order_status, NOT NULL | |
| changed_by | bigint | FK → app_user | |
| comment | text | | Например причина отклонения |
| changed_at | timestamptz | NOT NULL, по умолчанию now() | |

**tracking_status** — справочник подстатусов логистики и таможни.

| Поле | Тип | Ограничения | Комментарий |
|---|---|---|---|
| tracking_status_id | bigint | PK | |
| track_type | text | NOT NULL, CHECK in (`logistics`, `customs`) | |
| code | text | NOT NULL | UNIQUE вместе с `track_type` |
| name | text | NOT NULL | |
| sort_order | smallint | NOT NULL | |
| requires_reason | boolean | NOT NULL | Истина для «запрошены документы» (FR-12) |

**tracking_event** — события логистики и таможни.

| Поле | Тип | Ограничения | Комментарий |
|---|---|---|---|
| event_id | bigint | PK | |
| order_id | bigint | FK → customer_order, NOT NULL | |
| tracking_status_id | bigint | FK → tracking_status, NOT NULL | |
| reason | text | | Обязательна, если `requires_reason` |
| recorded_by | bigint | FK → app_user, NOT NULL | |
| recorded_at | timestamptz | NOT NULL, по умолчанию now() | Записи не меняются |

### 4.3 Подбор и осмотр

**car_variant** — вариант автомобиля, предложенный агентом.

| Поле | Тип | Ограничения | Комментарий |
|---|---|---|---|
| variant_id | bigint | PK | |
| order_id | bigint | FK → customer_order, NOT NULL | |
| brand | text | NOT NULL | |
| model | text | NOT NULL | |
| price_rub | numeric(12,2) | NOT NULL, CHECK > 0 | |
| production_year | smallint | | |
| mileage_km | integer | CHECK ≥ 0 | |
| colour | text | | |
| description | text | | |
| client_decision | text | NOT NULL, по умолчанию `pending`, CHECK in (`pending`, `accepted`, `rejected`) | |
| rejection_reason | text | | Необязательно (FR-04) |
| decided_at | timestamptz | | От него идёт срок 7 дней для отклонённых вариантов |
| created_by | bigint | FK → app_user, NOT NULL | |
| created_at | timestamptz | NOT NULL | |
| purged_at | timestamptz | | Когда удалены фото и описание (NFR-06) |

Частичный уникальный индекс: не более одного варианта со значением `accepted` на заказ.

**inspection_report** — отчёт об осмотре.

| Поле | Тип | Ограничения | Комментарий |
|---|---|---|---|
| report_id | bigint | PK | |
| variant_id | bigint | FK → car_variant, NOT NULL, UNIQUE | Один отчёт на вариант |
| matches_listing | boolean | NOT NULL | Заключение «соответствует объявлению» (FR-06) |
| comment | text | CHECK: при `matches_listing = false` обязателен | |
| created_by | bigint | FK → app_user, NOT NULL | |
| published_at | timestamptz | | Пусто, пока отчёт не опубликован |
| client_decision | text | CHECK in (`confirm`, `back_to_selection`, `cancel`) | Пусто до решения (FR-07) |
| decided_at | timestamptz | | |

**media_file** — фото и видео.

| Поле | Тип | Ограничения | Комментарий |
|---|---|---|---|
| file_id | bigint | PK | |
| kind | text | NOT NULL, CHECK in (`photo`, `video`) | |
| storage_key | text | NOT NULL, UNIQUE | Путь в хранилище |
| mime_type | text | NOT NULL | |
| size_bytes | bigint | NOT NULL | |
| variant_id | bigint | FK → car_variant | Владелец: ровно одно из трёх полей |
| report_id | bigint | FK → inspection_report | |
| claim_id | bigint | FK → claim | |
| uploaded_by | bigint | FK → app_user, NOT NULL | |
| uploaded_at | timestamptz | NOT NULL | |

Ограничение `CHECK`: заполнено ровно одно из полей `variant_id`, `report_id`, `claim_id`. В полной модели добавляется четвёртый владелец (`catalog_item_id`).

### 4.4 Платежи и претензии

**payment_type** — справочник типов платежей (FR-19).

| Поле | Тип | Ограничения | Комментарий |
|---|---|---|---|
| payment_type_id | bigint | PK | |
| code | text | NOT NULL, UNIQUE | `deposit`, `purchase_and_delivery`, `agent_fee`, `refund` |
| name | text | NOT NULL | |
| is_refund | boolean | NOT NULL | Определяет направление платежа |

**payment** — платёж или возврат.

| Поле | Тип | Ограничения | Комментарий |
|---|---|---|---|
| payment_id | bigint | PK | |
| order_id | bigint | FK → customer_order, NOT NULL | |
| payment_type_id | bigint | FK → payment_type, NOT NULL | |
| amount_rub | numeric(12,2) | NOT NULL, CHECK > 0 | Направление определяется типом |
| status | text | NOT NULL, CHECK in (`expected`, `received`) | |
| due_date | date | | Для напоминаний (FR-08) |
| paid_at | timestamptz | CHECK: обязателен при `received` | |
| recorded_by | bigint | FK → app_user, NOT NULL | |
| created_at | timestamptz | NOT NULL | |

**claim** — претензия.

| Поле | Тип | Ограничения | Комментарий |
|---|---|---|---|
| claim_id | bigint | PK | |
| order_id | bigint | FK → customer_order, NOT NULL | |
| description | text | NOT NULL | |
| status | text | NOT NULL, CHECK in (`submitted`, `accepted`, `rejected`, `settled`) | |
| submitted_by | bigint | FK → app_user, NOT NULL | |
| submitted_at | timestamptz | NOT NULL | Правило 14 дней проверяется по `customer_order.handed_over_at` |
| decision_reason | text | CHECK: обязательно при `rejected` | FR-16 |
| decided_by | bigint | FK → app_user | |
| decided_at | timestamptz | | |
| settled_at | timestamptz | | |

---

## 5. Трассировка таблиц к требованиям

| Таблицы | Требования |
|---|---|
| app_user | FR-01, FR-19 |
| consent | NFR-08 |
| user_notification | FR-15 |
| audit_log | NFR-03 |
| customer_order | FR-02, FR-09, FR-13 |
| order_status, order_status_history | FR-14, FR-04, FR-07 |
| tracking_status, tracking_event | FR-11, FR-12 |
| car_variant | FR-03, FR-04, NFR-06 |
| inspection_report | FR-06, FR-07 |
| media_file | FR-03, FR-06, FR-16 |
| payment_type, payment | FR-08 |
| claim | FR-16 |

---

## 6. Ограничения, которые не видны на диаграмме

Эти правила реализуются в `schema.sql` (индексы, `CHECK`, триггеры) или в приложении:

| Правило | Где реализуется |
|---|---|
| Один принятый вариант на заказ | Частичный уникальный индекс по `car_variant(order_id)` где `client_decision = 'accepted'` |
| У фото ровно один владелец | `CHECK` в `media_file` |
| Комментарий обязателен при `matches_listing = false` | `CHECK` в `inspection_report` |
| Причина обязательна при отказе по претензии | `CHECK` в `claim` |
| У платежа со статусом «получен» есть дата | `CHECK` в `payment` |
| Причина обязательна при `requires_reason` | Триггер или приложение |
| Претензию можно подать в течение 14 дней после передачи | Триггер или приложение |
| История статусов и события не меняются и не удаляются | Права доступа или триггер |
| Хотя бы один канал уведомлений включён | Приложение |
| Роль пользователя соответствует полю (`client_id` — клиент, `agent_id` — агент) | Триггер или приложение |

---

## 7. Проверка модели запросами

На диаграмме ER-01 можно ответить на следующие вопросы (запросы пишутся к `schema.sql`):

1. Вся история статусов заказа в хронологическом порядке с именами авторов.
2. Можно ли клиенту ещё подать претензию по заказу (14 дней с передачи)?
3. Какие отклонённые варианты старше 7 дней ещё не очищены?
4. Сколько клиент уже заплатил по каждому типу платежа и есть ли просроченные счета?
5. Текущий подстатус логистики и таможни для каждого активного заказа.

Остальные проверки (каталог, возвраты, документы, задержки уведомлений) требуют таблиц вне диаграммы ER-01.
