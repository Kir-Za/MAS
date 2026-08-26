# ADR Получение источников и фиксация Source Snapshot

**Статус:** Предложен

---

## Контекст и проблема

AI Crawler получает данные из Git, Jira и Confluence и передаёт их в общий ingestion pipeline для последующей materialization в Qdrant и Neo4j.

Источники имеют разные модели получения данных:

- Git предоставляет repositories, branches, commits и файлы;
- Jira предоставляет issues, tracked fields и changelog;
- Confluence предоставляет spaces, pages, page versions, macros и attachments.

Canonical ingestion кода выполняется для `master` и `release/vXX`-веток репозиториев сервисов и shared-библиотек. Feature- и `dev`-ветки в canonical ingestion не входят.

Qdrant и Neo4j должны обрабатывать один и тот же входной материал. Источники и Crawler не имеют общей транзакции, поэтому получение данных может быть частичным, повторным или завершиться ошибкой.

Snapshot хранится только на время текущей обработки. После успешного завершения обработки snapshot и raw payload удаляются, а manifest сохраняется дольше для аудита, контроля границ crawl и обнаружения удалений. Replay snapshot не используется: после технического сбоя выполняется новый crawl с новым `snapshot_id`.

**Проблема:** как получать данные из Git, Jira и Confluence с фиксацией единого входа для RAG и Graph, корректно отслеживать границы incremental crawl и не создавать ложные выводы об удалении объектов при неполном получении данных.

---

## Факторы решения

* Единый входной материал для RAG и Graph.
* Идемпотентность повторного crawl.
* Отделение source revision от snapshot.
* Корректное обнаружение удалений.
* Минимальное время хранения raw payload.
* Возможность full и incremental crawl.
* Безопасное продвижение cursor только после успешной обработки.
* Совместимость с `entity_revision_observation` и `entity_presence_observation`.
* Отсутствие replay snapshot.
* Разные модели revision boundary для Git, Jira и Confluence.

---

## Решение

Мы используем отдельный Connector для каждого источника и временный immutable `Source Snapshot`.

Общий порядок:

```text
Connector получает данные
    ↓
Snapshot Builder создаёт snapshot и manifest
    ↓
snapshot передаётся в ingestion pipeline
    ↓
RAG и Graph обрабатывают один snapshot
    ↓
materialization и reconciliation завершаются
    ↓
cursor фиксируется
    ↓
snapshot и raw payload удаляются
    ↓
manifest сохраняется
```

### 1. Connector-модель

Используются:

- Git Connector;
- Jira Connector;
- Confluence Connector.

Connector:

- получает данные из исходной системы;
- сохраняет source-native identifiers;
- фиксирует source revisions и source timestamps, если они доступны;
- формирует границы crawl;
- поддерживает full и incremental получение;
- обрабатывает pagination и source API errors;
- формирует `cursor_after`;
- передаёт данные Snapshot Builder.


### 2. Source Snapshot

`Source Snapshot` — временная immutable-копия source records, полученных для одного `source_system` и одного заданного `source_scope` в рамках crawl.

Snapshot содержит:

```text
snapshot_id
source_system
source_scope
ingestion_run_id
observed_at
source_latest_known
coverage
completeness_status
cursor_before
cursor_after
manifest_ref
raw_payload_ref
```

`observed_at` — время, когда Connector начал или завершил фиксацию данных snapshot согласно принятой реализации. Для отдельных records и observations source time хранится через `source_changed_at`.

`source_latest_known` — последняя source boundary, которую Connector успешно наблюдал и зафиксировал для данного scope:

- Git — commit/ref;
- Confluence — page/update boundary;
- Jira — changelog/change boundary.

`ingestion_run_id` группирует snapshots одного запуска, но не создаёт общую транзакцию и не означает одновременное получение данных.

### 3. Source scope

`source_scope` имеет source-specific формат:

```text
Git:
  repository_id + branch_or_release_reference

Confluence:
  space_key

Jira:
  project_key или явно заданный набор issues
```

