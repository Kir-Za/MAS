# ADR Классификация контента и маршрутизация обработки

**Статус:** Предложен

---

## Контекст и проблема

AI Crawler получает из Git, Jira и Confluence source records разной природы: исходный код, документацию, таблицы, диаграммы, макросы, вложения, Jira issues и служебные атрибуты.

Один source record может содержать несколько типов контента. Например:

- Confluence page может включать текст, таблицы, диаграммы и attachments;
- Git file может содержать несколько code symbols;
- Jira issue содержит текстовые поля, структурированные поля и changelog.

Разные типы контента требуют разных правил парсинга, нормализации, Quality Gate, chunking, embedding и Graph Extraction. Неправильная классификация может привести к потере структуры, неверному `content_hash`, ошибочному `entity_version_id` или загрязнению Neo4j техническими объектами.

Классификация выполняется после получения Source Snapshot. `content_hash` вычисляется до Quality Gate, а Quality Gate выполняется до Identity Resolution.

**Проблема:** как определить `source_container`, `content_type` и `processing_status` для каждого content unit и направить его на корректную обработку, сохранив единый результат классификации для RAG и Graph.

---

## Факторы решения

* Согласованность классификации между RAG и Graph.
* Сохранение структуры таблиц, диаграмм и кода.
* Совместимость с Quality Gate до Identity Resolution.
* Детерминированный выбор parser/extractor.
* Отсутствие автоматического превращения неизвестного или опасного контента в `NARRATIVE_TEXT`.
* Поддержка нескольких content units внутри одного source record.
* Совместимость с entity-level `content_hash`.
* Отсутствие технических nodes в Neo4j.
* Возможность полного re-crawl при изменении правил классификации.
* Поддержка Git, Jira и Confluence без введения source-specific content types.
* Использование Jira text fields и Confluence narrative units как источников semantic relation candidates на этапе Graph Extraction.

---

## Решение

Мы выполняем классификацию в две последовательные фазы:

```text
Source Record из Source Snapshot
    ↓
Coarse Classification
    ↓
Structural Classification
    ↓
Content Units для последующей обработки
```

Классификация не назначает `entity_uid`, `entity_version_id` или `derived_key`. Она только определяет контейнер, тип контента, processing status и маршрут дальнейшей обработки.

### 1. Coarse Classification

Coarse Classification выполняется сразу после получения source record.

Она определяет:

- `source_container`;
- подходящий parser/extractor;
- возможность перехода к Structural Classification.

Используются:

- `source_system`;
- `object_type`;
- MIME type;
- file extension;
- Git file mode;
- Confluence macro name;
- HTML/XML/storage structure;
- Jira object или field type.

Приоритет правил:

1. явный source-native type;
2. MIME type или attachment type;
3. зарегистрированный macro adapter;
4. структурная разметка;
5. language/tool detection;
6. fallback policy.

Если подходящий extractor отсутствует, source record получает:

```text
processing_status = UNSUPPORTED
content_type = UNSUPPORTED
```

Если тип потенциально может быть определён несколькими вариантами, source record передаётся в Structural Classification с:

```text
processing_status = AMBIGUOUS
```

### 2. Structural Classification

Structural Classification выполняется выбранным parser/extractor.

Она:

- разбирает source record на content units;
- определяет границы каждого unit;
- присваивает каждому unit `content_type`;
- присваивает каждому unit `processing_status`;
- формирует `unit_source_anchor`;
- устанавливает `parent_unit_id` для навигации по in-memory IR;
- для code units извлекает LSP/framework-specific structural fields.

Для кода Structural Classification использует LSP и framework extractors и может определить:

```text
symbol_kind
qualified_symbol
full_symbol_span_text
```

Structural Classification не назначает identity.

### 3. Гранулярность классификации

Классификация применяется к content unit — минимальной единице с отдельной processing policy.

Один source record может породить несколько content units:

```text
Confluence page
  → narrative units
  → table units
  → diagram units
  → attachment units

Git code file
  → class units
  → method units
  → function units

Jira issue
  → structured issue unit
  → policy-defined field units
```

Source record, content unit и logical entity не являются взаимозаменяемыми понятиями:

- source record — полученный объект источника;
- content unit — единица обработки;
- logical entity — объект identity-модели.


### 4. `source_container`

`source_container` описывает способ упаковки контента в исходной системе:

```text
PAGE
  страница Confluence;

FILE
  файл Git без отдельной code classification;

CODE_FILE
  файл Git, содержащий исходный код;

MACRO
  макрос Confluence;

ATTACHMENT
  вложение Confluence;

FIELD
  поле Jira issue;

CHANGELOG
  история изменений Jira issue.
```

`source_container` не является `content_type`.

Например:

```text
source_container = MACRO
content_type = DIAGRAM
```

или:

```text
source_container = ATTACHMENT
content_type = BINARY_ATTACHMENT
```

### 5. `content_type`

`content_type` описывает семантический тип обрабатываемого content unit:

```text
NARRATIVE_TEXT
  абзацы, заголовки, списки и обычный текст;

CODE
  функция, метод, класс, интерфейс, enum или module;

TABLE
  таблица как структурированная матрица;

DIAGRAM
  PlantUML, draw.io, Gliffy или другая диаграмма;

STRUCTURED_RECORD
  Jira issue с policy-defined полями;

BINARY_ATTACHMENT
  бинарное вложение или файл, предназначенный
  для binary processing;

UNSUPPORTED
  контент, для которого нет поддерживаемой обработки.
```

`METADATA` не является `content_type`. Служебные атрибуты сохраняются как source metadata, provenance или temporal metadata.

`MACRO` также не является `content_type`: это `source_container`. Содержимое macro классифицируется отдельно.

### 6. `processing_status`

`processing_status` описывает результат классификации и запуска выбранного extractor:

```text
CLASSIFIED
  тип определён, extractor выбран,
  структурная обработка может продолжаться;

AMBIGUOUS
  тип или способ обработки не определён однозначно;

UNSUPPORTED
  для контента нет поддерживаемого extractor;

FAILED
  выбранный extractor запущен, но завершился
  терминальной ошибкой.
```

`processing_status` не является `quality_verdict`. 


### 7. Приоритет и результат неоднозначной классификации

Неизвестный macro или binary content не классифицируется автоматически как `NARRATIVE_TEXT`.

Правила:

- безопасный текстовый source unit может получить `NARRATIVE_TEXT`, если fallback policy однозначно разрешает такую классификацию;
- неизвестный macro получает `AMBIGUOUS` или `UNSUPPORTED`, но не автоматически `NARRATIVE_TEXT`;
- бинарный content без известного формата получает `BINARY_ATTACHMENT` только при наличии соответствующего binary processing path;
- потенциально кодовый, табличный или диаграммный content при недостаточной уверенности получает `AMBIGUOUS`;
- потенциально опасный или неинтерпретируемый content не передаётся в LLM как обычный текст.

Дальнейшее решение:

- безопасный текстовый fallback → `NARRATIVE_TEXT`;
- структурно неоднозначный content → `AMBIGUOUS` и дальнейшая обработка через Quality Gate/HITL policy;
- отсутствие extractor → `UNSUPPORTED`;
- сбой выбранного extractor → `FAILED`.

### 8. Правила для Git

Git file не является автоматически одной logical entity.

Для Git:

- `FILE` используется для файлов, обрабатываемых как единое текстовое или structured content;
- `CODE_FILE` используется для файлов, в которых extractor выделяет code units;
- один code file может породить несколько code units;
- каждый code symbol получает собственный `unit_source_anchor`;
- `symbol_kind`, `qualified_symbol` и другие structural fields передаются в pipeline далее.

Если язык или конструкция не поддерживаются LSP/framework extractor'ом:

- unit не получает `content_type=CODE`;
- он может быть классифицирован как `NARRATIVE_TEXT` только при явном разрешении source policy;
- иначе получает `UNSUPPORTED`.

Классификация файла как narrative не должна скрывать, что исходный объект был code-like и не прошёл структурный анализ.

### 9. Правила для Confluence

Confluence page может содержать несколько content units:

```text
PAGE
  → NARRATIVE_TEXT
  → TABLE
  → DIAGRAM
  → CODE
  → ATTACHMENT/BINARY_ATTACHMENT
```

Правила:

- page является source container и может быть logical entity;
- content units получают собственные `content_type`;
- macro классифицируется по имени macro adapter'ом;
- содержимое macro классифицируется отдельно;
- таблица не превращается в narrative text;
- диаграмма не превращается в narrative text без явного fallback policy;
- attachment классифицируется отдельно от page content;
- page-level `content_hash` формируется за пределами данной ADR;
- `parent_unit_id` используется только в in-memory IR.

Если attachment не получен или macro не разобран, это фиксируется через `processing_status`. Классификация не создаёт автоматически domain entity в Neo4j.

**Confluence narrative units как источники semantic relation candidates:**

Для Confluence units с `content_type = NARRATIVE_TEXT` или содержащих текстовое описание, классификация сохраняет:

- текст unit'а для последующего использования в semantic candidate generation на этапе работы с графом;
- source context: `source_system = confluence`, `source_scope`, `source_object_id = page_id`, `source_revision_id = page_version`, `unit_source_anchor`;
- связь с `entity_uid` после определения сущности.

