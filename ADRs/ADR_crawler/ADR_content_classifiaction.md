# Классификация контента и маршрутизация обработки

**Статус:** Предложен

## Контекст

AI Crawler получает из Git, Jira и Confluence разнородные записи: код, обычный текст, таблицы, диаграммы, макросы, вложения, метаданные. Одна запись может содержать несколько типов. Разные виды содержимого требуют разных стратегий чтения, парсинга, деления на чанки.

Классификация применяется к **content unit** — минимальной единице с собственным правилом обработки. Определение границ content unit не входит в данную ADR. Классификация должна быть согласована между RAG и Graph. 

## Решение

### 1. Двухфазная классификация

Классификация выполняется в две фазы:

**Фаза 1 — Coarse classification (определение контейнера и выбор extractor'а)**

Выполняется сразу после получения source record. Определяет `source_container` и выбирает специализированный extractor. Использует: source system, MIME type, file extension, macro name, HTML structure.

**Фаза 2 — Structural classification (выделение content units внутри контейнера)**

Выполняется выбранным extractor'ом. Разбирает содержимое контейнера на content units, каждому присваивает `content_type` и `processing_status`. Для кода — LSP: `symbol_kind`, `qualified_symbol`, границы символа.

### 2. Переход между фазами

```text
Source record от Connector
    ↓
Фаза 1: Coarse classification
    ├── Определён source_container (PAGE, CODE_FILE, MACRO...)
    ├── Выбран extractor (LSP для .py, Confluence parser для page...)
    └── Если extractor не найден → processing_status = UNSUPPORTED
    ↓
Фаза 2: Structural classification (запуск выбранного extractor'а)
    ├── Разбор контейнера → content units
    ├── Каждому unit присвоен content_type
    ├── Каждому unit присвоен processing_status
    └── Если extractor упал → processing_status = FAILED
```

### 3. Приоритет классификации в рамках фаз

Приоритет применяется на Фазе 1 для выбора extractor'а:

1. **Явный source-native type.** Confluence API возвращает тип объекта; Git API возвращает file mode.
2. **MIME type / attachment type.** `image/png` → `BINARY_ATTACHMENT`; `application/vnd.jgraph.mxfile+xml` → `DIAGRAM`.
3. **Зарегистрированный macro adapter.** Для Confluence — имя macro'а ищется в реестре adapter'ов.
4. **Структурная разметка.** Анализ HTML/XML структуры.
5. **Language/tool detection.** Файл содержит `def`, `class` → Python code.
6. **Fallback policy.** Неизвестный MIME + нет adapter'а → `UNSUPPORTED`.

Неизвестный macro или бинарный контент НЕ классифицируется как `NARRATIVE_TEXT`. Получает `AMBIGUOUS` и направляется в ручное разрешение.

### 4. `content_type`

Семантический тип content unit:

- `NARRATIVE_TEXT` — абзацы, заголовки, списки. HTML-теги `p`, `h1`-`h6`, `li`, `a`; текстовые поля Jira; `*.md`, `*.txt`.
- `CODE` — функция, метод, класс, интерфейс, enum. LSP extraction; `symbol_kind` из LSP; code macro.
- `TABLE` — таблица как целостная структура. HTML `table`; table macro; табличные данные.
- `DIAGRAM` — PlantUML, draw.io, Gliffy. Diagram macro; MIME диаграммы.
- `STRUCTURED_RECORD` — Jira issue целиком со всеми policy-полями.
- `BINARY_ATTACHMENT` — вложение: изображение, файл, документ.
- `UNSUPPORTED` — контент, который система не может обработать.

### 5. `source_container`

Способ упаковки в источнике:

- `PAGE` — страница Confluence (имеет `page_id`)
- `FILE` — файл без кода (`*.md`, `*.txt`, `*.json`, `*.yaml`)
- `CODE_FILE` — файл с исходным кодом (`*.py`, `*.java`, `*.js`)
- `MACRO` — макрос Confluence (`ac:structured-macro`)
- `ATTACHMENT` — вложение Confluence (имеет MIME type)
- `FIELD` — поле Jira issue
- `CHANGELOG` — история изменений Jira issue

### 6. `processing_status`

Результат классификации:

- `CLASSIFIED` — контент успешно классифицирован, extractor определён.
- `AMBIGUOUS` — классификация неоднозначна, автоматический выбор невозможен.
- `UNSUPPORTED` — система не может обработать в принципе (нет extractor'а, без поддержки LSP).
- `FAILED` — extractor запущен, но завершился с ошибкой.


### 7. Код без LSP-поддержки

Код в статусе `UNSUPPORTED` не получает `entity_uid`, не вычисляется `content_hash`. Может индексироваться как `NARRATIVE_TEXT` только как часть файла/документа с `content_hash` от текста файла/документа.

### 8. Ключевые правила по источникам

**Git:** файл не является одной entity. LSP выделяет code units. Файл может породить несколько code entities.

**Confluence:** page — одна entity с одним `content_hash` из всех units. `content_hash` вычисляется только из полученных units.

**Jira:** issue — `STRUCTURED_RECORD`. Один `content_hash` из всех policy-полей. Changelog — вход для observation, не самостоятельная entity.

### 9. Инварианты

- Один source record может содержать несколько content units
- `content_hash` на уровне entity, не unit
- `content_type` не назначает `entity_uid`
- `MACRO` и `METADATA` не являются `content_type`
- Низкая уверенность не приводит к narrative fallback
- Graph extraction — только по graph policy (отдельная ADR)
- Изменение правил классификации → полный re-crawl
- `UNSUPPORTED`, `AMBIGUOUS`, `FAILED` не скрываются

### 10. Хранение результата

Для успешно обработанных units: `content_type`, `source_container`, `processing_status` сохраняются в Qdrant payload и Neo4j node properties.

Для `UNSUPPORTED` и `FAILED`: узел `UnsupportedUnit` в Neo4j с полями `source_system`, `source_scope`, `source_revision_id`, `reason`, `observed_at`. Без `snapshot_id` — snapshot удаляется после обработки.

## Последствия

### Положительные

- Двухфазная классификация устраняет конфликт между выбором extractor'а и identity resolution
- `content_hash` на уровне entity сохраняет дедупликацию версий
- Структурный контент не превращается в narrative автоматически
- Graph не загрязняется техническими сущностями

### Отрицательные

- Требуется два уровня классификации
- LSP обязателен для кода
- Неоднозначный контент задерживается в ручном разрешении
- Изменение классификации требует полного re-crawl

## Рассмотренные альтернативы

### Универсальный extractor
**Минусы:** теряется структура таблиц и диаграмм. Отклонено.

### Обрабатывать неоднозначный контент как NARRATIVE_TEXT
**Минусы:** недоверенный контент попадает в LLM. Отклонено.

### `content_hash` per-unit
**Минусы:** content_hash на уровне entity означает что entity версионируется целиком. Per-unit content_hash разбил бы одну entity на несколько версионируемых частей, что противоречит определению entity.

### Graph entity для каждого content unit
**Минусы:** загрязнение графа. Отклонено.