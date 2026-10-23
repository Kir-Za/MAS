# ADR Проверка качества входных данных

**Статус:** Предложен

---

## Контекст и проблема

После классификации и парсинга каждый processing unit представлен структурированными данными:

- `content_type`;
- `source_container`;
- `processing_status`;
- `structured_content`;
- `content_hash`, если исходное содержимое доступно;
- source и provenance metadata.

Quality Gate выполняется до Identity Resolution. Поэтому он не должен назначать или проверять наличие:

- `entity_uid`;
- `entity_version_id`;
- `derived_key`;
- `entity_alias`;
- `entity_redirect`.

Quality Gate должен определить, пригоден ли unit для Identity Resolution и последующей обработки в Chunking, Embedding и Graph Extraction. При этом `content_hash` может быть вычислен для unit с ошибкой extraction, но это не означает пригодность unit для downstream materialization.

Одна processing entity может состоять из нескольких units. Ошибка необязательного unit не должна автоматически блокировать всю entity, но unit, входящий в `canonical_content_payload`, обязателен для полной публикации.

**Проблема:** как детерминированно проверить качество processing units до Identity Resolution и определить влияние их состояния на обработку entity, не смешивая `quality_verdict` с identity и `canonical_status`.

---

## Факторы решения

* Выполнение Quality Gate до Identity Resolution.
* Исключение identity resolution для заведомо непригодных units.
* Отделение доступности source content от успешности extraction.
* Согласованность verdict для RAG и Graph.
* Сохранение entity-level `content_hash`.
* Защита от публикации неполного `canonical_content_payload`.
* Различение обязательных и необязательных units.
* Различение quality failure, quarantine и технического сбоя.
* Отсутствие технических quality nodes в Neo4j.
* Совместимость с временным snapshot и cursor semantics.
* Отсутствие `tenant_id`, ACL metadata, Bamboo, `content_policy_version`, `content_version` и отдельного `crawl_mode`.

---

## Решение

Мы выполняем Quality Gate в два уровня:

1. проверяем каждый processing unit;
2. агрегируем verdicts в пределах processing entity.

Quality Gate выполняется после парсинга и до идентификации:

```text
Source Snapshot
→ Classification
→ Parsing and Normalization
→ content_hash
→ Quality Gate
→ Identity Resolution
→ Version and Observation Finalization
```

Quality Gate вычисляет `quality_verdict`, но не устанавливает `canonical_status`.

### 1. Входные данные Quality Gate

