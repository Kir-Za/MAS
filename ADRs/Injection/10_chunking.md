# ADR Формирование Qdrant-представлений

**Статус:** Предложен

---

## Контекст и проблема

После прохождения Quality Gate в Chunker передаются processing units со статусом `quality_verdict=PASS`. Для units, участвующих в materialization, определены:

- `entity_uid`;
- `entity_version_id`;
- `content_hash`;
- `source_scope`;
- `source_revision_id`.

Chunker формирует Qdrant representations для последующего embedding и записи в Qdrant. Chunking не изменяет identity, content hash или temporal metadata.

Типы контента требуют разных правил обработки:

- narrative text можно разделять по структурным границам;
- code, tables, diagrams и Jira structured records нельзя произвольно разрезать;
- большие code entities могут не помещаться в допустимый размер одного embedding;
- повторная обработка одного состояния должна формировать те же Qdrant point IDs.

**Проблема:** как сформировать детерминированный набор Qdrant representations, сохраняющий структуру контента и identity сущностей, не допуская произвольного разрезания и конфликтов при повторной materialization.

---

## Факторы решения

* Сохранение `entity_uid`, `entity_version_id` и `content_hash`.
* Детерминированность `chunk_id`.
* Идемпотентность повторной обработки.
* Совместимость с tokenizer и `embedding_hard_limit`.
* Сохранение структуры tables, diagrams, code и Jira records.
* Отсутствие дублирования parent и child representations.
* Корректная работа в разных `source_scope`.
* Совместимость с временной моделью версий.
* Отделение Chunker от Qdrant Writer и Embedding Server.
* Возможность формирования current/historical Qdrant projections.
* Chunker формирует Qdrant representations, но не создаёт semantic relations и не назначает identity. `text` и `embed_text` используются только последующим embedding stage для semantic matching.

---

## Решение

Chunker получает только units, которые:

```text
quality_verdict = PASS
entity_uid определён
entity_version_id определён
content_hash определён
```

Chunker не создаёт processing units и не выполняет Quality Gate или Identity Resolution повторно.

### 1. Вход Chunker

```text
content_type
source_container
structured_content
source_system
snapshot_id
source_scope
source_object_id
source_revision_id
unit_source_anchor
parent_unit_id
entity_uid
entity_version_id
content_hash
quality_verdict
```

`parent_unit_id` используется только для навигации по in-memory IR и не входит в `chunk_id` или постоянный Qdrant payload.

Units со следующими состояниями не передаются в Chunker:

```text
quality_verdict = REJECTED
quality_verdict = QUARANTINED
entity_uid отсутствует
entity_version_id отсутствует
```

### 2. Processing Units и Qdrant Representations

Processing unit:

- создаётся Parser/IR;
- проходит Quality Gate;
- имеет собственный `quality_verdict`;
- может иметь собственные `entity_uid` и `entity_version_id`.

Qdrant representation:

- создаётся Chunker;
- не проходит Quality Gate повторно;
- не является новой processing unit;
- не получает новый `entity_uid`;
- наследует identity unit или parent entity согласно representation policy;
- получает собственные `chunk_anchor` и `chunk_id`.

Обычный Qdrant representation наследует identity unit:

```text
entity_uid
entity_version_id
content_hash
```

При structural fallback representation может относиться к parent entity, даже если её текст извлечён из проверенного child processing unit. Это исключение подробно определено в разделе 5.

### 3. Параметры Chunker

Используются следующие конфигурационные константы:

```text
target_chunk_size = 500 tokens
min_chunk_size = 100 tokens
max_chunk_size = 800 tokens
overlap_tokens = 50 tokens
embedding_hard_limit
```

`embedding_hard_limit` определяется embedding model/server policy. Chunker использует тот же tokenizer, что и Embedding Server, и учитывает резерв для special tokens.

Параметры:

- не входят в `entity_uid`;
- не входят в `content_hash`;
- не входят в `entity_version_id`;
- не входят в `chunk_id` напрямую.

Изменение параметров требует полного re-crawl и полного replacement текущих Qdrant representations.

### 4. Source scope и каноническая сериализация

`source_scope` используется для различения representations одинакового контента в разных scope:

```text
Git:
  source_scope = repository_id + branch_or_release_reference

Confluence:
  source_scope = space_key

Jira:
  source_scope = project_key
```

Для формирования `chunk_id` Chunker создаёт техническую переменную `canonical_source_scope`:

```text
Git:
  "git\0" + repository_id + "\0" + branch_or_release_reference

Confluence:
  "confluence\0" + space_key

Jira:
  "jira\0" + project_key
```

Все компоненты канонической строки:

1. нормализуются в NFC;
2. сериализуются в UTF-8;
3. экранируются перед конкатенацией;
4. соединяются в фиксированном порядке;
5. используют пустую строку для отсутствующего компонента.

`canonical_source_scope` не сохраняется в Qdrant payload.

### 5. Атомарность и structural decomposition

Атомарность означает, что Chunker не разрезает representation по произвольной token boundary.

Structural decomposition разрешена только по заранее выделенным Parser/IR структурным единицам и не меняет:

```text
entity_uid
entity_version_id
content_hash
```

#### Code

В обычном режиме:

```text
Code entity верхнего уровня,
укладывающаяся в max_chunk_size
→ один atomic Qdrant representation
```

Вложенные symbols входят в parent span, а отдельные child chunks не создаются.

Если class превышает embedding hard limit:

1. Parser/IR заранее выделяет class и method processing units.
2. Quality Gate проверяет все эти units.
3. Chunker не создаёт новые units.
4. Class representation не создаётся.
5. Каждый method unit со статусом `quality_verdict=PASS` может стать отдельной Qdrant representation.
6. Method representation наследует identity parent class:
   ```text
   entity_uid
   entity_version_id
   content_hash
   ```
7. `unit_source_anchor` метода используется только для локализации и citation.
8. Child method identity не публикуется через этот Qdrant representation.
9. Graph identity method обрабатывается независимо в Graph pipeline.

Fallback method representation является представлением parent class entity, а не materialization method entity в Qdrant.

#### Table

Обычная таблица является одним atomic representation.

Если таблица превышает embedding hard limit, допускается structural decomposition:

- строки обрабатываются в исходном порядке;
- headers повторяются в каждом representation;
- граница не проводится внутри `rowspan`/`colspan`;
- строки не повторяются между группами;
- каждая группа получает anchor вида:
  ```text
  canonical(unit_source_anchor) + ":rows=" + N + "-" + M
  ```
- `N` и `M` — индексы в canonical row order;
- row group наследует identity parent table;
- row group не является новой processing unit или domain entity.

#### Diagram

Диаграмма не режется произвольно.

Если исходная diagram representation превышает embedding hard limit, используется structural description, подготовленное на этапе парсинга.

#### Structured record

Jira issue является atomic representation, если помещается в `embedding_hard_limit`.

Если issue не помещается:

- допускается field-group representation;
- каждая группа получает anchor вида:
  ```text
  canonical(unit_source_anchor) + ":fields=" + canonical(group_name)
  ```
- field group наследует identity Jira issue;
- field group не является отдельной entity;
- rejected field group блокирует полную публикацию, если соответствующее поле входит в `canonical_content_payload`.

#### Binary attachment

Binary attachment не режется.

Если attachment превышает допустимый размер:

- Qdrant representation не создаётся;
- metadata attachment обрабатывается согласно Entity Policy;
- attachment не становится Neo4j domain entity автоматически;
- факт невозможности Qdrant materialization передаётся в далее.

### 6. Narrative chunking

`NARRATIVE_TEXT` разбивается по структурным границам.

Алгоритм:

1. Heading без собственного содержательного текста не образует отдельный chunk.
2. Heading включается в `section_hierarchy`.
3. Heading-only fragment присоединяется к следующему содержательному chunk.
4. Новый heading закрывает текущий chunk, если он содержит текст.
5. Paragraphs группируются до `target_chunk_size`.
6. Paragraph, превышающий `max_chunk_size`, разбивается по предложениям.
7. Предложение разбивается только по безопасным tokenizer boundaries.
8. Если предложение нельзя безопасно разбить, оно обрабатывается как oversized representation.
9. Overlap применяется только между narrative chunks.
10. В начало следующего chunk добавляются последние `overlap_tokens` токенов предыдущего chunk.
11. Overlap уменьшается до безопасной структурной границы, если полное значение невозможно.
12. Если overlap нарушает `max_chunk_size`, он детерминированно сокращается или не используется.
13. Overlap не пересекает границы code, table и diagram blocks.
14. Fragment меньше `min_chunk_size` присоединяется к следующему содержательному chunk, если это не нарушает `max_chunk_size`.
15. Если присоединение к следующему chunk невозможно, fragment присоединяется к предыдущему.
16. Если оба варианта невозможны, fragment сохраняется отдельно.
17. Последний chunk может быть меньше `min_chunk_size`.
18. Atomic units не объединяются с соседними units.

