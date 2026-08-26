# ADR Завершение ingestion и фиксация cursor

**Статус:** Предложен

---

## Контекст и проблема

После обработки Source Snapshot результаты Qdrant и Neo4j проходят Reconciliation. До этого момента ingestion считается незавершённым: проекции могут быть записаны частично, а `cursor_after` ещё не может заменить текущую позицию cursor.

`cursor_after` показывает границу, до которой connector получил данные. Но продвижение cursor до этой границы раньше завершения обязательной materialization может привести к потере изменений при следующем incremental crawl.

Snapshot является временным и удаляется после завершения обработки. Replay snapshot не используется: при техническом сбое выполняется новый crawl с новым `snapshot_id`.

`canonical_status` имеет per-run семантику и хранится в `IngestionRunResult`, а не в `ProcessingUnit` или `EntityRevisionObservation`.

**Проблема:** как определить окончание ingestion-прогона, безопасно зафиксировать `cursor_after`, удалить временный snapshot и установить `canonical_status`, не теряя необработанные изменения и не выдавая частичный результат за полный.

---

## Факторы решения

* Нельзя продвигать cursor до завершения обязательной materialization.
* Нельзя удалять snapshot до фиксации результата текущей обработки.
* Qdrant и Neo4j не имеют общей транзакции.
* `PARTIAL` и `FAILED` Reconciliation не должны приводить к `CANONICAL_PUBLISHED`.
* `QUALITY_REJECTED` является терминальным результатом качества, а не техническим сбоем.
* `QUARANTINED`, unresolved identity и technical failure требуют отдельной обработки.
* Повторный crawl создаёт новый `snapshot_id`.
* Финальный статус должен быть идемпотентным.
* `cursor_after` должен обновляться без гонки с параллельным crawl.
* Snapshot cleanup не должен удалять manifest и ingestion metadata.
* Отсутствие semantic candidate или незавершённый candidate не блокируют ingestion и cursor.
* Незавершённая обязательная semantic relation блокирует cursor только если это установлено Graph Policy.

---

## Решение

Мы разделяем завершение ingestion на четыре независимых действия:

```text
1. Finalize processing result
2. Commit cursor
3. Cleanup temporary snapshot
4. Publish final ingestion state
```

Для успешного ingestion используется порядок:

```text
Materialization
→ Reconciliation
→ IngestionRunResult
→ Cursor Commit
→ Snapshot Cleanup
```

`canonical_status` фиксируется до Cursor Commit, а сам ingestion run считается завершённым только после успешной фиксации cursor и cleanup.

### 1. Входные данные Completion

Completion получает:

```text
snapshot_id
ingestion_run_id
source_system
source_scope
cursor_before
cursor_after
coverage
completeness_status
required_materialization_targets
quality result
identity resolution result
qdrant_materialization_result
graph_materialization_result
reconciliation_result
semantic_candidates
semantic_domain_relations
```

`cursor_after` является границей, подготовленной Connector'ом. Он не считается зафиксированным cursor store до успешного Cursor Commit.

### 2. `IngestionRunResult`

Для каждого обрабатываемого source scope создаётся или идемпотентно обновляется:

```text
IngestionRunResult:
  result_id
  ingestion_run_id
  snapshot_id
  source_system
  source_scope
  cursor_before
  cursor_after
  canonical_status
  materialization_targets
  materialization_status
  reconciliation_status
  semantic_candidate_status
  created_at
  updated_at
  published_at
```

`IngestionRunResult` фиксирует результат конкретного ingestion run и не изменяет:

```text
entity_uid
entity_version_id
content_hash
source_revision_id
```

### 3. Определение `canonical_status`

#### `CANONICAL_PUBLISHED`

Устанавливается, если одновременно выполнены:

```text
quality gate пройден
+ identity разрешена для всех materialized entities
+ все required_materialization_targets успешно обработаны
+ reconciliation_status = COMPLETE
+ все обязательные semantic relations (если требуются Graph Policy) применены
```

