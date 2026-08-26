# ADR Отказоустойчивость и восстановление ingestion

**Статус:** Предложен

---

## Контекст и проблема

Ingestion pipeline получает данные из Git, Jira и Confluence, обрабатывает их и материализует результаты в Qdrant и Neo4j. Источники, processing components и целевые хранилища могут временно становиться недоступными или завершать обработку с ошибкой.

В проекте используется временный `Source Snapshot`: после завершения обработки snapshot удаляется, а долговременный replay snapshot не поддерживается. Поэтому повторная обработка после сбоя выполняется через новый crawl с новым `snapshot_id`.

При этом необходимо различать:

- терминальный результат качества;
- незавершённое identity или HITL-решение;
- технический сбой;
- частичную materialization;
- ошибку cursor commit;
- ошибку snapshot cleanup.

Без единой recovery policy возможны:

- потеря изменений при продвижении cursor до завершения materialization;
- повторная обработка с потерей исходной source boundary;
- создание дублей в Qdrant или Neo4j;
- маскировка технической ошибки под `QUALITY_REJECTED`;
- бесконечные повторы одного и того же сбоя;
- удаление snapshot до сохранения результата обработки.

**Проблема:** как классифицировать ошибки ingestion, выполнять безопасное восстановление без replay snapshot, сохранять идемпотентность и гарантировать, что cursor продвигается только после завершения обязательной обработки.

---

## Факторы решения

* Snapshot временный и не используется для долговременного replay.
* После технического сбоя выполняется новый crawl с новым `snapshot_id`.
* Cursor нельзя продвигать до завершения обязательной materialization и reconciliation.
* `QUALITY_REJECTED` не является техническим failure.
* `QUARANTINED` и unresolved identity не являются quality rejection.
* Частичная запись в Qdrant и Neo4j должна быть обнаружена reconciliation.
* Повторные операции не должны создавать дубли.
* Ошибки должны иметь ограниченный retry budget.
* Неисправимые ошибки должны попадать в DLX или эскалацию.
* Recovery не должен изменять `entity_uid`, `entity_version_id`, `content_hash` или `source_revision_id`.
* Ошибка одного source scope не должна автоматически останавливать остальные независимые scopes.
* `tenant_id`, ACL metadata, Bamboo, `content_policy_version`, `content_version` и отдельный `crawl_mode` не используются.

---

## Решение

Мы применяем отказоустойчивость как сквозную policy для всех этапов от получения источников до завершения.

Общий принцип:

```text
Временная ошибка операции
    ↓
ограниченный retry операции
    ↓
успех → продолжение текущего этапа

ошибка получения данных
    ↓
snapshot не создаётся или помечается FAILED
    ↓
новый crawl

ошибка после создания snapshot
    ↓
текущий snapshot не переигрывается
    ↓
фиксация recovery result
    ↓
новый crawl с новым snapshot_id

partial materialization
    ↓
reconciliation/recovery decision
    ↓
при необходимости новый crawl

неисправимый сбой
    ↓
DLX / escalation
```

Resilience не изменяет source data, identity, temporal metadata или результаты контроля качества.

### 1. Классы ошибок

Ошибки разделяются на следующие классы.

#### Временная ошибка операции

Ошибка, для которой допустима повторная попытка той же технической операции:

- кратковременный network timeout;
- временная недоступность API;
- rate limit;
- временная ошибка Qdrant, Neo4j или embedding server;
- временная ошибка cursor store.

Retry выполняется с ограничением количества попыток и с увеличением интервала между попытками.

#### Ошибка получения источника

Ошибка, из-за которой source data не была надёжно получена:

- не завершена pagination;
- не подтверждена source boundary;
- не загружен обязательный payload;
- потерян или повреждён cursor;
- Git ref не может быть корректно разрешён;
- Confluence или Jira change boundary не восстановлена.

Такая ошибка не создаёт пригодный snapshot и не продвигает cursor.

#### Ошибка обработки snapshot

Ошибка после получения snapshot:

- падение parser;
- ошибка контроля качества;
- ошибка разрешения идентичности;
- ошибка извлечения графа;
- ошибка разбиения на фрагменты;
- ошибка встраивания;
- ошибка materialization.

Snapshot не используется для replay. После фиксации технического результата выполняется новый crawl.

#### Частичная materialization

Одна materialization branch завершилась, а другая нет, либо записан неполный набор points/entities.

Такая ошибка не является результатом отклонения контролем качества. Она обрабатывается сверкой и не позволяет установить `CANONICAL_PUBLISHED` или продвинуть cursor.

#### Терминальный результат качества

