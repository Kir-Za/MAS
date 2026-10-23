# ADR Извлечение сущностей и подготовка графа

**Статус:** Предложен

---

## Контекст и проблема

После получения, классификации, парсинга, нормализации и Quality Gate данные представлены проверенными processing units:

- `structured_content`;
- `content_type`;
- `processing_status`;
- `quality_verdict`;
- `content_hash`.

Identity Resolution выполняется после Quality Gate. Для units, прошедших Quality Gate и identity resolution, доступны:

- `entity_uid`;
- `entity_version_id`;
- `source_revision_id`;
- `source_scope`;
- provenance.

Neo4j должен содержать доменные сущности и связи, а также согласованные provenance и semantic lineage relations. Технические ошибки, `REJECTED` units и `UnsupportedUnit` не должны становиться доменными узлами графа.

Graph Extraction не должен повторно назначать identity, пересчитывать `content_hash` или самостоятельно интерпретировать source records. Он должен использовать тот же `structured_content`, который был сформирован на этапе парсинга и прошёл Quality Gate.

**Проблема:** как детерминированно извлечь доменные сущности и связи из проверенных processing units, подготовить согласованный набор для Neo4j и не допустить повторного identity resolution или загрязнения графа техническими объектами.

---

## Факторы решения

* Согласованность с `entity_uid` и `entity_version_id`, назначенными на предыдущих этапах.
* Использование одного `structured_content` для Graph Extraction и Chunking.
* Разделение domain entities, provenance relations и semantic lineage relations.
* Отсутствие технических quality nodes в Neo4j.
* Поддержка кода, Confluence и Jira.
* Сохранение source revision и temporal context.
* Детерминированность повторной обработки одного snapshot.
* Совместимость с eventual consistency Qdrant и Neo4j.
* Независимость структурного Graph Extraction от Qdrant и embedding.
* Возможность выполнить структурный Graph Extraction параллельно с Chunking после общей identity/version barrier.
* Semantic domain relations (`DESCRIBES`, `DOCUMENTS`, `SEMANTIC_ASSOCIATION`) требуют двухфазного подхода: структурная фаза до Qdrant, семантическая фаза после Qdrant.

---

## Решение

Мы выполняем Graph Extraction в две фазы:

```text
Structural Graph Extraction (до Qdrant)
    ↓
Chunking
    ↓
Embedding
    ↓
Qdrant Materialization
    ↓
Semantic Candidate Search (по Qdrant)
    ↓
Semantic Relation Apply (после Qdrant)
    ↓
Reconciliation
```

**Structural Graph Extraction** выполняется параллельно с Chunking после общей identity/version barrier.

**Semantic Relation Apply** выполняется после Qdrant materialization и использует Qdrant для semantic candidate generation.

Graph Extraction использует только units, которые:

```text
quality_verdict = PASS
entity_uid определён
entity_version_id определён
structured_content доступен
```

Graph Extraction не изменяет:

```text
entity_uid
entity_version_id
content_hash
entity_revision_observation
entity_presence_observation
```

### 1. Входные данные

Graph Extraction получает:

```text
source_system
snapshot_id
source_scope
source_object_id
source_revision_id
source_container
content_type
structured_content
content_hash
entity_uid
entity_version_id
unit_source_anchor
provenance
quality_verdict
```

`parent_unit_id` используется только внутри in-memory IR и не передаётся как постоянный идентификатор графа.

Units с:

```text
quality_verdict = REJECTED
quality_verdict = QUARANTINED
entity_uid отсутствует
entity_version_id отсутствует
```

не передаются в Graph Extraction.

### 2. Границы ответственности

Graph Extraction отвечает за:

- извлечение доменных сущностей из `structured_content`;
- извлечение структурных доменных отношений;
- извлечение семантических доменных отношений (`DESCRIBES`, `DOCUMENTS`, `SEMANTIC_ASSOCIATION`) на основе semantic candidate generation;
- привязку результата к `entity_uid` и `entity_version_id`;
- подготовку provenance и lineage relations к записи;
- формирование `GraphMaterializationSet`.