#### `QUALITY_REJECTED`

Устанавливается, если:

- есть обязательный `REJECTED` unit;
- причина является терминальным quality result;
- отсутствует технический failure, препятствующий фиксации результата.

`QUALITY_REJECTED` является финальным результатом текущего run. Он не означает, что source object был удалён.

#### `EXTRACTED`

Устанавливается, если:

- есть обязательный `QUARANTINED` unit;
- identity не разрешена;
- materialization частичная;
- Reconciliation не завершён;
- `required_materialization_targets = NONE`;
- текущий run не может быть опубликован как canonical.

`EXTRACTED` не является успешным завершением ingestion и не разрешает Cursor Commit, если причина связана с незавершённой обработкой, unresolved identity, quarantine или partial materialization.

### 4. Влияние semantic candidates на завершение

Отсутствие semantic candidate или low-confidence candidate не блокирует ingestion и не блокирует cursor.

Правила:

```text
1. Semantic candidate без active relation:
   → НЕ блокирует ingestion
   → НЕ блокирует cursor
   → candidate сохраняется в ingestion metadata

2. Low-confidence semantic candidate:
   → НЕ блокирует ingestion
   → НЕ блокирует cursor
   → candidate сохраняется для последующего анализа

3. Незавершённая обязательная semantic relation:
   → блокирует cursor ТОЛЬКО если это установлено Graph Policy
   → если semantic relation не обязательна по Graph Policy → НЕ блокирует cursor

4. Semantic candidate generation failure:
   → НЕ блокирует ingestion
   → НЕ блокирует cursor
   → ошибка фиксируется в ingestion metadata
```

### 5. Правила Cursor Commit

Cursor Commit разрешён только после успешного завершения обработки заявленного source scope.

#### Cursor может быть продвинут

```text
canonical_status = CANONICAL_PUBLISHED
reconciliation_status = COMPLETE
```

либо:

```text
canonical_status = QUALITY_REJECTED
```

если quality result терминальный, snapshot полностью обработан, а materialization не требуется для rejected entity.

В случае `QUALITY_REJECTED` cursor фиксируется только после сохранения quality result в ingestion metadata.

#### Cursor не продвигается

```text
technical failure
partial materialization
reconciliation_status = PARTIAL
reconciliation_status = FAILED
unresolved identity
обязательный QUARANTINED
неполный snapshot
completeness_status = PARTIAL
completeness_status = FAILED
обязательная semantic relation не применена (только если Graph Policy требует этого)
```

При этих состояниях выполняется новый crawl согласно Identity Resolution и Resilience. Новый crawl получает новый `snapshot_id`.

### 6. Атомарность Cursor Commit

Cursor Commit выполняется с проверкой значения `cursor_before`.

```text
Если текущее значение cursor
равно cursor_before:
  → записать cursor_after
  → cursor_status = VALID

Если текущее значение cursor
отличается от cursor_before:
  → Cursor Commit отклонить
  → создать recovery/reconciliation case
  → не выполнять blind overwrite
```

Это предотвращает ситуацию, когда более старый ingestion run перезаписывает cursor, уже продвинутый другим run.

Для Git дополнительно проверяется совместимость `cursor_after` с актуальной ref history. Для Jira и Confluence проверяется source-specific boundary.

### 7. Cursor status

После успешного Cursor Commit:

```text
cursor_status = VALID
position = cursor_after
saved_at = время фиксации cursor
```

Если source revision больше не существует или Git history была переписана:

```text
cursor_status = INVALID
```

и следующий crawl выполняется как `FULL_SCOPE`.

Если изменились connector rules:

```text
cursor_status = RESET_REQUIRED
```

и выполняется полный re-crawl согласно Source Snapshot.

`cursor_status` не является частью `canonical_status`.

### 8. Snapshot Cleanup

