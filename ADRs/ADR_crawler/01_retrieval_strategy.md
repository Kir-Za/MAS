# ADR Стратегия retrieval и объединение результатов

**Статус:** Предложен

---

## Контекст и проблема

Qdrant и Neo4j являются двумя проекциями общего домена:

- Qdrant выполняет dense+sparse поиск по `Chunk`;
- Neo4j хранит логические сущности, отношения, provenance и temporal context.

Оба хранилища используют общие:

```text
entity_uid
entity_version_id
source_scope
source_revision_id
```

но обновляются независимо и могут временно содержать разные состояния данных. Qdrant может содержать `Chunk`, для которого ещё отсутствуют соответствующие graph relations, а Neo4j может содержать entity, для которой Qdrant representation ещё не создана.

Проект работает для одной группы разработки одного продукта. `tenant_id` не используется. ACL проверяется на уровне интерфейса и не входит в retrieval metadata или hard-filter данного ADR.

Запросы различаются по характеру:

- обычные QA-запросы требуют семантического и лексического поиска;
- запросы с явно указанными сущностями требуют graph lookup и enrichment;
- relational и multi-hop запросы требуют обхода Neo4j;
- неопределённые запросы должны использовать оба источника;
- исторические запросы требуют выбора версии по `version_context` и temporal semantics.

**Проблема:** как выбирать источник и порядок retrieval, объединять результаты Qdrant и Neo4j и обрабатывать временную рассогласованность без потери recall и без выдачи несовместимых версий данных.

---

## Факторы решения

* Recall для обычных, relational и multi-hop запросов.
* Корректное применение `source_scope` и `version_context`.
* Отсутствие прямого сравнения разнородных scores.
* Сохранение provenance и `entity_uid`/`entity_version_id`.
* Работа при partial или stale materialization.
* Ограниченная latency и нагрузка на Qdrant/Neo4j.
* Единая retrieval-логика для Planner и других потребителей.
* Отсутствие зависимости от полной консистентности двух хранилищ.
* Отсутствие cross-tenant логики, так как в проекте нет tenant-модели.

---

## Решение

Мы принимаем адаптивный hybrid retrieval с тремя режимами:

```text
QDRANT_PRIMARY
GRAPH_PRIMARY
PARALLEL
```

Режим выбирается Retrieval Service на основании `retrieval_mode`, переданного Planner, либо результата классификации запроса.

### 1. Граница ответственности

Planner отвечает за:

- формирование пользовательского запроса;
- выбор или запрос retrieval mode;
- декомпозицию multi-hop задачи;
- порядок последующих hop-ов;
- объединение результатов разных retrieval-вызовов на уровне задачи.

Retrieval Service отвечает за:

- выполнение одного retrieval plan;
- поиск в Qdrant и/или Neo4j;
- применение version/scope constraints;
- enrichment;
- fusion результатов в рамках одного вызова;
- возврат provenance и consistency metadata.

Retrieval Service не отвечает за:

- создание новых сущностей;
- изменение `entity_uid`;
- изменение `entity_version_id`;
- ingestion;
- Graph Extraction;
- бесконечный traversal;
- самостоятельную многошаговую декомпозицию запроса.

### 2. Выбор режима

#### `QDRANT_PRIMARY`

Используется для:

- обычных QA-запросов;
- narrative/free-text запросов;
- запросов без надёжно определённых entity anchors.

Qdrant выполняет hybrid dense+sparse поиск по `text`/embedding-представлениям. Neo4j может быть вызван после поиска для enrichment найденных `entity_uid`.

Graph enrichment не исключает результаты Qdrant и не изменяет их порядок, если запрос не содержит отдельного graph-priority правила.

#### `GRAPH_PRIMARY`

Используется для:

- явного поиска сущности;
- запросов по зависимостям;
- graph-only запросов;
- relational/multi-hop запросов, для которых структура важнее семантического сходства.

Neo4j выполняет entity lookup и ограниченный traversal. Найденные `entity_uid` используются для:

- получения связанных chunks из Qdrant;
- формирования допустимого source/version scope;
- enrichment provenance и relations.

Если Qdrant не содержит соответствующий Chunk, entity может быть возвращена как metadata/candidate с `content_pending`, но не как текстовый контекст для LLM.

#### `PARALLEL`

Используется, если:

- intent не определён;
- classifier confidence ниже установленного порога;
- запрос одновременно содержит semantic и relational признаки;
- запрос признан критичным policy.

