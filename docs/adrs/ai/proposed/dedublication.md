# ADR Дедупликация и фиксация semantic lineage

**Статус:** Предложен

---

## Контекст и проблема

После этапа  каждый прошедший Quality Gate и успешно разрешённый processing unit имеет:

- `entity_uid`;
- `entity_version_id`;
- `content_hash`;
- `EntityRevisionObservation`;
- source и temporal metadata.

Одна и та же версия содержимого может наблюдаться повторно:

- при повторном crawl;
- в нескольких `source_scope`;
- после возврата к прежнему состоянию;
- в разных ingestion runs.

Повторное наблюдение не должно создавать новый `entity_version_id`, но должно сохранять отдельную observation и provenance.

Помимо очевидной deduplication, необходимо выявлять более сложные отношения:

- копирование кода;
- разделение кодовой сущности;
- замещение документов;
- связь Jira-задачи с commit и code entity version.

Similarity-based detection не является достаточным основанием для автоматического объединения сущностей или создания lineage relation.

**Проблема:** как дедуплицировать content-equivalent состояния, сохранить provenance и безопасно формировать semantic lineage relations без ложного объединения разных сущностей и без потери истории observations.

---

## Факторы решения

* Сохранение всех `EntityRevisionObservation`.
* Разделение content deduplication, materialization deduplication и semantic lineage.
* Отсутствие автоматического merge и `entity_redirect`.
* Поддержка Git, Jira и Confluence.
* Защита от ложных `COPIED_FROM`, `SPLIT_FROM` и `REPLACES`.
* Source-specific revision и scope context.
* Идемпотентность повторной обработки.
* Совместимость с `FULL_SCOPE`/`PARTIAL_SCOPE` и `completeness_status`.
* Последовательное применение identity-dependent relations.

---

## Решение

Мы разделяем дедупликацию и lineage на три уровня:

```text
1. Exact content deduplication
2. Provenance relations
3. Semantic lineage detection and apply
```

Модуль выполняется после определения версии и до Graph Extraction/Chunking:

```text
Version and Observation Finalization
    ↓
Exact Deduplication
    ↓
Provenance Pass
    ↓
Semantic Lineage Candidate Generation
    ↓
Lineage Apply
    ↓
Graph Extraction / Chunking
```

Модуль не изменяет:

```text
entity_uid
entity_version_id
content_hash
EntityRevisionObservation
```

Merge и `entity_redirect` выполняются Canonical Identity Resolver.

**Область ответственности:** exact deduplication, provenance relations и semantic lineage (`COPIED_FROM`, `SPLIT_FROM`, `REPLACES`)..

### 1. Exact content deduplication

Если для одной `entity_uid` уже существует такой же `content_hash`:

```text
entity_version_id переиспользуется
новая EntityRevisionObservation сохраняется
новая logical content version не создаётся
```

Exact deduplication выполняется по:

```text
entity_uid + content_hash
```

`source_scope` не входит в identity content version. Одна и та же content-equivalent version может наблюдаться в разных scopes.

Materialization deduplication является отдельной операцией и выполняется по representation identity:

```text
entity_uid
+ entity_version_id
+ source_scope
+ chunk_anchor
```

Exact deduplication не удаляет:

- observations;
- provenance;
- temporal history;
- lineage evidence;
- исторические relations.

### 2. Provenance relations

Provenance relations связывают source objects/events с entities или другими source objects. Они не являются semantic lineage relations.

#### `IMPLEMENTED_BY_COMMIT`

Связывает Jira issue с Git commit:

```text
Jira Issue → Git Commit
```

Семантика: commit атрибутирован Jira issue.

#### `TOUCHED_IN`

Связывает Git commit с исходным файлом:

```text
Git Commit → Source File
```

Семантика: commit изменил файл или его область, независимо от того, изменилась ли конкретная code entity.

#### `AFFECTS`

Связывает Git commit с версией code entity:

```text
Git Commit → Code Entity Version
```

Семантика: commit изменил или косвенно затронул конкретную code entity version.

#### `REALIZED_IN`

Связывает Jira issue с реализованной версией code entity:

```text
Jira Issue → Code Entity Version
```

`REALIZED_IN` создаётся только если:

- Jira attribution подтверждена;
- связанный commit изменил конкретную code entity;
- соответствующая entity version входит в canonical scope;
- identity code entity финализирована.

