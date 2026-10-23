# ADR Reconciliation и проверка межхранилищной согласованности

**Статус:** Предложен

---

## Контекст и проблема

После Graph Extraction (обе фазы) и Qdrant Materialization AI Crawler материализует одну входную обработку в двух независимых проекциях:

- Neo4j содержит доменные entities, relations, provenance и lineage, включая **семантические доменные relations** (`DESCRIBES`, `DOCUMENTS`, `SEMANTIC_ASSOCIATION`);
- Qdrant содержит Chunks, dense/sparse vectors и retrieval metadata.

Qdrant и Neo4j не имеют общей транзакции. Поэтому возможны состояния, при которых:

- запись в Qdrant выполнена, а запись в Neo4j нет;
- запись в Neo4j выполнена, а embedding или Qdrant materialization нет;
- записан неполный набор chunks;
- остались stale Qdrant points;
- materialization выполнена для другой source revision;
- одна из ветвей завершилась с ошибкой после успешной записи другой;
- семантические relations применены в Neo4j, но соответствующие Qdrant points ещё не записаны, или наоборот.

Согласно Source Snapshot, `cursor_after` продвигается и временный snapshot удаляется только после успешного reconciliation. При ошибке downstream snapshot не используется для replay: восстановление выполняется новым чтением источника с новым `snapshot_id`.

**Проблема:** как определить, что обязательные materialization в Qdrant и Neo4j согласованы для конкретного `snapshot_id`, entity и `source_scope`, включая семантические доменные relations, и когда разрешены cursor commit, snapshot cleanup и переход к `CANONICAL_PUBLISHED`.

---

## Факторы решения

* Отсутствие общей транзакции между Qdrant и Neo4j.
* Идемпотентность повторной materialization.
* Согласованность по `snapshot_id`, `entity_uid`, `entity_version_id` и `source_scope`.
* Отсутствие ложного `CANONICAL_PUBLISHED`.
* Сохранение historical representations.
* Корректная очистка stale Qdrant points.
* Отделение технического сбоя от quality rejection.
* Отсутствие replay временного snapshot согласно Source Snapshot.
* Возможность определить, когда можно продвинуть cursor.
* Поддержка разных `required_materialization_targets`.
* Семантические relations могут быть применены после Qdrant и Neo4j materialization.
* Semantic candidate без active relation не считается ошибкой reconciliation.

---

## Решение

Мы выполняем reconciliation после завершения обязательных materialization branches:

```text
Graph Materialization (включая семантические relations)
+
Qdrant Materialization
→ Reconciliation
→ IngestionRunResult
→ Cursor Commit
→ Snapshot Cleanup
```

Reconciliation выполняется для конкретного:

```text
snapshot_id
+ ingestion_run_id
+ source_scope
```

При необходимости результат проверяется на уровне отдельной:

```text
entity_uid
+ entity_version_id
```

Reconciliation не изменяет:

```text
entity_uid
entity_version_id
content_hash
source_revision_id
```

Он проверяет результаты materialization и определяет, может ли текущий ingestion run считаться завершённым.

### 1. Входные данные

Reconciliation получает:

```text
snapshot_id
ingestion_run_id
source_system
source_scope
source_revision_id
entity_uid
entity_version_id
content_hash
required_materialization_targets
qdrant_materialization_result
neo4j_materialization_result
expected_set
semantic_candidates
semantic_domain_relations
```

Для Qdrant проверяются:

```text
expected_chunk_ids
processed_chunk_ids
failed_chunk_ids
skipped_chunk_ids
stale_chunk_ids
```

Для Neo4j проверяются:

```text
expected_entity_uids
expected_entity_version_ids
expected_relation_ids
expected_semantic_relation_ids
graph_materialization_result
```

Если конкретный target не входит в `required_materialization_targets`, его отсутствие не считается ошибкой reconciliation.

### 2. Границы проверки

Reconciliation проверяет:

1. наличие всех обязательных materializations;
2. соответствие materialization исходным identity;
3. соответствие `source_scope`;
4. соответствие `source_revision_id`;
5. соответствие `entity_version_id` и `content_hash`;
6. наличие обязательных Qdrant chunks;
7. наличие обязательных Neo4j entities и relations;
8. наличие обязательных семантических доменных relations, если они требуются Graph Policy;
9. отсутствие stale points в current projection;
10. отсутствие materialization с более старой source revision;
11. завершённость cleanup для текущего набора.

Reconciliation не проверяет:

- качество исходного контента;
- корректность `entity_uid`;
- корректность `content_hash`;
- semantic correctness lineage;
- полноту domain model сверх требований соответствующего Graph Policy;
- пользовательские ACL на уровне интерфейса.

### 3. Материализация Qdrant

Qdrant materialization считается согласованной, если:

- все обязательные Chunks из `expected_set` имеют соответствующие points;
- `Chunk.chunk_id` совпадает с point ID;
- point содержит тот же:
  - `entity_uid`;
  - `entity_version_id`;
  - `content_hash`;
  - `source_scope`;
  - `source_revision_id`;
- vectors созданы успешно;
- отсутствуют незавершённые обязательные cleanup operations;
- stale points current projection удалены или деактивированы согласно Qdrant policy.

Исторические points не считаются ошибкой только потому, что они отсутствуют в current expected set. Их сохранение регулируется temporal policy Version Finalization и Qdrant policy Qdrant Materialization.

### 4. Материализация Neo4j

Neo4j materialization считается согласованной, если:

- все обязательные entities записаны;
- entities имеют корректные `entity_uid` и `entity_version_id`;
- обязательные domain relations записаны;
- обязательные семантические доменные relations (`DESCRIBES`, `DOCUMENTS`, `SEMANTIC_ASSOCIATION`) записаны, если они требуются Graph Policy;
- provenance и lineage relations применены согласно Deduplication и Semantic Lineage;
- relations не ссылаются на отсутствующие обязательные endpoints;
- source scope и source revision соответствуют текущему snapshot;
- повторная запись не создала дубликаты.

Semantic candidate без active relation не считается ошибкой reconciliation. Если candidate generation завершился, но relation не была подтверждена (не прошла evidence/policy decision), это не блокирует reconciliation.

### 5. Согласование Qdrant и Neo4j

Cross-store consistency считается достигнутой, если для каждой materialized entity выполняется:

```text
Qdrant:
  entity_uid
  entity_version_id
  source_scope
  source_revision_id

совместимы с:

Neo4j:
  entity_uid
  entity_version_id
  source_scope
  source_revision_id
```

При этом:

- Qdrant point не должен ссылаться на неизвестный `entity_uid`;
- Neo4j entity не должна иметь обязательную Qdrant representation, если Qdrant входит в required target;
- отсутствие Graph representation допустимо, если Neo4j не входит в required target;
- отсутствие Qdrant representation допустимо, если Qdrant не входит в required target;
- `source_revision_id` сравнивается по source-specific правилам, а не как произвольная строка;
- семантические доменные relations проверяются на наличие в Neo4j, если они были подтверждены и должны быть материализованы;
- semantic candidates без active relation игнорируются при проверке согласованности.

### 6. Статусы reconciliation

`reconciliation_status` принимает значения:

```text
COMPLETE
PARTIAL
FAILED
```

#### `COMPLETE`

Все обязательные targets успешно материализованы, expected sets проверены, stale cleanup завершён, cross-store contradictions не обнаружены, все обязательные семантические relations записаны в Neo4j (если они требуются Graph Policy).

#### `PARTIAL`

Часть materialization или cleanup выполнена, но:

- отсутствует обязательный point или entity;
- одна branch завершилась, другая нет;
- часть stale points не обработана;
- обнаружено расхождение, требующее восстановления;
- некоторые обязательные семантические relations отсутствуют в Neo4j;
- некоторые semantic candidates требуют дополнительного анализа.

`PARTIAL` не разрешает cursor commit и не позволяет установить `CANONICAL_PUBLISHED`.

#### `FAILED`

Проверка или восстановление невозможно из-за:

- недоступности Qdrant или Neo4j;
- повреждённого materialization result;
- невозможности сопоставить source revision;
- нарушения обязательного identity contract;
- ошибки reconciliation.