Qdrant и Neo4j выполняются параллельно. Результаты объединяются Retrieval Service.

Если классификатор недоступен, Retrieval Service использует `PARALLEL`, а при невозможности его выполнить — `QDRANT_PRIMARY`.

### 3. Query classification

Классификация выполняется быстрым детерминированным classifier по:

- наличию известных identifiers;
- маркерам отношений;
- маркерам traversal;
- explicit graph-only формулировкам;
- переданному `version_context`.

LLM-классификация допускается только для неопределённых случаев и не является обязательной частью каждого запроса.

Если confidence ниже настроенного порога, выбирается `PARALLEL`. Порог является конфигурацией retrieval и не входит в identity или content metadata.

### 4. Fusion

В `PARALLEL` Retrieval Service использует Reciprocal Rank Fusion.

RRF применяется к позициям результатов, а не к исходным абсолютным scores:

```text
RRF_score(d) =
  Σ 1 / (k + rank_i(d))
```

где `k` — фиксированная retrieval-константа проекта.

Исходные semantic и graph scores напрямую не сравниваются.

Для `GRAPH_PRIMARY` применяется детерминированный graph priority:

- результаты, непосредственно подтверждённые graph lookup/traversal, размещаются выше;
- Qdrant candidates не исключаются, если не нарушают обязательные scope/version constraints;
- при равном приоритете используется исходный порядок источника.

Fusion удаляет дубликаты на уровне результата по:

```text
entity_uid
+ entity_version_id
+ chunk_id, если он присутствует
```

Отдельные результаты с разным `entity_version_id` не объединяются в один content item.

### 5. Scope и version context

Retrieval применяет `version_context`:

```text
version_context:
  source_revision_id: optional
  branch | release_tag: optional
  source_scope: optional
  as_of: optional timestamp
```

Приоритет ограничений:

1. точный `source_revision_id`;
2. явно заданные `branch`/`release_tag`;
3. явно заданный `source_scope`;
4. `as_of`;
5. default scope.

Конфликтующие параметры приводят к:

```text
INVALID_VERSION_CONTEXT
```

без silent fallback.

Если контекст не задан:

- для Git используется canonical `master`;
- для release-запроса используется соответствующий явно заданный release scope;
- для Confluence используется последняя доступная версия в допустимом space;
- для Jira используется последнее доступное состояние issue.

ACL проверяется интерфейсом до или в момент выдачи результата и не является полем retrieval payload в рамках данного ADR.

### 6. Cross-store join

Связь между Qdrant и Neo4j выполняется по:

```text
entity_uid
```

Для version-sensitive retrieval дополнительно проверяются:

```text
entity_version_id
source_scope
source_revision_id
as_of
```

`content_hash` используется для идентификации версии содержимого, но не является основным cross-store join key.

Qdrant `chunk_id` используется для идентификации конкретного Qdrant representation и не является идентификатором Neo4j entity.

### 7. Eventual consistency

Retrieval не ожидает полного статуса ingestion или reconciliation.

Если Graph содержит entity, но Qdrant representation отсутствует:

```text
content_pending = true
```

Entity возвращается как metadata/candidate и не используется как текстовый контекст.

Если Qdrant содержит Chunk, но Neo4j ещё не содержит соответствующую relation:

```text
graph_incomplete = true
```

Chunk может быть возвращён, но отсутствие graph relation не трактуется как отсутствие отношения.

Если source revision между проекциями несовместима:

```text
consistency_status = degraded
```

Такой результат может быть возвращён только с provenance и явной consistency metadata. Для операций, требующих полной версиионной согласованности, Retrieval Service должен исключить несовместимый результат.

### 8. Деградация

При недоступности или timeout одного источника:

- Neo4j недоступен:
  - выполняется Qdrant-only retrieval;
  - результат содержит `graph_unavailable=true`;
- Qdrant недоступен:
  - выполняется Graph-only retrieval;
  - entity и relations могут быть возвращены как metadata;
  - текстовый контекст возвращается только при наличии доступной representation;
  - результат содержит `qdrant_unavailable=true`;
- оба источника недоступны:
  - возвращается retrieval failure;
  - LLM не получает пустой или неподтверждённый контекст как успешный результат.

Если одна ветка превысила latency budget, Retrieval Service завершает запрос по доступным результатам и маркирует его как degraded/partial согласно retrieval result contract.

### 9. Multi-hop

Planner выполняет multi-hop orchestration.

Для каждого hop Retrieval Service предоставляет атомарные операции:

```text
entity lookup
related entities lookup
bounded graph traversal
chunks by entity_uid
semantic search in source_scope
```

Каждый hop возвращает:

- найденные identifiers;
- source/version context;
- provenance;
- ограничения для следующего вызова;
- consistency metadata.

Retrieval Service не продолжает traversal бесконечно и не создаёт новые hop-ы без retrieval plan от Planner.

### 10. Результат retrieval

Каждый результат должен содержать:

```text
retrieval_result:
  source_system
  source_scope
  source_revision_id
  entity_uid
  entity_version_id
  chunk_id, если найден
  content_hash, если доступен
  provenance
  consistency_status
  content_pending
  graph_incomplete
  source_unavailable
```

Технические flags имеют значение только для конкретного retrieval result и не изменяют identity, temporal или ingestion status.

## Инварианты

1. Qdrant и Neo4j являются проекциями общего домена, а не независимыми моделями identity.
2. Retrieval не изменяет `entity_uid`, `entity_version_id` или `content_hash`.
3. Обычный QA по умолчанию использует `QDRANT_PRIMARY`.
4. Явно структурный запрос может использовать `GRAPH_PRIMARY`.
5. Неопределённый запрос использует `PARALLEL`.
6. Исходные scores Qdrant и Neo4j напрямую не сравниваются.
7. Fusion выполняется Retrieval Service.
8. `source_revision_id`, `entity_version_id` и `source_scope` проверяются для version-sensitive retrieval.
9. `content_hash` не является основным cross-store join key.
10. Отсутствие relation в Neo4j не доказывает отсутствие отношения.
11. Entity без Qdrant content не передаётся LLM как текстовый контекст.
12. Недоступность одного источника приводит к явной деградации, а не к молчаливой ошибке.
13. Конфликтующий `version_context` приводит к `INVALID_VERSION_CONTEXT`.
14. Planner оркестрирует multi-hop, Retrieval Service выполняет атомарные операции.
15. Retrieval Service не выполняет бесконечный traversal.
16. `tenant_id` и ACL metadata не используются.
17. `snapshot_id` не является retrieval identity и не используется как cross-store join key.
18. Исторические и актуальные версии не объединяются в один результат.
19. Один результат не может одновременно ссылаться на несовместимые source revisions.
20. Retrieval не блокируется ожиданием полного ingestion `COMPLETED`.

---

## Новые переменные

| Переменная | Тип | Описание | Точка появления |
|---|---|---|---|
| `retrieval_mode` | enum | Режим retrieval: `QDRANT_PRIMARY`, `GRAPH_PRIMARY`, `PARALLEL` | Planner или Retrieval Service |
| `query_intent` | enum | Классификация запроса для выбора retrieval mode | Query Classifier |
| `classifier_confidence` | float | Уверенность классификатора в диапазоне $[0,1]$ | Query Classifier |
| `k` | integer | Параметр RRF для fusion | Retrieval configuration |
| `retrieval_result` | структура | Результат одного retrieval operation | Retrieval Service |
| `consistency_status` | enum | Состояние согласованности результата: `consistent` или `degraded` | Retrieval Service |
| `content_pending` | boolean | Graph entity найдена, но соответствующий Qdrant content ещё отсутствует | Retrieval Service |
| `graph_incomplete` | boolean | Chunk найден, но соответствующая graph information ещё отсутствует | Retrieval Service |
| `graph_unavailable` | boolean | Neo4j недоступен или не ответил в пределах запроса | Retrieval Service |
| `qdrant_unavailable` | boolean | Qdrant недоступен или не ответил в пределах запроса | Retrieval Service |
| `source_unavailable` | boolean | Источник результата недоступен при выполнении retrieval | Retrieval Service |
| `INVALID_VERSION_CONTEXT` | status | Ошибка несовместимого или противоречивого version context | Retrieval Service |

`entity_uid`, `entity_version_id`, `content_hash`, `chunk_id`, `source_scope`, `source_revision_id`, `version_context`, `provenance` и `snapshot_id` используются согласно ранее принятым ADR и в этой ADR не переопределяются.

---

## Последствия

### Положительные

* Обычные QA-запросы получают быстрый Qdrant-first retrieval.
* Структурные и graph-only запросы не ограничиваются top-k Qdrant.
* Неопределённые запросы используют оба источника.
* RRF устраняет необходимость прямой калибровки semantic и graph scores.
* Неполнота Neo4j не приводит к молчаливой потере Qdrant-контекста.
* Неполнота Qdrant не маскируется под полноценный текстовый контекст.
* Version-aware retrieval снижает риск объединения несовместимых состояний.
* Multi-hop orchestration отделена от низкоуровневых retrieval operations.
* Временная недоступность одного источника приводит к контролируемой деградации.
* Единый Retrieval Service предотвращает расхождение retrieval-логики между потребителями.

