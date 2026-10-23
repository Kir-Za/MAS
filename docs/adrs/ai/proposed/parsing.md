# ADR Парсинг и нормализация источников

**Статус:** Предложен

---

## Контекст и проблема

После классификации source records из Git, Jira и Confluence классифицированы по `source_container`, `content_type` и `processing_status`. До контроля качества и разрешения идентичности их необходимо преобразовать в детерминированное представление.

Параллельно формируются два связанных артефакта:

- `structured_content` — типизированное представление для контроля качества, разбиения на фрагменты и извлечения графа;
- `canonical_content_payload` — нормализованное source-derived представление для вычисления `content_hash`.

Оба артефакта строятся из одного source content и должны быть согласованы. Парсинг не должен зависеть от Qdrant или Neo4j и не должен назначать identity.

Технические различия источника и формата — CRLF/LF, BOM, кодировки, порядок ключей structured records и незначащие пробелы — не должны создавать ложные изменения содержимого. При этом нормализация не должна менять смысл кода, таблиц, диаграмм, Jira records или Confluence structure.

`content_hash` вычисляется до контроля качества и разрешения идентичности. 

**Проблема:** как детерминированно преобразовать Classified Content Unit в структурированный IR и source-derived canonical payload, чтобы downstream-компоненты использовали единый результат, а `content_hash` не зависел от случайностей формата или успешности вторичного extraction.

---

## Факторы решения

* Детерминированность `content_hash`.
* Согласованность входа для контроля качества, разбиения на фрагменты и извлечения графа.
* Вычисление `content_hash` до разрешения идентичности.
* Сохранение структуры кода, таблиц, диаграмм и Confluence pages.
* Корректная canonical serialization Jira records.
* Отделение source content от semantic inference.
* Корректная обработка недоступного и частично разобранного контента.
* Отсутствие влияния разбиения на фрагменты, встраивания и извлечения графа на identity.
* Совместимость с временным снимком.
* Отсутствие `content_policy_version` и `content_version`; изменение правил требует полного повторного обхода.
* Наличие детерминированного текстового представления для cross-source semantic matching на этапе работы с графом.

---

## Решение

Мы выполняем парсинг и нормализацию после классификации формируя два артефакта - один для нормализации, другой для оценки качества. 

### 1. Parsed IR

Typed Parser формирует `ParsedIR` для processing unit:

```text
ParsedIR:
  source_system
  snapshot_id
  source_scope
  source_object_id
  source_revision_id
  unit_source_anchor
  parent_unit_id
  content_type
  source_container
  structured_content
  processing_status
  raw_payload_checksum
```

`parent_unit_id` является временным идентификатором IR-дерева и используется только во время обработки текущего снимка. Он не является identity и не сохраняется в постоянных materialized records.

`unit_source_anchor` локализует unit внутри `source_object_id`. Он не заменяет `entity_uid`, `entity_version_id` или `chunk_id`.

### 2. `structured_content`

`structured_content` — типизированное представление unit, используемое:

- контроль качества;
- разбиение на фрагменты;
- извлечение графа.

Парсер не должен повторно разбирать тот же source content независимо для разных downstream-компонентов.

Минимальная структура зависит от `content_type`.

#### `NARRATIVE_TEXT`

Сохраняются:

- headings и их порядок;
- paragraphs;
- списки и вложенность;
- ссылки;
- inline formatting;
- дочерние units, если они есть.

#### `CODE`

Сохраняются:

- `full_symbol_span_text`;
- `module_path`;
- `qualified_symbol`;
- `symbol_kind`;
- `signature_fingerprint`;
- `declared_interfaces_or_base_types`;
- source boundaries symbol.

LSP и framework extractors используются для структурного разбора. Семантические relations, callers/callees и inferred types не являются частью canonical content payload.

#### `TABLE`

Сохраняются:

- headers;
- rows;
- cell boundaries;
- `rowspan` и `colspan`;
- alignment;
- исходный порядок строк и ячеек.

Таблица не преобразуется в неструктурированный narrative text.

