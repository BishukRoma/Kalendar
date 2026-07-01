# RealtyCalendar — Dev & Architecture Knowledge Base (Notion)

> Источники: Dev, Тех поддержка, Маркетинг, Проекты компании
> Дата: 2026-07-01

---

## 1. Архитектура RC — RealtyCalendar Platform (RCP)

### Стек
- **Frontend**: SPA (React)
- **Backend монолит**: Ruby on Rails
- **Platform Consumer**: Node.js / NestJS — обрабатывает события из Kafka
- **Platform API**: Rails + Grape API — принимает события от Consumer через HTTP
- **Event Bus**: Apache Kafka (Yandex Cloud Managed Service for Kafka)
- **Аналитика**: Amplitude (behavioral) + Snowflake (data warehouse)
- **БД миграции**: без lock timeout (используют `algorithm: :concurrently`)
- **CI**: RSpec, Rubocop

### Идеология разработки
- **Ruby on Rails монолит** + отдельный NestJS-потребитель Kafka
- **Event Driven Architecture (EDA)** — всё работает через события
- Паттерны (refactoring.guru): Abstract Factory, Bridge, Facade, Strategy, Template Method, Builder, Adapter
- Линтеры + постепенный рефакторинг «по касанию» (+20-50% времени на задачу → долгосрочно ускоряет)
- БД индексы — только `algorithm: :concurrently` в отдельной миграции с `disable_ddl_transaction!`

---

## 2. RCP — Процесс импорта броней с площадок

### Инициирование (Шаг 1)
**CronTask**: `CronTasks::Events::ImportExtranets` — запускает по крону, создаёт батч событий `Events::PlatformImport`.

`PlatformImport` создаёт либо `RequestHTTPPlatform` (для площадок с 1 запросом), либо `RequestHTTPIntegration` (для площадок с N запросами).

### Два типа площадок (Шаг 2)

**Тип A — один запрос (AbstractExtranetPlatform)**
Площадки: **CIAN, CBooking, Kufar**

```
ImportModule::PlatformRequestFactory.call(platform)
→ Events::RequestHTTPPlatform
  → success: Events::PlatformImportedSuccessful
  → fail:    Events::PlatformImportFailed
```

**Тип B — много запросов per integration (AbstractPlatform)**
Площадки: **OneTwoTrip, 101Hotels, Avito**

```
ImportModule::IntegrationRequestFactory.call(integration)
→ Events::RequestHTTPIntegration
  → success: Events::IntegrationImported
  → fail:    Events::IntegrationImportFailed (+ счётчик ошибок → invalid статус)
```

Оба пути на выходе дают поток `Events::EventCalendarImport` + `Events::EventCalendarImportInvalid`.

### Обработка брони (Шаг 3)
`Events::EventCalendarImport` содержит:
- `Types::RemoteBooking` — унифицированная структура для всех площадок
- `ImportModule::LocalBooking` — bridge к локальной записи брони

**Стратегии** (через `ImportModule::StrategyFactory`):
- `Strategy::Create` — новый EventCalendar
- `Strategy::Update` — обновление
- `Strategy::Cancel` — отмена
- `Strategy::CreateCanceled` — создать уже отменённым
- `Strategy::Noop` — ничего не делать

После стратегии:
- `ImportModule::NotifierFactory` → уведомления (Extranet или Unified)
- `ImportModule::EventFactory` → исходящие события:
  - `Events::EventCalendarCreated`
  - `Events::EventCalendarUpdated`
  - `Events::EventCalendarCanceled`

---

## 3. RCP — Процесс экспорта на площадки

### Инициирование (Шаг 1)
Источники событий:
- Процесс импорта выше
- **Пользователь RC** (создал/изменил/отменил бронь в интерфейсе)

```
EventCalendarCreated / Updated / Canceled
→ Events::EventCalendarExport
→ Events::ApartmentExport
→ Events::SynchronizationExport × (N интеграций × M типов синхронизации)
```

**Пример**: квартира с CIAN + OneTwoTrip + Hotels101:
- OneTwoTrip: Availability + Prices + Restrictions = 3 события
- Hotels101: Availability + Prices + Restrictions = 3 события
- CIAN: All = 1 событие
- **Итого: 7 SynchronizationExport**

### Генерация HTTP запроса (Шаг 2)
```
Events::SynchronizationExport
→ ExportModule::RequestFactory(SyncParams)
→ *Platform::Requests.export(SyncParams)
→ Events::RequestHTTP
→ запись IntegrationExport (статус: pending)
```

**Классы-адаптеры** для каждой площадки:
- `*Platform::Availabilities::MessageEnhancer + Request`
- `*Platform::Prices::MessageEnhancer + Request`
- `*Platform::Restrictions::MessageEnhancer + Request`

