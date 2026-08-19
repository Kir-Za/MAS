# ADR-008: Парсинг и нормализация источников

**Статус:** Предложен

## Контекст

После классификации source containers разбиваются на content units. Дальше эти units проходят через парсинг и нормализацию, после чего возникают два независимых артефакта:

1. **`content_hash`** — для identity, отвечающий на вопросы "что это", "какая версия"?
2. **IR** — для обработки, для ответа на вопрос как это устроено внутри.

Служебные символы (CRLF/LF, кодировки, порядок ключей, пробелы, служебный шум) разрушает identity-модель, создавая ложные `entity_version_id`. Решение должно провести нормализацию так, что бы не менять смысл кода, таблиц, диаграмм, Jira records или Confluence structure.

## Решение

Предлагается схема с двумя потоками данных:
- **Поток 1 — Identity**. Работает с нормализованной строкой, не со структурой реализуя цепочку Normalization → canonical_content_payload → content_hash → entity_version_id (после entity_uid).
- **Поток 2 — Processing**. Работает с типизированной структурой. Typed Parser → Parsed IR (structured_content) → Quality Gate → Chunking → Graph Extraction.


### Зачем нужен `content_hash`

`content_hash` — это отпечаток содержимого сущности. Нужен чтобы:

- Определить изменилась ли сущность.
- Связать версию содержимого с `entity_version_id`.
- Различить «та же версия» и «новая версия».

`content_hash` вычисляется из **source content** (не из **IR**), это гарантирует что при повторной успешной обработке тот же source даст тот же hash. В `content_hash` входит только то, что **буквально написано** в источнике:

```text
— Нормализованный текст
— module_path
— Сигнатура из объявления
— Interfaces/base types из объявления
— Исходный текст failed unit'а (если физически доступен)
```

НЕ входит то, что система **вывела** или **поняла**:

```text
— LSP-derived relations (callers, callees)
— Semantic inference о типах
— Результаты framework-specific extraction
```

#### Что такое `canonical_content_payload`

`canonical_content_payload` — строка, которая хешируется для получения `content_hash`. `canonical_content_payload` формируется из source content после нормализации.

#### Общие правила нормализации

Допускается:

- Кодировка в UTF-8 после корректного decode
- Удаление BOM
- CRLF и CR → LF
- Удаление trailing whitespace где это не меняет смысл
- Детерминированная serialization structured data
- Удаление служебных полей

Запрещено:

- Безусловное `tab → 4 spaces`
- Глобальное схлопывание whitespace
- Произвольное изменение порядка элементов
- `errors="ignore"` или `errors="replace"`


### Зачем нужен IR

IR (Intermediate Representation) — структурированное представление для downstream-компонентов. Нужен чтобы **один раз распарсить** и передать в downstream-компонент:

- **Quality Gate** проверяет headers, rows пустые/нет и т.д.
- **Chunking** определяет размер деления объектов таблица целиком.
- **Graph Extraction** содержит ли объект entities?

Без IR каждый компонент парсил бы сырой HTML заново — три разных парсера, три разных результата.


#### Формат Parsed IR

```text
ParsedIR:
  source_system
  snapshot_id
  source_scope
  source_object_id         — file_path, page_id, issue_key
  source_revision_id
  unit_source_anchor       — qualified_symbol + symbol_kind + source_span
  parent_unit_id           — временный ID, в пределах snapshot_id
  content_type
  source_container
  structured_content       — типизированное представление
  processing_status
  raw_payload_checksum
```


### Что происходит при failed extraction

```text
Диаграмма не разобралась:

Поток Identity:
  Исходный текст диаграммы попадает в canonical_content_payload
  → content_hash вычисляется
  → entity_version_id создаётся

Поток Processing:
  processing_status = FAILED
  → downstream пропускается (chunking, graph extraction)
  → Quality Gate решает публиковать или нет
```

Identity и processing разделены. Сущность может иметь `entity_version_id`, но не быть опубликованной.


### Правила по типам контента

**Код:**
- `content_hash` состав: `full_symbol_span_text`, `module_path`, interfaces/base types из объявления
- Границы span: от объявления до конца тела, включая decorators и docstring
- Для whitespace-significant языков: если нормализация меняет AST — используется исходный span

**Confluence:**
- Page — entity с одним `content_hash` из всех units
- Порядок units сохраняется
- Dangling links сохраняются как есть

**Jira:**
- Issue — entity, `content_hash` из отслеживаемых policy-полей
- Object keys сортируются, даты в UTC ISO-8601
- Changelog — источник `source_revision_id`, не content unit

**Таблицы:**
- Не преобразуются в narrative text
- Atomic IR: headers, rows, spans, alignment

**Диаграммы:**
- Если DSL невалиден — исходное представление в payload, processing_status = FAILED

**Binary attachments:**
- MIME, filename, byte size, SHA-256 checksum
- UTF-8 rules не применяются к raw bytes
- Если attachment недоступен и входит в hash page — page hash не вычисляется


## Последствия

### Положительные

- Два независимых потока: identity и processing не влияют друг на друга
- `content_hash` стабилен относительно extraction success/failure
- IR даёт одинаковую структуру всем downstream-компонентам
- Failed unit не меняет hash при доступном raw payload

### Отрицательные

- Требуется source-specific parsing и нормализация
- Два потока увеличивают сложность
- Для whitespace-significant языков hash из исходного span
- Изменение rules требует полного re-crawl

## Рассмотренные альтернативы

### Единый Markdown
Минусы: теряются структура и семантика. Отклонено.

### Универсальная нормализация
Минусы: меняет смысл кода. Отклонено.

### Minification кода
Минусы: нарушает границы символа. Отклонено.

### `content_hash` из extraction result
Минусы: нестабилен относительно extraction success/failure. Отклонено.