#### Contract provenance relation

```text
ProvenanceEdge:
  relation_id
  relation_type
  source_system_from
  source_system_to
  source_object_id_from
  source_object_id_to
  source_scope_from
  source_scope_to
  source_revision_id_from
  source_revision_id_to
  entity_uid_from
  entity_version_id_from
  entity_uid_to
  entity_version_id_to
  observed_at
  resolution_method
```

Обязательные поля зависят от `relation_type`.

#### Idempotency keys

```text
IMPLEMENTED_BY_COMMIT:
  issue_key + commit_sha

TOUCHED_IN:
  commit_sha + source_file_path

AFFECTS:
  commit_sha + entity_uid_to + entity_version_id_to

REALIZED_IN:
  issue_key + commit_sha
  + entity_uid_to + entity_version_id_to
```

Повторная обработка одинакового provenance evidence не создаёт дубликат.

### 3. Semantic lineage relations

Semantic lineage relations связывают только финализированные `entity_uid`.

```text
LineageRelation:
  relation_id
  relation_type
  entity_uid_from
  entity_uid_to
  source_system_from
  source_system_to
  source_scope_from
  source_scope_to
  valid_from
  valid_until
  valid_until_unknown
  valid_time_source
  edge_status
```

Допустимые типы:

```text
COPIED_FROM
SPLIT_FROM
REPLACES
```

`entity_uid_from` и `entity_uid_to` имеют следующий смысл:

```text
COPIED_FROM:
  производная entity → source entity

SPLIT_FROM:
  новая entity → исходная entity

REPLACES:
  новый документ → заменённый документ
```

`entity_redirect` не является `LineageRelation`.

**Важно:** Semantic domain relations (`DESCRIBES`, `DOCUMENTS`, `SEMANTIC_ASSOCIATION`) не входят в ADR. Они относятся к domain relations.

### 4. Lineage relation lifecycle

```text
edge_status:
  ACTIVE
  RETIRED
  RETRACTED
```

Правила:

```text
ACTIVE → RETIRED:
  relation больше не действует по temporal/business policy,
  но исторически корректна;

ACTIVE → RETRACTED:
  relation признана ошибочной или отменена
  через explicit correction или HITL;

RETIRED:
  не восстанавливается автоматически;

RETRACTED:
  не участвует в retrieval и не становится ACTIVE
  без новой explicit correction или HITL decision.
```

`valid_from` и `valid_until` описывают temporal applicability. `edge_status` не заменяет temporal validity.

Новое evidence само по себе:

- не создаёт новую logical relation;
- не изменяет `edge_status`;
- не закрывает `valid_until`.

### 5. Lineage evidence

Каждое evidence semantic relation хранится отдельно:

```text
LineageEvidence:
  evidence_id
  relation_id
  relation_type
  entity_uid_from
  entity_uid_to
  source_system_from
  source_system_to
  source_object_id_from
  source_object_id_to
  source_scope_from
  source_scope_to
  source_revision_id_from
  source_revision_id_to
  confidence
  observed_at
  resolution_method
```

`LineageEvidence` фиксирует конкретное доказательство relation и не имеет собственного `edge_status`.

Для evidence используется следующий idempotency key:

```text
evidence_key =
  relation_id
  + source_system_from
  + source_system_to
  + source_object_id_from
  + source_object_id_to
  + source_scope_from
  + source_scope_to
  + source_revision_id_from
  + source_revision_id_to
  + resolution_method
```

Повторное evidence с тем же ключом не создаёт дубликат.

### 6. `COPIED_FROM`

Candidate generation выполняется отдельно от создания active relation.

#### Narrative text

Используются:

```text
MinHash + LSH
Jaccard similarity по нормализованным 5-граммам
```

Similarity является candidate evidence и не создаёт active relation самостоятельно.

#### Code

Используются:

- совпадение или совместимость `signature_fingerprint`;
- AST normalization;
- удаление комментариев;
- alpha-renaming локальных переменных;
- normalized tree comparison.

Similarity и AST distance используются для формирования candidate.

Active `COPIED_FROM` создаётся только при наличии:

- explicit source/reference evidence;
- подтверждённого VCS copy evidence;
- либо HITL decision.

Generated, vendor и template-derived code не создают automatic `COPIED_FROM` только по similarity.

Если направление копирования не доказано, active relation не создаётся.

### 7. `SPLIT_FROM`

