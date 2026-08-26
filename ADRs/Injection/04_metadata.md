# ADR Контракт метаданных и трассировки ingestion

**Статус:** Предложен

---

## Контекст и проблема

После парсинга данные из Git, Jira и Confluence представлены классифицированными и распарсенными content units. До Quality Gate, Identity Resolution, Version Finalization, Graph Extraction и Chunking эти данные должны передаваться в едином формате.

Разные этапы добавляют собственные значения:

- Snapshot фиксирует источник, scope, snapshot, revision и границы crawl;
- Классификация определяет `source_container`, `content_type`, `processing_status` и локализацию unit;
- Парсинг формирует `structured_content`, `canonical_content_payload`, `content_hash` и code-specific fields;
- Quality Gate формирует `quality_verdict`;
- Identify назначает `entity_uid`;
- Version формирует `entity_version_id` и observations.

Без общего контракта этапы могут по-разному трактовать один и тот же source record, терять связь с `snapshot_id` или смешивать source metadata, identity, quality и publication status.

**Проблема:** как определить единый поэтапно обогащаемый контракт `ProcessingUnit`, который сохраняет source context и provenance, но не смешивает ответственность соседних ingestion ADR.

---

## Факторы решения

* Единый формат передачи данных между ingestion stages.
* Сохранение связи с `snapshot_id` и `source_revision_id`.
* Совместимость с ADR — Source Snapshot.
* Совместимость с ADR Classification.
* Совместимость с ASR — Parsed IR и `content_hash`.
* Возможность выполнить Quality Gate до Identity Resolution.
* Отсутствие identity-полей до прохождения Quality Gate.
* Отделение временного `parent_unit_id` от постоянных идентификаторов.
* Согласованность RAG и Graph pipeline.
* Отсутствие `tenant_id`, ACL metadata, Bamboo, `content_policy_version`, `content_version` и отдельного `crawl_mode`.
* Необходимость сохранения source context для semantic domain relations (`DESCRIBES`, `DOCUMENTS`, `SEMANTIC_ASSOCIATION`) на этапе Graph Extraction. Semantic relations должны иметь provenance, позволяющий связать их с конкретными source objects, scope, revision и unit anchors.

---

## Решение

Мы вводим поэтапно обогащаемый контракт `ProcessingUnit`. Он передаётся между ingestion stages и содержит только данные, необходимые для текущего и последующих этапов.

Общий поток обогащения:

```text
Source Snapshot
→ ProcessingUnit после классификации
→ Parsed ProcessingUnit после парсинга
→ Quality Gate result после QG
→ Identity-resolved ProcessingUnit идентификации entity
→ Version-finalized ProcessingUnit после версионирования
```

`ProcessingUnit` не является постоянной domain entity и не заменяет:

- `EntityRevisionObservation`;
- `EntityPresenceObservation`;
- `IngestionRunResult`;
- `EntityAlias`;
- `EntityRedirect`;
- `ProvenanceEdge`;
- `LineageRelation`;
- семантические domain relations (формируются на Graph Extraction).

### 1. Контракт `ProcessingUnit` после парсинга

```text
ProcessingUnit:
  # Source context
  source_system
  source_scope
  source_object_id
  source_revision_id
  source_changed_at
  snapshot_id
  ingestion_run_id
  observed_at
  coverage
  completeness_status

  # Classification
  source_container
  content_type
  processing_status
  unit_source_anchor
  parent_unit_id

  # Parsing and normalization
  structured_content
  content_hash
  raw_payload_checksum
  child_unit_source_anchor

  # Code-specific fields
  module_path
  qualified_symbol
  symbol_kind
  signature_fingerprint
  full_symbol_span_text
  declared_interfaces_or_base_types

  # Provenance
  provenance:
    connector_version
    extractor_version
    parser_version
    quality_gate_version
    identity_resolver_version
```

### 2. Source context

Поля source context передаются из snapshot и не изменяются последующими этапами:

```text
source_system
source_scope
source_object_id
source_revision_id
snapshot_id
ingestion_run_id
observed_at
coverage
completeness_status
```

`source_changed_at` передаётся, если источник его предоставляет. Его отсутствие не является ошибкой: `valid_from` может быть определён с помощью `observed_at` или source history.