Это не влияет на `content_hash` и не создаёт relations на этапе классификации. Классификация только отмечает unit как доступный для semantic matching.

### 10. Правила для Jira

Jira issue классифицируется как:

```text
content_type = STRUCTURED_RECORD
source_container = PAGE или FIELD
```

Внутренние части issue обрабатываются по policy:

- summary, description и acceptance criteria могут быть text units;
- status, resolution и другие tracked fields являются structured attributes;
- comments могут быть `NARRATIVE_TEXT`, если входят в policy;
- changelog является temporal/source input, а не обычным content unit.

Jira issue остаётся одной logical entity согласно ADR parsing. Отдельные поля не получают собственные `entity_uid`, если это явно не предусмотрено отдельной entity policy.

**Jira text fields как источники semantic relation candidates:**

Для Jira units с `content_type = STRUCTURED_RECORD` или выделенных текстовых полей (`summary`, `description`, `comments`), классификация сохраняет:

- текст полей для последующего использования в semantic candidate generation на этапе работы с графом;
- source context: `source_system = jira`, `source_scope`, `source_object_id = issue_key`, `source_revision_id`, `unit_source_anchor`;
- связь с `entity_uid` после идентификации сущностей.

Это не влияет на `content_hash` и не создаёт relations на этапе классификации. Классификация только отмечает текстовые поля как доступные для semantic matching.

### 11. Взаимодействие с parser и `content_hash`

Classification result передаётся в Parser и Normalizer.

`content_hash` вычисляется после классификации и необходимой structural parsing, но до Quality Gate и Identity Resolution:

```text
content classification
→ parsing/normalization
→ canonical_content_payload
→ content_hash
→ Quality Gate
→ Identity Resolution
```

Классификация не изменяет `entity_uid` или `entity_version_id`, поскольку на этом этапе они ещё не назначаются.

Изменение правил классификации требует полного re-crawl. Старые materializations не должны молча смешиваться с результатами новой классификации.

### 12. Хранение результата классификации

Для units, прошедших обработку и materialization, сохраняются:

```text
source_container
content_type
processing_status
```

в составе соответствующих materialized metadata согласно downstream-контрактам.

Для `UNSUPPORTED` и `FAILED` сохраняется ingestion metadata:

```text
source_system
snapshot_id
source_scope
source_object_id
source_revision_id
unit_source_anchor
content_type
processing_status
observed_at
```

Эти units не записываются в Neo4j как domain nodes.

`parent_unit_id` не сохраняется как постоянный идентификатор. Он используется только внутри IR и действует в пределах `snapshot_id`.

### 13. Артефакты

ADR формирует для каждого content unit:

```text
source_container
content_type
processing_status
unit_source_anchor
parent_unit_id
```

Для code units дополнительно формируются structural fields, необходимые следующим этапам:

```text
symbol_kind
qualified_symbol
full_symbol_span_text
```

## Инварианты

1. Классификация выполняется после создания Source Snapshot.
2. Один source record может содержать несколько content units.
3. `source_container` описывает упаковку контента, а `content_type` — его семантический тип.
4. `MACRO` и `METADATA` не являются значениями `content_type`.
5. Один content unit получает один согласованный classification result для RAG и Graph.
6. Классификация не назначает `entity_uid` и `entity_version_id`.
7. `content_hash` вычисляется после классификации и parsing, но до Quality Gate и Identity Resolution.
8. `processing_status` не заменяет `quality_verdict`.
9. Неизвестный macro или binary content не превращается автоматически в `NARRATIVE_TEXT`.
10. Rejected или unsupported code не преобразуется в narrative без явного source policy.
11. Таблицы, диаграммы и code units сохраняют свой структурный `content_type`.
12. Changelog Jira является temporal/source input и не является обычным content unit.
13. `parent_unit_id` используется только для in-memory IR.
14. `unit_source_anchor` локализует unit, но не является `entity_uid`.
15. Graph Extraction создаёт domain entities только согласно graph policy.
16. Технические ошибки классификации не создают domain nodes в Neo4j.
17. Изменение правил классификации требует полного re-crawl.
18. `UNSUPPORTED`, `AMBIGUOUS` и `FAILED` не скрываются.
19. Границы content unit определяются classification/extraction pipeline и передаются в последующие ADR.
20. Классификация не изменяет source identity и temporal metadata.
21. Confluence narrative units и Jira text fields сохраняются как источники для semantic relation candidates на этапе работы с графом, но не создают relations на этапе классификации.

---

## Новые переменные

