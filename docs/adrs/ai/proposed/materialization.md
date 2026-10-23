# ADR Материализация представлений в Qdrant

**Статус:** Предложен

---

## Контекст и проблема

После Chunker формирует `Chunk`, а после Embedding формирует `EmbeddingResult`. Эти артефакты должны быть записаны в Qdrant как точки с dense/sparse vectors и retrieval metadata.

Qdrant не участвует в общей транзакции с Neo4j. Повторная обработка одного snapshot, source revision или Chunk возможна, а изменение chunking или embedding требует полной замены текущего набора представлений. При этом исторические representations должны оставаться доступными для `as_of`-запросов, а устаревшие representations не должны попадать в default retrieval.

**Проблема:** как идемпотентно записывать и заменять Qdrant representations, сохраняя связь с `entity_uid`, `entity_version_id`, `source_scope` и provenance, не смешивая текущие и исторические данные.

---

## Факторы решения

* Идемпотентность повторной materialization.
* Совместимость с `chunk_id`, сформированным Chunker.
* Согласованность Chunk и EmbeddingResult.
* Разделение current и historical representations.
* Отсутствие смешения старого и нового набора chunks.
* Возможность восстановления после частичной записи.
* Совместимость с Reconciliation.
* Сохранение source и temporal metadata.
* Отсутствие `tenant_id` и ACL в Qdrant payload согласно текущему scope проекта.
* Qdrant используется для semantic candidate generation на этапе Graph Extraction (семантическая фаза). Points становятся доступными для candidate search после успешной записи необходимых points.

---

## Решение

Мы используем Qdrant как хранилище материализованных Qdrant representations. Каждая точка Qdrant соответствует одному `Chunk` и имеет идентификатор `chunk_id`.

Qdrant Materialization выполняется после:

```text
Chunker
→ Embedding
→ Qdrant Materialization
```

Материализация выполняется только для Chunk, который:

```text
quality_verdict = PASS
entity_uid определён
entity_version_id определён
content_hash определён
chunk_id сформирован
EmbeddingResult.status = SUCCEEDED
```

### 1. Входные данные

Qdrant Writer получает:

```text
Chunk:
  chunk_id
  entity_uid
  entity_version_id
  content_hash
  content_type
  source_container
  source_system
  source_scope
  source_object_id
  source_revision_id
  unit_source_anchor
  chunk_anchor
  text
  chunk_index
  is_atomic
  is_oversized
  hierarchy
  section_hierarchy

EmbeddingResult:
  chunk_id
  dense_vector
  sparse_vector
  embedding_model
  embedding_model_version
  token_count
  embedding_status
```

`chunk_id` в `Chunk` и `EmbeddingResult` должен совпадать. Несовпадение считается ошибкой materialization.

### 2. Qdrant point

Для каждого успешно обработанного Chunk создаётся одна Qdrant point:

```text
Point:
  id: chunk_id
  vector:
    dense: dense_vector
    sparse: sparse_vector
  payload:
    entity_uid
    entity_version_id
    content_hash
    content_type
    source_container
    source_system
    source_scope
    source_object_id
    source_revision_id
    unit_source_anchor
    chunk_anchor
    text
    chunk_index
    is_atomic
    is_oversized
    hierarchy
    section_hierarchy
```

`embed_text` в payload не сохраняется.

`chunk_id` является единственным идентификатором Qdrant point. Qdrant native point ID не используется для cross-store join.

`parent_unit_id` не записывается в Qdrant, поскольку является временным идентификатором in-memory IR.

### 3. Проверка перед записью

До upsert Qdrant Writer проверяет:

- наличие обязательных полей Chunk;
- совпадение `Chunk.chunk_id` и `EmbeddingResult.chunk_id`;
- наличие dense vector;
- наличие sparse vector;
- размерность dense vector;
- соответствие vector model policy;
- отсутствие превышения `embedding_hard_limit`;
- корректность `source_scope`;
- корректность `entity_uid` и `entity_version_id`;
- соответствие `content_hash` между Chunk и materialization request.

Если проверка не пройдена, point не записывается, а результат фиксируется как ошибка materialization.

### 4. Идемпотентный upsert

Запись выполняется идемпотентным upsert по `chunk_id`.

Повторная запись того же `chunk_id`:

- не создаёт новую point;
- обновляет point только если materialization относится к допустимой актуальной source revision;
- не должна заменять более новую representation устаревшим результатом;
- должна сохранять одинаковые identity и source metadata.

Если payload или vectors отличаются, Writer проверяет актуальность source revision согласно source-specific правилам и stale-writer policy.

Повторный replay/re-crawl одного content-equivalent состояния может использовать тот же `chunk_id`.