Для `PRESENT` source records `source_revision_id` должен быть определён Connector'ом. Если источник не предоставляет native revision, Connector формирует deterministic composite ID.

Для отсутствующих объектов `source_revision_id` может отсутствовать; такие факты фиксируются через `EntityPresenceObservation`, а не через обычный `ProcessingUnit`.

### 3. Classification context

Этап классификации добавляет:

```text
source_container
content_type
processing_status
unit_source_anchor
parent_unit_id
```

`source_container` описывает способ упаковки контента:

```text
PAGE
FILE
CODE_FILE
MACRO
ATTACHMENT
FIELD
CHANGELOG
```

`content_type` описывает семантический тип unit:

```text
NARRATIVE_TEXT
CODE
TABLE
DIAGRAM
STRUCTURED_RECORD
BINARY_ATTACHMENT
UNSUPPORTED
```

`processing_status` описывает результат classification/extractor:

```text
CLASSIFIED
AMBIGUOUS
UNSUPPORTED
FAILED
```

`parent_unit_id` используется только для навигации по in-memory IR. Он не является identity и не передаётся в постоянные materializations.

`unit_source_anchor` является source-local ссылкой на конкретный fragment внутри `source_object_id`. Он не заменяет `entity_uid`, `entity_version_id` или `chunk_id`.

### 4. Parsing and normalization context

Парсинг добавляет:

```text
structured_content
content_hash
raw_payload_checksum
child_unit_source_anchor
```

`structured_content` является единым структурированным входом для:

- Quality Gate;
- Graph Extraction;
- Chunking.

`content_hash` вычисляется из `canonical_content_payload` до Quality Gate и Identity Resolution. Он не зависит от:

```text
entity_uid
entity_version_id
quality_verdict
chunking
embedding
graph extraction
```

`raw_payload_checksum` является fingerprint исходного payload. Он не заменяет raw payload и не обеспечивает replay после удаления временного snapshot.

`child_unit_source_anchor` используется для structural fallback и локализации дочернего unit. Он не является identity.

### 5. Metadata для semantic relations

Для поддержки semantic domain relations (`DESCRIBES`, `DOCUMENTS`, `SEMANTIC_ASSOCIATION`) на этапе Graph Extraction, `ProcessingUnit` сохраняет source context, необходимый для построения provenance semantic relations.

После прохождения Identity Resolution и Version Finalization каждая сторона потенциальной semantic relation имеет:

```text
entity_uid              # назначен в Identity Resolution
entity_version_id       # создан в Version Finalization
EntityRevisionObservation  # создан в Version Finalization
source_system
source_scope
source_object_id
source_revision_id
unit_source_anchor
```

Эти поля используются Graph Extraction для:

- идентификации endpoint'ов semantic relation;
- привязки relation к конкретным source объектам и ревизиям;
- формирования provenance semantic relation;
- сохранения source context для аудита и отладки.

**Semantic relation provenance включает:**

```text
entity_uid_from
entity_uid_to
entity_version_id_from
entity_version_id_to
source_system_from
source_system_to
source_scope_from
source_scope_to
source_object_id_from
source_object_id_to
source_revision_id_from
source_revision_id_to
unit_source_anchor_from
unit_source_anchor_to
observed_at
similarity_score
confidence
resolution_method
```

Эти поля не добавляются в `ProcessingUnit` как отдельные поля, но являются результатом работы Graph Extraction (семантическая фаза) и сохраняются в `GraphMaterializationSet` и Neo4j.

### 6. Identity и version fields

До прохождения Quality Gate `ProcessingUnit` не содержит обязательных:

```text
entity_uid
entity_version_id
derived_key
identity_status
```

После QG:

```text
quality_verdict = PASS
```

означает, что unit пригоден для Identity Resolution, но identity ещё может отсутствовать.

После идентификации:

```text
entity_uid
derived_key
identity_status
```

заполняются для resolved identity.

Если identity не разрешена:

```text
quality_verdict = PASS
entity_uid отсутствует
identity_status = AMBIGUOUS
```