`QUALITY_REJECTED` означает, что данные обработаны и отвергнуты правилами контроля качества. Это не технический failure.

После фиксации quality result cursor может быть продвинут согласно завершению, если отсутствуют другие незавершённые ошибки.

### 2. Retry policy

Retry применяется только к операциям, которые могут завершиться успешно при повторе.

Для retry используются:

- ограниченный `retry_count`;
- задержка с увеличением интервала;
- ограничение общего времени;
- повторная проверка source boundary;
- защита от одновременных конфликтующих операций.

После исчерпания retry budget:

```text
retry_status = EXHAUSTED
```

и формируется recovery decision.

Retry не должен:

- создавать новый `entity_uid`;
- создавать дублирующий `entity_version_id`;
- продвигать cursor;
- считать техническую ошибку `QUALITY_REJECTED`;
- переигрывать удалённый snapshot как replay.

Повторная техническая операция внутри текущего source read допустима. Повторное чтение источника после завершения неудачного crawl является новым crawl и создаёт новый `snapshot_id`.

### 3. Recovery после ошибки получения данных

Если ошибка произошла до создания `READY` snapshot:

```text
snapshot не передаётся downstream
cursor не продвигается
выполняется повторное чтение источника
```

Если source boundary невозможно подтвердить:

```text
cursor_status = INVALID
```

или:

```text
cursor_status = RESET_REQUIRED
```

в зависимости от причины, после чего выполняется full re-crawl.

Если источник временно недоступен, connector не должен создавать `COMPLETE` snapshot на основании неполного ответа.

### 4. Recovery после ошибки обработки

Если snapshot был создан, но один из downstream-этапов завершился техническим failure:

```text
текущий snapshot не используется для replay
cursor не продвигается
технический результат сохраняется
выполняется новый crawl с новым snapshot_id
```

Новый crawl начинается от прежней cursor boundary. Он может получить уже изменённое состояние источника; это фиксируется как новая source observation и не считается продолжением несуществующего replay.

Если старый snapshot ещё удерживается в рамках незавершённой текущей попытки, он может использоваться для диагностики до принятия recovery decision. После принятия решения о новом crawl raw payload старого snapshot удаляется согласно завершению, а manifest и recovery metadata сохраняются.

### 5. Recovery при partial materialization

При partial materialization:

```text
canonical_status ≠ CANONICAL_PUBLISHED
cursor не продвигается
```

Сверка сначала определяет, может ли расхождение быть устранено уже записанными materialization operations.

Если восстановление невозможно без повторного получения источника:

```text
текущий snapshot не используется для replay
выполняется новый crawl
новый snapshot_id
```

Повторная materialization должна использовать уже существующие:

```text
entity_uid
entity_version_id
content_hash
chunk_id
```

если соответствующее source state и scope совпадают.

### 6. Recovery при неразрешённой identity

Если контроль качества пройден, но разрешение идентичности не смог определить identity:

```text
quality_verdict = PASS
identity_status = AMBIGUOUS
entity_uid отсутствует
```

то:

- materialization не выполняется;
- создаётся `UnresolvedIdentityRecord`;
- cursor не продвигается для соответствующего source scope;
- после HITL или дополнительного свидетельства выполняется новый crawl;
- новый crawl получает новый `snapshot_id`;
- решение применяется только при совпадении source и unit context.

Если новый crawl относится к другой ревизии или другому `unit_source_anchor`, разрешение идентичности выполняется заново.

### 7. Recovery при `QUARANTINED`

Для обязательного `QUARANTINED` unit:

```text
materialization не выполняется
cursor не продвигается
```

Snapshot не используется для долговременного replay. После HITL:

- `APPROVE` — новый crawl может обработать unit как `PASS`;
- `REJECT` — новый crawl обработает unit как `REJECTED`;
- `TIMEOUT` — применяется policy HITL/risk management.

Один quarantined unit блокирует только те scopes, для которых это предусмотрено Entity Policy. Если политика допускает независимую обработку других scopes, они могут продолжать обработку.

### 8. Dead-letter queue и escalation

Если ошибка не устраняется после retry budget или требует ручного вмешательства, создаётся DLX/escalation record.

В DLX попадают:

- неисправимые connector errors;
- неоднозначные source boundary;
- повторяющиеся materialization failures;
- невозможность сверки;
- ошибки cursor commit;
- ошибки, требующие ручного исправления конфигурации.

DLX record должен быть связан с:

```text
source_system
source_scope
snapshot_id, если snapshot был создан
ingestion_run_id
source_revision_id, если известен
retry_count
recovery_decision
```

Повторная попытка после исправления выполняется как новый crawl или явно разрешённая новая processing attempt. Старый snapshot не используется как replay source.