| Переменная | Тип | Описание | Точка появления |
|---|---|---|---|
| `source_container` | enum | Способ упаковки контента: `PAGE`, `FILE`, `CODE_FILE`, `MACRO`, `ATTACHMENT`, `FIELD`, `CHANGELOG` | Coarse Classification |
| `content_type` | enum | Семантический тип content unit | Structural Classification |
| `processing_status` | enum | Результат классификации и запуска extractor: `CLASSIFIED`, `AMBIGUOUS`, `UNSUPPORTED`, `FAILED` | Structural Classification |
| `unit_source_anchor` | string | Ссылка на конкретный фрагмент source object | Structural Classification/Parser |
| `parent_unit_id` | string | Временный ID родительского unit в IR-дереве | Parser; только in-memory |
| `symbol_kind` | enum | Тип code symbol, извлечённый LSP | LSP/framework extractor |
| `qualified_symbol` | string | Полное имя code symbol | LSP/framework extractor |
| `full_symbol_span_text` | string | Полный source span code entity | LSP/framework extractor |


---

## Последствия

### Положительные

* Coarse и Structural Classification разделяют выбор контейнера и выделение content units.
* RAG и Graph получают согласованный результат классификации.
* Структура code, tables, diagrams и Confluence pages сохраняется до downstream processing.
* Классификация не смешивается с identity resolution.
* `content_hash` вычисляется до Quality Gate и не зависит от последующего verdict.
* Неизвестные macros и binary objects не маскируются под обычный текст.
* Технические classification failures не загрязняют Neo4j.
* Изменение classification rules согласуется с политикой полного re-crawl.
* Confluence narrative units и Jira text fields сохраняют source контекст для последующего semantic matching на этапе работы с графом.

### Отрицательные

* Для одного source record требуется обрабатывать несколько content units.
* Нужны source-specific classifiers и extractors.
* LSP/framework extraction обязателен для полноценной структурной обработки поддерживаемого кода.
* Неоднозначный content может потребовать HITL или быть исключён.
* Изменение правил классификации требует полного re-crawl.
* Часть content units может не получить Graph или Qdrant materialization согласно policy.

---

## Рассмотренные альтернативы

### Неявная классификация внутри каждого extractor

Каждый downstream extractor самостоятельно определяет тип контента и способ обработки.

**Плюсы:**

* меньше централизованных правил;
* проще первоначальная реализация отдельных extractors;
* можно быстро добавлять source-specific обработку.

**Минусы:**

* RAG и Graph могут классифицировать один content unit по-разному;
* разные extractors могут по-разному формировать `content_hash`;
* сложнее тестировать и повторять pipeline;
* ошибки классификации обнаруживаются слишком поздно;
* возрастает риск дублирования source-specific logic.

**Решение:** отклонено.

### Классификация только после Identity Resolution

Сначала назначать identity, затем определять content type и processing route.

**Плюсы:**

* resolver получает source record с уже известной identity;
* проще связывать ошибки с entity;
* можно использовать entity context при классификации.

**Минусы:**

* identity выполняется для неподдерживаемого или повреждённого контента;
* для code identity нужны структурные признаки, которые ещё не извлечены;
* возрастает стоимость обработки;
* ошибки классификации могут привести к неверному identity resolution.

**Решение:** отклонено.

### Универсальный extractor для всех типов контента

Все source records преобразуются одним extractor'ом в текстовое представление.

**Плюсы:**

* простая архитектура;
* один downstream contract;
* меньше отдельных parser/extractor компонентов.

**Минусы:**

* теряется структура таблиц, диаграмм и кода;
* ухудшается `content_hash`;
* Graph Extraction получает недостаточно данных;
* binary и macro content обрабатываются некорректно;
* возрастает риск ложных entities и relations.

**Решение:** отклонено.

### Обрабатывать любой неоднозначный контент как `NARRATIVE_TEXT`

При низкой уверенности всегда использовать текстовый fallback.

**Плюсы:**

* меньше блокирующих случаев;
* больше контента попадает в Qdrant;
* не требуется HITL для большинства неоднозначных units.

**Минусы:**

* code, tables и diagrams могут потерять структуру;
* неизвестный binary или macro content может попасть в LLM;
* возрастает риск неверного `content_hash`;
* Graph Extraction получает недостоверный source representation.

**Решение:** отклонено. Безопасный текстовый fallback разрешён только source-specific policy.

### Создавать Graph entity для каждого content unit

Каждый content unit автоматически превращается в узел Neo4j.

**Плюсы:**

* простое отображение IR в Graph;
* максимальная сохранность технических объектов;
* проще реализовать первичную трассировку units.

**Минусы:**

* технические units загрязняют domain graph;
* metadata, macros и attachments становятся ложными domain entities;
* увеличивается размер графа;
* ухудшается graph retrieval;
* нарушается граница между content unit и logical entity.

**Решение:** отклонено. Graph entities создаются только по graph policy. присутствие content unit не означает наличие domain entity.