`FAILED` не является Quality Gate rejection.

### 7. Divergence

При расхождении формируется `divergence`:

```text
divergence:
  source_scope
  entity_uid
  entity_version_id
  qdrant_status
  neo4j_status
  missing_qdrant_chunks
  extra_qdrant_chunks
  missing_neo4j_entities
  missing_neo4j_relations
  missing_semantic_relations
  revision_mismatch
  content_hash_mismatch
  semantic_candidate_status
  reason
```

`divergence` используется для диагностики и формирования recovery decision. Он не изменяет identity или temporal fields.

### 8. Repair и recovery

Если расхождение может быть исправлено идемпотентной операцией, Reconciliation формирует:

```text
repair_required = true
```

и передаёт задачу в materialization/recovery policy.

Если восстановление требует повторного получения источника:

1. текущий snapshot не используется для replay;
2. cursor не продвигается;
3. выполняется новый crawl;
4. создаётся новый `snapshot_id`;
5. новый snapshot обрабатывается как отдельный ingestion attempt.

Если snapshot ещё физически доступен до завершения текущей операции, он может использоваться для диагностики. Это не превращает его в долговременный replay source.

### 9. Решение о завершении ingestion

Reconciliation возвращает `COMPLETE`, если:

```text
all required materialization targets = successful
+ expected sets verified
+ no blocking divergence
+ stale cleanup completed
+ all required semantic relations applied (if required by Graph Policy)
```

Только после `reconciliation_status=COMPLETE` Completion может:

- создать успешный `IngestionRunResult`;
- установить `canonical_status=CANONICAL_PUBLISHED`, если выполнены остальные условия;
- зафиксировать `cursor_after`;
- удалить временный snapshot.

Для:

```text
required_materialization_targets = NONE
```

reconciliation не создаёт успешную публикацию в Qdrant/Neo4j. Итоговый `canonical_status` определяется согласно Quality Gate и Completion.

### 10. Связь с cursor

| Состояние reconciliation | Cursor |
|---|---|
| `COMPLETE` | Может быть продвинут после Cursor Commit |
| `PARTIAL` | Не продвигается |
| `FAILED` | Не продвигается |
| Blocking divergence | Не продвигается |
| Quality rejection без technical failure | Обрабатывается по policy Quality Gate/Completion |
| Semantic candidate без active relation | Не блокирует cursor |

Reconciliation не продвигает cursor самостоятельно. Он только возвращает решение о готовности к Cursor Commit.

### 11. Связь с `canonical_status`

Reconciliation не устанавливает `canonical_status`.

Он предоставляет результат, на основании которого Completion определяет:

```text
reconciliation_status = COMPLETE
+ Quality Gate passed
+ identity resolved
+ required targets materialized
→ CANONICAL_PUBLISHED
```

Если reconciliation не завершён:

```text
reconciliation_status = PARTIAL | FAILED
→ CANONICAL_PUBLISHED запрещён
```

Отсутствие semantic candidate или low-confidence candidate не блокирует `CANONICAL_PUBLISHED`, если semantic relations не являются обязательными по Graph Policy.

### 12. Повторная проверка и идемпотентность

Повторная проверка одного materialization result:

- не создаёт новые entities;
- не создаёт новые Qdrant points;
- не изменяет `entity_uid`;
- не изменяет `entity_version_id`;
- не создаёт новый source observation;
- повторно проверяет expected sets и divergence;
- **повторно проверяет наличие семантических relations, но не создаёт их заново.**

Одинаковый reconciliation result для того же:

```text
snapshot_id
+ ingestion_run_id
+ source_scope
```

не создаёт дубликат результата.

## Инварианты