### 9. Cursor recovery

Cursor commit выполняется согласно завершению.

Cursor не продвигается при:

```text
technical failure
partial materialization
reconciliation_status = PARTIAL
reconciliation_status = FAILED
unresolved identity
обязательный QUARANTINED
completeness_status = PARTIAL
completeness_status = FAILED
```

Cursor может быть продвинут при:

```text
QUALITY_REJECTED
```

если:

- это терминальный quality result;
- source snapshot обработан полностью в рамках заявленного coverage;
- отсутствуют технические ошибки;
- quality result сохранён в ingestion metadata;
- завершение разрешает cursor commit.

Операция Cursor Commit защищается проверкой `cursor_before`. Ошибка конкурентного обновления не приводит к перезаписи более нового cursor.

### 10. Изоляция ошибок по source scope

Каждый source scope обрабатывается независимо, если это допускает `ingestion_run_id` и orchestration policy.

Сбой:

```text
Git repository A
```

не должен автоматически отменять обработку:

```text
Git repository B
Confluence space C
Jira project D
```

Если scopes образуют обязательную логическую группу, например для общего materialization run, группа может быть остановлена согласно Entity Policy. Это должно быть явно задано до запуска обработки.

### 11. Идемпотентность recovery

Повторное выполнение recovery decision не должно:

- создавать новый `entity_uid`;
- создавать новый `entity_version_id` для того же source state;
- создавать дублирующую `EntityRevisionObservation` для того же snapshot;
- продвигать cursor более одного раза;
- удалять manifest;
- создавать дублирующий DLX record для той же ошибки.

Новый crawl всегда создаёт новый `snapshot_id`, но не должен создавать новую логическую identity только из-за нового crawl.

### 12. Границы ответственности

Resilience отвечает за:

- классификацию технических ошибок;
- retry policy;
- recovery decision;
- DLX и escalation;
- восстановление cursor flow;
- повторное чтение источника после сбоя;
- координацию со сверкой и завершением;
- защиту от retry storm и повторного blind overwrite.

Resilience не отвечает за:

- правила контроля качества;
- назначение `entity_uid`;
- вычисление `content_hash`;
- вычисление `entity_version_id`;
- parsing и classification;
- разбиение на фрагменты;
- встраивание;
- извлечение графа;
- Qdrant или Neo4j write logic;
- semantic lineage;
- retention и hard delete.

## Инварианты

1. Технический failure не маскируется под `QUALITY_REJECTED`.
2. `QUALITY_REJECTED` является терминальным quality result текущего run.
3. После технического сбоя новый crawl получает новый `snapshot_id`.
4. Snapshot не используется как долговременный replay source.
5. Cursor не продвигается при техническом failure.
6. Cursor не продвигается при partial materialization.
7. Cursor не продвигается при неразрешённой identity.
8. Cursor не продвигается при обязательном `QUARANTINED`.
9. Cursor может быть продвинут после terminal `QUALITY_REJECTED`, если нет других ошибок и завершение разрешает commit.
10. Retry применяется только к retryable operations.
11. Исчерпание retry budget приводит к recovery decision, DLX или escalation.
12. Повторный crawl не создаёт новую логическую identity только из-за нового `snapshot_id`.
13. Recovery не изменяет `entity_uid`, `entity_version_id`, `content_hash` или `source_revision_id`.
14. Повторная обработка одного snapshot не создаёт новую `EntityRevisionObservation`.
15. Ошибка одного независимого source scope не останавливает остальные scopes без явной policy.
16. Конкурентный Cursor Commit не приводит к blind overwrite.
17. `quality_verdict`, `canonical_status`, `materialization_status`, `reconciliation_status` и `retry_status` имеют разную семантику.
18. Manifest и ingestion metadata сохраняются после удаления временного raw snapshot.
19. `tenant_id`, ACL metadata, Bamboo, `content_policy_version`, `content_version` и `crawl_mode` не используются.

## Новые переменные

| Переменная | Тип | Описание | Точка появления |
|---|---|---|---|
| `retry_count` | integer | Количество выполненных retry для технической операции | Retry Coordinator |
| `retry_status` | enum | Состояние retry: `NOT_STARTED`, `IN_PROGRESS`, `SUCCEEDED`, `EXHAUSTED` | Retry Coordinator |
| `recovery_decision` | enum | Решение после ошибки: `RETRY_OPERATION`, `NEW_CRAWL`, `DLX`, `ESCALATE`, `IGNORE` | Recovery Coordinator |
| `DLX` | storage/queue | Хранилище неисправимых или эскалированных ошибок | DLX Handler |
| `dlq_record` | структура | Запись об ошибке, переданной в DLX | DLX Handler |
| `retryable` | boolean | Признак допустимости повторной попытки операции | Retry Policy |
| `technical_failure` | структура/status | Результат технического сбоя, не являющегося quality rejection | Error Classifier |
| `source_boundary_conflict` | status | Невозможность безопасно подтвердить source boundary | Connector/Recovery Coordinator |
| `recovery_attempt_id` | `uuid` | Идентификатор recovery attempt для аудита и идемпотентности | Recovery Coordinator |

