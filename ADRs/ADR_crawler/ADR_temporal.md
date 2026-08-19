# ADR Версии и temporal semantics

**Статус:** Предложен

## Контекст

Одна и та же версия сущности может наблюдаться несколько раз (повторный crawl страницы, несколько observation в разных scope), и актуальность версии не является глобальным свойством — она зависит от контекста (ветка, пространство, момент времени). Глобальный флаг «текущая версия» такие случаи не покрывает.

Под наблюдением (observation) понимается факт обнаружения сущности в конкретном состоянии в конкретном источнике в конкретный момент времени. Это запись о том, что crawler увидел сущность, а не сама сущность и не её версия.

Observation создается на основе snapshot — неизменяемого снимка состояния источника, полученного AI Crawler в рамках одного ingestion-прогона. Является единицей обработки: все сущности, извлечённые из этого снимка, имеют общий `snapshot_id`.

## Решение

### 1. Bi-temporal модель и принцип независимости времени

- **Valid time** (`valid_from`, `valid_until`) — период действия состояния в предметной области источника.
- **System time** (`observed_at`, `ingested_at`, `published_at`) — когда crawler узнал, обработал и опубликовал состояние.

**Инвариант:** system time никогда не используется для определения актуальности. Более поздняя публикация не делает версию «более актуальной» в предметном времени.

### 2. `entity_revision_observation`

Данный параметр metadata требует расширения для учета временных меток:

```text
entity_revision_observation:
  entity_uid
  entity_version_id
  source_system
  source_revision_id
  source_scope
  valid_from
  valid_until           -- вычисляется; может быть unknown
  valid_time_source
  source_changed_at
  observed_at
  ingested_at
  provenance
```

Создаётся только при фактическом наблюдении версии. Несколько observation для одной `entity_version_id` (revert, повторный crawl, параллельный scope) — нормальный случай; они не дублируют содержимое, но сохраняют полную историю появлений.

`valid_from` — момент, начиная с которого версия сущности начала действовать в предметной области (время коммита, изменения задачи и т.д.).

`valid_time_source` — указывает, откуда взято значение `valid_from`, может принимать следующие значения:
- `source` когда источник предоставил явное значение (время коммита, время создания версии страницы, время изменения задачи);
- `inferred`, когда источник не предоставил явного значения, система использовала `observed_at` как запасной вариант.
- `derived`	если `valid_from` вычислен на основе других данных (например, из Git-истории для прошлых коммитов при историческом crawl).

Переменные, введенные с целью аудита, отладки:
- `source_changed_at` фиксирует когда источник зарегистрировал изменение, независимо от того когда crawler его обнаружил. `source_changed_at` и `valid_from` почти всегда совпадают, потому что источник обычно предоставляет одну временную метку для изменения. Разница возникает только если источник даёт две разных метки (например отложенная публикация Confluence) или если `valid_from` вычисляется не из источника а из других данных.
- `observed_at` — временная метка, когда crawler впервые зафиксировал данное состояние сущности.
- `ingested_at` — временная метка, когда crawler завершил обработку observation и создал запись `entity_revision_observation`.


### 3. `entity_presence_observation` — отдельно от версии

Отсутствие сущности в snapshot — не просто "не наблюдение версии", а отдельный факт:

```text
entity_presence_observation:
  entity_uid
  snapshot_id
  source_scope
  presence_status   -- PRESENT | ABSENT_CONFIRMED | NOT_OBSERVED
                     -- | ACCESS_UNKNOWN | EXTRACTOR_GAP
  reason
  observed_at
```
`presence_status` определяет жизненный цикл записи о наблюдаемой сущности:
- `PRESENT`	Сущность обнаружена в snapshot'е. Создаётся `entity_revision_observation`.
- `ABSENT_CONFIRMED`	Подтверждённое отсутствие в полном корректном snapshot. Закрывает valid interval.
- `NOT_OBSERVED`	Сущность не обнаружена, но snapshot может быть неполным (shallow checkout).
- `ACCESS_UNKNOWN`	Нет доступа к источнику — невозможно определить присутствие.
- `EXTRACTOR_GAP`	Сбой extractor'а — сущность могла быть, но не извлечена


`source_scope` определяется следующим образом:
```text
source_scope:
  repository_id | space_id
  branch_or_release_reference  -- для кода: master; release_tag — тег внутри той же линии
```

### 4. Композиция `content_hash`

- **Confluence:** текст, заголовки/иерархия, таблицы, списки, макросы, ссылки, anchors, диаграммы и вложения — входят безусловно (условное «если входит» исключено).
- **Jira:** снимок всех отслеживаемых policy-полей на момент `source_changed_at`.
- **ADR/документы с жизненным циклом:** только текст документа; статус не входит.


### 6. Jira: identity ревизии и версии