#### `DIAGRAM`

Сохраняются:

- исходное DSL/XML или другое source representation;
- тип диаграммы;
- labels;
- nodes;
- edges, если они извлечены.

Отсутствие edges не делает диаграмму автоматически невалидной.

#### `STRUCTURED_RECORD`

Для Jira сохраняются:

- отслеживаемые поля;
- их source values;
- source order только там, где он семантически значим;
- даты и другие structured values в canonical form.

Changelog является source revision input, но не обычным content unit.

#### `BINARY_ATTACHMENT`

Сохраняются:

- MIME;
- filename;
- byte size;
- raw payload checksum;
- извлечённые дочерние units, если extraction выполнен.

К raw bytes не применяются правила текстовой нормализации.

### 3. `canonical_content_payload`

Entity Normalizer формирует `canonical_content_payload` из source content после детерминированной нормализации.

Payload строится на уровне logical entity, а не произвольного отдельного chunk. Границы entity определяются по source-specific правилами:

- Git code symbol — отдельная code entity;
- Confluence page — page entity с дочерними units;
- Jira issue — issue entity с отслеживаемыми полями.

`canonical_content_payload` содержит только значения, выведенные непосредственно из source content и его детерминированной структуры. Он не содержит:

- semantic inference;
- callers/callees;
- graph relations;
- embedding;
- chunk metadata;
- результаты контроля качества;
- `entity_uid`;
- `entity_version_id`.

Для code entity в payload входят:

```text
full_symbol_span_text
module_path
declared_interfaces_or_base_types
```

Сигнатура входит в `full_symbol_span_text`. LSP-derived relations и типы, выведенные из контекста, не входят.

### 4. Детерминированная нормализация

Допускаются:

- корректное декодирование в UTF-8;
- удаление BOM;
- преобразование `CRLF` и `CR` в `LF`;
- удаление trailing whitespace там, где оно незначимо;
- canonical serialization structured data;
- удаление служебных полей, исключённых source-specific content policy.

Запрещаются:

- `errors="ignore"` и `errors="replace"` для текстовых данных;
- безусловная замена табов на фиксированное число пробелов;
- глобальное схлопывание whitespace;
- произвольное изменение порядка элементов;
- удаление markup без source-specific правила;
- включение результатов semantic extraction в `canonical_content_payload`.

Для языков со значимыми отступами нормализация не должна менять синтаксический смысл. Если безопасная эквивалентность не подтверждена, используется исходное source representation без опасной трансформации.

### 5. Вычисление `content_hash`

После формирования `canonical_content_payload` вычисляется:

```text
content_hash = hash(canonical_content_payload)
```

`content_hash`:

- вычисляется до контроля качества;
- вычисляется до разрешения идентичности;
- не зависит от `entity_uid`;
- не зависит от `entity_version_id`;
- не зависит от разбиения на фрагменты и встраивания;
- не зависит от извлечения графа;
- используется разрешением идентичности как свидетельство;
- используется для формирования `entity_version_id`.

Если source content доступен, `content_hash` может быть вычислен даже при неуспешном структурном extraction. Это не означает успешную обработку или право на materialization.

Если source content физически недоступен, `content_hash` не вычисляется.

### 6. Правила по источникам

#### Git

Git file является source container и может содержать несколько code units.

Для code units:

- LSP/framework extractor выделяет symbols;
- каждый symbol получает свой `unit_source_anchor`;
- `full_symbol_span_text` включает полный span symbol;
- вложенные symbols обрабатываются согласно code policy;
- module path и declaration-level attributes нормализуются детерминированно.

Если LSP или другой обязательный extractor не может обработать code unit:

- source content может быть сохранён для hash, если доступен;
- `processing_status` получает `FAILED` или `UNSUPPORTED`;
- контроль качества определяет дальнейшую судьбу unit;
- identity не назначается до прохождения контроля качества.

#### Confluence

Confluence page может содержать:

- narrative text;
- tables;
- diagrams;
- code blocks/macros;
- attachments.