### 7. `chunk_anchor`

`chunk_anchor` — технический anchor конкретной Qdrant representation внутри unit.

Типовые формы:

```text
Atomic:
  canonical(unit_source_anchor) + ":atomic"

Narrative:
  canonical(unit_source_anchor)
  + ":section=" + canonical(structural_path)
  + ":fragment=" + canonical(local_fragment_index)

Structural child:
  canonical(child_unit_source_anchor) + ":structural"

Table row group:
  canonical(unit_source_anchor) + ":rows=" + N + "-" + M

Jira field group:
  canonical(unit_source_anchor) + ":fields=" + canonical(group_name)
```

Все компоненты anchor:

- нормализуются в NFC;
- сериализуются в UTF-8;
- экранируются;
- используют фиксированный порядок;
- не зависят от runtime object IDs или случайных UUID.

`structural_path` — канонический путь unit в Parsed IR, создаваемый Parser.

`local_fragment_index` — детерминированный порядковый номер fragment внутри конкретного `entity_version_id` и processing result. Он не гарантирует стабильность между разными версиями entity.

Перед вычислением `chunk_id` Chunker проверяет уникальность `chunk_anchor` в пределах:

```text
entity_uid
+ entity_version_id
+ canonical_source_scope
```

При коллизии Chunker не публикует chunk set для entity и передаёт ошибку в materialization/reconciliation pipeline.

### 8. `chunk_id`

`chunk_id` является UUIDv5 Qdrant representation.

```text
chunk_name =
  entity_uid + "\0"
  + entity_version_id + "\0"
  + canonical_source_scope + "\0"
  + chunk_anchor

chunk_id =
  UUIDv5(FIXED_CHUNK_NAMESPACE, chunk_name)
```

`FIXED_CHUNK_NAMESPACE` — постоянная константа проекта. Она:

- создаётся при первоначальной фиксации схемы Qdrant;
- не изменяется между deployment;
- не зависит от `snapshot_id`;
- не зависит от `source_revision_id`;
- не зависит от `chunk_index`.

`chunk_id` идентифицирует content-equivalent Qdrant representation в конкретном `source_scope`, а не отдельное observation.

### 9. `text` и embedding input

Chunker формирует:

```text
text
section_hierarchy
```

`text` — исходный текст representation, включая overlap, если overlap применён. После создания Chunk он не изменяется.

`embed_text` вычисляется на этапе embedding:

```text
embed_text =
  section_hierarchy.join(" > ") + "\n" + text
```

`embed_text`:

- не является полем Chunk;
- не сохраняется в Qdrant payload;
- не является LLM-based contextual enrichment;
- вычисляется Embedding stage.

Chunker не создаёт semantic relations и не назначает identity. `text` и `embed_text` используются только последующим embedding stage для:

- генерации векторов;
- semantic matching на этапе работы с графом  Semantic Relation Apply (после Qdrant materialization).

Если `embed_text` превышает `embedding_hard_limit`:

1. `section_hierarchy` сокращается детерминированно;
2. `text` не изменяется;
3. если лимит всё ещё превышен, применяется oversized policy;
4. embedding не создаётся из произвольной части `text` без нового chunk boundary;
5. Qdrant target обрабатывается согласно Entity Policy.