`source_scope` не является identity сущности. Он используется для:

- определения области crawl;
- temporal filtering;
- presence detection;
- materialization scope;
- различения current и historical representations.

### 4. Coverage

`coverage` принимает значения:

```text
FULL_SCOPE
PARTIAL_SCOPE
```

`FULL_SCOPE` означает полный явно заданный scope текущего crawl:

- полный текущий tree заданной Git revision;
- все страницы заданного Confluence space;
- все задачи заданного Jira project.

`PARTIAL_SCOPE` означает:

- диапазон Git revisions;
- Jira change set;
- изменившиеся Confluence pages;
- явно заданное подмножество объектов.

`coverage` не является отдельным `crawl_mode`. Режим crawl выводится из его значения.

Только `FULL_SCOPE` вместе с `completeness_status=COMPLETE` даёт право рассматривать отсутствие объекта как evidence для `ABSENT_CONFIRMED`, если source-specific правила допускают такой вывод.

### 5. Completeness status

`completeness_status` принимает значения:

```text
COMPLETE
PARTIAL
FAILED
```

- `COMPLETE` — весь заявленный coverage получен;
- `PARTIAL` — часть заявленного coverage не получена;
- `FAILED` — snapshot непригоден для обработки.

`COMPLETE` означает полноту именно заявленного coverage, а не полноту всей истории источника.

`PARTIAL` и `FAILED` snapshot не позволяют автоматически создавать `ABSENT_CONFIRMED`.

### 6. Manifest

Manifest — долговременно сохраняемое оглавление snapshot.

Manifest содержит:

```text
snapshot_id
source_system
source_scope
coverage
completeness_status
cursor_before
cursor_after
source_latest_known
objects
errors
manifest_checksum
```

Каждый элемент `objects` содержит:

```text
source_object_id
source_revision_id
object_type
raw_payload_ref
raw_payload_checksum
```

`object_type` имеет source-specific значения:

```text
code_file
file
page
attachment
issue
```

Manifest не содержит:

- `entity_uid`;
- `entity_version_id`;
- `unit_source_anchor`;
- `parent_unit_id`;
- `content_type`.

Эти значения появляются на последующих этапах classification, parsing и identity resolution.

Manifest используется для:

- аудита состава crawl;
- проверки completeness;
- сравнения полных scope между crawl;
- обнаружения удалений;
- диагностики пропущенных объектов;
- связи с временно хранившимся raw payload.

После удаления raw payload ссылка `raw_payload_ref` сохраняется только как историческая ссылка и не гарантирует возможность replay.

### 7. Raw payload storage

Raw payload хранится во временном object storage до завершения обработки snapshot.

В raw payload могут входить:

- Git metadata и необходимые source files;
- Jira issue payload и changelog;
- Confluence page content, macros и attachments.

Raw payload:

- не изменяется после фиксации snapshot;
- доступен компонентам текущего ingestion pipeline;
- удаляется после успешного завершения обработки и разрешённого cursor commit;
- не используется как долговременный архив.

Manifest и ingestion metadata сохраняются после удаления raw payload.

### 8. Git Connector

Git Connector получает:

```text
repository_id
branch_or_ref
cursor_position
```

Для canonical ingestion допустимы:

```text
master
release/vXX
```

Git Connector фиксирует:

```text
commit_sha
parent_commits
commit_metadata
changed_paths
source_files
checkout_completeness
```

Для incremental crawl обрабатывается диапазон между предыдущей и текущей ref boundary.

Если обнаружены:

- force-push;
- reset;
- rebase;
- потеря cursor;
- отсутствие предыдущего commit в ожидаемой ancestry;
- невозможность определить корректную revision boundary;

выполняется полный crawl или reconciliation crawl. Shallow/partial checkout не позволяет создавать `ABSENT_CONFIRMED`.

`source_changed_at` для Git соответствует времени commit, если оно доступно и принято source policy. Git commit является `source_revision_id`, но не является `entity_uid` или `entity_version_id`.