Quality Gate получает `ProcessingUnit`:

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
content_hash
observed_at
```

Поля `entity_uid` и `entity_version_id` не требуются и не должны использоваться Quality Gate:

```text
entity_uid        — отсутствует до Identity Resolution;
entity_version_id — отсутствует до Version Finalization.
```

`parent_unit_id` используется только для навигации по in-memory IR.

### 2. Unit-level `quality_verdict`

Для каждого unit Quality Gate вычисляет один из verdicts:

```text
PASS
QUARANTINED
REJECTED
```

#### `PASS`

Unit:

- структурно валиден;
- соответствует своему `content_type`;
- имеет необходимое содержимое;
- не нарушает установленные limits;
- может быть передан в Identity Resolution.

`PASS` означает пригодность для Identity Resolution, но не означает, что identity уже разрешена.

#### `QUARANTINED`

Unit:

- нельзя безопасно классифицировать или интерпретировать автоматически;
- требует HITL или другого разрешённого policy decision;
- не передаётся в Identity Resolution, Chunking, Embedding или Graph Extraction до разрешения.

Безопасный текстовый fallback может получить `PASS`, если выполнены условия ADR-002. Структурно неоднозначный, неизвестный или потенциально опасный контент получает `QUARANTINED`.

#### `REJECTED`

Unit:

- пуст;
- повреждён;
- не поддерживается;
- не содержит обязательной структуры;
- превышает соответствующий limit;
- имеет терминальную ошибку parsing или extraction;
- не имеет доступного исходного содержимого.

`REJECTED` unit не передаётся в Identity Resolution, Chunking, Embedding или Graph Extraction.

Если исходное содержимое доступно, `content_hash` может быть вычислен для `REJECTED` unit. Это необходимо для фиксации source state, но не разрешает Identity Resolution или materialization.

### 3. Mapping `processing_status` в `quality_verdict`

| `processing_status` | Условие | `quality_verdict` |
|---|---|---|
| `CLASSIFIED` | Structural validation и limits пройдены | `PASS` |
| `CLASSIFIED` | Структура пуста, повреждена или нарушен limit | `REJECTED` |
| `AMBIGUOUS` | Разрешён безопасный текстовый fallback согласно ADR-002 | `PASS` |
| `AMBIGUOUS` | Контент структурно неоднозначен или потенциально опасен | `QUARANTINED` |
| `UNSUPPORTED` | Нет поддерживаемого extractor | `REJECTED` |
| `FAILED` | Терминальная ошибка parsing/extraction | `REJECTED` |

Технический сбой connector, сети или инфраструктуры не является `quality_verdict`.

### 4. Правила по content type

#### `NARRATIVE_TEXT`

`PASS`, если:

- есть хотя бы один сущестсвующий текстовый блок;
- структура доступна;
- `content_hash` вычислен;
- размер не превышает `text_limit`.

`REJECTED`, если:

- текст пуст;
- безопасное декодирование невозможно;
- структура недоступна;
- превышен `text_limit`.

Dangling links сами по себе не являются причиной rejection.

#### `CODE`

`PASS`, если:

- `full_symbol_span_text` существует;
- присутствуют обязательные для extractor fields;
- `content_hash` вычислен;
- размер не превышает `code_limit`.

Обязательные поля могут включать:

```text
module_path
qualified_symbol
symbol_kind
signature_fingerprint
full_symbol_span_text
```

`REJECTED`, если:

- span пуст;
- отсутствуют обязательные structural fields;
- границы symbol не определены;
- язык или конструкция не поддерживаются;
- превышен `code_limit`.

Rejected code не преобразуется в `NARRATIVE_TEXT`.

#### `TABLE`

`PASS`, если:

- `rows >= 1`;
- `cols >= 1`;
- matrix ячеек восстановлена;
- `rowspan` и `colspan` корректны;
- размер не превышает `table_limit`.

Headers не обязательны.

`REJECTED`, если:

- таблица пуста;
- matrix повреждена;
- spans невалидны;
- превышен `table_limit`.

#### `DIAGRAM`

`PASS`, если:

- raw representation доступно и существует;
- DSL валиден либо для image-based diagram доступен extractor;
- размер не превышает `diagram_limit`.

Наличие edges не обязательно.

`REJECTED`, если:

- source representation пуст;
- raw content недоступен;
- DSL повреждён;
- extractor отсутствует;
- превышен `diagram_limit`.

Если raw representation доступно, `content_hash` может быть вычислен при `REJECTED`, но unit не передаётся downstream.

#### `STRUCTURED_RECORD`

Для Jira issue `PASS`, если:

- `tracked_fields` содержит необходимые policy-поля;
- structured representation корректна;
- canonical serialization возможна;
- `content_hash` вычислен.

`REJECTED`, если:

- отсутствуют обязательные tracked fields;
- structured representation повреждена;
- сериализация невозможна;
- превышен соответствующий limit.

Changelog является source revision input и не является обычным content unit.

#### `BINARY_ATTACHMENT`

`PASS`, если:

- raw payload доступен;
- `raw_payload_checksum` вычислен;
- размер больше нуля;
- формат поддерживается зарегистрированным extractor;
- размер не превышает `binary_limit`.

`REJECTED`, если:

- payload недоступен;
- файл пуст;
- формат не поддерживается;
- превышен `binary_limit`.

Raw binary не проходит текстовую нормализацию.

### 5. Resource limits

Quality Gate использует отдельные конфигурационные константы:

```text
text_limit
code_limit
table_limit
diagram_limit
binary_limit
```

Limits применяются к соответствующему `content_type`. Для каждого limit заранее определяются:

- единица измерения;
- объект применения;
- действие при превышении.

Превышение limit приводит к:

```text
quality_verdict = REJECTED
```

Limits не входят в:

```text
content_hash
entity_uid
entity_version_id
```

Изменение limits является изменением Quality Gate rules и требует полного re-crawl согласно процессу создания snapshot.

### 6. Entity Policy

`Entity Policy` — конфигурация ingestion, которая определяет требования к processing entity.

Policy выбирается по:

```text
source_system
content_type
типу logical entity
```

Entity Policy определяет:

```text
required_units
optional_units
required_materialization_targets
```

`required_materialization_targets` принимает значения:

```text
QDRANT
NEO4J
QDRANT_AND_NEO4J
NONE
```

Entity Policy:

- не изменяет `canonical_content_payload`;
- не изменяет `content_hash`;
- не изменяет `entity_uid`;
- не изменяет `entity_version_id`;
- не может объявить optional unit, который входит в entity-level `canonical_content_payload`.

Unit, входящий в `canonical_content_payload`, является обязательным для полной публикации независимо от downstream policy.

### 7. Агрегация verdicts

Качество entity определяется после проверки units:

```text
Все обязательные units = PASS
  → entity допускается к Identity Resolution
  → после identity допускается materialization