### 5. Проверка staleness

Перед записью Writer проверяет актуальность materialization относительно:

```text
entity_uid
entity_version_id
source_scope
source_revision_id
```

Порядок source revisions определяется source-specific правилами:

- Git — commit ancestry и текущая revision boundary;
- Confluence — page version;
- Jira — source revision/changelog ordering.

`ingestion_run_id` не используется как критерий новизны. Он применяется только для аудита и связывания результатов одного запуска.

Устаревший результат не записывается поверх более новой materialization и получает статус `STALE` в результате materialization.

### 6. Current и historical representations

Qdrant materialization различает:

```text
Historical representation:
  point старой entity_version_id или старого valid-time состояния,
  доступный для исторического retrieval.

Current representation:
  point, допустимый для default retrieval согласно source_scope
  и temporal policy Version Finalization.
```

Qdrant Writer не удаляет исторические points только потому, что появилась новая версия entity.

Исключение исторических points из default retrieval выполняется через current projection или соответствующие retrieval filters, определённые Qdrant/Retrieval policy.

### 7. Expected set

Для каждой полной materialization operation формируется:

```text
expected_set:
  entity_uid
  source_scope
  expected_chunk_ids
```

`expected_chunk_ids` — полный набор points, который должен присутствовать в текущей materialization данного `entity_uid` и `source_scope`.

Points, существующие в текущем scope, но отсутствующие в `expected_chunk_ids`, считаются stale candidates. Они:

- не удаляются до проверки historical/current policy;
- удаляются или деактивируются только в current projection;
- не должны затрагивать points другого `source_scope`;
- передаются в Reconciliation, если невозможно безопасно определить их статус.

### 8. Replacement current projection

При изменении параметров Chunking или Embedding выполняется:

```text
full re-crawl
→ новый Chunk set
→ новый EmbeddingResult set
→ формирование expected_set
→ запись новых points
→ cleanup/деактивация отсутствующих current points
→ Reconciliation
```

Старые historical representations не удаляются автоматически.

До завершения replacement Qdrant Writer и Retrieval policy должны исключать смешение старого и нового current set. Конкретный механизм переключения и критерий завершения определяются в рамках Reconciliation и связанной Retrieval/Qdrant policy.

### 9. Материализация oversized representations

Если Chunk создан после structural fallback, он записывается как обычная Qdrant point с унаследованными:

```text
entity_uid
entity_version_id
content_hash
```

Для fallback representations:

- `unit_source_anchor` используется для локализации;
- `chunk_anchor` идентифицирует representation;
- `is_atomic` отражает неделимость конкретного Chunk;
- `is_oversized` отражает состояние конкретного созданного Chunk;
- parent identity не заменяется child identity.

Если Chunk не создан из-за невозможности безопасного embedding, Qdrant point не создаётся. Результат фиксируется в ingestion metadata и учитывается при определении `materialization_status`.

### 10. Использование Qdrant для semantic candidate generation

Qdrant используется для semantic candidate generation на этапе Graph Extraction (семантическая фаза).

Points становятся доступными для candidate search после успешной записи необходимых Qdrant points.

#### Доступность points для candidate search

Points доступны для semantic candidate search, когда:

```text
materialization_status = COMPLETE для соответствующего entity_uid и source_scope
```

или:

```text
конкретный Chunk записан и доступен для поиска
```

#### Scope и фильтры для candidate search

При semantic candidate search (Graph Extraction, семантическая фаза) используются следующие фильтры:

```text
source_scope:
  - ограничивает поиск указанным scope (repository, space, project)
  - может быть опущен для cross-scope поиска

source_system:
  - фильтр по source_system (git, confluence, jira)
  - используется для поиска только в документации или только в коде

content_type:
  - фильтр по content_type (NARRATIVE_TEXT, CODE, STRUCTURED_RECORD)
  - используется для поиска только narrative-текстов или только кода

entity_uid:
  - поиск только для конкретной entity
  - используется для поиска связанных сущностей

source_revision_id:
  - фильтр по revision (актуальная версия)
  - используется для поиска только актуальных representations
```

Правила доступа для candidate search:

1. Candidate search выполняется только после Qdrant materialization (после завершения записи необходимых points).
2. Поиск выполняется среди сущностей с финализированными `entity_uid` и `entity_version_id`.
3. Результат поиска: `similarity_score` для каждой пары.
4. Candidate search не изменяет Qdrant points.
5. Candidate search не влияет на materialization status.
6. Отсутствие результатов candidate search не блокирует ingestion.

### 11. Результат Qdrant materialization

Для каждого запроса materialization формируется:

```text
qdrant_materialization_result:
  materialization_status
  processed_chunk_ids
  skipped_chunk_ids
  failed_chunk_ids
  stale_chunk_ids
  expected_set
```

`materialization_status` принимает значения:

```text
PENDING
COMPLETE
PARTIAL
FAILED
```

- `COMPLETE` — все обязательные Qdrant points записаны;
- `PARTIAL` — часть points записана, часть пропущена или неуспешна;
- `FAILED` — materialization не выполнена;
- `PENDING` — результат ещё не завершён.

Результат передаётся в Reconciliation для сверки с Neo4j materialization.

### 12. Ошибки

Qdrant Writer не изменяет:

```text
entity_uid
entity_version_id
content_hash
source_revision_id
text
```

При ошибках:

- point не записывается либо получает только подтверждённый результат upsert;
- ошибка фиксируется в `qdrant_materialization_result`;
- техническое восстановление выполняется через Reconciliation и Resilience;
- `canonical_status` не устанавливается непосредственно Qdrant Writer.

### 13. Границы ответственности

Qdrant Materialization отвечает за:

- создание Qdrant point;
- проверку Chunk и EmbeddingResult;
- идемпотентный upsert;
- staleness check;
- формирование expected set;
- подготовку current projection cleanup;
- формирование Qdrant materialization result;
- обеспечение доступности Qdrant points для semantic candidate search после успешной materialization.

Qdrant Materialization не отвечает за:

- вычисление `content_hash`;
- назначение `entity_uid`;
- создание `entity_version_id`;
- повторную классификацию;
- Quality Gate;
- Graph Extraction (структурная фаза);
- финальный `canonical_status`;
- межхранилищное Reconciliation;
- окончательное решение о retention исторических points;
- semantic candidate generation — это ответственность Graph Extraction (семантическая фаза).

---

## Инварианты

1. Qdrant Writer принимает только Chunk с `quality_verdict=PASS` и разрешённой identity.
2. `Chunk.chunk_id` и `EmbeddingResult.chunk_id` должны совпадать.
3. Qdrant point ID равен `chunk_id`.
4. Qdrant Writer не пересчитывает `entity_uid`, `entity_version_id` или `content_hash`.
5. `parent_unit_id` не сохраняется в Qdrant payload.
6. `embed_text` не сохраняется в Qdrant payload.
7. Повторный upsert того же `chunk_id` идемпотентен.
8. Устаревший результат не может перезаписать более новую materialization.
9. Новизна source revision определяется source-specific правилами, а не `ingestion_run_id`.
10. Historical representations не удаляются автоматически при появлении новой версии.
11. Expected set формируется отдельно для каждого `entity_uid + source_scope`.
12. Cleanup одного `source_scope` не затрагивает другой scope.
13. Qdrant Writer не создаёт domain entities и relations Neo4j.
14. Oversized fallback не изменяет identity родительской entity.
15. Отсутствующий Chunk не имеет `is_oversized`; факт отказа фиксируется в materialization metadata.
16. `materialization_status` не заменяет `canonical_status`.
17. `CANONICAL_PUBLISHED` не устанавливается Qdrant Writer самостоятельно.
18. Qdrant materialization result передаётся в Reconciliation.
19. Смешение старого и нового current chunk set должно быть исключено до завершения replacement.
20. Технические ошибки materialization не изменяют source identity и temporal identity.
21. Qdrant points становятся доступными для semantic candidate search после успешной записи необходимых points.
22. Semantic candidate search не изменяет Qdrant points и не влияет на materialization status.
23. Отсутствие результатов candidate search не блокирует ingestion.

---

## Новые переменные

| Переменная | Тип | Описание | Точка появления |
|---|---|---|---|
| `Point` | структура | Материализованная Qdrant point с vectors и payload | Qdrant Writer |
| `materialization_targets` | enum/collection | Обязательные targets materialization для текущего run | Entity Policy/Materialization Coordinator |
| `materialization_status` | enum | Состояние materialization: `PENDING`, `COMPLETE`, `PARTIAL`, `FAILED` | Qdrant Writer/Materialization Coordinator |
| `expected_set` | структура/collection | Ожидаемый набор `chunk_id` для `entity_uid + source_scope` | Qdrant Writer |
| `qdrant_materialization_result` | структура | Результат записи Qdrant points | Qdrant Writer |
| `processed_chunk_ids` | collection | Успешно обработанные Chunk IDs | Qdrant Writer |
| `skipped_chunk_ids` | collection | Chunk IDs, пропущенные по policy | Qdrant Writer |
| `failed_chunk_ids` | collection | Chunk IDs, для которых запись завершилась ошибкой | Qdrant Writer |
| `stale_chunk_ids` | collection | Chunk IDs, признанные устаревшими относительно expected set | Qdrant Writer/Reconciliation |
| `STALE` | status value | Устаревший результат materialization, не применённый поверх новой версии | Qdrant Writer |
| `current_projection` | structure/policy | Набор representations, доступный для default retrieval | Qdrant/Retrieval policy |
| `historical_projection` | structure/policy | Набор representations, доступный для historical retrieval | Qdrant/Retrieval policy |