### Отрицательные

* Адаптивный routing сложнее безусловного Qdrant-first.
* Parallel mode увеличивает нагрузку на оба хранилища.
* Требуется отдельный query classifier и evaluation его ошибок.
* Retrieval должен обрабатывать degraded results и consistency flags.
* Графовые запросы зависят от качества entity resolution и полноты Neo4j.
* Исторический и version-aware retrieval сложнее обычного semantic search.
* Multi-hop запросы требуют координации Planner и Retrieval Service.
* Новые result flags должны быть учтены context bundle и downstream agent logic.

---

## Рассмотренные альтернативы

### Безусловный Qdrant-first retrieval

Любой запрос сначала выполняется в Qdrant, после чего найденные entities при необходимости обогащаются через Neo4j.

**Плюсы:**

* простая маршрутизация;
* предсказуемая latency;
* низкая нагрузка на Neo4j;
* хорошо подходит для обычного QA;
* проще эксплуатация и тестирование.

**Минусы:**

* теряются graph-only запросы;
* multi-hop ограничен результатами top-k Qdrant;
* entity, не найденная семантически, не получает graph-driven context;
* хуже поддерживаются запросы по зависимостям и отношениям.

**Решение:** отклонено как универсальная стратегия. Используется как основной режим обычного QA и fallback.

### Всегда параллельный поиск

Каждый запрос выполняется одновременно в Qdrant и Neo4j.

**Плюсы:**

* не требуется intent routing;
* минимизируется риск ошибочного выбора режима;
* естественная поддержка fusion и graceful degradation.

**Минусы:**

* постоянная нагрузка на оба хранилища;
* выше latency и стоимость;
* сложнее capacity planning;
* graph traversal выполняется даже для запросов без структурного смысла.

**Решение:** отклонено как режим по умолчанию. Используется для неопределённых и критичных запросов.

### Graph-first retrieval

Сначала выполняется поиск и traversal в Neo4j, затем по найденным entities подтягиваются chunks из Qdrant.

**Плюсы:**

* хорошо подходит для relational и multi-hop запросов;
* поддерживает graph-only discovery;
* позволяет искать по структуре независимо от лексического сходства.

**Минусы:**

* зависит от полноты графа;
* плохо работает с narrative/free-text запросами;
* может иметь высокую latency;
* ошибка entity resolution ограничивает recall;
* неполный граф может молча исключить релевантный контент.

**Решение:** отклонено как универсальная стратегия. Используется для явно структурных запросов.

### Score-based fusion

Объединять абсолютные scores Qdrant и Neo4j через взвешенную формулу.

**Плюсы:**

* можно учитывать силу сигнала каждого источника;
* потенциально выше качество после калибровки.

**Минусы:**

* scores имеют разную шкалу и семантику;
* требуется калибровка по datasets;
* параметры нестабильны при изменении моделей и графа;
* сложнее объяснять порядок результатов.

**Решение:** отклонено для базовой реализации. Используется rank-based RRF.

### Раздельные retrieval API для каждого потребителя

Каждый агент самостоятельно решает, обращаться ли к Qdrant или Neo4j и как объединять результаты.

**Плюсы:**

* максимальная гибкость агентов;
* независимое развитие сценариев;
* простые API отдельных хранилищ.

**Минусы:**

* дублирование routing, filtering, fusion и consistency logic;
* разные агенты могут получить разные результаты;
* сложнее безопасность, наблюдаемость и тестирование;
* version-aware join реализуется неодинаково.

**Решение:** отклонено для общего production-пути. Низкоуровневые API допускаются внутри Retrieval Service.

### Строгий consistency gate

Не возвращать данные, пока Qdrant и Neo4j не имеют статус полностью согласованной materialization.

**Плюсы:**

* минимальный риск выдачи неполного контекста;
* проще объяснять результат;
* меньше degraded states.

**Минусы:**

* ухудшается доступность;
* растёт latency;
* свежие данные недоступны до завершения reconciliation;
* временный сбой одного хранилища блокирует весь retrieval.

**Решение:** отклонено для общего retrieval. Используется только для сценариев, которым явно требуется строгая version consistency.