Есть обязательный QUARANTINED
  → entity не передаётся в materialization
  → текущий ingestion result = EXTRACTED

Есть обязательный REJECTED
  → entity не передаётся в materialization
  → текущий ingestion result = QUALITY_REJECTED

Необязательный REJECTED или QUARANTINED
  → entity может продолжить обработку,
    если unit не входит в canonical_content_payload
```

Если unit входит в `canonical_content_payload` и имеет `REJECTED` или `QUARANTINED`, полная публикация entity запрещена.

Quality Gate не устанавливает `CANONICAL_PUBLISHED`. Этот статус определяется позднее после:

```text
Identity Resolution
→ Version Finalization
→ Graph/Chunking processing
→ Embedding/materialization
→ Reconciliation
```

### 8. Identity readiness

`quality_verdict=PASS` означает:

```text
unit пригоден для Identity Resolution
```

Но не означает:

```text
entity_uid назначен
entity_version_id создан
```

После Quality Gate:

```text
PASS
→ Identity Resolution
→ entity_uid
→ entity_version_id
→ EntityRevisionObservation
```

Если Identity Resolution не может определить identity:

```text
quality_verdict = PASS
identity_status = AMBIGUOUS
```

В этом случае:

- materialization блокируется;
- создаётся `UnresolvedIdentityRecord`;
- текущий `canonical_status` не становится `QUALITY_REJECTED`;
- Quality Gate не пересчитывается автоматически.

### 9. Обработка результатов Quality Gate

Для `REJECTED` и `QUARANTINED` units сохраняется ingestion metadata:

```text
source_system
snapshot_id
source_scope
source_object_id
source_revision_id
unit_source_anchor
content_type
processing_status
quality_verdict
quality_reason
content_hash
observed_at
```

`quality_reason` объясняет quality decision и не заменяет `reason` из `EntityPresenceObservation`.

Rejected и quarantined units:

- не становятся domain nodes в Neo4j;
- не передаются в Chunking;
- не передаются в Embedding;
- не передаются в Graph Extraction.

### 10. Влияние на cursor и snapshot

`QUALITY_REJECTED` без технического сбоя является терминальным quality result текущего run. После сохранения quality result и завершения необходимых операций cursor может быть продвинут.

Для обязательного `QUARANTINED`:

- текущий run не завершается успешной публикацией;
- cursor не продвигается до HITL resolution;
- snapshot не используется для долгосрочного replay;
- после решения выполняется новый crawl с новым `snapshot_id`.

Для unresolved identity:

- materialization блокируется;
- cursor не продвигается;
- после resolution выполняется новый crawl.

Технические failures обрабатываются не преобразуются в `QUALITY_REJECTED`.

### 11. HITL

Переход:

```text
QUARANTINED → PASS
```

допускается только после HITL или другого решения, разрешённого отдельной policy.

HITL decision хранится и используется согласно отдельному ADR. В рамках этого ADR фиксируется только влияние решения:

```text
APPROVE
  → при новом crawl unit может получить PASS;