Page-level `canonical_content_payload` строится в детерминированном порядке source units.

В page payload входят:

- нормализованный текст и структура;
- tables;
- macros;
- links и anchors;
- diagrams;
- attachments.

Если attachment входит в page-level content policy и физически недоступен, полный page-level `content_hash` не вычисляется. Если attachment не относится к page-level payload, он обрабатывается как отдельный linked unit согласно Entity Policy.

Перемещение страницы внутри space при сохранении `page_id` не меняет source identity, но изменение hierarchy может изменить content hash, если hierarchy входит в page payload.

#### Jira

Jira issue представляется как `STRUCTURED_RECORD`.

`canonical_content_payload` строится из отслеживаемых policy-полей:

- object keys сериализуются в лексикографическом порядке;
- даты приводятся к UTC ISO-8601;
- `null` и отсутствующее поле различаются;
- порядок семантически упорядоченных массивов сохраняется;
- changelog не включается в payload автоматически.

Один changelog batch является source revision input. Изменение отслеживаемых полей создаёт новое content-equivalent состояние; изменение поля вне policy не создаёт новую `entity_version_id`.

### 7. Ошибки парсинга и extraction

Нужно различать доступность source content и успешность структурного extraction.

#### Source content доступен, extraction завершился ошибкой

```text
canonical_content_payload → формируется из доступного source representation
content_hash              → может быть вычислен
processing_status         → FAILED
контроль качества         → определяет quality_verdict
```

Такой unit не считается автоматически пригодным для разбиения на фрагменты или извлечения графа и может требовать HITL.

#### Source content физически недоступен

```text
content_hash       → не вычисляется
processing_status  → FAILED
контроль качества  → REJECTED
identity           → не назначается
```

#### Частично разобранный контейнер

Если обязательная структура потеряна, unit получает `FAILED` и передаётся контролю качества. Если отдельный дочерний unit не разобран, это не должно автоматически уничтожать доступную часть entity; итог определяется Entity Policy.

### 8. Взаимодействие с контролем качества

Этап формирует:

```text
structured_content
canonical_content_payload
content_hash
```

Контроль качества использует их для quality validation:

- не пересчитывает `content_hash`;
- не назначает `entity_uid`;
- не назначает `entity_version_id`;
- не изменяет Parsed IR source content;
- определяет `quality_verdict`.

Изменение `quality_verdict` не меняет `content_hash`.

### 9. Хранение артефактов

`structured_content` и `canonical_content_payload` используются в памяти в рамках текущей обработки снимка.

После завершения обработки:

- `canonical_content_payload` может быть удалён;
- `structured_content` может быть удалён;
- `raw source payload` удаляется согласно циклу снимка;
- `content_hash` сохраняется в observations и materialized records;
- `raw_payload_checksum` сохраняется как fingerprint;
- classification, quality и provenance metadata сохраняются согласно соответствующим ADR.

Для `FAILED` и `UNSUPPORTED` units результат обработки сохраняется в ingestion metadata согласно.

### 10. Source-derived text representation для cross-source matching

Для поддержки semantic relation discovery на этапе работы с графом, Parser формирует детерминированное текстовое представление для каждого processing unit, предназначенное для cross-source сравнения.

**Для `NARRATIVE_TEXT` (Confluence):**

```text
cross_source_text =
  заголовки + "\n" + абзацы + "\n" + списки + "\n" + ссылки
```

Порядок элементов сохраняется. Текст нормализуется согласно правилам раздела 4, но не включается в `canonical_content_payload` если не входит в состав entity-level payload.

**Для `STRUCTURED_RECORD` (Jira):**

```text
cross_source_text =
  summary + "\n" + description + "\n" + acceptance_criteria + "\n" + comments
```

Поля, включённые в `cross_source_text`, определяются Entity Policy. Текст полей нормализуется согласно правилам раздела 4.

**Для `CODE` (Git):**

```text
cross_source_text =
  qualified_symbol + "\n" + module_path + "\n"
  + docstring + "\n" + field_names + "\n" + method_names
```