1. Reconciliation выполняется после materialization обязательных targets.
2. Reconciliation не изменяет identity и temporal fields.
3. `COMPLETE` возможен только после проверки всех required targets.
4. `PARTIAL` и `FAILED` не позволяют продвинуть cursor.
5. `PARTIAL` и `FAILED` не позволяют установить `CANONICAL_PUBLISHED`.
6. Qdrant и Neo4j сравниваются в рамках одного `snapshot_id` и совместимого source scope.
7. Исторические Qdrant representations не удаляются только из-за появления новой версии.
8. Stale cleanup не затрагивает другой `source_scope`.
9. Отсутствие необязательного target не является ошибкой.
10. Наличие lineage candidate без active relation не является ошибкой, если candidate сохранён согласно Deduplication и Semantic Lineage.
11. `source_revision_id` сравнивается по source-specific правилам.
12. Reconciliation не считается Quality Gate rejection.
13. Reconciliation не создаёт новые `entity_uid` или `entity_version_id`.
14. Reconciliation не создаёт новый `entity_revision_observation`.
15. Snapshot не используется для долговременного replay.
16. При recovery через новый crawl создаётся новый `snapshot_id`.
17. Cursor Commit и snapshot cleanup выполняются только после `reconciliation_status=COMPLETE`.
18. `canonical_status` устанавливается только в Completion.
19. Семантические domain relations проверяются на наличие в Neo4j, если они требуются Graph Policy.
20. Semantic candidate без active relation не считается ошибкой reconciliation и не блокирует cursor.
21. Отсутствие semantic relation не блокирует `CANONICAL_PUBLISHED`, если semantic relations не являются обязательными по Graph Policy.

## Новые переменные

| Переменная | Тип | Описание | Точка появления |
|---|---|---|---|
| `reconciliation_status` | enum | Результат проверки: `COMPLETE`, `PARTIAL`, `FAILED` | Reconciliation Worker |
| `divergence` | структура | Описание расхождения между materializations | Reconciliation Worker |
| `repair_required` | boolean | Требуется ли восстановительная операция | Reconciliation Worker |
| `expected_entity_uids` | collection | Ожидаемые Neo4j entities для текущей materialization | Graph Materialization/Reconciliation |
| `expected_entity_version_ids` | collection | Ожидаемые версии Neo4j entities | Graph Materialization/Reconciliation |
| `expected_relation_ids` | collection | Ожидаемые обязательные graph relations | Graph Materialization/Reconciliation |
| `expected_semantic_relation_ids` | collection | Ожидаемые обязательные семантические domain relations | Graph Materialization/Reconciliation |
| `graph_materialization_result` | структура | Результат записи Neo4j entities и relations | Graph Apply |
| `qdrant_materialization_result` | структура | Результат записи Qdrant points | Qdrant Writer |
| `missing_qdrant_chunks` | collection | Ожидаемые, но отсутствующие Qdrant chunks | Reconciliation Worker |
| `extra_qdrant_chunks` | collection | Лишние points в текущем scope | Reconciliation Worker |
| `missing_neo4j_entities` | collection | Ожидаемые, но отсутствующие Neo4j entities | Reconciliation Worker |
| `missing_neo4j_relations` | collection | Ожидаемые, но отсутствующие Neo4j relations | Reconciliation Worker |
| `missing_semantic_relations` | collection | Ожидаемые, но отсутствующие семантические domain relations | Reconciliation Worker |
| `semantic_candidate_status` | string | Статус semantic candidates для диагностики | Reconciliation Worker |
| `revision_mismatch` | boolean | Несовпадение source revisions между branches | Reconciliation Worker |
| `content_hash_mismatch` | boolean | Несовпадение `content_hash` между проекциями | Reconciliation Worker |

---

## Последствия

### Положительные

* Рассогласование Qdrant и Neo4j обнаруживается явно.
* `CANONICAL_PUBLISHED` не устанавливается при partial materialization.
* Cursor не продвигается до завершения проверки.
* Исторические representations защищены от ошибочного cleanup.
* Необязательные materialization targets не блокируют ingestion.
* Повторная materialization остаётся идемпотентной.
* Recovery через новый crawl согласуется с временным жизненным циклом snapshot.
* Reconciliation не смешивается с Quality Gate, Identity Resolution или retention.
* Семантические relations проверяются на наличие в Neo4j, если они требуются Graph Policy.
* Semantic candidate без active relation не блокирует cursor и не считается ошибкой.
* Отсутствие semantic relation не блокирует `CANONICAL_PUBLISHED`.