Graph Extraction не отвечает за:

- назначение `entity_uid`;
- выполнение merge и создание `entity_redirect`;
- вычисление `content_hash`;
- принятие Quality Gate verdict;
- генерацию semantic lineage candidates;
- запись технических quality failures как domain nodes;
- запись в Neo4j (выполняется Graph Apply).

Финальная запись в Neo4j выполняется единым Graph Apply в рамках downstream materialization.

### 3. Объект извлечения

Graph Extraction различает:

```text
Domain Entity
  логическая сущность предметной области с entity_uid;

Domain Relation
  предметная связь между domain entities;

Provenance Relation
  связь source object/event с entity или другим source object;

Semantic Lineage Relation
  связь между подтверждёнными entity_uid (COPIED_FROM, SPLIT_FROM, REPLACES);

Semantic Domain Relation
  связь между сущностями на основе семантической близости
  (DESCRIBES, DOCUMENTS, SEMANTIC_ASSOCIATION);

Lineage Evidence
  доказательство semantic lineage relation.
```

`entity_uid` и `entity_version_id` используются только для domain entities. Source objects и events, например Git commit или Jira issue, используют source-specific identifiers.

### 4. Извлечение по типам контента

#### CODE

Graph Extraction извлекает из code `structured_content`:

- code entities, уже разрешённые Identity Resolver;
- наследование;
- реализацию интерфейсов;
- вызовы;
- импорты;
- зависимости;
- framework-specific relations, если они разрешены graph policy.

Извлечение не создаёт новый `entity_uid`. Если LSP или framework extractor обнаружил symbol, отсутствующий в Identity Resolution, он не записывается как подтверждённая domain entity в текущем Graph Apply.

#### `NARRATIVE_TEXT`

Graph Extraction извлекает только сущности и связи, предусмотренные graph policy. Обычный текст не превращается автоматически в набор узлов графа.

Извлекаемые сущности должны иметь:

- source provenance;
- `entity_uid`, если они являются уже разрешёнными entities;
- confidence/evidence, если разрешение выполняется на этом этапе policy.

Неразрешённые сущности не становятся active graph nodes без Identity Resolution.

#### `TABLE`

Таблица может использоваться для извлечения domain entities и relations, если её структура соответствует graph policy.

Таблица не создаёт graph nodes автоматически только из-за наличия строк. Требуется определить, что строки и колонки представляют доменные объекты или отношения.

#### `DIAGRAM`

Из диаграммы извлекаются:

- компоненты;
- сервисы;
- модули;
- зависимости;
- архитектурные связи;

только если diagram extraction успешно сформировал структурированное представление.

Невалидная или неподдерживаемая диаграмма не создаёт domain nodes.

#### `STRUCTURED_RECORD`

Для Jira issue Graph Extraction может извлекать:

- issue entity;
- связи с другими Jira issues;
- связи с кодовыми entities, если они подтверждены;
- status или другие domain attributes, если они входят в graph policy.

Changelog является source revision/temporal input и не становится отдельной domain entity без специальной policy.

#### `BINARY_ATTACHMENT`

Attachment не становится graph entity автоматически. Graph Extraction может использовать извлечённые из attachment units, если они:

- успешно классифицированы;
- прошли Quality Gate;
- имеют необходимую identity;
- разрешены graph policy.

### 5. Domain relations

Graph Extraction формирует только отношения, разрешённые graph policy. Каждая relation содержит:

```text
relation_type
entity_uid_from
entity_uid_to
entity_version_id_from
entity_version_id_to
source_system_from
source_system_to
source_scope_from
source_scope_to
source_revision_id_from
source_revision_id_to
provenance
```

Для отношений, связывающих source objects/events с entities, применяется relation-specific schema:

```text
IMPLEMENTED_BY_COMMIT
TOUCHED_IN
AFFECTS
REALIZED_IN
```