REJECT
  → при новом crawl unit получает REJECTED;

TIMEOUT
  → применяется default action HITL/risk policy.
```

Решение HITL относится к конкретному:

```text
source_system
source_scope
source_object_id
source_revision_id
unit_source_anchor
```

Если новый crawl относится к другой ревизии или другому unit, решение переоценивается.

## Инварианты

1. Quality Gate выполняется после формирования `structured_content` и `content_hash`.
2. Quality Gate выполняется до Identity Resolution.
3. Quality Gate не вычисляет и не изменяет `entity_uid`.
4. Quality Gate не вычисляет и не изменяет `entity_version_id`.
5. Quality Gate не пересчитывает `content_hash`.
6. `quality_verdict` является unit-level результатом текущего ingestion run.
7. `processing_status`, `quality_verdict`, `completeness_status`, `presence_status` и `canonical_status` имеют разную семантику.
8. `PASS` означает пригодность для Identity Resolution, но не означает resolved identity.
9. `FAILED` parsing/extraction с доступным source content получает `REJECTED`, а не `PASS`.
10. Наличие `content_hash` у `REJECTED` unit не разрешает materialization.
11. Безопасный текстовый `AMBIGUOUS` fallback может получить `PASS` только по правилам классификации.
12. Структурно неоднозначный или потенциально опасный `AMBIGUOUS` unit получает `QUARANTINED`.
13. Unit, входящий в `canonical_content_payload`, обязателен для полной публикации.
14. Quality Gate не устанавливает `CANONICAL_PUBLISHED`.
15. `CANONICAL_PUBLISHED` возможен только после identity readiness, обязательной materialization и reconciliation.
16. `REJECTED` и `QUARANTINED` units не создают domain nodes Neo4j.
17. `parent_unit_id` используется только внутри in-memory IR.
18. Quality results сохраняются в ingestion metadata, а не изменяют immutable manifest.
19. `QUALITY_REJECTED` является терминальным quality result, а не техническим failure.
20. Обязательный `QUARANTINED` и unresolved identity не продвигают cursor.
21. Snapshot не используется для долгосрочного replay.
22. Решение HITL применяется только к совпадающим source и unit identifiers.
23. Изменение Quality Gate rules требует полного re-crawl.
24. `tenant_id`, ACL metadata, Bamboo, `content_policy_version`, `content_version` и `crawl_mode` не используются.

## Новые переменные

| Переменная | Тип | Описание | Точка появления |
|---|---|---|---|
| `quality_verdict` | enum | Unit-level verdict: `PASS`, `QUARANTINED`, `REJECTED` | Quality Gate |
| `quality_reason` | string | Структурированная причина quality decision | Quality Gate |
| `Entity Policy` | configuration | Правила обязательности units и materialization targets | Ingestion configuration |
| `required_units` | collection | Units, обязательные для полной публикации entity | Entity Policy |
| `optional_units` | collection | Units, rejection которых не блокирует entity при соблюдении content payload rules | Entity Policy |
| `required_materialization_targets` | enum/collection | Обязательные targets: `QDRANT`, `NEO4J`, `QDRANT_AND_NEO4J`, `NONE` | Entity Policy |


## Последствия

### Положительные

* Quality Gate отбраковывает непригодный контент до Identity Resolution.
* `PASS` имеет однозначную семантику: unit готов к identity resolution.
* `FAILED` extraction не маскируется под успешную обработку.
* Наличие `content_hash` сохраняется даже для доступного, но отклонённого source content.
* Обязательные и необязательные units обрабатываются различно.
* Unit, входящий в `canonical_content_payload`, не может быть молча исключён из полной публикации.
* Quality status отделён от identity и publication status.
* Технические quality failures не загрязняют Neo4j domain graph.
* RAG и Graph используют единый quality verdict.
* Cursor policy для quality rejection, quarantine и unresolved identity согласована с ingestion lifecycle.

### Отрицательные

* Требуются Entity Policy и отдельное ingestion metadata.
* Обязательный `QUARANTINED` может задерживать обработку source scope.
* HITL resolution требует нового crawl, поскольку snapshot не хранится для replay.
* Для каждого `content_type` нужен отдельный validator.
* Предварительная проверка качества требует обработки source content до Identity Resolution.
* Изменение Quality Gate rules требует полного re-crawl.

## Рассмотренные альтернативы

### Выполнять Identity Resolution до Quality Gate

Сначала назначать identity, а затем проверять качество content unit.

**Плюсы:**

* identity доступна всем последующим этапам;
* проще связать quality result с entity;
* можно использовать entity context при проверке.

**Минусы:**

* identity resolution выполняется для пустых, повреждённых и unsupported units;
* создаются identity records для данных, которые не будут опубликованы;
* усложняется обработка rejected units;
* повышается риск ложного identity mapping.

**Решение:** отклонено. Quality Gate выполняется до Identity Resolution.

### Обрабатывать любой `AMBIGUOUS` как `NARRATIVE_TEXT`

Использовать текстовый fallback для любого неоднозначного unit.

**Плюсы:**

* меньше блокирующих случаев;
* больше контента попадает в Qdrant;
* меньше HITL cases.

**Минусы:**

* code, tables и diagrams теряют структуру;
* неизвестный binary или macro content может попасть в LLM;
* возрастает риск неверного `content_hash`;
* Graph Extraction получает недостоверное представление.

**Решение:** отклонено. Безопасный текстовый fallback разрешается только правилами классификации.

### Полностью отбраковывать entity при ошибке любого unit

Не публиковать entity, если любой её unit получил `REJECTED` или `QUARANTINED`.

**Плюсы:**

* простая агрегация;
* entity всегда представлена полностью;
* отсутствуют частичные representations.

**Минусы:**

* один необязательный attachment или macro блокирует полезный контент;
* снижается recall;
* техническая ошибка распространяется на всю entity.

**Решение:** отклонено. Используется Entity Policy с обязательными и необязательными units.

### Присваивать `PASS` unit с failed extraction

Считать доступность исходного содержимого достаточным основанием для передачи unit дальше, даже если extraction завершился ошибкой.

**Плюсы:**

* можно сохранить больше content states;
* `content_hash` уже вычислен;
* меньше терминальных quality rejections.

**Минусы:**

* unit формально прошёл Quality Gate, но фактически не пригоден для downstream;
* Chunking и Graph Extraction не имеют валидного structured input;
* возникает расхождение между `content_hash` и опубликованным представлением;
* обязательный failed unit может быть ошибочно исключён из materialization.

**Решение:** отклонено. Доступный source content позволяет вычислить `content_hash`, но failed extraction получает `REJECTED`.

### Создавать технические quality nodes в Neo4j

Записывать rejected и unsupported units в граф как `UnsupportedUnit`.

**Плюсы:**

* ошибки видны в общем хранилище;
* можно выполнять graph-based диагностику.

**Минусы:**

* технические записи загрязняют domain graph;
* усложняются traversal, retention и retrieval;
* quality failure ошибочно выглядит как domain entity.

**Решение:** отклонено. Quality results хранятся в ingestion metadata.
