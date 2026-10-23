# ADR Финализация версий и наблюдений

**Статус:** Предложен

---

## Контекст и проблема

Для units, прошедших Quality Gate и получивших разрешённый `entity_uid`, необходимо зафиксировать:

- content-equivalent состояние сущности;
- конкретную source revision;
- факт наблюдения этого состояния;
- период его действия в заданном `source_scope`;
- связь с временным `snapshot_id` и ingestion run.

Одна и та же версия сущности может наблюдаться несколько раз:

- при повторном crawl;
- в разных source scopes;
- после возврата к прежнему состоянию;
- в разных snapshots.

При этом `entity_version_id` не должен дублироваться для одинакового `entity_uid` и `content_hash`, а история наблюдений не должна теряться.

Git использует commit DAG и допускает merge, rebase, cherry-pick, force-push и revert. Confluence имеет версии страниц. Jira имеет changelog и последовательность изменений. Поэтому единый глобальный флаг текущей версии и единая линейная логика для всех источников неприменимы.

**Проблема:** как финализировать `entity_version_id`, создать корректные observation-записи и вычислять актуальность версии по source scope и времени, не смешивая identity, source revision, valid time и system time.

---

## Факторы решения

* `entity_uid` уже назначен на этапе идентификации сущности и не должен изменяться этим этапом.
* `entity_version_id` должен определяться через `entity_uid` и `content_hash`.
* Повторное наблюдение не должно создавать новую content-equivalent version.
* Каждый новый факт наблюдения должен сохраняться отдельно.
* Актуальность должна определяться по `valid_from`/`valid_until`, а не по времени публикации.
* Git history может быть нелинейной.
* `source_changed_at` может отсутствовать.
* `ABSENT_CONFIRMED` допустим только при подтверждённом полном source scope.
* Unresolved identity не создаёт `EntityRevisionObservation`.
* `canonical_status` относится к `IngestionRunResult`, а не к observation.
* Snapshot временный; его provenance должен быть перенесён в постоянные записи.
* `tenant_id`, ACL metadata, Bamboo, `content_policy_version`, `content_version` и отдельный `crawl_mode` не используются.
* Semantic candidate generation начинается только после финализации обеих identities. Обе стороны semantic relation кандидата должны иметь финализированные `entity_uid`, `entity_version_id` и `EntityRevisionObservation`.

---

## Решение

Мы выполняем финализацию версии и observation после идентификации сущности:

```text
PASS unit + resolved identity
    ↓
entity_version_id
    ↓
EntityRevisionObservation
```

Отдельно обрабатываем отсутствие сущности:

```text
absence signal
    ↓
EntityPresenceObservation
```

В рамках определения версии не назначаем identity, не пересчитываем `content_hash`, не выполняем Quality Gate и не определяем `canonical_status`.

### 1. Входные параметры

На этапе определения версии мы получаем `ProcessingUnit` после Identity Resolution:

```text
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

content_type
source_container
processing_status
unit_source_anchor
structured_content
content_hash
raw_payload_checksum

quality_verdict
entity_uid
derived_key
identity_status
provenance
```

Для создания `EntityRevisionObservation` обязательны:

```text
quality_verdict = PASS
entity_uid присутствует
content_hash присутствует
source_revision_id присутствует
```

Unit с:

```text
identity_status = AMBIGUOUS
entity_uid отсутствует
```

не передаётся в финализацию версии. Для него сохраняется `UnresolvedIdentityRecord`.

### 2. Формирование `entity_version_id`

Для resolved identity вычисляется:

```text
entity_version_id = hash(entity_uid + content_hash)
```

Если для той же пары:

```text
entity_uid + content_hash
```

уже существует версия, используется существующий `entity_version_id`.

Если такой пары ещё нет, создаётся новая content-equivalent version.

`entity_version_id` не включает:

```text
snapshot_id
ingestion_run_id
source_revision_id
source_scope
observed_at
ingested_at
published_at
```

Эти значения относятся к observation, provenance или system time.

### 3. `EntityRevisionObservation`

Для каждого фактического наблюдения resolved entity создаётся отдельная запись:

```text
EntityRevisionObservation:
  observation_id
  entity_uid
  entity_version_id

  source_system
  source_scope
  source_object_id
  source_revision_id
  unit_source_anchor
  source_changed_at

  valid_from
  valid_until
  valid_until_unknown
  valid_time_source

  is_initial_baseline

  observed_at
  ingested_at

  provenance:
    snapshot_id
    ingestion_run_id
    raw_payload_checksum
    connector_version
    extractor_version
    parser_version
    quality_gate_version
    identity_resolver_version

  created_at
```

Observation создаётся только для:

```text
presence_status = PRESENT
quality_verdict = PASS
resolved identity
```

`canonical_status` и `published_at` не входят в observation, они хранятся в `IngestionRunResult`.

**Prerequisite для semantic candidate generation:**

После завершения определения версии обе стороны потенциального semantic relation кандидата имеют финализированные:

```text
entity_uid
entity_version_id
EntityRevisionObservation
```

Это является обязательным условием для того, чтобы граф мог формировать semantic domain relations (`DESCRIBES`, `DOCUMENTS`, `SEMANTIC_ASSOCIATION`) между сущностями из разных источников. Semantic candidate generation не начинается до тех пор, пока обе endpoint identities не финализированы и не имеют соответствующих observations.

### 4. Повторные наблюдения

Повторное наблюдение существующей content-equivalent version:

```text
entity_uid + content_hash
```

не создаёт новый `entity_version_id`, но создаёт новый `observation_id`.

Новые observations сохраняют различия:

- `snapshot_id`;
- `ingestion_run_id`;
- `source_revision_id`;
- `source_scope`;
- `observed_at`;
- `ingested_at`;
- provenance.

Таким образом, последовательность:

```text
V1 observed at A
V1 observed at B
V2 observed at C
V1 observed at D
```

сохраняет один content state V1, но несколько фактов его наблюдения.

Повторная обработка того же `snapshot_id` не создаёт дополнительную observation. Новый crawl с новым `snapshot_id` создаёт новую observation, если source state фактически был наблюдён заново.

### 5. Valid time и system time

Разделяются два временных измерения.

#### Valid time

```text
valid_from
valid_until
```

Описывает период действия состояния в предметной области или source scope.

#### System time

```text
observed_at
ingested_at
published_at
```

Описывает, когда crawler увидел, обработал или опубликовал состояние.

`published_at` не влияет на `valid_from`, `valid_until` или выбор версии для `as_of`.

### 6. Определение `valid_from`

`valid_from` определяется source-specific правилами:

| Источник | Правило |
|---|---|
| Git | timestamp commit, если он принят как source evidence |
| Confluence | timestamp создания page version |
| Jira | timestamp changelog change |
| Нет достоверного source time | `observed_at`, `valid_time_source=inferred` |
| Время восстановлено из source history | вычисленное значение, `valid_time_source=derived` |

`source_changed_at`:

- обязателен при `valid_time_source=source`;
- отсутствует при `valid_time_source=inferred`;
- может отсутствовать при `valid_time_source=derived`.

Равенство:

```text
source_changed_at = valid_from
```

не требуется.

### 7. Определение `valid_until`

`valid_until` вычисляется только в рамках конкретного `source_scope`.

#### Git

`valid_until` устанавливается, если найден однозначный successor той же entity в той же ветке или release scope.

При:

- merge;
- divergent history;
- rebase;
- force-push;
- неоднозначном successor;

устанавливается:

```text
valid_until = null
valid_until_unknown = true
```

#### Confluence

В линейной истории страницы:

```text
valid_until =
  timestamp следующей подтверждённой page version
```

#### Jira

В линейной истории issue:

```text
valid_until =
  timestamp следующего подтверждённого changelog change
```

Если порядок или граница следующего состояния недостоверны, `valid_until` остаётся неизвестным.

### 8. Повторное появление прежней версии

При последовательности:

```text
V1 → V2 → V1
```

последнее наблюдение повторно ссылается на существующий `entity_version_id` V1.

При этом:

- создаётся новая `EntityRevisionObservation`;
- сохраняется новый `source_revision_id`;
- сохраняется новый период действия V1;
- исторический переход не теряется.

`REVERTED` не является отдельным глобальным статусом версии. Это характеристика последовательности observations в конкретном source scope.