### 9. Jira Connector

Jira Connector получает:

```text
project_key
cursor_position
```

Для Jira сохраняются:

```text
issue_key
tracked_fields
changelog
source_changed_at
source_revision_id
```

`source_revision_id` использует native changelog/event ID, если он доступен. В противном случае Connector формирует детерминированный composite ID согласно Version and Observation Finalization.

Если несколько полей изменены одной операцией, они формируют один change batch, если это подтверждается Jira API. При невозможности восстановить границы batch source record помечается неполным.

Jira Connector не создаёт provenance relations `IMPLEMENTED_BY_COMMIT` или `REALIZED_IN`. Эти связи формируются позже на основании Git и Jira data согласно Deduplication and Semantic Lineage.

### 10. Confluence Connector

Confluence Connector получает:

```text
space_key
cursor_position
```

Для страниц сохраняются:

```text
page_id
page_version
title
hierarchy
content
macros
attachments
timestamps
archive_delete_signals
```

Incremental crawl использует page version, update boundary, API cursor или их комбинацию.

Повторное получение той же page version не создаёт новую source revision. Новый crawl создаёт новый `snapshot_id`, а дальнейшее создание observation определяется Version and Observation Finalization.

Перемещение страницы внутри space при сохранении `page_id` не меняет source identity страницы. Перенос между spaces или sites не считается автоматически продолжением сущности и передаётся в Identity Resolution согласно ADR-003.

Если attachment не получен:

- соответствующий source record помечается неполным;
- отсутствие attachment не считается подтверждённым удалением;
- page snapshot не считается полностью пригодным для соответствующего content scope, если attachment входит в заявленный coverage.

### 11. Cursor

Cursor хранится отдельно для каждого source scope.

Cursor содержит:

```text
source_scope
position
saved_at
cursor_status
```

`position` имеет source-specific значение:

- Git — commit/ref;
- Confluence — page/update boundary;
- Jira — changelog/change boundary.

`cursor_before` фиксирует значение cursor до начала crawl. `cursor_after` фиксирует границу, до которой Connector получил данные.

Cursor store обновляется только после:

```text
успешной обработки snapshot
+ завершения обязательной materialization
+ успешного reconciliation
```

Если downstream processing завершился ошибкой:

- cursor не продвигается;
- текущий snapshot не используется для replay;
- выполняется новый crawl от прежней cursor boundary с новым `snapshot_id`.

Если cursor position больше недействительна:

```text
cursor_status = INVALID
```

и выполняется полный crawl.

Если изменились правила Connector:

```text
cursor_status = RESET_REQUIRED
```

и выполняется полный re-crawl.

### 12. Full и incremental crawl

Full crawl используется:

- при первом запуске;
- после `INVALID` или `RESET_REQUIRED`;
- после изменения Connector rules;
- после обнаружения пропусков;
- для периодической проверки полного scope;
- для обнаружения удалений.

Incremental crawl используется для штатного получения:

- новых Git commits;
- новых Jira changes;
- изменившихся Confluence pages.

Incremental crawl не позволяет делать вывод об отсутствии объектов, не вошедших в его change range.

### 13. Snapshot lifecycle

Snapshot проходит следующий жизненный цикл:

```text
CREATED
→ READY
→ PROCESSING
→ PROCESSED
→ CLEANED
```

Snapshot становится `READY` только после фиксации manifest и доступности raw payload для ingestion.

Snapshot удаляется после:

- завершения обязательной materialization;
- завершения reconciliation;
- фиксации итогового ingestion result;
- успешного cursor commit.

При техническом сбое или незавершённом reconciliation snapshot не используется как replay source. После сохранения технического результата и принятия recovery decision выполняется новый crawl.

### 14. Idempotency

Повторный crawl может получить тот же source revision, но новый snapshot.

Повторная обработка:

- не должна создавать новую source identity только из-за нового `snapshot_id`;
- не должна создавать дублирующую materialization;
- должна использовать существующие `entity_uid`, `entity_version_id` и `chunk_id`, если они соответствуют тому же source state и scope;
- должна создавать observation согласно Version and Observation Finalization для нового факта получения source state.

`source_revision_id` и source-native object identity используются Connector'ом для дедупликации повторно полученных source records.

### 15. Удаления

Connector не удаляет entities или representations из Qdrant и Neo4j.

Connector только фиксирует:

- explicit deletion/archive signals;
- состав полного scope;
- отсутствие объектов в рамках coverage.

Дальнейшее создание `entity_presence_observation` выполняется после snapshot processing согласно Metadata Contract и Version and Observation Finalization.

`ABSENT_CONFIRMED` допускается только при:

```text
FULL_SCOPE
+ COMPLETE
+ source-specific coverage позволяет делать вывод об отсутствии
```

Для `PARTIAL_SCOPE` отсутствие объекта означает `NOT_OBSERVED`, если источник не предоставил explicit deletion signal.

---

## Инварианты

1. Один `snapshot_id` представляет один immutable входной набор для текущего ingestion operation.
2. RAG и Graph получают один и тот же `snapshot_id` и source revision context.
3. Connector не назначает `entity_uid` и не формирует `entity_version_id`.
4. Snapshot не содержит identity, назначенную downstream-компонентами.
5. `coverage` принимает только `FULL_SCOPE` или `PARTIAL_SCOPE`.
6. Отдельный `crawl_mode` не используется.
7. `COMPLETE` означает полноту заявленного coverage, а не всей истории источника.
8. `PARTIAL_SCOPE` не даёт права на `ABSENT_CONFIRMED`, кроме explicit deletion signal.
9. Manifest immutable и сохраняется дольше raw snapshot.
10. Raw payload удаляется после завершения обработки и cursor commit.
11. Replay snapshot не используется.
12. При сбое выполняется новый crawl с новым `snapshot_id`.
13. Cursor продвигается только после обязательной materialization и reconciliation.
14. Ошибка cursor commit не приводит к blind overwrite.
15. `source_revision_id` и `snapshot_id` имеют разную семантику.
16. `source_object_id` и `object_type` фиксируются до classification и parsing.
17. `unit_source_anchor`, `parent_unit_id` и `content_type` появляются после Connector stage.
18. Connector не удаляет данные из Qdrant или Neo4j.
19. Отсутствие объекта в неполном или частичном coverage не считается удалением.
20. Изменение Connector rules требует полного re-crawl.
21. `tenant_id`, Bamboo, `content_policy_version`, `content_version` и `crawl_mode` не используются.

---

## Новые переменные

| Переменная | Тип | Описание | Точка появления |
|---|---|---|---|
| `SourceSnapshot` | структура | Временный immutable набор source records для одного source scope и crawl | Snapshot Builder |
| `manifest` | структура | Долговременное оглавление snapshot и его состава | Snapshot Builder |
| `SourceRecordRef` | структура | Ссылка на source object, revision и raw payload | Snapshot Builder |
| `raw_payload_ref` | string | Ссылка на временно сохранённый raw payload | Snapshot Builder/object storage |
| `objects` | collection | Объекты, вошедшие в manifest | Snapshot Builder |
| `cursor` | структура | Техническая позиция incremental crawl для source scope | Connector/cursor store |
| `position` | source-specific | Значение позиции cursor | Connector/cursor store |
| `cursor_status` | enum | Состояние cursor: `VALID`, `INVALID`, `RESET_REQUIRED` | Connector |
| `checkout_completeness` | enum | Полнота Git checkout: `COMPLETE`, `SHALLOW`, `PARTIAL` | Git Connector |

---

## Последствия

### Положительные

