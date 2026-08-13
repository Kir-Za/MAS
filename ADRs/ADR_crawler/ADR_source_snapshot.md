# ADR Получение источников, Source Snapshot и хранение исходных данных

**Статус:** Предложен

## Контекст

AI Crawler получает данные из Git, Jira и Confluence для обработки общим ingestion pipeline и материализации в Qdrant и Neo4j. Canonical ingestion кода — только `master` и `release/vXX`-ветки. Qdrant и Neo4j должны обрабатывать один входной материал.

Источники и Crawler не имеют общей транзакции. Получение данных может быть частичным, повторным, с задержкой. Необходимы: immutable snapshot, cursors, full/incremental crawl, replay, хранение исходных данных, фиксация отсутствия объектов, идемпотентность.

## Решение

Предлагается схема работы через snapshot источника:
1. Получить данные от источника
2. Построить snapshot и manifest
3. Обработать snapshot в RAG и Graph
4. После успешной обработки — продвинуть cursor

### Source Snapshot

Snapshot — это зафиксированная копия состояния источника на момент получения. Один раз получили данные из Git/Confluence/Jira — положили в хранилище и больше не меняем, т.е. каждый snapshot иммутабелен после фиксации manifest. 
Каждый snapshot содержит: 
- `snapshot_id` - уникальный идентификатор конкретного Source Snapshot'а;
- `source_system` - идентификатор системы-источника, из которой получена сущность;
- `source_scope` - контекст, в котором была получена сущность. Определяет где именно в источнике сущность находилась;
- `ingestion_run_id`— уникальный идентификатор одного запуска ingestion pipeline, объединяющий все snapshot'ы, созданные в рамках этого запуска;
- `observed_at` временная метка, когда crawler впервые зафиксировал данное состояние сущности;
- `source_latest_known` - последняя source boundary (коммит, версия документа и т.д.), которую connector успешно наблюдал и зафиксировал в snapshot/manifest;
- `coverage`;
- `completeness_status`;
- ссылки на manifest;
- raw payload.

`coverage` — что пытались получить, может иметь статус:
- `FULL_SCOPE` полный scope, т.е. планируется выполнить full crawl;
- `PARTIAL_SCOPE` т.е. хотим получить диапазон, change set, подмножество иначе говоря выполнить incremental crawl.

`completeness_status` — результат получения данных относительно заявленного coverage.Отвечает на вопрос: «насколько успешно получено то, что планировали получить». Может принимать значения: 
- `COMPLETE` - заявленный coverage получен полностью;
- `PARTIAL` - часть заявленного coverage не получена;
- `FAILED` - snapshot непригоден для обработки.

Snapshot живёт **только до завершения обработки**. После успешной публикации результатов в Qdrant и Neo4j snapshot и связанный raw payload удаляются. Replay не предусмотрен — при сбое выполняется повторное чтение источника.

Только `FULL_SCOPE` + `COMPLETE` даёт право на  установку параметра `presence_status` из `entity_presence_observation` в статус `ABSENT_CONFIRMED`.


### Manifest и хранение

Manifest это оглавление snapshot'а. Содержит: 
- `snapshot_id`, 
- `source`, 
- `coverage`,
- список объектов, 
- `source revision` каждого,
- cursor до/после, 
- `observed_at`,
- `completeness_status`, 
- ошибки, 
- checksum, 
- ссылки на raw payload. 

Manifest **хранится дольше** чем snapshot — используется для обнаружения удалённых сущностей путём сравнения со следующим manifest'ом.


### Cursors

Cursor — это "отметка", которая помнит где мы остановились при прошлом чтении источника. 
Cursor:
- `source_scope` — какой репозиторий/space/проект,
- `position` — где остановились (коммит, версия страницы, время изменения),
- `saved_at` — когда сохранили,
- `cursor_status` — `VALID` | `INVALID` | `RESET_REQUIRED`

После успешной обработки snapshot'а (а не после его фиксации) `cursor_status` может быть
- `VALID`, позиция существует в источнике, можно доверять и выполнять	Incremental crawl.
- `INVALID` позиции больше нет (force-push, reset) выполняем	Full crawl.
- `RESET_REQUIRED`	изменилась connector policy	 - Full re-crawl.

### Git Connector