`source_revision_id` — это `issue_key + позиция изменения + source_changed_at` с сохранением hash исходного payload для проверки идентичности при ресинхронизации. Один changelog-batch (все поля, изменённые одной операцией) образует один снимок и одну `entity_version_id`.

### 7. ADR/документы с ЖЦ: статус — метаданные, не content

Смена статуса (`PROPOSED → ACCEPTED` и т.п.) не входит в `content_hash` и не создаёт новую `entity_version_id`, если текст не изменился — фиксируется как метаданные новой `entity_revision_observation` с тем же `entity_version_id`. 

### 8. `valid_until`: линейная и общая история

Для кода (только `master`) история линейна: `valid_until` версии из коммита A — `valid_from` следующего коммита в `master`, изменившего тот же `entity_uid`, вычисляется однозначно. В ветках релизов `valid_until`по аналогии с `master` имеет линейную историю относительно `source_scope` релиза.

### 9. Разрешение `as_of` при нескольких observation

Система должна выбрать одну версию из нескольких observation'ов. Среди `entity_revision_observation` данного `entity_uid` в запрошенном `source_scope` выбирается наблюдение, где `valid_from <= as_of` и (`valid_until` неизвестен либо `as_of < valid_until`), с максимальным `valid_from` среди подходящих — по valid time, не по `observed_at`/`published_at`. Если подходящей observation нет — результат `NOT_APPLICABLE_FOR_AS_OF`.

### 10. Контракт `version_context`

Единый набор параметров которые понимают все компоненты: Retrieval Service, Graph Query API, Planner.

```text
version_context:
  source_revision_id: optional   -- точный режим, наивысший приоритет
  branch | release_tag: optional
  source_scope: optional
  as_of: optional timestamp
```

Конфликтующие параметры (например, `as_of` предшествует `valid_from` указанной `source_revision_id`, или `branch` несовместим с ревизией) приводят к явному `INVALID_VERSION_CONTEXT`, а не к молчаливому выбору одного из параметров.

### 11. Default scope и freshness

```text
latest_allowed_source_revision:
  source_latest_known            -- по Source Snapshot/cursor
  ingestion_latest_processed
```

`source_latest_known` — последнее известное системе состояние источника. 

`ingestion_latest_processed` — последнее состояние источника, которое Crawler успешно обработал и для которого созданы entity_revision_observation. 

Если `ingestion_latest_processed` отстаёт от `source_latest_known`, результат сопровождается freshness/processing warning, а не выдаётся как безусловно текущий.

## Последствия

### Положительные

- Явные `valid_from`/`valid_until` на observation делают `as_of`-запросы детерминированными без скрытых допущений.
- Отдельная `entity_presence_observation` не путает подтверждённое отсутствие с обычным наблюдением версии.
- Линейность `master` даёт однозначный `valid_until` для основного сценария без излишнего запрета.
- Явный `INVALID_VERSION_CONTEXT` предотвращает расхождение результатов у разных потребителей при конфликтующих параметрах запроса.


### Отрицательные

- Дополнительные поля (`valid_time_source`, `entity_presence_observation`) увеличивают объём metadata и число сущностей, которые должны понимать downstream-потребители.
- Строгая обработка `INVALID_VERSION_CONTEXT` требует от клиентов явной валидации контекста перед запросом, а не полагания на «умный» fallback.


## Рассмотренные альтернативы

### Хранить только последнюю версию

**Минусы:** невозможен исторический поиск и аудит; revert и supersession неразличимы. Отклонено.

### Глобальный флаг `is_current`

**Минусы:** не отражает параллельные scope; исторические запросы требуют обходных решений. Отклонено.

### `source_revision_id` как `entity_version_id`

**Минусы:** разная семантика источников; дублирование при повторном наблюдении. Отклонено (согласуется с ADR-002/003).

### `valid_from`/`valid_until` на уровне `entity_version_id`, без scope и без observation

**Минусы:** одна версия может быть актуальна в разных scope одновременно; конфликтующие интервалы при разных наблюдениях. Отклонено в пользу хранения интервала на уровне observation.

### Включение ACL в `source_scope`

**Минусы:** изменение прав доступа искусственно создаёт видимость новой temporal-версии; cache и join начинают зависеть от authorization context. Отклонено — ACL вынесен в отдельный `access_scope`.

### `published_at` как основание для определения актуальности версии

**Минусы:** более поздняя публикация (например, после reconciliation) не означает более позднюю предметную актуальность — риск инверсии valid time относительно system time. Отклонено.

### Хранение отсутствия сущности как обычной `entity_revision_observation`

**Минусы:** создаёт ложное утверждение о факте наблюдения версии там, где версия не наблюдалась. Отклонено в пользу отдельной `entity_presence_observation`.