Существующие `snapshot_id`, `ingestion_run_id`, `source_scope`, `source_revision_id`, `cursor_before`, `cursor_after`, `cursor_status`, `quality_verdict`, `canonical_status`, `materialization_status`, `reconciliation_status`, `entity_uid`, `entity_version_id`, `content_hash` и `chunk_id` используются согласно предыдущим ADR и в этой ADR не переопределяются.

## Последствия

### Положительные

* Ошибки классифицируются отдельно от Quality Gate rejection.
* Новый crawl после сбоя согласуется с временным жизненным циклом snapshot.
* Cursor не продвигается до завершения обязательной обработки.
* Partial materialization не маскируется под успешную публикацию.
* Retry ограничен и не приводит к бесконечному циклу.
* DLX сохраняет неисправимые ошибки для последующего разбора.
* Recovery не изменяет identity и version semantics.
* Ошибка одного независимого source scope не блокирует остальные.
* Конкурентная обработка не должна перезаписывать более новый cursor.
* Повторный crawl не создаёт identity-дубликаты.

### Отрицательные

* При техническом сбое источник необходимо читать повторно.
* Между первоначальным и повторным crawl source state может измениться.
* Snapshot может временно удерживаться до принятия recovery decision.
* Частичный сбой может задержать cursor для всего source scope.
* Требуются retry coordinator, DLX и recovery policy.
* Ошибка cursor store может потребовать полного или reconciliation crawl.
* Новые технические статусы требуют отдельной обработки в monitoring и operations.

## Рассмотренные альтернативы

### Продвигать cursor сразу после получения snapshot

Cursor фиксируется сразу после успешного получения данных, до завершения downstream processing.

**Плюсы:**

* источник перечитывается реже;
* ниже задержка connector stage;
* downstream может работать независимо.

**Минусы:**

* неполная materialization может навсегда потерять изменения;
* Qdrant и Neo4j могут остаться рассогласованными;
* повторное чтение не гарантирует прежний source state;
* нарушается порядок завершения получения источников и завершения.

**Решение:** отклонено.

### Повторно обрабатывать тот же snapshot

Сохранять snapshot до успешного завершения всех retry и выполнять replay после сбоя.

**Плюсы:**

* повторная попытка использует тот же вход;
* проще восстановить partial materialization;
* не зависит от изменения источника между попытками.

**Минусы:**

* противоречит принятой политике временного snapshot;
* требует долговременного replay lifecycle;
* увеличивает storage и retention complexity;
* не используется в текущем проекте.

**Решение:** отклонено.

### Считать любой технический failure `QUALITY_REJECTED`

Продвигать cursor и завершать ingestion как quality rejection после любой ошибки.

**Плюсы:**

* простое состояние;
* не требуется отдельный recovery path;
* cursor не блокируется надолго.

**Минусы:**

* технический сбой ошибочно трактуется как плохие данные;
* изменения могут быть потеряны;
* невозможно отличить источник проблемы от качества content;
* Qdrant и Neo4j могут остаться неполными.

**Решение:** отклонено.

### Останавливать все source scopes при ошибке одного scope

Любая ошибка одного repository, project или space отменяет весь `ingestion_run_id`.

**Плюсы:**

* проще обеспечить групповую согласованность;
* проще диагностировать общий run;
* меньше вариантов частичного результата.

**Минусы:**

* независимые источники блокируют друг друга;
* снижается доступность;
* увеличивается задержка обработки;
* ошибка одного scope распространяется на весь ingestion.

**Решение:** отклонено для независимых scopes. Групповая остановка допускается только при явной Entity Policy.

### Не использовать DLX

Повторять обработку до успешного результата либо окончательно отбрасывать ошибку.

**Плюсы:**

* меньше компонентов;
* проще начальная реализация;
* не требуется ручной DLX workflow.

**Минусы:**

* возможен бесконечный retry loop;
* неисправимые сообщения не отделяются от временных;
* ошибки теряются или перегружают pipeline;
* отсутствует управляемый процесс ручного восстановления.

**Решение:** отклонено. Для неисправимых ошибок используется DLX или escalation.