Входные параметры:
- `repository_id` - уникальный идентификатор Git-репозитория.
- `branch_or_ref` - ветка или ref для чтения.
- `cursor_position` (для incremental) - последний обработанный коммит из cursor, для full crawl — отсутствует.

Выходные параметры:
- `snapshot_id` - уникальный идентификатор созданного snapshot'а.
- `source_system` - идентификатор системы-источника.
- `source_scope` -   `repository_id` + branch, где были получены данные.
- `coverage` - `FULL_SCOPE` | `PARTIAL_SCOPE` (диапазон коммитов).
- `completeness_status` - `COMPLETE` | `PARTIAL` | `FAILED` — результат получения.
- `commit_sha` - идентификатор каждого коммита, вошедшего в snapshot.
- `parent_commits` - родительские коммиты для каждого коммита, нужны для ancestry-анализа.
- `commit_metadata` - автор, committer, время, сообщение коммита, provenance-данные.
- `changed_paths` - какие файлы изменились в каждом коммите, для provenance.
- `source_files` - содержимое изменённых файлов.
- `checkout_completeness` - полон ли checkout (full clone) или shallow. Влияет на `ABSENT_CONFIRMED`.
- `cursor_after` - новый last_observed_commit, записывается после успешной обработки.

### Jira Connector

Входные параметры:
- `project_key` - проект Jira, из которого читаются задачи.
- `cursor_position` (для incremental) -последнее обработанное изменение из cursor.

Выходные параметры:
- `snapshot_id` - уникальный идентификатор созданного snapshot'а.
- `source_system` - идентификатор системы-источника.
- `source_scope` - project_key или набор issues.
- `coverage` - `FULL_SCOPE` (все задачи проекта) или `PARTIAL_SCOPE` (диапазон изменений).
- `completeness_status` - `COMPLETE` | `PARTIAL` | `FAILED`.
- `issue_key` - уникальный идентификатор задачи.
- `tracked_fields` - снимок отслеживаемых policy-полей на момент изменения.
- `changelog` - история изменений задачи: какие поля, когда, кем.
- `source_changed_at` - время когда изменение произошло в Jira.
- `source_revision_id` - changelog event ID или composite (`issue_key` + позиция + время).
- `cursor_after` - новая source change boundary.


### Confluence Connector

Входные параметры:
- `space_key` - пространство Confluence для чтения.
- `cursor_position` (для incremental) последняя обработанная версия страницы.

Выходные параметры:
- `snapshot_id` - уникальный идентификатор созданного snapshot'а.
- `source_system` - идентификатор системы-источника.
- `source_scope` - space_key.
- `coverage` - `FULL_SCOPE` (все страницы space) или `PARTIAL_SCOPE` (изменившиеся страницы).
- `completeness_status` - `COMPLETE` | `PARTIAL` | `FAILED`.
- `page_id` - уникальный идентификатор страницы.
- `page_version` - номер версии страницы, каждое изменение это новая версия.
- `title` - заголовок страницы.
- `hierarchy` - положение страницы в иерархии: родительские страницы, порядок.
- `content` - содержимое страницы в storage format (HTML с макросами).
- `macros` - встроенные элементы: таблицы, диаграммы, код, ссылки.
- `attachments` - вложения страницы: файлы, изображения.
- `timestamps` - время создания и изменения версии страницы.
- `archive_delete_signals` - сигналы архивирования или удаления, если API их возвращает.

## Последствия

### Положительные

- Простой жизненный цикл: snapshot живёт недолго, не требуется длительное object storage.
- Cursor продвигается после обработки — меньше риск потери данных при сбое.
- Manifest сохраняет возможность обнаружения удалений.

### Отрицательные

- Невозможен replay без повторного чтения источника.
- Нет длительного аудита исходных данных.
- Изменение connector policy требует полного re-crawl — дорого.
- При сбое downstream источник запрашивается повторно — нагрузка на внешние системы.

## Рассмотренные альтернативы

### Длительное хранение snapshot для replay
**Минусы:** требуется durable object storage, сложнее жизненный цикл. Отклонено — проект допускает повторное чтение источника.

### `content_policy_version` для различения policy
**Минусы:** усложнение модели. Отклонено — при изменении policy выполняется полный re-crawl.

### `crawl_mode` как отдельное поле
**Минусы:** дублирует coverage. Отклонено — режим выводится из coverage.