создаётся `UnresolvedIdentityRecord`, а `ProcessingUnit` не передаётся в Version Finalization или materialization.

После версионирования для resolved identity добавляются:

```text
entity_version_id
valid_from
valid_until?
valid_until_unknown?
valid_time_source
is_initial_baseline?
```

`ProcessingUnit` не становится постоянным хранилищем этих значений: они переносятся в `EntityRevisionObservation`.

### 7. Provenance

`provenance` содержит версии компонентов, участвовавших в обработке:

```text
provenance:
  connector_version
  extractor_version
  parser_version
  quality_gate_version
  identity_resolver_version
```

Поля появляются по мере прохождения этапов:

| Поле | Точка появления |
|---|---|
| `connector_version` | Source Snapshot |
| `extractor_version` | Classification / Parsing |
| `parser_version` | Parsing |
| `quality_gate_version` | Quality Gate |
| `identity_resolver_version` | Identity Resolution |

Версия компонента не входит в:

```text
entity_uid
content_hash
entity_version_id
```

После создания `EntityRevisionObservation` постоянный provenance также содержит:

```text
snapshot_id
ingestion_run_id
source_revision_id
raw_payload_checksum
```

`source_revision_id` хранится в `EntityRevisionObservation` как отдельное top-level поле. Он не дублируется внутри provenance.

### 8. Состояния ProcessingUnit по этапам

#### После классификации

```text
source context
classification fields
unit_source_anchor
parent_unit_id
```

#### После парсинга

```text
structured_content
content_hash
raw_payload_checksum
code-specific fields
```

#### После Quality Gate

```text
quality_verdict
quality_reason
```

`quality_verdict` и `quality_reason` являются результатом текущего ingestion run и сохраняются в ingestion metadata.

#### После Identity Resolution

```text
entity_uid
derived_key
identity_status
```

или:

```text
UnresolvedIdentityRecord
```

#### После Version Finalization

```text
entity_version_id
valid_from
valid_until
valid_until_unknown
valid_time_source
is_initial_baseline
```

и создаётся `EntityRevisionObservation`.

### 9. Persistent records

`ProcessingUnit` используется in-memory. После завершения обработки постоянные данные распределяются по специализированным records:

```text
EntityRevisionObservation
EntityPresenceObservation
UnresolvedIdentityRecord
IngestionRunResult
EntityAlias
EntityRedirect
ProvenanceEdge
LineageRelation
```

Semantic domain relations (`DESCRIBES`, `DOCUMENTS`, `SEMANTIC_ASSOCIATION`) сохраняются в Neo4j через `GraphMaterializationSet` и имеют полный provenance:

```text
entity_uid_from
entity_uid_to
entity_version_id_from
entity_version_id_to
source_system_from
source_system_to
source_scope_from
source_scope_to
source_object_id_from
source_object_id_to
source_revision_id_from
source_revision_id_to
unit_source_anchor_from
unit_source_anchor_to
similarity_score
confidence
resolution_method
observed_at
```

`parent_unit_id` не сохраняется в постоянных materializations.

`structured_content` и `canonical_content_payload` могут быть удалены после завершения Graph Extraction и Chunking. Долговременные записи сохраняют необходимые:

```text
content_hash
raw_payload_checksum
source context
identity
version
provenance
```

### 10. Валидация контракта

На каждом этапе проверяется наличие полей, обязательных для данного этапа.

#### Source validation

Обязательны:

```text
source_system
source_scope
source_object_id
snapshot_id
ingestion_run_id
observed_at
coverage
completeness_status
```

Для `PRESENT` source record также обязателен `source_revision_id`.

#### Classification validation

Обязательны:

```text
source_container
content_type
processing_status
unit_source_anchor
```

#### Parsing validation

Для `processing_status=CLASSIFIED` обязательны:

```text
structured_content
content_hash
raw_payload_checksum
```

#### Quality validation

Для обработанного unit обязателен:

```text
quality_verdict
```

Для `quality_verdict=REJECTED` обязателен:

```text
quality_reason
```

#### Identity validation

Для `quality_verdict=PASS` после идентификации должно выполняться одно из условий:

```text
entity_uid присутствует
```

или:

```text
UnresolvedIdentityRecord создан
```