### 9. Контролируемое обновление `valid_until`

`EntityRevisionObservation` является неизменяемой, за исключением контролируемого обновления `valid_until` и связанных технических полей аудита.

`valid_until` может быть установлен после появления подтверждённого successor.

Повторная установка того же значения идемпотентна и не создаёт новую observation.

Обновление не изменяет:

```text
entity_uid
entity_version_id
source_revision_id
content_hash
observed_at
```

Корректировка фиксируется в `update_history`.

### 10. `EntityPresenceObservation`

Если entity не обнаружена в snapshot, создаётся отдельная presence observation:

```text
EntityPresenceObservation:
  observation_id
  entity_uid
  snapshot_id
  source_scope
  source_object_id
  source_revision_id
  source_latest_known

  presence_status
  reason
  observed_at
  coverage
  completeness_status
```

Допустимые `presence_status`:

```text
ABSENT_CONFIRMED
NOT_OBSERVED
ACCESS_UNKNOWN
EXTRACTOR_GAP
```

`PRESENT` не используется в `EntityPresenceObservation`. При `PRESENT` создаётся `EntityRevisionObservation`.

### 11. Правила `ABSENT_CONFIRMED`

`ABSENT_CONFIRMED` создаётся только если:

```text
coverage = FULL_SCOPE
completeness_status = COMPLETE
```

и source-specific scope действительно позволяет сделать вывод об отсутствии объекта.

Для `PARTIAL_SCOPE` отсутствие объекта означает:

```text
NOT_OBSERVED
```

если источник не предоставил explicit deletion signal.

Для `ABSENT_CONFIRMED` должна быть доступна source boundary:

```text
source_revision_id
```

или:

```text
source_latest_known
```

Для `NOT_OBSERVED`, `ACCESS_UNKNOWN` и `EXTRACTOR_GAP` эти поля могут отсутствовать.

### 12. Выбор версии по `as_of`

Для запроса с `as_of`:

1. выбираются observations указанного `source_scope`;
2. исключаются observations с:
   ```text
   valid_from > as_of
   ```
3. observation с:
   ```text
   valid_until != null
   ```
   подходит, если:
   ```text
   as_of < valid_until
   ```
4. observation с:
   ```text
   valid_until = null
   valid_until_unknown = false
   ```
   считается открытым интервалом;
5. observation с:
   ```text
   valid_until = null
   valid_until_unknown = true
   ```
   не выбирается автоматически и требует дополнительного scope или явной policy;
6. среди подходящих выбирается observation с максимальным `valid_from`.

Выбор выполняется по valid time, а не по:

```text
observed_at
ingested_at
published_at
```

Если подходящей observation нет:

```text
NOT_APPLICABLE_FOR_AS_OF
```

### 13. `version_context`

Все downstream-компоненты используют единый контракт:

```text
version_context:
  source_revision_id
  branch | release_tag
  source_scope
  as_of
```

`version_context` не является частью `ProcessingUnit` и не входит в `entity_version_id`.

Конфликтующие параметры приводят к:

```text
INVALID_VERSION_CONTEXT
```

Без silent fallback.

### 14. Хранение и mapping

`EntityRevisionObservation` сохраняет:

```text
source_system
source_scope
source_object_id
source_revision_id
unit_source_anchor
source_changed_at
valid_from
valid_until
valid_until_unknown
valid_time_source
entity_uid
entity_version_id
observed_at
ingested_at
is_initial_baseline
provenance
```

`provenance` сохраняет:

```text
snapshot_id
ingestion_run_id
raw_payload_checksum
connector_version
extractor_version
parser_version
quality_gate_version
identity_resolver_version
```

`canonical_status` и `published_at` сохраняются в `IngestionRunResult`, а не в observation.

### 15. Границы ответственности

Модуль определения версии отвечает за:

- переиспользование или создание `entity_version_id`;
- создание `EntityRevisionObservation`;
- создание `EntityPresenceObservation`;
- вычисление `valid_from`;
- вычисление `valid_until`;
- установку `valid_until_unknown`;
- установку `valid_time_source`;
- сохранение source и system time;
- подготовку данных для Deduplicatioin и последующей materialization;
- обеспечение prerequisite для semantic candidate generation — финализация `entity_uid`, `entity_version_id` и `EntityRevisionObservation` для обеих сторон потенциального semantic relation.