Docstring извлекается из `full_symbol_span_text`, если доступен. Имена полей и методов извлекаются из структуры класса.

**Для `TABLE` и `DIAGRAM`:**

Cross-source text representation формируется из структурного описания:

```text
cross_source_text =
  headers + "\n" + row_values + "\n" + labels + "\n" + node_names
```

**Свойства:**

- `cross_source_text` формируется детерминированно из того же source content;
- не влияет на `canonical_content_payload`;
- не влияет на `content_hash`;
- не является identity;
- не является `embed_text` (формируется отдельно на этапе встраивания);
- доступен для semantic matching только после разрешения идентичности и финализации версий (когда назначен `entity_uid`);
- не сохраняется в постоянных materialized records после завершения приёма данных.

### 11. Изменение правил

Проект не использует переменные типа:

```text
content_policy_version
content_version
```

При изменении правил парсинга или нормализации:

1. выполняется полный повторный обход;
2. создаются новые снимки;
3. повторно формируются `structured_content`, `canonical_content_payload` и `content_hash`;
4. старые materializations не перезаписываются неявно;
5. итоговая замена materializations выполняется согласно Qdrant/Neo4j и политике сверки.

## Инварианты

1. `content_hash` вычисляется до контроля качества и разрешения идентичности.
2. `entity_uid` и `entity_version_id` не участвуют в вычислении `content_hash`.
3. `content_hash` не вычисляется из разбиения на фрагменты, встраивания или извлечения графа.
4. `structured_content` используется как единый вход контроля качества, разбиения на фрагменты и извлечения графа.
5. `canonical_content_payload` формируется только из source-derived значений.
6. Semantic inference и graph relations не входят в `canonical_content_payload`.
7. Source content, processing unit и logical entity не смешиваются.
8. Один source record может породить несколько content units.
9. `parent_unit_id` является временным IR-идентификатором и не сохраняется в постоянных materializations.
10. `unit_source_anchor` локализует unit, но не является identity.
11. Binary payload не проходит текстовую UTF-8 нормализацию.
12. Таблицы, диаграммы и code units не преобразуются в narrative text без явного source-specific правила.
13. Неуспешный extraction не делает unit автоматически пригодным для downstream materialization.
14. Отсутствие source content не позволяет вычислить `content_hash`.
15. Контроль качества не пересчитывает `content_hash`.
16. Изменение правил parsing/normalization требует полного повторного обхода.
17. `content_policy_version` и `content_version` не используются.
18. Парсер и нормализатор не записывают данные в Qdrant или Neo4j.
19. Одинаковый source input при одинаковых правилах даёт одинаковые `structured_content` и `content_hash`.
20. `cross_source_text` формируется детерминированно, не влияет на `content_hash` и используется только для semantic matching после разрешения идентичности и финализации версий.

## Новые переменные

| Переменная | Тип | Описание | Точка появления |
|---|---|---|---|
| `ParsedIR` | структура | Промежуточное структурированное представление processing unit | Typed Parser |
| `structured_content` | JSON | Типизированное содержимое processing unit для контроля качества, разбиения на фрагменты и извлечения графа | Typed Parser |
| `canonical_content_payload` | строка | Детерминированное source-derived представление для вычисления `content_hash` | Entity Normalizer |
| `content_hash` | строка | Хеш нормализованного содержимого logical entity | Entity Normalizer |
| `raw_payload_checksum` | строка | Fingerprint исходного source payload | Connector/Normalizer |
| `child_unit_source_anchor` | строка? | Source-local anchor дочернего processing unit для structural processing | Parser |
| `cross_source_text` | строка? | Детерминированное текстовое представление для cross-source semantic matching на этапе работы с графом | Parser |


## Последствия

### Положительные