---

## Последствия

### Положительные

* Qdrant point имеет однозначную связь с Chunk и EmbeddingResult.
* Повторный upsert не создаёт дубликаты.
* Устаревшие writers не могут молча перезаписать новые representations.
* Исторические и текущие representations разделены концептуально.
* Cleanup ограничен конкретным `entity_uid` и `source_scope`.
* Изменение chunking/embedding не изменяет identity сущностей.
* Oversized fallback поддерживается без ложного создания child entities.
* Partial Qdrant materialization передаётся в Reconciliation, а не скрывается.
* `embed_text` не дублируется в payload.
* Qdrant Writer не смешивает materialization с identity, parsing или Graph Extraction.
* Semantic candidate search не влияет на materialization и не блокирует ingestion.

### Отрицательные

* Требуется expected-set calculation и cleanup current projection.
* Current/historical filtering должен быть согласован с Retrieval policy.
* Partial writes требуют Reconciliation и recovery.
* Stale protection зависит от source-specific revision ordering.
* Ошибка embedding или превышение hard limit может сделать Qdrant target недоступным для отдельного Chunk.
* При изменении Chunking или Embedding требуется полный re-crawl и replacement current projection.
* Появляются дополнительные materialization results и технические статусы.
* Semantic candidate generation зависит от завершения Qdrant materialization, что увеличивает общую latency.

---

## Рассмотренные альтернативы

### Записывать Chunk и vectors в Qdrant без отдельной materialization policy

Сразу выполнять upsert каждого результата embedding без expected set, stale protection и разделения текущих и исторических points.

**Плюсы:**

* простая реализация;
* минимальное количество компонентов;
* низкая задержка happy path.

**Минусы:**

* старые points остаются после изменения chunking;
* устаревший writer может перезаписать свежие данные;
* невозможно надёжно определить partial materialization;
* усложняется Reconciliation;
* historical и current representations смешиваются.

### Использовать `snapshot_id` как Qdrant point ID

Создавать новый point ID для каждого crawl.

**Плюсы:**

* простое различение запусков;
* отсутствие перезаписи между snapshots;
* удобная диагностика отдельных crawl.

**Минусы:**

* повторный crawl создаёт дубликаты;
* растёт Qdrant index;
* identity content не используется для идемпотентности;
* cleanup становится существенно сложнее.

### Удалять все старые points при появлении новой версии

Хранить только текущий набор Qdrant representations.

**Плюсы:**

* простой default retrieval;
* меньше storage;
* не требуется сложный current filter.

**Минусы:**

* невозможен historical retrieval;
* теряется контекст старых release и incident analysis;
* rollback и аудит становятся неполными;
* revert нельзя корректно представить как повторное состояние.

### Хранить только исторические points и фильтровать актуальность при каждом запросе

Не удалять Qdrant points, а полностью полагаться на `source_scope` и temporal filtering Retrieval Service.

**Плюсы:**

* сохраняется история;
* проще ingestion cleanup;
* не требуется удалять points при каждой новой версии.

**Минусы:**

* retrieval обязан всегда корректно применять temporal filters;
* ошибка фильтра приводит к выдаче устаревшего контента;
* увеличивается размер Qdrant index;
* сложнее поддерживать high-performance current retrieval.

### Выполнять cleanup внутри Chunker

Chunker одновременно создаёт chunks, пишет Qdrant и удаляет устаревшие points.

**Плюсы:**

* единое место для формирования и удаления chunks;
* меньше промежуточных materialization results.

**Минусы:**

* Chunker получает ответственность за состояние Qdrant;
* сложнее обрабатывать partial writes;
* нарушается разделение Chunker и Qdrant Materialization;
* выше риск удалить historical representations;
* усложняется Reconciliation с Neo4j.

### Позволить embedding server самостоятельно разбивать oversized Chunk

Передавать большой Chunk в embedding server, который возвращает несколько vectors.

**Плюсы:**

* больше oversized content получает Qdrant coverage;
* Chunker становится проще;
* token handling централизуется.

**Минусы:**

* embedding server начинает определять chunk boundaries;
* нарушается контракт `chunk_id`;
* теряется контроль над anchors и provenance;
* expected set становится непредсказуемым;
* нарушается ответственность Chunker за формирование Qdrant representations.