* RAG и Graph обрабатывают один и тот же immutable source input.
* Manifest позволяет обнаруживать удаления без долговременного хранения raw snapshot.
* Cursor продвигается только после завершения обязательной обработки.
* Повторный crawl не создаёт identity-дубликаты из-за нового `snapshot_id`.
* Source-specific connectors изолируют различия Git, Jira и Confluence.
* Частичный coverage не приводит к ложному `ABSENT_CONFIRMED`.
* Изменение Connector rules обрабатывается полным re-crawl.
* Replay не требуется и не добавляет отдельный lifecycle snapshot.
* `unit_source_anchor` и `parent_unit_id` не появляются до соответствующих downstream-этапов.

### Отрицательные

* При техническом сбое требуется повторное обращение к внешнему источнику.
* Между первым и повторным crawl источник может измениться.
* Manifest не позволяет восстановить удалённый raw payload.
* Неполный incremental crawl может задержать presence resolution.
* Full crawl увеличивает нагрузку на Git, Jira и Confluence.
* Сбой после получения snapshot, но до cursor commit, может привести к повторному чтению уже полученных source records.
* Для каждого источника требуется отдельный cursor и source-specific boundary logic.

---

## Рассмотренные альтернативы

### Долговременное хранение snapshot для replay

Snapshot хранится до завершения всех retry и reconciliation.

**Плюсы:**

* повторная обработка использует точно тот же вход;
* не требуется повторно обращаться к источнику;
* проще восстанавливать partial materialization;
* сохраняется полный raw audit.

**Минусы:**

* увеличиваются стоимость и объём object storage;
* усложняется lifecycle и retention;
* появляется отдельный механизм replay;
* противоречит принятой политике временного snapshot и нового crawl после сбоя.

### Продвижение cursor сразу после получения snapshot

Cursor фиксируется до завершения downstream processing.

**Плюсы:**

* источник читается реже;
* downstream можно обрабатывать независимо;
* ниже задержка connector stage.

**Минусы:**

* незавершённая materialization может навсегда потерять изменения;
* повторное чтение не гарантирует прежнее состояние;
* Qdrant и Neo4j могут остаться рассогласованными;
* нарушается требование завершения pipeline до cursor commit.

### Только полный crawl

Каждый запуск получает полный заданный scope.

**Плюсы:**

* проще обнаруживать удаления;
* меньше зависимость от cursor;
* проще проверять полноту текущего состояния.

**Минусы:**

* высокая нагрузка на источники;
* большая задержка ingestion;
* плохо масштабируется на все repositories, spaces и projects;
* не подходит для частого обновления.

### Только incremental crawl

Обрабатывать только изменения после последней cursor boundary.

**Плюсы:**

* минимальная нагрузка;
* низкая задержка;
* небольшой объём обрабатываемых данных.

**Минусы:**

* не позволяет надёжно обнаруживать удаления;
* зависит от корректности cursor и порядка событий;
* пропущенные изменения могут сохраняться до следующего recovery;
* требуется периодическая полная сверка.

### Читать источники напрямую в RAG и Graph

Каждая downstream-ветвь самостоятельно читает Git, Jira и Confluence.

**Плюсы:**

* меньше промежуточных компонентов;
* потенциально ниже задержка простого сценария;
* не требуется общий snapshot storage.

**Минусы:**

* RAG и Graph могут получить разные source revisions;
* невозможно обеспечить единый вход;
* дублируется source-specific логика;
* сложнее reconciliation и диагностика;
* возрастает риск рассогласования materializations.

### Хранить raw snapshot только в Redis

Использовать Redis для manifest и raw source payload.

**Плюсы:**

* Redis уже используется для cursor и transient state;
* быстрый доступ;
* проще начальная интеграция.

**Минусы:**

* Redis не является authoritative storage больших raw payload;
* eviction и TTL могут привести к потере входных данных;
* неудобно хранить attachments и Git files;
* отсутствует надёжный долговременный manifest storage;
* snapshot становится зависимым от transient state Redis.

**Решение:** отклонено. Redis используется для cursor и transient state, а временный raw payload и долговременный manifest хранятся в object storage.