#### Version validation

Для resolved identity после определения версии обязательны:

```text
entity_uid
entity_version_id
valid_from
valid_time_source
```

### 11. Границы ответственности

ADR отвечает за:

- единый `ProcessingUnit` contract;
- обязательность полей по этапам;
- source context propagation;
- provenance propagation;
- mapping между ProcessingUnit и постоянными records;
- запрет смешения временных и постоянных идентификаторов;
- обеспечение наличия source context для semantic domain relations на этапе Graph Extraction.

ADR не отвечает за:

- получение источников;
- классификацию;
- parsing;
- вычисление `content_hash`;
- Quality Gate;
- Identity Resolution;
- version finalization;
- deduplication;
- lineage;
- Graph Extraction;
- Chunking;
- Embedding;
- Qdrant/Neo4j materialization;
- reconciliation;
- retention;
- формирование semantic domain relations (это Graph Extraction).

## Инварианты

1. `ProcessingUnit` является единым поэтапно обогащаемым объектом текущего ingestion.
2. Source context не меняется downstream-компонентами.
3. `source_revision_id` обязателен для `PRESENT` source records.
4. `source_changed_at` может отсутствовать.
5. `content_hash` вычисляется до Quality Gate и Identity Resolution.
6. `entity_uid` и `entity_version_id` не участвуют в вычислении `content_hash`.
7. `quality_verdict` является unit-level результатом Quality Gate.
8. `canonical_status` не является полем `ProcessingUnit`.
9. `entity_uid` появляется только после Quality Gate.
10. `entity_version_id` появляется только после назначения `entity_uid`.
11. `parent_unit_id` является временным IR-идентификатором.
12. `parent_unit_id` не используется для identity, cross-store join или постоянной materialization.
13. `unit_source_anchor` локализует unit, но не является identity.
14. `source_revision_id` не дублируется внутри provenance.
15. Версии компонентов не входят в identity и content hash.
16. `structured_content` является единым входом Quality Gate, Graph Extraction и Chunking.
17. `ProcessingUnit` не записывается напрямую в Qdrant или Neo4j.
18. `REJECTED` и `UNSUPPORTED` units не создают `EntityRevisionObservation`.
19. `PASS` без resolved identity приводит к `UnresolvedIdentityRecord`.
20. Только resolved `PASS` unit может создать `EntityRevisionObservation`.
21. `EntityRevisionObservation`, `EntityPresenceObservation` и `IngestionRunResult` имеют разные назначения.
22. `tenant_id`, ACL metadata, Bamboo, `content_policy_version`, `content_version` и `crawl_mode` не используются.
23. Semantic domain relations (`DESCRIBES`, `DOCUMENTS`, `SEMANTIC_ASSOCIATION`) имеют полный provenance, включающий source context обеих endpoint'ов.
24. Source context для semantic relations доступен после прохождения Identity Resolution и Version Finalization.

## Новые переменные

| Переменная | Тип | Описание | Точка появления |
|---|---|---|---|
| `ProcessingUnit` | структура | Единый поэтапно обогащаемый контракт processing unit | Metadata |
| `provenance` | структура | Версии компонентов, участвующих в обработке, и source trace context | Snapshot-Identify |
| `quality_verdict` | enum | Unit-level результат Quality Gate: `PASS`, `QUARANTINED`, `REJECTED` | Quality Gate |
| `quality_reason` | string? | Причина Quality Gate decision | Quality Gate |
| `UnresolvedIdentityRecord` | структура | Постоянная запись PASS unit с неразрешённой identity | Identity Resolution |
| `IngestionRunResult` | структура | Результат публикации entity в конкретном ingestion run | Completion |
| `valid_until_unknown` | boolean? | Признак неизвестной temporal boundary | Version Finalization |
| `lineage_detection_status` | enum? | In-memory статус lineage processing | Deduplication |
| `semantic_relation_provenance` | структура | Provenance semantic domain relation включая source context endpoint'ов | Graph Extraction |

## Последствия

### Положительные