В текущем scope `SPLIT_FROM` применяется только к Code. Разделение Confluence pages этим ADR не моделируется.

Candidate может быть создан, если:

- одна старая code entity перестала наблюдаться;
- две или более новые entities появились в том же commit range;
- найден content overlap;
- LSP или structural analysis связывает новые symbols со старой entity.

Active `SPLIT_FROM` допускается только если:

```text
coverage = FULL_SCOPE
completeness_status = COMPLETE
```

и одновременно:

- все endpoint `entity_uid` финализированы;
- старая entity имеет подтверждённое отсутствие;
- найдено structural lineage evidence;
- исключены более вероятные rename, move, merge и delete объяснения;
- либо relation подтверждена HITL.

При:

```text
PARTIAL_SCOPE
PARTIAL
FAILED
NOT_OBSERVED
EXTRACTOR_GAP
```

создаётся только candidate.

### 8. `REPLACES`

`REPLACES` создаётся только при explicit replacement evidence.

#### ADR в Git

- YAML front-matter содержит поле `replaces`;
- target старого ADR однозначно разрешён.

#### Confluence

- новая page содержит ссылку на старую page через `target_page_id`;
- либо API предоставляет явный `replaced_by` signal с target.

Следующие события сами по себе не создают `REPLACES`:

- archival;
- deletion/trash;
- move;
- copy;
- отсутствие старой страницы;
- совпадение заголовка;
- высокое текстовое сходство.

Если явные, машиночитаемые утверждения (сигналы) из источника или метаданных конфликтуют:

- active relation не создаётся автоматически;
- сохраняется candidate;
- выполняется HITL или explicit correction (правки руками);
- существующая relation не переводится в `RETRACTED` автоматически.

Повторное обнаружение той же replacement relation создаёт новое evidence, но не новую logical relation.

### 9. Candidate records

Неподтверждённые результаты хранятся в ingestion metadata:

```text
CandidateRecord:
  candidate_id
  detection_type
  entity_uid_from
  entity_uid_to
  source_system_from
  source_system_to
  source_object_id_from
  source_object_id_to
  source_scope_from
  source_scope_to
  source_revision_id_from
  source_revision_id_to
  similarity_score
  threshold
  confidence
  observed_at
  candidate_status
  resolution_method
  relation_id
```

`entity_uid_from` и `entity_uid_to` могут отсутствовать, если identity ещё не разрешена. В этом случае используются source object identifiers.

Допустимые `detection_type`:

```text
COPIED_FROM
SPLIT_FROM
REPLACES
```

Допустимые `candidate_status`:

```text
CANDIDATE
ACCEPTED
DISMISSED
APPLIED
EXPIRED
```

Жизненный цикл:

```text
CANDIDATE → ACCEPTED → APPLIED
CANDIDATE → DISMISSED
CANDIDATE → EXPIRED
```

`ACCEPTED` может быть результатом:

- `RULE`;
- `SOURCE`;
- `HITL`.

После `APPLIED` сохраняется `relation_id`. Повторное применение candidate идемпотентно.

### 10. Sequential Apply Phase

Финальные identity-dependent relations применяются только после Identity Resolution и Version Finalization.

Порядок:

```text
1. Identity decisions
2. entity_alias/entity_redirect
3. Version and Observation Finalization
4. Provenance relations
5. Semantic lineage relations
6. Передача результатов в Graph Materialization
```

Candidate generation может выполняться до Apply Phase, но active relations не создаются для:

```text
PROVISIONAL
AMBIGUOUS
unresolved entity_uid
```

---

## Инварианты