Модуль определения версии не отвечает за:

- получение источников;
- классификацию;
- парсинг;
- вычисление `content_hash`;
- Quality Gate;
- Identity Resolution;
- deduplication и semantic lineage relations;
- Chunking;
- Embedding;
- Graph Extraction;
- Qdrant/Neo4j materialization;
- `canonical_status`;
- Cursor Commit;
- Snapshot Cleanup;
- retention;
- формирование semantic domain relations.

## Инварианты

1. Выполняется только после Quality Gate и Identity Resolution.
2. `entity_uid` не изменяется на этапе определения версии.
3. `entity_version_id = hash(entity_uid + content_hash)`.
4. `entity_version_id` не включает `snapshot_id`, `ingestion_run_id`, `source_revision_id` или system time.
5. Одинаковые `entity_uid` и `content_hash` используют один `entity_version_id`.
6. Каждое новое фактическое наблюдение создаёт отдельную `EntityRevisionObservation`.
7. Повторная обработка того же `snapshot_id` не создаёт новую observation.
8. `EntityRevisionObservation` создаётся только для `PRESENT + PASS + resolved identity`.
9. Unresolved identity не создаёт `EntityRevisionObservation`.
10. `PRESENT` не является значением `EntityPresenceObservation`.
11. `source_changed_at` может отсутствовать.
12. `valid_time_source=source` требует `source_changed_at`.
13. `valid_time_source=inferred` использует `observed_at`.
14. `valid_until` вычисляется только в конкретном `source_scope`.
15. `valid_until_unknown=true` не трактуется как обычный открытый интервал.
16. `published_at` не определяет valid time.
17. `canonical_status` не хранится в `EntityRevisionObservation`.
18. `ABSENT_CONFIRMED` требует полного и корректного scope либо explicit deletion signal.
19. Отсутствие в partial/change-set scope не является подтверждённым удалением.
20. `source_object_id` и `unit_source_anchor` сохраняются для локализации observation.
21. `EntityRevisionObservation` не изменяет identity и `content_hash`.
22. `version_context` не входит в identity или version ID.
23. Конфликтующий `version_context` приводит к `INVALID_VERSION_CONTEXT`.
24. `tenant_id`, ACL metadata, Bamboo, `content_policy_version`, `content_version` и `crawl_mode` не используются.
25. Semantic candidate generation начинается только после финализации обеих identities. Обе стороны semantic relation кандидата должны иметь финализированные `entity_uid`, `entity_version_id` и `EntityRevisionObservation`.

## Новые переменные

| Переменная | Тип | Описание | Точка появления |
|---|---|---|---|
| `EntityRevisionObservation` | структура | Факт наблюдения resolved entity version в source revision | Version Finalization |
| `EntityPresenceObservation` | структура | Факт отсутствия или невозможности подтвердить присутствие entity | Version Finalization |
| `valid_from` | timestamp | Начало действия состояния в valid time | Version Finalization |
| `valid_until` | timestamp | Конец действия состояния в заданном source scope | Version Finalization |
| `valid_until_unknown` | boolean | Признак неизвестной верхней temporal boundary | Version Finalization |
| `valid_time_source` | enum | Источник `valid_from`: `source`, `inferred`, `derived` | Version Finalization |
| `update_history` | collection | История контролируемых изменений `valid_until` | Version Finalization |
| `NOT_APPLICABLE_FOR_AS_OF` | status | Нет подходящей версии для заданного времени | Version Finalization |
| `INVALID_VERSION_CONTEXT` | status | Противоречивый version context | Version Finalization/вход downstream |
| `is_initial_baseline` | boolean | Признак версии, полученной при initial baseline | Version Finalization |


## Последствия

### Положительные