### 10. Выход Chunk

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
```

`parent_unit_id` не входит в постоянный Chunk, поскольку является временным IR identifier.

`hierarchy` — source-specific metadata. Для Confluence она содержит page hierarchy. Если hierarchy неприменима, значение отсутствует.

`section_hierarchy` содержит путь заголовков внутри текущей entity.

`is_atomic=true` означает, что конкретный Chunk не подлежит дальнейшему произвольному разбиению. Это не означает, что Chunk представляет всю parent entity.

`is_oversized=true` означает, что конкретный созданный Chunk потребовал oversized policy. Если Chunk не создан, `is_oversized` отсутствует, а причина фиксируется в ingestion metadata через `quality_reason`.

### 11. Current и historical representations

Chunker не принимает решение об удалении исторических Qdrant points.

Он формирует полный набор ожидаемых Chunk representations для переданного:

```text
entity_uid
+ source_scope
```

Исторические representations сохраняются для `as_of` retrieval.

Исключение устаревших representations из current projection, cleanup и безопасная замена chunk set выполняются Qdrant Writer и Reconciliation.

При изменении chunking parameters требуется:

```text
full re-crawl
→ формирование нового expected chunk set
→ replacement current projection
```

### 12. Инварианты

1. Chunker принимает только units с `quality_verdict=PASS` и разрешённой identity.
2. Chunker не создаёт processing units в обход Quality Gate.
3. Chunking не пересчитывает `entity_uid`, `entity_version_id` или `content_hash`.
4. Structural fallback использует уже проверенные child units либо технические representations, не являющиеся processing units.
5. Fallback representations явно наследуют identity parent entity.
6. `parent_unit_id` используется только внутри in-memory IR.
7. Atomic units не режутся по произвольным token boundaries.
8. `chunk_anchor` детерминирован и проверяется на коллизии.
9. `chunk_id` является UUIDv5 с фиксированной namespace-константой.
10. `snapshot_id` и `source_revision_id` не входят в `chunk_id`.
11. `text` не изменяется после создания Chunk.
12. `embed_text` создаётся только на этапе векторизации и не сохраняется в Chunk payload.
13. Размер embedding input проверяется по `embed_text`.
14. `is_atomic` означает неделимость Chunk, а не полноту entity.
15. `is_oversized` существует только у созданного Chunk.
16. Historical representations не удаляются Chunker автоматически.
17. Chunker формирует expected set, но не реализует cleanup и replacement.
18. Изменение Chunking parameters требует полного re-crawl.
19. Один и тот же source state при одинаковом scope и anchor формирует тот же `chunk_id`.
20. Qdrant Writer не должен публиковать смешанный старый и новый chunk set.
21. Chunker не создаёт semantic relations и не назначает identity.
22. `text` и `embed_text` используются только последующим embedding stage для генерации векторов и semantic matching.

---

## Новые переменные

| Переменная | Тип | Описание | Точка появления |
|---|---|---|---|
| `canonical_source_scope` | `string` | Каноническая сериализация `source_scope` для формирования `chunk_id` | Chunker |
| `chunk_anchor` | `string` | Детерминированный anchor Qdrant representation внутри processing unit | Chunker |
| `structural_path` | `string` | Канонический путь unit в Parsed IR | Parser |
| `local_fragment_index` | `integer` | Детерминированный номер narrative fragment внутри processing result | Chunker |
| `chunk_index` | `integer` | Порядковый номер Chunk внутри entity representation set | Chunker |
| `section_hierarchy` | `collection/string` | Путь заголовков внутри entity | Chunker |
| `is_atomic` | `boolean` | Признак запрета дальнейшего произвольного разбиения Chunk | Chunker |
| `is_oversized` | `boolean` | Признак того, что созданный Chunk требует oversized policy | Chunker |
| `FIXED_CHUNK_NAMESPACE` | `constant` | Неизменяемый namespace для UUIDv5 chunk IDs | Проектная конфигурация до первой materialization |


---

## Последствия

### Положительные

* Qdrant representations не смешиваются с processing units.
* Identity parent entity сохраняется при structural fallback.
* Child units проходят Quality Gate до использования Chunker.
* Таблицы, диаграммы, code и Jira records не разрезаются произвольно.
* `chunk_id` детерминирован и не зависит от временного `snapshot_id`.
* Повторный crawl одного content-equivalent состояния поддерживает идемпотентный Qdrant upsert.
* `text` сохраняется неизменным для цитирования и context assembly.
* `embed_text` отделён от постоянного payload и вычисляется на этапе embedding.
* Historical и current Qdrant representations разделены концептуально.
* Chunker не создаёт semantic relations и не назначает identity, сохраняя чёткое разделение ответственности.

### Отрицательные

* Parser должен заранее выделять child units для oversized code.
* Quality Gate проверяет methods, которые в обычном режиме могут не иметь отдельных Qdrant representations.
* Большие atomic units могут не получить Qdrant representation при отсутствии безопасного fallback.
* Structural fallback требует дополнительных правил identity inheritance и citation.
* Изменение chunking parameters требует полного re-crawl и replacement current projection.
* Требуется поддерживать отдельные current и historical Qdrant projections или эквивалентные filtering/cleanup-механизмы.

---

## Рассмотренные альтернативы

### Фиксированный размер для всех content types

Все source units разбиваются на chunks одинакового размера.

**Плюсы:**

* простая реализация;
* единая конфигурация;
* предсказуемое число chunks.

**Минусы:**

* таблицы и диаграммы могут быть разрезаны в смысловых границах;
* code symbols могут стать нечитаемыми;
* Jira structured records могут потерять связанность полей;
* ухудшается качество retrieval и Graph Extraction.

### Один полный parent chunk вместе с child chunks

Для class/page/table одновременно создаётся полный parent chunk и отдельные chunks дочерних units.

**Плюсы:**

* сохраняется общий контекст parent entity;
* можно искать как общую сущность, так и отдельные fragments;
* oversized parent может дополняться дочерними representations.

**Минусы:**

* дублируется содержимое и embedding;
* ухудшается ranking;
* возрастает стоимость хранения и поиска;
* усложняется deduplication и context assembly.

### Использовать только child chunks для всех code entities

Даже небольшие классы и модули всегда представляются через методы и функции.

**Плюсы:**

* меньше oversized problems;
* точечный поиск по методам;
* меньший размер отдельных embeddings.

**Минусы:**

* теряется контекст класса и module-level declaration;
* отдельный метод может быть непонятен без parent information;
* усложняется citation и reconstruction;
* representation класса не соответствует его полному content state.

### Считать `chunk_index` единственным идентификатором

`chunk_id` формируется только из entity identity и порядкового номера.

**Плюсы:**

* минимальная формула;
* простая генерация;
* удобный порядок chunks.

**Минусы:**

* порядок fragments может меняться при локальной структурной правке;
* недостаточно различаются source scopes;
* возможны stale points и ошибочные upsert;
* не отражается structural identity fragment.

### Включать `snapshot_id` в `chunk_id`

Каждый новый crawl создаёт новые Qdrant point IDs.

**Плюсы:**

* простое различение processing runs;
* исключается перезапись point из разных snapshots;
* удобна диагностика отдельных crawl.

**Минусы:**

* повторный crawl создаёт дубликаты;
* content deduplication не распространяется на Qdrant;
* растёт объём индекса;
* snapshot является временным и не должен определять постоянную representation identity.

### Разрешить произвольное усечение `text` при превышении embedding limit

Если `embed_text` слишком велик, передавать в embedding только его часть.

**Плюсы:**

* representation почти всегда создаётся;
* меньше rejected oversized units;
* проще embedding pipeline.

**Минусы:**

* embedding не представляет полный `text`;
* ухудшается объяснимость retrieval;
* пользовательская цитата может не соответствовать embedded content;
* возможна потеря критически важной части кода, таблицы или документа.

### Выполнять cleanup и replacement внутри Chunker

Chunker сам удаляет старые Qdrant points и переключает current projection.

**Плюсы:**

* единая логика формирования и удаления;
* меньше межкомпонентных контрактов;
* потенциально проще локальный pipeline.

**Минусы:**

* Chunker начинает отвечать за storage consistency;
* сложнее обрабатывать partial writes;
* нарушается разделение ответственности с Qdrant Writer и Reconciliation;
* повышается риск удалить historical representations;
* Chunker становится зависимым от состояния Qdrant.

### Не использовать oversized fallback

Большие atomic units не индексируются в Qdrant.

**Плюсы:**

* простая и безопасная политика;
* отсутствует риск искажения representation;
* не требуется child decomposition.

**Минусы:**

* крупные классы, таблицы и Jira issues становятся недоступны для vector retrieval;
* снижается recall;
* часть domain context доступна только через Graph или source metadata.