Для semantic lineage применяются relations:

```text
COPIED_FROM
SPLIT_FROM
REPLACES
```

Для семантических доменных отношений (cross-source relations между документацией и кодом) используются:

```text
DESCRIBES
  Confluence page/document → Code entity

DOCUMENTS
  Document/ADR → Code entity, service, module или relation

SEMANTIC_ASSOCIATION
  Обобщённая семантическая связь между сущностями,
  когда более точный тип связи не установлен
```

Семантические доменные отношения имеют следующие правила:

- обе endpoint entities уже имеют `entity_uid`;
- обе endpoint entities имеют version context;
- similarity создаёт только candidate;
- similarity не назначает identity;
- similarity не выполняет merge;
- similarity не создаёт `entity_redirect`;
- active relation создаётся только после установленного evidence/policy decision;
- relation имеет source provenance;
- relation не входит в `content_hash`;
- relation не заменяет `COPIED_FROM`, `SPLIT_FROM` и `REPLACES`.

`entity_redirect` и merge применяются Canonical Identity Resolver и не реализуются Graph Extraction как самостоятельная lineage relation.

### 6. Двухфазный семантический поиск

#### Фаза 1: Structural Graph Extraction

Выполняется **до** Qdrant materialization:

- извлечение доменных сущностей;
- извлечение структурных отношений (наследование, импорты, зависимости);
- подготовка `GraphMaterializationSet` для структурной части;
- передача структурных relations в Graph Apply.

#### Фаза 2: Semantic Candidate Generation и Apply 

Выполняется **после** Qdrant materialization:

1. **Semantic Candidate Search:**
   - Для каждого narrative/текстового unit (Confluence, Jira summary/description, code docstring) выполняется поиск по Qdrant.
   - Используются dense и sparse vectors (BGE-M3).
   - Поиск выполняется среди сущностей с финализированными `entity_uid` и `entity_version_id`.
   - Результат: `similarity_score` для каждой пары.

2. **Candidate Generation:**
   - Если `similarity_score >= threshold`, создаётся `CandidateRecord`.
   - Тип обнаружения: `DESCRIBES`, `DOCUMENTS` или `SEMANTIC_ASSOCIATION`.
   - Сохраняется source контекст обеих сторон.

3. **Evidence/Policy Decision:**
   - Кандидат может быть принят (`ACCEPTED`) по:
     - `RULE` — автоматическое правило (например, несколько evidence совпадают);
     - `SOURCE` — явное утверждение (например, ссылка в Confluence);
     - `HITL` — ручное подтверждение.
   - Принятый кандидат становится активной semantic domain relation.

4. **Semantic Relation Apply:**
   - Активная relation записывается в `GraphMaterializationSet`.
   - Сохраняется `confidence`, `resolution_method`, source и version context.

5. **Graph Apply:**
   - Semantic domain relations записываются в Neo4j вместе со структурными relations.

**Важно:** Отсутствие semantic candidate или low-confidence candidate **не блокирует** ingestion. Cursor продвигается после обязательной materialization.

### 7. Candidate/Evidence Policy для семантических доменных отношений

Для семантических доменных отношений используется следующий контракт:

```text
SemanticCandidate:
  candidate_id
  detection_type: enum[DESCRIBES|DOCUMENTS|SEMANTIC_ASSOCIATION]
  entity_uid_from
  entity_uid_to
  entity_version_id_from
  entity_version_id_to
  source_system_from
  source_system_to
  source_scope_from
  source_scope_to
  source_revision_id_from
  source_revision_id_to
  similarity_score
  confidence
  observed_at
  candidate_status: enum[CANDIDATE|ACCEPTED|DISMISSED|APPLIED|EXPIRED]
  resolution_method: enum[RULE|SOURCE|HITL]
```

`similarity_score` — сырой результат алгоритма similarity (cosine similarity embeddings).

`confidence` — итоговая оценка evidence policy после учёта similarity и других evidence.

Порог для `ACCEPTED` определяется policy:

```text
similarity_score >= 0.85 → может быть ACCEPTED по RULE
similarity_score >= 0.70 → может быть ACCEPTED по SOURCE (при наличии дополнительного evidence)
similarity_score < 0.70 → только CANDIDATE, требует HITL
```

### 8. Provenance и lineage

Graph Extraction использует результаты этапа дедупликации:

- применённые provenance relations;
- применённые semantic lineage relations;
- `LineageEvidence`;
- unresolved candidates, если они должны быть сохранены в ingestion metadata.

Graph Extraction не превращает candidate в active relation. Active relation может быть записана только после:

- Identity Finalization;
- завершения соответствующего evidence/policy decision;
- проверки наличия финализированных endpoint identities.

Повторное обнаружение существующей relation не должно создавать дубликат.

### 9. Temporal context

Каждая извлечённая domain entity и relation привязывается к:

```text
source_scope
source_revision_id
entity_version_id
```

Graph Extraction не изменяет:

```text
valid_from
valid_until
valid_until_unknown
valid_time_source
```

### 10. Результат Graph Extraction

Graph Extraction формирует:

```text
GraphMaterializationSet:
  snapshot_id
  source_scope
  source_revision_id
  domain_entities
  structural_domain_relations
  semantic_domain_relations      # DESCRIBES, DOCUMENTS, SEMANTIC_ASSOCIATION
  provenance_relations
  lineage_relations
  lineage_evidence
  semantic_candidates
```

`GraphMaterializationSet` не является записью Neo4j. Это подготовленный набор для Graph Apply.

В набор не включаются:

- `REJECTED` units;
- `QUARANTINED` units;
- `UnsupportedUnit`;
- unresolved active relations;
- entities без разрешённой identity;
- relations без обязательных endpoint identifiers.

### 11. Graph Apply

Graph Apply выполняется после завершения Graph Extraction и получает один согласованный `GraphMaterializationSet`.

Порядок применения:

```text
1. Entity nodes
2. Entity version references
3. Structural domain relations
4. Semantic domain relations (DESCRIBES, DOCUMENTS, SEMANTIC_ASSOCIATION)
5. Provenance relations
6. Semantic lineage relations
7. Lineage evidence references
8. Graph materialization result
```

Relations не должны записываться раньше обязательных endpoint nodes.

Graph Apply должен быть идемпотентным для одного:

```text
snapshot_id
+ source_scope
+ source_revision_id
```


### 12. Параллельность с Chunking

После определения версий и завершения identity/version finalization **структурный** Graph Extraction и Chunking могут выполняться параллельно:

```text
Structural Graph ∥ Chunking
```

Параллельность допускается только потому, что обе ветви используют:

```text
snapshot_id
source_scope
source_revision_id
entity_uid
entity_version_id
content_hash
structured_content
```

**Семантическая** фаза выполняется **после** Qdrant materialization и не может выполняться параллельно с Chunking.

Финальная запись в Neo4j выполняется единым Graph Apply. Lineage relations не записываются отдельным независимым writer-треком.

---

## Инварианты