Snapshot и его raw payload удаляются только после:

```text
IngestionRunResult зафиксирован
+ quality/materialization result сохранён
+ Reconciliation завершён либо принято terminal quality decision
+ Cursor Commit успешно выполнен
```

После cleanup сохраняются:

- manifest согласно Source Snapshot;
- `IngestionRunResult`;
- quality metadata;
- observations;
- materialization results;
- reconciliation results;
- provenance и checksums;
- semantic candidates (если они были созданы).

Snapshot не удаляется, если:

- Cursor Commit не выполнен;
- snapshot нужен для текущей незавершённой операции;
- результат обрабатывается в рамках активного recovery;
- сохранение snapshot временно требуется для диагностики.

Поскольку replay не используется, после принятия решения о новом crawl старый snapshot может быть удалён после сохранения failure/recovery metadata, даже если cursor остаётся неизменным.

### 9. Обработка terminal quality rejection

Для terminal `QUALITY_REJECTED`:

```text
quality result сохраняется
canonical_status = QUALITY_REJECTED
cursor_after фиксируется
cursor commit выполняется
snapshot и raw payload удаляются
```

Повторное чтение источника не выполняется автоматически только из-за `QUALITY_REJECTED`. Новый crawl потребуется при:

- изменении Quality Gate rules;
- исправлении source content;
- изменении connector;
- ручном решении о повторной обработке.

### 10. Обработка quarantine и unresolved identity

Для обязательного `QUARANTINED` или unresolved identity:

```text
canonical_status = EXTRACTED
cursor не продвигается
snapshot не используется для replay
после HITL/resolution выполняется новый crawl
```

Решение HITL или identity decision применяется только к соответствующему:

```text
source_system
source_scope
source_object_id
source_revision_id
unit_source_anchor
```

Если новый crawl получил другую ревизию или другой unit, решение переоценивается.

### 11. Обработка partial materialization

Если одна или несколько обязательных materialization branches завершились частично:

```text
canonical_status = EXTRACTED
reconciliation_status = PARTIAL
cursor не продвигается
snapshot не используется для replay
выполняется новый crawl
```

Если Reconciliation обнаружил расхождение после записи только одной проекции, это не является `QUALITY_REJECTED`. Это техническая ошибка materialization и обрабатывается Reconciliation и Resilience.

### 12. Обработка semantic candidates при завершении

```text
Если semantic candidate создан, но relation не подтверждена:
  → candidate сохраняется в ingestion metadata
  → ingestion завершается без блокировки
  → cursor продвигается (если остальные условия выполнены)

Если semantic candidate generation завершился ошибкой:
  → ошибка фиксируется в ingestion metadata
  → ingestion завершается без блокировки
  → cursor продвигается (если остальные условия выполнены)

Если semantic relation обязательна по Graph Policy и не применена:
  → cursor НЕ продвигается
  → canonical_status = EXTRACTED
  → выполняется новый crawl после исправления
```

### 13. Идемпотентность завершения

Повторная обработка одного:

```text
ingestion_run_id
+ snapshot_id
+ source_scope
```

не должна:

- создавать новый `IngestionRunResult`;
- повторно продвигать cursor;
- изменять уже зафиксированный `cursor_after`;
- удалять manifest;
- создавать конфликтующий final status.

Повторный Cursor Commit с тем же `cursor_before` и `cursor_after` считается успешным, если cursor store уже содержит `cursor_after`.

Повторный Snapshot Cleanup считается успешным, если временный snapshot уже удалён, а manifest и metadata сохранены.

### 14. Границы ответственности

Completion отвечает за:

- создание `IngestionRunResult`;
- определение финального `canonical_status`;
- Cursor Commit;
- Snapshot Cleanup;
- проверку порядка завершения;
- передачу final state в monitoring и downstream retrieval;
- учёт semantic candidates при завершении (отсутствие candidate не блокирует cursor).

Completion не отвечает за:

- вычисление `content_hash`;
- Identity Resolution;
- Quality Gate;
- Chunking;
- Embedding;
- Graph Extraction;
- Qdrant/Neo4j materialization;
- reconciliation logic;
- retry classification;
- формирование semantic relations (это Graph Extraction).

---

## Инварианты

1. Cursor Commit выполняется только после завершения обязательной обработки.
2. `cursor_after` не считается зафиксированным до успешного Cursor Commit.
3. `cursor_before` используется для защиты от конкурентного overwrite.
4. `QUALITY_REJECTED` не является техническим failure.
5. `QUARANTINED`, unresolved identity и partial materialization не продвигают cursor.
6. `CANONICAL_PUBLISHED` невозможен без успешного Reconciliation обязательных targets.
7. `required_materialization_targets = NONE` не приводит к `CANONICAL_PUBLISHED`.
8. Snapshot удаляется только после фиксации результата и разрешённого Cursor Commit.
9. Manifest не удаляется вместе со snapshot.
10. Replay snapshot не используется.
11. Новый crawl после failure получает новый `snapshot_id`.
12. `canonical_status` хранится в `IngestionRunResult`, а не в `ProcessingUnit`.
13. `canonical_status` имеет per-run семантику.
14. Completion не изменяет `entity_uid`, `entity_version_id`, `content_hash` или `source_revision_id`.
15. Повторный Cursor Commit с теми же границами идемпотентен.
16. Snapshot Cleanup идемпотентен.
17. Технический сбой materialization не маскируется под `QUALITY_REJECTED`.
18. Ошибка конкурентного Cursor Commit не приводит к blind overwrite.
19. Manifest и ingestion metadata сохраняются после удаления raw snapshot.
20. `cursor_status` не заменяет `canonical_status`.
21. **Отсутствие semantic candidate не блокирует ingestion и cursor.**
22. **Low-confidence semantic candidate не блокирует cursor.**
23. **Незавершённая обязательная semantic relation блокирует cursor только если это установлено Graph Policy.**
24. **Semantic candidate generation failure не блокирует ingestion и cursor.**

---

## Новые переменные

| Переменная | Тип | Описание | Точка появления |
|---|---|---|---|
| `IngestionRunResult` | структура | Финальный результат обработки source scope в конкретном ingestion run | Ingestion Orchestrator, Completion |
| `result_id` | `uuid` | Идентификатор `IngestionRunResult` | Completion |
| `materialization_targets` | enum/collection | Targets, обязательные для публикации текущего результата | Entity Policy/Completion |
| `materialization_status` | enum | Итоговое состояние materialization targets | Qdrant Materialization/Reconciliation |
| `reconciliation_status` | enum | Итог Reconciliation: `COMPLETE`, `PARTIAL`, `FAILED` | Reconciliation |
| `semantic_candidate_status` | enum | Статус semantic candidates: `PENDING`, `PROCESSED`, `SKIPPED`, `BLOCKING` | Completion |
| `cursor_commit_status` | enum | Результат фиксации cursor: `COMMITTED`, `REJECTED`, `SKIPPED` | Completion |
| `snapshot_cleanup_status` | enum | Результат удаления временного snapshot: `CLEANED`, `DEFERRED`, `FAILED` | Completion |
| `updated_at` | timestamp | Время изменения `IngestionRunResult` | Completion |
| `published_at` | timestamp | Время canonical publication | Completion при `CANONICAL_PUBLISHED` |


---

## Последствия

### Положительные