### Отрицательные

* Требуется отдельный Reconciliation Worker.
* Нужно формировать и проверять expected sets для Qdrant и Neo4j.
* Разные источники требуют разных правил сравнения `source_revision_id`.
* Partial materialization может задерживать cursor и удерживать временный snapshot.
* Полная проверка current и historical projections увеличивает стоимость обработки.
* Корректность результата зависит от согласованности materialization contracts Graph Extraction и Qdrant Materialization.
* Проверка семантических relations добавляет сложность в reconciliation.

---

## Рассмотренные альтернативы

### Считать ingestion завершённым после успешной записи только в Qdrant

Считать Graph materialization необязательной и продвигать cursor после успешной записи в Qdrant.

**Плюсы:**

* простой pipeline;
* меньше задержка;
* ниже риск блокировки из-за Neo4j;
* Qdrant быстрее становится доступен для поиска.

**Минусы:**

* Neo4j может навсегда остаться неполным;
* retrieval будет использовать рассогласованные проекции;
* невозможно корректно выполнять graph enrichment;
* cursor может быть продвинут до восстановления Graph branch.

**Решение:** отклонено, если Neo4j входит в required materialization targets.

### Считать ingestion завершённым после успешной записи только в Neo4j

Считать Qdrant materialization вторичной и продвигать cursor после Graph Apply.

**Плюсы:**

* сохраняется структурная проекция;
* можно выполнять graph-driven retrieval;
* проще контролировать entity identity и relations.

**Минусы:**

* Qdrant может остаться неполным;
* обычный semantic retrieval будет возвращать устаревшие или неполные данные;
* embedding processing может потеряться после продвижения cursor;
* нарушается симметрия required targets.

**Решение:** отклонено, если Qdrant входит в required materialization targets.

### Использовать общую распределённую транзакцию между Qdrant и Neo4j

Пытаться сделать запись в оба хранилища атомарной через distributed transaction.

**Плюсы:**

* формально строгая консистентность;
* не требуется отдельная модель partial materialization;
* проще определить commit/rollback.

**Минусы:**

* Qdrant и Neo4j не предоставляют общей транзакции в используемой архитектуре;
* растут latency и операционная сложность;
* повышается связанность хранилищ;
* не устраняется необходимость recovery при внешних сбоях.

**Решение:** отклонено. Используется eventual consistency и reconciliation.

### Удалять старые Qdrant points без проверки Neo4j

При новой materialization сразу удалять все points, отсутствующие в новом expected set.

**Плюсы:**

* простой current retrieval;
* меньше stale data;
* не требуется отдельный historical cleanup policy.

**Минусы:**

* можно удалить historical representations;
* ошибка expected set приводит к потере данных;
* невозможно безопасно восстановиться после partial write;
* cleanup может затронуть другой source scope.

**Решение:** отклонено. Cleanup выполняется только в рамках проверенного scope и с учётом historical policy.

### Продвигать cursor сразу после создания snapshot

Считать получение source достаточным для продвижения cursor, а downstream обрабатывать независимо.

**Плюсы:**

* внешние источники читаются реже;
* downstream можно восстанавливать отдельно;
* ниже задержка connector'а.

**Минусы:**

* неполный downstream ingestion может привести к потере изменений;
* повторное чтение источника может не вернуть прежнее состояние;
* Qdrant и Neo4j могут навсегда разойтись;
* нарушается текущая политика Source Snapshot.

**Решение:** отклонено. Cursor продвигается только после завершения обязательной materialization и reconciliation.

### Считать отсутствие semantic relation ошибкой reconciliation

Блокировать `CANONICAL_PUBLISHED` при отсутствии любой semantic relation, даже если она не обязательна по Graph Policy.

**Плюсы:**

* максимальная полнота графа;
* гарантируется наличие всех возможных semantic relations.

**Минусы:**

* отсутствие semantic relation может блокировать ingestion;
* low-confidence candidates требуют HITL, что задерживает cursor;
* semantic similarity не всегда точна;
* увеличивается latency ingestion.

**Решение:** отклонено. Semantic candidate без active relation не блокирует cursor.