1. Graph Extraction выполняется только для units, прошедших Quality Gate.
2. Graph Extraction не назначает и не изменяет `entity_uid`.
3. Graph Extraction не вычисляет и не изменяет `content_hash` и `entity_version_id`.
4. `REJECTED` и `QUARANTINED` units не создают domain nodes.
5. `UnsupportedUnit` не записывается в Neo4j как domain entity.
6. `parent_unit_id` используется только внутри in-memory IR.
7. Graph Extraction использует тот же `structured_content`, что и Chunking.
8. Source objects/events не смешиваются с domain entities.
9. Provenance relations и semantic lineage relations используют разные контракты.
10. `entity_redirect` и merge регулируются ADR-003.
11. Active lineage relations используют только финализированные `entity_uid`.
12. Candidate не становится active relation без предусмотренного evidence/policy decision.
13. Relations не записываются раньше обязательных endpoint entities.
14. Graph Extraction не изменяет temporal fields из ADR-004.
15. Повторная обработка одного snapshot и source revision идемпотентна.
16. Graph Extraction не записывает данные в Neo4j напрямую.
17. Структурный Graph Extraction и Chunking могут выполняться параллельно после общей identity/version barrier.
18. Результат Graph Apply участвует в reconciliation вместе с Qdrant materialization.
19. Semantic domain relations (`DESCRIBES`, `DOCUMENTS`, `SEMANTIC_ASSOCIATION`) формируются только после Qdrant materialization.
20. Semantic similarity не используется для identity resolution.
21. Semantic similarity не используется для merge или `entity_redirect`.
22. Semantic domain relations имеют отдельный lifecycle и не смешиваются с lineage relations.
23. Отсутствие semantic candidate или low-confidence candidate не блокирует cursor и ingestion.
24. Семантические кандидаты сохраняются в ingestion metadata для последующего HITL или повторного анализа.

---

## Новые переменные

| Переменная | Тип | Описание | Точка появления |
|---|---|---|---|
| `GraphMaterializationSet` | структура | Подготовленный набор domain entities, relations, provenance и lineage для Graph Apply | Graph Extraction |
| `structural_domain_relations` | collection | Структурные доменные отношения (наследование, импорты) | Structural Graph Extraction |
| `semantic_domain_relations` | collection | Семантические доменные отношения (`DESCRIBES`, `DOCUMENTS`, `SEMANTIC_ASSOCIATION`) | Semantic Relation Apply |
| `domain_entities` | collection | Извлечённые подтверждённые доменные сущности | Graph Extraction |
| `domain_relations` | collection | Все доменные отношения (структурные + семантические) | Graph Extraction |
| `provenance_relations` | collection | Provenance relations для Graph Apply | Graph Extraction/Deduplication |
| `lineage_relations` | collection | Подтверждённые semantic lineage relations | Deduplication/Graph Extraction |
| `lineage_evidence` | collection | Evidence semantic lineage relations | Deduplication/Graph Extraction |
| `semantic_candidates` | collection | Кандидаты семантических доменных отношений | Semantic Relation Apply |
| `entity_uid_from` | `uuid` | Identity исходного endpoint relation | Graph Relation Builder |
| `entity_uid_to` | `uuid` | Identity целевого endpoint relation | Graph Relation Builder |
| `entity_version_id_from` | `string` | Версия исходного entity endpoint | Graph Relation Builder |
| `entity_version_id_to` | `string` | Версия целевого entity endpoint | Graph Relation Builder |
| `source_system_from` | enum | Source system исходного endpoint | Relation Builder |
| `source_system_to` | enum | Source system целевого endpoint | Relation Builder |
| `source_scope_from` | `string` | Scope исходного endpoint | Relation Builder |
| `source_scope_to` | `string` | Scope целевого endpoint | Relation Builder |
| `source_revision_id_from` | `string` | Revision исходного endpoint | Relation Builder |
| `source_revision_id_to` | `string` | Revision целевого endpoint | Relation Builder |

---

## Последствия

### Положительные

* Graph Extraction использует единую identity и temporal model.
* RAG и Graph обрабатывают один и тот же `structured_content`.
* Технические quality failures не загрязняют Neo4j.
* Provenance relations отделены от semantic lineage.
* Semantic domain relations (`DESCRIBES`, `DOCUMENTS`, `SEMANTIC_ASSOCIATION`) отделены от lineage.
* Graph Apply получает согласованный набор для записи в Neo4j.
* Структурный Graph Extraction может выполняться параллельно с Chunking.
* `entity_redirect` и merge не дублируются в Graph Extraction.
* Повторная обработка одного snapshot и source revision поддерживает идемпотентность.
* Graph materialization может быть проверена вместе с Qdrant в reconciliation.
* Semantic similarity используется только для domain relations, а не для identity.
* Отсутствие semantic relation не блокирует ingestion.
* Semantic candidates сохраняются для последующего HITL.