* Cursor не продвигается при partial materialization, unresolved identity или quarantine.
* Окончание ingestion отделено от обработки отдельных units.
* `QUALITY_REJECTED` не заставляет бесконечно перечитывать заведомо отклонённые данные.
* Snapshot удаляется только после фиксации необходимого результата.
* Manifest и quality metadata сохраняются после удаления raw payload.
* Защита по `cursor_before` предотвращает потерю прогресса при конкурентных crawl.
* Повторный Cursor Commit и Snapshot Cleanup идемпотентны.
* Новый crawl после технического сбоя соответствует принятой модели Source Snapshot.
* `CANONICAL_PUBLISHED` не маскирует partial Qdrant/Neo4j materialization.
* Отсутствие semantic candidate не блокирует ingestion и cursor.
* Low-confidence candidate не блокирует cursor.

### Отрицательные

* Ошибка на позднем этапе может задержать Cursor Commit и следующий incremental crawl.
* При отсутствии snapshot replay технический failure требует повторного обращения к внешнему источнику.
* Требуется отдельное хранение `IngestionRunResult`, reconciliation result и technical metadata.
* Некорректная concurrency policy cursor может привести к необходимости full/reconciliation crawl.
* Snapshot может временно сохраняться дольше обычного при незавершённом Reconciliation или cleanup.
* Если Graph Policy требует обязательной semantic relation, её отсутствие может заблокировать cursor.

---

## Рассмотренные альтернативы

### Продвигать cursor сразу после создания snapshot

Cursor фиксируется сразу после успешного получения source data, независимо от результатов downstream processing.

**Плюсы:**

* внешние источники читаются реже;
* downstream можно обрабатывать независимо;
* ниже задержка connector'а.

**Минусы:**

* downstream может навсегда потерять изменения;
* повторное чтение не гарантирует получение прежнего source state;
* Qdrant и Neo4j могут остаться рассогласованными;
* нарушается принцип завершения ingestion после обязательной materialization.

### Продвигать cursor после успешной записи только в Qdrant

Qdrant считается достаточным для завершения обработки.

**Плюсы:**

* быстрый semantic retrieval;
* меньше зависимость от Neo4j;
* проще happy path.

**Минусы:**

* Graph projection может остаться неполной;
* graph enrichment будет работать на устаревших или отсутствующих данных;
* cursor будет продвинут до восстановления Neo4j;
* нарушается `required_materialization_targets`.

### Продвигать cursor после успешной записи только в Neo4j

Neo4j считается авторитетным target для завершения обработки.

**Плюсы:**

* структурные entities и relations гарантированно записаны;
* можно выполнять graph retrieval.

**Минусы:**

* Qdrant может навсегда остаться неполным;
* semantic retrieval потеряет часть контекста;
* embedding failure не будет восстановлен новым incremental crawl;
* решение зависит от типа content и target policy.

### Хранить snapshot длительно для replay

Сохранять raw snapshot до успешного завершения всех retry и reconciliation.

**Плюсы:**

* повторная обработка использует тот же вход;
* не требуется повторно обращаться к источнику;
* проще восстанавливать partial writes.

**Минусы:**

* увеличивается стоимость object storage;
* усложняется retention;
* появляется отдельный lifecycle replay snapshot;
* противоречит принятой политике Source Snapshot, согласно которой при сбое выполняется новый crawl.

### Удалять snapshot сразу после передачи в downstream

Удалять временный snapshot после успешной фиксации manifest, не дожидаясь materialization и reconciliation.

**Плюсы:**

* минимальное время хранения;
* проще cleanup;
* меньше storage.

**Минусы:**

* невозможно проверить или повторить обработку того же входа;
* при partial write источник может уже измениться;
* усложняется расследование ошибки;
* cursor может быть продвинут при неполном результате.

### Считать отсутствие semantic relation блокирующим для cursor

Не продвигать cursor при отсутствии любой semantic relation.

**Плюсы:**

* максимальная полнота графа;
* гарантируется наличие всех возможных semantic relations.

**Минусы:**

* отсутствие semantic relation может блокировать ingestion;
* low-confidence candidates требуют HITL, что задерживает cursor;
* semantic similarity не всегда точна;
* увеличивается latency ingestion.

**Решение:** отклонено. Semantic candidate без active relation не блокирует cursor.