1. Exact content deduplication не удаляет observations или provenance.
2. Exact content deduplication выполняется по `entity_uid + content_hash`, без ограничения `source_scope`.
3. Materialization deduplication выполняется отдельно с учётом `source_scope` и `chunk_anchor`.
4. `entity_version_id` не пересчитывается.
5. Provenance relations не смешиваются с semantic lineage relations.
6. `IMPLEMENTED_BY_COMMIT` связывает Jira issue с Git commit.
7. `REALIZED_IN` связывает Jira issue с Code Entity Version.
8. Semantic lineage relations используют только финализированные `entity_uid`.
9. `LineageEvidence` хранится отдельно от `LineageRelation`.
10. Повторное evidence с тем же `evidence_key` не создаёт дубликат.
11. Новое evidence не изменяет logical relation автоматически.
12. Similarity не создаёт active `COPIED_FROM` без explicit source evidence или HITL.
13. Active `SPLIT_FROM` требует Code, `FULL_SCOPE`, `COMPLETE`, structural evidence и finalized identities.
14. Archival и deletion не являются `REPLACES`.
15. `REPLACES` требует explicit replacement evidence.
16. `edge_status` не заменяет `valid_from` и `valid_until`.
17. Candidate с unresolved identity не становится active relation автоматически.
18. Final Apply Phase выполняется после Identity Resolution и Version Finalization.
19. Provenance relation повторно создаётся идемпотентно по relation-specific key.
20. `source_system_from/to` и source object context обязательны для cross-source relations.
21. Модуль не создаёт и не изменяет `entity_uid`.
22. Модуль не записывает domain nodes непосредственно в Neo4j.

---

## Новые переменные

| Переменная | Тип | Описание | Точка появления |
|---|---|---|---|
| `ProvenanceEdge` | структура | Связь source object/event с entity или другим source object | Provenance Pass |
| `LineageRelation` | структура | Логическая semantic lineage relation между `entity_uid` | Lineage Apply |
| `LineageEvidence` | структура | Отдельное evidence существования lineage relation | Lineage Apply |
| `CandidateRecord` | структура | Неподтверждённый результат dedup/lineage detection | Candidate Generation |
| `relation_id` | `uuid` | Стабильный идентификатор logical relation | Relation Apply |
| `evidence_id` | `uuid` | Идентификатор отдельного lineage evidence | Evidence Store |
| `evidence_key` | `string` | Идемпотентный ключ lineage evidence | Evidence Store |
| `candidate_id` | `uuid` | Идентификатор candidate record | Candidate Generation |
| `candidate_status` | enum | Состояние candidate: `CANDIDATE`, `ACCEPTED`, `DISMISSED`, `APPLIED`, `EXPIRED` | Candidate Store |
| `entity_uid_from` | `uuid` | Identity исходного endpoint semantic relation | Lineage Apply |
| `entity_uid_to` | `uuid` | Identity целевого endpoint semantic relation | Lineage Apply |
| `source_system_from` | enum | Source system исходного endpoint | Relation Builder |
| `source_system_to` | enum | Source system целевого endpoint | Relation Builder |
| `source_object_id_from` | `string` | Source object исходного endpoint | Relation Builder |
| `source_object_id_to` | `string` | Source object целевого endpoint | Relation Builder |
| `source_scope_from` | `string` | Scope исходного endpoint | Relation Builder |
| `source_scope_to` | `string` | Scope целевого endpoint | Relation Builder|
| `source_revision_id_from` | `string` | Revision исходного endpoint | Relation Builder |
| `source_revision_id_to` | `string` | Revision целевого endpoint | Relation Builder |
| `confidence` | `float` | Итоговая уверенность evidence policy | Lineage Apply |
| `similarity_score` | `float` | Сырой результат алгоритма similarity | Candidate Generation |
| `threshold` | `float` | Порог, использованный при candidate generation | Candidate Generation |
| `detection_type` | enum | Тип обнаруженного candidate | Candidate Generation |
| `edge_status` | enum | Lifecycle logical relation: `ACTIVE`, `RETIRED`, `RETRACTED` | Lineage Apply |
| `resolution_method` | enum | Способ принятия решения: `RULE`, `SOURCE`, `HITL` | Relation/Evidence Apply |

---

## Последствия

### Положительные

* Exact deduplication не уничтожает историю observations.
* Materialization deduplication отделена от content deduplication.
* Provenance и semantic lineage используют разные контракты.
* Jira issue может быть связана как с commit, так и с конкретной code entity version.
* Similarity не создаёт active lineage relation автоматически.
* Неполный snapshot не приводит к автоматическому `SPLIT_FROM`.
* Архивирование и удаление не смешиваются с replacement semantics.
* Candidate lifecycle поддерживает rule-based, source-based и HITL decisions.
* Повторная обработка provenance и evidence идемпотентна.
* Модуль не дублирует ответственность Identity Resolver или Graph Apply.
* Semantic domain relations (`DESCRIBES`, `DOCUMENTS`, `SEMANTIC_ASSOCIATION`) отделены от lineage.

### Отрицательные