### Результат (Шаг 3)
- HTTP ≥ 400 → `Events::SynchronizationFailed` → IntegrationExport = fail → retry (`Events::SynchronizationRetry`)
- HTTP < 400 → `Events::SynchronizationExported` → IntegrationExport = success

---

## 4. Channel Manager — архитектура форм

Каждая площадка = набор файлов:
```
form.rb          # форма подключения (JSON-based)
model.rb         # модель данных
actions.rb       # действия (connect, disconnect, refresh)
fields.rb        # поля формы
save.rb          # логика сохранения
validation.rb    # валидации
controller       # HTTP контроллер
routes           # маршруты
```

**Platform Generator** — автогенерация скелета для новой площадки.

---

## 5. Планы разработки RC 2024

### Q1-Q2 2024 (выпущено)
| Фича | Описание |
|---|---|
| Новое мобильное приложение | Полная пересборка, новая фин. статистика, права горничных |
| Права доступа горничных | Скрыть цены и контакты гостя для роли горничной |
| Обновлённая фин. статистика | ADR, средняя длительность, сравнение периодов, учёт комиссии OTA |
| Яндекс Путешествия | Полная интеграция (апарты + мини-отели) |
| Монета. Статистика | Детализация платежей/залогов через Монету |
| 1С | Выгрузка платёжных данных |
| WhatsApp | Триггерные сообщения гостям (без диалога) |
| Новая форма регистрации | Поэтапная, с метриками |

### Q3-Q4 2024
| Фича | Описание |
|---|---|
| Оффлайн шахматка (мобайл) | Просмотр без интернета (без редактирования) |
| Channel Manager в мобайле | Управление каналами со смартфона |
| Автосообщения | Автоматические сообщения гостям |
| Улучшение залога | UX/механика взятия залога |
| Отчёт по овербукингам | Аналитика случаев овербукинга |
| Обновление Суточно | Меньше сбоев и овербукингов |

---

## 6. Аналитический стек RC
- **Amplitude** — поведенческие метрики, product analytics
- **Snowflake** — data warehouse для сложной аналитики
- **Zendesk / Omnidesk** — тикеты тех поддержки
- **CarrotQuest** (заменяется) — feature flags, диалоги с пользователями

---

## 7. Тех. поддержка — структура обращений

Основные разделы базы знаний поддержки:
- Синхронизация (API + iCal)
- Настройки RealtyCalendar
- Мобильное приложение (iOS + Android)
- Работа с AmoCRM
- Логи
- **Алгоритмы решения тикетов** (2-я линия)
- Ответы по Модулю бронирования 2.0

---

## 8. Маркетинг

- **Партнёрская программа** — порядок работы, подарки партнёрам
- **Казахстан** — гипотеза по партнёрской экспансии
- Видеоотзывы, работа с клиентами для сайта
- Маркетинг «под ключ» для отелей (отдельный продукт)

---

## 9. Ключевые инсайты для нашей архитектуры

### RC уже использует Kafka — подтверждает наш выбор
RC перешли с polling/cron на EDA через Kafka (Yandex Cloud). Это доказывает, что для high-throughput event flow Kafka — правильный выбор.

### Разделение площадок по типу импорта
Важно при проектировании Channel Service:
- **Extranet-тип** (1 запрос → все новые брони) — проще, быстрее
- **Integration-тип** (N запросов per объект) — дороже, нужен rate limiting

### Архитектура экспорта — каскад событий
При любом изменении брони цепочка:
`Booking change → EventCalendarExport → ApartmentExport → SynchronizationExport × N`

Для квартиры с 3 площадками — 7+ событий на 1 изменение брони. При 1000 броней/час — 7000+ событий. **Kafka с Kafka Streams / consumer groups — единственный адекватный вариант.**

### Retry с ограничением
- При ошибке HTTP → счётчик ошибок per integration
- При достижении лимита → integration → статус `invalid`
- Автоматические retry через `Events::SynchronizationRetry`

### Наша доработка поверх RC-архитектуры
1. **Redis SETNX lock** при создании брони — мгновенная блокировка, быстрее Kafka loop
2. **Outbox pattern** — атомарная запись в БД + Kafka (у RC этого нет явно)
3. **Idempotency keys** на всех событиях (у RC — частично через `internal_id`)
4. **ClickHouse для аналитики** (у RC — Snowflake, для нас дешевле и быстрее)

### Стратегии создания/обновления/отмены — взять готовую модель
RC Strategy pattern (`Create / Update / Cancel / CreateCanceled / Noop`) — чистый и масштабируемый подход. Применить у себя в Booking Service.