### Отрицательные

* Требуется отдельный `GraphMaterializationSet` перед Graph Apply.
* Graph policy должна определять, какие content units и relations являются доменными.
* Ошибки LSP, diagram extraction или структурного разбора могут уменьшить полноту графа.
* Необходимо поддерживать разные схемы provenance, lineage и semantic domain relations.
* Параллельные Graph и Qdrant branches требуют reconciliation.
* Семантическая фаза (Semantic Relation Apply) выполняется после Qdrant, что увеличивает общую latency.
* Semantic candidate generation требует вычислительных ресурсов (Qdrant search).

---

## Рассмотренные альтернативы

### Записывать данные в Neo4j непосредственно из Graph Extraction

Graph Extraction сразу создаёт или обновляет nodes и relations в Neo4j.

**Плюсы:**

* меньше промежуточных артефактов;
* проще короткий pipeline;
* ниже задержка между extraction и записью.

**Минусы:**

* extraction и materialization становятся неразделимыми;
* сложнее retry и idempotency;
* частичная ошибка может оставить неполный граф;
* relations могут записываться раньше endpoint nodes;
* сложнее reconciliation и тестирование.

### Выполнять Graph Extraction после завершения Chunking и Qdrant materialization

Сначала полностью формировать и записывать Qdrant, затем строить граф.

**Плюсы:**

* последовательный pipeline;
* проще визуально контролировать порядок;
* можно использовать информацию о созданных chunks.

**Минусы:**

* Graph не должен зависеть от готовности Qdrant;
* увеличивается latency ingestion;
* отказ Qdrant блокирует построение Neo4j;
* нарушается независимость materialization branches.

### Извлекать граф только из Qdrant chunks

Строить Neo4j entities и relations на основании уже сформированных chunks.

**Плюсы:**

* единый вход для текстового и графового контура;
* меньше parser/extractor вызовов;
* проще связать Graph с Qdrant points.

**Минусы:**

* chunking может потерять структурные сведения;
* chunk не обязан содержать полную entity;
* code и diagram relations извлекаются хуже;
* Qdrant representation не является авторитетным source IR;
* увеличивается зависимость Graph от стратегии chunking.

### Создавать Graph node для каждого processing unit

Каждый unit независимо материализуется как node Neo4j.

**Плюсы:**

* простое отображение IR в граф;
* сохраняется больше технических объектов;
* проще первичная трассировка units.

**Минусы:**

* технические units загрязняют domain graph;
* `UNSUPPORTED`, metadata и attachments начинают выглядеть как domain entities;
* ухудшается graph retrieval;
* растёт объём nodes и relations;
* нарушается граница между content unit и logical entity.

### Выполнять semantic lineage и Graph Extraction одним общим алгоритмом

В одном этапе одновременно извлекать сущности, отношения и lineage.

**Плюсы:**

* один проход по данным;
* потенциально меньше вычислительных затрат;
* можно использовать общие структурные признаки.

**Минусы:**

* смешиваются extraction и cross-entity comparison;
* сложнее контролировать identity и evidence;
* возрастает риск ложных lineage relations;
* трудно независимо переигрывать lineage detection;
* нарушается разделение provenance и semantic lineage.

**Решение:** использовать общий проверенный IR, но разделять Graph Extraction, lineage candidate generation и финальный Graph Apply.

### Включить semantic domain relations (lineage)

Рассматривать `DESCRIBES`, `DOCUMENTS`, `SEMANTIC_ASSOCIATION` как часть lineage detection.

**Плюсы:**

* единый механизм для всех cross-entity relations;
* проще централизованное применение;
* меньше этапов.

**Минусы:**

* смешиваются разные типы relations с разной семантикой;
* `DESCRIBES` (документация → код) не является lineage;
* усложняется lifecycle управления;
* нарушается разделение ответственности.

**Решение:** отклонено. Semantic domain relations формируются на этапе работы с графом.