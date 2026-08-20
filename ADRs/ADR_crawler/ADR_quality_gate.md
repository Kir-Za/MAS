# ADR Quality Gate для входных данных

**Статус:** Предложен

## Контекст

После классификации и парсинга каждый processing unit представлен структурированным `ParsedIR`. Entity Normalizer вычислил `content_hash`, Canonical Identity Resolver назначил `entity_uid` и сформировал `entity_version_id`.

Quality Gate определяет, может ли unit быть передан в Chunking, Embedding/Qdrant, Graph Extraction/Neo4j. Финальный `canonical_status` определяется Ingestion Orchestrator.

Вводятся отдельные лимиты по типам данных как константы.

## Решение

Проводить анализ качества отдельных processing unit как первый этап, и проводить анализ качества всего processing entity как второй этап.

Первый этап определяет `quality_verdict` (статусы unit'ов): `PASS`,`QUARANTINED`, `REJECTED`. 
Второй этап проводит оценку всей processing entity и собирает итоговый статус `canonical_status`: `OBSERVED`, `EXTRACTED`, `QUALITY_REJECTED`, `CANONICAL_PUBLISHED`.

Прохождение контроля качества позволяет переходить к процессу materialization данных (процесс записи обработанных данных в целевые хранилища, преобразование результатов обработки в физические записи, доступные для поиска и навигации).


### Входные данные Quality Gate

```text
source_system
snapshot_id
source_scope
source_object_id
source_revision_id
unit_source_anchor
parent_unit_id
content_type
source_container
processing_status
structured_content
raw_payload_checksum
entity_uid              — может отсутствовать
content_hash            — может отсутствовать
entity_version_id       — может отсутствовать
```

`observed_at` доступен через snapshot context.

На основе входных данных происходит вычисление переменной `quality_verdict`. Эта переменная может принимать значения `PASS`, `QUARANTINED`, `REJECTED` согласно следующей логике:

| processing_status | Условие | quality_verdict |
|---|---|---|
| CLASSIFIED | валидация и limits пройдены | PASS |
| CLASSIFIED | структура пуста или limit нарушен | REJECTED |
| AMBIGUOUS | безопасный текстовый fallback | PASS |
| AMBIGUOUS | структурно неоднозначный контент | QUARANTINED |
| UNSUPPORTED | нет extractor'а | REJECTED |
| FAILED | source content недоступен | REJECTED |
| FAILED | source content доступен, extraction failed | REJECTED (downstream пропущен) |


**Ключевое различие для `processing_status` = `FAILED`:**

```text
source content доступен + extraction failed:
  — content_hash вычислен
  — entity_version_id создаётся
  — quality_verdict = PASS
  — downstream для этого unit пропускается

source content недоступен:
  — content_hash не вычисляется
  — quality_verdict = REJECTED
```

Переход `QUARANTINED` → `PASS` — только через HITL. 

**Критерии безопасного текстового fallback для `AMBIGUOUS`**

`AMBIGUOUS` → `PASS` только если выполнены **все** условия:

```text
1. source_container ∈ {PAGE, FILE, FIELD}
2. Содержимое текстовое (декодируемо как UTF-8)
3. Нет маркеров кода:
   — def, class, import (Python)
   — function, const, let (JavaScript)
   — другие language-specific маркеры
4. Нет маркеров таблицы:
   — HTML <table>
   — Markdown |...|...|
5. Нет маркеров диаграммы:
   — digraph (Graphviz)
   — @startuml (PlantUML)
6. Нет бинарных сигнатур
7. Размер ≤ text limit
```

`AMBIGUOUS → QUARANTINED` если:

```text
— Содержимое похоже на code/table/diagram
— Бинарный контент без явного типа
— Неизвестный macro
```

**Правила валидации по `content_type` при `processing_status = CLASSIFIED`**


#### `NARRATIVE_TEXT`

```text
PASS:
  — хотя бы один непустой текстовый блок
  — структура доступна
  — размер ≤ text limit

REJECTED:
  — пустой текст
  — decode невозможен
  — превышен limit
```

#### `CODE`

```text
PASS:
  — full_symbol_span_text непуст
  — компоненты derived_key: qualified_symbol, symbol_kind, signature_fingerprint
  — компоненты content_hash: full_symbol_span_text, module_path, declared_interfaces_or_base_types
  — размер ≤ code limit

REJECTED:
  — span пуст
  — отсутствуют компоненты derived_key или content_hash
  — превышен limit
```

#### `TABLE`

```text
PASS:
  — rows >= 1, cols >= 1
  — структура таблицы, представленная как матрица ячеек восстановлена
  — rowspan/colspan корректны
  — размер ≤ table limit

REJECTED:
  — пустая таблица
  — matrix повреждена
  — превышен limit
```

#### `DIAGRAM`

```text
PASS:
  — raw representation доступно и непусто
  — DSL: синтаксис валиден
  — image-based: зарегистрирован extractor
  — размер ≤ diagram limit

REJECTED:
  — source representation пуст
  — raw content недоступен
  — нет extractor'а
  — превышен limit
```

#### `STRUCTURED_RECORD`

```text
PASS:
  — tracked_fields содержит policy-поля
  — сериализация детерминирована

REJECTED:
  — tracked fields отсутствуют
  — JSON повреждён
```

#### `BINARY_ATTACHMENT`

```text
PASS:
  — raw payload доступен
  — raw_payload_checksum вычислен
  — byte_size > 0
  — формат поддерживается
  — размер ≤ binary limit

REJECTED:
  — payload недоступен
  — файл пуст
  — формат не поддерживается
  — превышен limit
```

**Обработка `REJECTED` и `QUARANTINED`**

```text
— Не передаются в Chunking, Embedding, Graph Domain Extraction
— Не записываются в Neo4j как domain nodes
— Фиксируются в ingestion metadata:
    source_system
    snapshot_id
    source_scope
    source_object_id
    source_revision_id
    unit_source_anchor
    content_type
    processing_status
    quality_verdict
    quality_reason            — структурированная строка: "категория:деталь"
    entity_uid                — если назначен
    content_hash              — если вычислен
    entity_version_id         — если создан
    observed_at
```

`quality_reason` — структурированная причина решения Quality Gate.


### Entity Policy

**Entity Policy** — конфигурация Quality Gate применяемая к processing entity (которая определяется как `source_object_id` + `parent_unit_id`). Entity Policy определяет:
- `required_units` - какие processing units обязательны без них публикация невозможна.
- `optional_units` — их `REJECTED`/`QUARANTINED` не блокирует entity.
- `required_materialization_targets` — `QDRANT | NEO4J | QDRANT_AND_NEO4J | NONE`

Выбирается Entity Policy по `source_system`, `content_type` и типу entity. 

Entity Policy не может объявить `optional` unit, который входит в `canonical_content_payload`. Состав `canonical_content_payload` имеет приоритет над Entity Policy.

Entity Policy хранится в конфигурации, не входит в identity-поля. Изменение → full re-crawl.


### 9. Агрегация и публикация

```text
Все обязательные units = PASS + identity разрешена (identity resolution этап дал смог определить сущность и мы получили все опциональные поля на входе):
  → materialization в целевые хранилища
  → canonical_status = EXTRACTED

Все обязательные units = PASS + но identity НЕ разрешена:
  → materialization блокируется
  → canonical_status = EXTRACTED

Есть обязательный unit в QUARANTINED:
  → materialization не начинается
  → canonical_status = EXTRACTED

Есть обязательный unit в REJECTED:
  → materialization не начинается
  → canonical_status = QUALITY_REJECTED

Необязательный unit в REJECTED или QUARANTINED:
  → entity может быть materialized,
    ЕСЛИ unit не входит в canonical_content_payload
  → canonical_status определяется обязательными unit
```


### Актуализация `canonical_status` за пределами Quality Gate

`CANONICAL_PUBLISHED` — после:
1. Quality Gate пройден
2. Identity разрешена
3. Materialization всех `required_materialization_targets` успешна

```text
required_materialization_targets = NONE:
  → CANONICAL_PUBLISHED не используется
  → canonical_status = EXTRACTED
```

### Влияние на cursor

```text
Приоритет 1: Технический failure или partial materialization
  → cursor НЕ продвигается
  → повторное чтение источника
  → идемпотентность через chunk_id/entity_version_id

Приоритет 2: Unresolved identity
  → cursor НЕ продвигается

Приоритет 3: Обязательный QUARANTINED
  → cursor НЕ продвигается для всего source_scope
  → все последующие изменения ждут HITL resolution

Приоритет 4: QUALITY_REJECTED
  → cursor может быть продвинут

Приоритет 5: Полностью обработанный и опубликованный
  → cursor может быть продвинут
```

### HITL

```text
APPROVE → при следующем crawl'е с совпадающим source_revision_id
          unit получает PASS без повторного QUARANTINED

REJECT  → при следующем crawl'е unit получает REJECTED

TIMEOUT → default action по HITL/risk policy
```

HITL decision привязывается к `source_system + source_scope + source_object_id + source_revision_id + unit_source_anchor`. Если новый crawl получил другую ревизию — решение переоценивается.

### Инварианты

1. Quality Gate не пересчитывает identity-поля
2. `processing_status`, `quality_verdict`, `canonical_status` — разные уровни
3. `content_hash` может существовать у REJECTED unit
4. Rejected code не превращается в NARRATIVE_TEXT
5. `QUARANTINED` не входит в `canonical_status`
6. Технические failures не создают domain nodes
7. Unit в `canonical_content_payload` — обязателен для публикации
8. `CANONICAL_PUBLISHED` — после identity + QualityGate + materialization targets
9. Один unit — одинаковый verdict для RAG и Graph
10. Изменение правил → full re-crawl

## Последствия

### Положительные

- Правила валидации явно ограничены CLASSIFIED
- Entity Policy не может переопределить состав `canonical_content_payload`
- Failed extraction при доступном source content не блокирует публикацию

### Отрицательные

- Обязательный QUARANTINED блокирует cursor для всего source_scope
- HITL APPROVE требует re-crawl
- Для каждого типа нужен валидатор
- Ingestion metadata — дополнительный компонент

## Рассмотренные альтернативы

### Partial materialization при QUARANTINED
Минусы: неполный контекст, расхождение с `content_hash`. Отклонено.

### Полная отбраковка entity при ошибке любого unit
Минусы: потеря полезного контента. Отклонено.

### Любой AMBIGUOUS как NARRATIVE_TEXT
Минусы: теряется структура, опасный контент в LLM. Отклонено.

### `UnsupportedUnit` как узел Neo4j
Минусы: загрязнение domain graph. Отклонено.