* Одна content-equivalent версия не дублируется при повторных observations.
* История повторного crawl, revert и появления версии в разных scopes сохраняется.
* Source revision и entity version не смешиваются.
* Git DAG учитывается при вычислении `valid_until`.
* `valid_until_unknown` отделён от обычного открытого интервала.
* Confluence, Jira и Git используют общий temporal contract с source-specific mapping.
* `ABSENT_CONFIRMED` защищён от ложных удалений при частичном crawl.
* `canonical_status` отделён от факта наблюдения версии.
* Source object и unit anchor сохраняются для последующей локализации и аудита.
* Qdrant и Neo4j получают согласованный version context.
* Semantic candidate generation имеет чёткий prerequisite: обе стороны кандидата имеют финализированные identity и observations, что исключает работу с provisional или неразрешёнными сущностями.

### Отрицательные

* Необходимо хранить отдельные revision и presence observations.
* Temporal queries сложнее обычного выбора последней версии.
* Git merge и divergent history могут оставлять `valid_until` неизвестным.
* Контролируемое обновление `valid_until` требует аудита.
* Повторные observations увеличивают объём постоянных metadata.
* Неполные source boundaries могут приводить к `NOT_APPLICABLE_FOR_AS_OF` или невозможности подтвердить удаление.
* Ошибочная source-specific mapping policy может повлиять на исторический retrieval.

## Рассмотренные альтернативы

### Хранить только последнюю версию

Сохранять только текущее состояние сущности без истории observations.

**Плюсы:**

* минимальный объём хранения;
* простая логика retrieval;
* не требуется вычислять temporal intervals.

**Минусы:**

* невозможен исторический и `as_of` retrieval;
* теряется история revert и повторных наблюдений;
* невозможно полноценно анализировать инциденты и старые releases;
* сложнее отличить удаление от отсутствия данных.

**Решение:** отклонено.

### Использовать `source_revision_id` как `entity_version_id`

Считать commit SHA, page version или Jira changelog revision идентификатором entity version.

**Плюсы:**

* прямая связь с исходной системой;
* простая трассировка;
* не требуется content-equivalent deduplication.

**Минусы:**

* разные источники имеют несовместимые revision semantics;
* один Git commit содержит множество entity versions;
* одинаковое содержимое дублируется при повторных revisions;
* source revision не идентифицирует логическое состояние конкретной entity.

**Решение:** отклонено.

### Создавать новую `entity_version_id` для каждого observation

Каждый crawl, snapshot или source revision получает новую entity version.

**Плюсы:**

* простая реализация;
* прямое соответствие observation и version;
* не требуется отдельная дедупликация content state.

**Минусы:**

* повторный crawl создаёт дубликаты;
* revert раздувает количество версий;
* растёт объём Qdrant/Neo4j materialization;
* теряется различие между состоянием и фактом его наблюдения.

**Решение:** отклонено. Observation и content-equivalent version разделены.

### Хранить `valid_from` и `valid_until` на `entity_version_id`

Считать temporal interval глобальным свойством content-equivalent version.

**Плюсы:**

* простая модель;
* быстрый temporal filtering;
* меньше отдельных metadata.

**Минусы:**

* одна версия может быть актуальна в нескольких scopes одновременно;
* нельзя корректно представить master и release;
* revert создаёт конфликтующие интервалы;
* temporal applicability смешивается с content identity.

**Решение:** отклонено. Temporal applicability определяется по observation и `source_scope`.

### Считать каждое отсутствие удалением

При отсутствии entity в новом crawl сразу закрывать её valid interval.

**Плюсы:**

* простая логика удаления;
* быстрое обновление current state;
* не требуется отдельный presence status.

**Минусы:**

* incremental crawl не содержит полный scope;
* технический сбой будет интерпретирован как удаление;
* возможна потеря исторической корректности;
* последующее восстановление entity станет неоднозначным.

**Решение:** отклонено. Отсутствие фиксируется через `EntityPresenceObservation`.

### Начинать semantic candidate generation до финализации identity

Формировать semantic relation candidates до завершения идентификации сущности или определения версии.

**Плюсы:**

* потенциально более раннее обнаружение semantic relations;
* можно использовать provisional identity для поиска.

**Минусы:**

* provisional identity может измениться;
* отсутствует полноценный version context;
* сложнее гарантировать корректность relation endpoints;
* риск создания relation между неподтверждёнными сущностями.

**Решение:** отклонено. Semantic candidate generation начинается только после финализации обеих identities.