* Все ingestion stages используют единый контракт `ProcessingUnit`.
* Source context и provenance не теряются при переходе между этапами.
* `content_hash`, identity, version и publication status имеют разные точки появления.
* Quality Gate может выполняться до Identity Resolution.
* `parent_unit_id` не смешивается с постоянной identity.
* `PRESENT`, rejected и unresolved identity cases имеют разные persistent records.
* RAG и Graph получают согласованный `structured_content`.
* Mapping в `EntityRevisionObservation` и `IngestionRunResult` становится явным.
* Source-specific optional fields не ошибочно объявляются обязательными для всех источников.
* Semantic domain relations имеют полный provenance, включающий source context обеих endpoint'ов, что обеспечивает трассируемость и аудит.

### Отрицательные

* Один `ProcessingUnit` содержит поля, обязательные на разных этапах, поэтому требуется условная валидация.
* Контракт становится шире по мере прохождения pipeline.
* Потребуется поддерживать отдельные persistent records для observations, unresolved identity, lineage и run results.
* Ошибка в mapping между in-memory unit и постоянной записью может привести к потере provenance.
* Изменение downstream-контрактов требует проверки совместимости `ProcessingUnit`.
* Semantic relation provenance требует дополнительных полей и структур в Graph Extraction.

## Рассмотренные альтернативы

### Независимый DTO для каждого этапа

Каждый ingestion stage получает и возвращает собственную структуру данных без общего `ProcessingUnit`.

**Плюсы:**

* каждый контракт содержит только необходимые поля;
* проще локально валидировать этап;
* изменения одного этапа меньше влияют на другие.

**Минусы:**

* требуется большое количество mapping-операций;
* повышается риск потери `snapshot_id`, source revision или provenance;
* RAG и Graph могут получать разные представления одного unit;
* сложнее отлаживать полный путь данных;
* возрастает вероятность несовместимых контрактов.

**Решение:** отклонено. Используется единый поэтапно обогащаемый `ProcessingUnit`.

### Передавать raw source record между всеми этапами

Каждый этап самостоятельно извлекает необходимые поля из исходной source record.

**Плюсы:**

* минимальная начальная модель;
* меньше явных промежуточных полей;
* легко добавлять source-specific metadata.

**Минусы:**

* downstream-компоненты начинают повторно парсить source content;
* классификация, parsing и normalization дублируются;
* возрастает риск разных `content_hash` и IR;
* Quality Gate, Chunking и Graph Extraction могут работать с разными представлениями.

**Решение:** отклонено. Используется Parsed IR и общий ProcessingUnit contract.

### Сохранять все поля в одном постоянном универсальном record

Один record хранит source data, IR, identity, observations, quality и publication status.

**Плюсы:**

* один объект для хранения и поиска;
* простая трассировка;
* не требуется отдельное mapping между records.

**Минусы:**

* смешиваются временные и постоянные данные;
* `canonical_status` ошибочно становится свойством source observation;
* растёт объём записи;
* усложняется lifecycle и retention;
* невозможно корректно представить несколько observations одной entity version.

**Решение:** отклонено. `ProcessingUnit` является in-memory контрактом, а постоянные факты сохраняются в специализированных records.

### Хранить provenance только в snapshot и manifest

Не передавать provenance в observations и materialized records.

**Плюсы:**

* меньше metadata в целевых записях;
* проще source record contract;
* snapshot является единой точкой происхождения.

**Минусы:**

* snapshot временный и удаляется после обработки;
* manifest не содержит всех сведений об extraction и identity;
* после удаления raw payload сложно восстановить происхождение результата;
* observations и Qdrant/Neo4j records теряют самостоятельную трассируемость.

**Решение:** отклонено. Необходимые provenance fields переносятся в постоянные observations и materialized records.

### Использовать `entity_uid` и `entity_version_id` до Quality Gate

Назначать identity до проверки качества контента.

**Плюсы:**

* каждый unit сразу связан с logical entity;
* проще агрегировать unit-level результаты;
* identity доступна Quality Gate.

**Минусы:**

* identity resolution выполняется для повреждённых и неподдерживаемых units;
* создаются identity records для данных, которые не будут опубликованы;
* усложняется обработка rejected units;
* возникает риск ложных identity mappings.

**Решение:** отклонено. Quality Gate выполняется до Identity Resolution.