* `content_hash` вычисляется до разрешения идентичности и может использоваться как свидетельство.
* Контроль качества получает проверяемый структурированный вход.
* Разбиение на фрагменты и извлечение графа используют один `structured_content`.
* Служебные различия формата не создают ложные content versions.
* Ошибка extraction не изменяет hash доступного source content.
* Неуспешный extraction не объявляется автоматически успешной обработкой.
* Confluence, Jira и Git получают source-specific normalization без разных identity-моделей.
* Изменение правил обрабатывается полным повторным обходом без введения дополнительных version variables.
* Временные IR-идентификаторы не попадают в постоянные materializations.
* `cross_source_text` обеспечивает детерминированное представление для semantic matching без влияния на `content_hash` и identity.

### Отрицательные

* Требуются отдельные Typed Parser и Entity Normalizer для разных source types.
* Нормализация кода не всегда может быть безопасно применена и может потребовать сохранения исходного span.
* Неполный или недоступный source content может привести к `REJECTED` unit или невозможности вычислить hash.
* Изменение правил parsing/normalization требует полного повторного обхода.
* Хранение source checksum не позволяет восстановить удалённый raw payload.
* Для Confluence необходимо явно определять, какие attachments входят в page-level payload.
* Ошибка parsing одного обязательного дочернего unit может блокировать публикацию всей entity согласно Entity Policy.
* `cross_source_text` увеличивает объём обрабатываемых данных и требует отдельного хранения до этапа semantic matching.

## Рассмотренные альтернативы

### Вычислять `content_hash` из Parsed IR

Использовать сериализованный `structured_content` как единственный источник `content_hash`.

**Плюсы:**

* один артефакт используется и для processing, и для hashing;
* меньше преобразований;
* проще реализовать повторную обработку.

**Минусы:**

* изменение структуры IR может изменить hash при неизменном source content;
* внутренние parser details могут стать частью identity;
* разные extractor versions могут давать разные hashes;
* semantic extraction может случайно попасть в version identity.

**Решение:** отклонено. Hash формируется из source-derived canonical payload.

### Вычислять `content_hash` после контроля качества

Сначала проверять качество, затем вычислять hash только для прошедших units.

**Плюсы:**

* меньше вычислений;
* rejected units не получают hash;
* проще исключить повреждённый content.

**Минусы:**

* контроль качества не может проверять стабильность hash;
* identity evidence становится доступен поздно;
* два одинаковых source content могут получить разный результат при различной обработке;
* усложняется диагностика rejected content.

**Решение:** отклонено. `content_hash` вычисляется до контроля качества, если source content доступен.

### Использовать единый текстовый формат для всех источников

Преобразовать Git, Jira и Confluence в обычный Markdown или plain text.

**Плюсы:**

* простой общий pipeline;
* меньше типов структур;
* проще интеграция с встраиванием.

**Минусы:**

* теряется структура таблиц и диаграмм;
* Jira fields смешиваются с changelog;
* ухудшается code parsing;
* усложняется извлечение графа;
* возрастает риск неверного `content_hash`.

**Решение:** отклонено.

### Выполнять отдельный parsing в каждом downstream-компоненте

Контроль качества, разбиватель на фрагменты и извлечение графа самостоятельно разбирают raw source content.

**Плюсы:**

* независимое развитие компонентов;
* меньше требований к общему IR;
* каждый компонент может использовать оптимальный parser.

**Минусы:**

* разные компоненты получают разные представления одного источника;
* дублируется parsing;
* увеличивается стоимость;
* возрастает риск расхождения контроля качества, разбиения на фрагменты и извлечения графа.

**Решение:** отклонено. Все downstream-компоненты используют один Parsed IR.

### Включать semantic extraction в `content_hash`

Добавлять в hash callers, callees, inferred types, graph relations и результаты framework extraction.

**Плюсы:**

* hash отражает более богатую семантику;
* изменения зависимостей могут автоматически создавать новую версию;
* потенциально выше точность semantic comparison.

**Минусы:**

* hash начинает зависеть от extractor и внешнего контекста;
* изменение извлечения графа создаёт ложные новые версии;
* нарушается воспроизводимость source identity;
* content version смешивается с semantic graph state.

**Решение:** отклонено.