* Требуется отдельное хранение logical relations, evidence и candidates.
* Semantic lineage detection требует вычислительных ресурсов.
* Низкоуверенные результаты создают candidates и могут потребовать HITL.
* Active `SPLIT_FROM` возможен только для полного и корректного Code scope.
* Merge и `entity_redirect` остаются зависимостью ADR-003.
* Финальная запись в Neo4j выполняется позднее, через Graph Apply.
* Идемпотентность provenance и evidence требует relation-specific keys.
* Для relation temporal validity требуется согласование с Graph schema.

---

## Рассмотренные альтернативы

### Объединить exact deduplication и semantic lineage

Использовать один алгоритм, который одновременно удаляет дубликаты и объединяет похожие сущности.

**Плюсы:**

* меньше этапов;
* проще первоначальная схема;
* потенциально ниже вычислительная стоимость.

**Минусы:**

* similarity ошибочно становится основанием для изменения identity;
* невозможно отличить одинаковое содержимое от копирования;
* возрастает риск false merge;
* теряется разделение content identity и semantic relation.

**Решение:** отклонено.

### Выполнять дедупликацию только внутри `source_scope`

Считать одинаковые сущности в разных scopes независимыми версиями или объектами.

**Плюсы:**

* проще изолировать branch/release данные;
* меньше cross-scope comparisons;
* проще локальный поиск.

**Минусы:**

* одна и та же `entity_uid` и `content_hash` могут получить несколько content versions;
* нарушается формула `entity_version_id`;
* дублируется историческое содержимое;
* усложняется cross-scope lineage.

**Решение:** отклонено. Scope учитывается на уровне materialization и retrieval, но не на уровне content-equivalent entity version.

### Создавать active `COPIED_FROM` по similarity threshold

Автоматически создавать relation при достижении Jaccard или AST similarity threshold.

**Плюсы:**

* высокая степень автоматизации;
* меньше HITL;
* быстрое обнаружение похожего кода и текста.

**Минусы:**

* типовой код, boilerplate и generated code дают false positives;
* similarity не определяет направление копирования;
* невозможно отличить копирование от независимой реализации;
* ошибка загрязняет graph lineage.

**Решение:** отклонено.

### Использовать один общий `ProvenanceEdge` для всех relations

Хранить provenance и semantic lineage в одной структуре.

**Плюсы:**

* меньше типов records;
* проще единый writer;
* единый интерфейс хранения.

**Минусы:**

* смешиваются source objects и logical entities;
* разные endpoint contracts становятся неоднозначными;
* усложняются idempotency и temporal filtering;
* `entity_uid` ошибочно начинает использоваться для commits и files.

**Решение:** отклонено.

### Создавать `SPLIT_FROM` при любом исчезновении entity и появлении двух новых

Считать такое изменение достаточным признаком split.

**Плюсы:**

* простое правило;
* быстрое обнаружение refactoring;
* не требуется сложный structural analysis.

**Минусы:**

* delete и два независимых новых symbols выглядят как split;
* неполный snapshot создаёт ложный split;
* rename/move могут быть ошибочно классифицированы;
* relations становятся недостоверными.

**Решение:** отклонено.

### Записывать lineage relations непосредственно в Graph Extraction

Graph Extraction сразу создаёт semantic lineage relations в Neo4j.

**Плюсы:**

* меньше промежуточных записей;
* relation доступна сразу после extraction;
* проще короткий happy path.

**Минусы:**

* смешиваются extraction и cross-entity comparison;
* сложнее применять identity finalization;
* возрастает риск записи relation на неактуальные identity;
* ухудшается идемпотентность и reconciliation.

**Решение:** отклонено. Deduplication формирует и применяет lineage decisions, а Graph Apply материализует их в Neo4j по отдельному контракту.

### Включить `DESCRIBES`, `DOCUMENTS`, `SEMANTIC_ASSOCIATION`

Рассматривать semantic domain relations как часть lineage detection и apply.

**Плюсы:**

* единый механизм для всех cross-entity relations;
* проще централизованное применение;
* меньше этапов.

**Минусы:**

* смешиваются разные типы relations с разной семантикой;
* `DESCRIBES` (документация → код) не является lineage;
* усложняется lifecycle управления;
* нарушается разделение ответственности.

**Решение:** отклонено. Semantic domain relations (`DESCRIBES`, `DOCUMENTS`, `SEMANTIC_ASSOCIATION`) формируются на этапе работы с графом.
