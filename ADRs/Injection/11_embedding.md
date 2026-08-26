# ADR Формирование embedding-представлений

**Статус:** Предложен

---

## Контекст и проблема

Chunker формирует `Chunk` — детерминированное Qdrant-представление processing unit с неизменяемыми:

- `entity_uid`;
- `entity_version_id`;
- `content_hash`;
- `chunk_id`;
- `text`;
- `section_hierarchy`.

Для генерации векторов требуется отдельное embedding-представление. Оно должно учитывать иерархический контекст, но не менять исходный `text`, сохранённый для цитирования и формирования контекста LLM.

Embedding-модель имеет собственный tokenizer и hard limit входной последовательности. Превышение этого лимита, неправильная размерность или недетерминированный результат могут привести к ошибке записи в Qdrant или к несогласованным представлениям одного `chunk_id`.

**Проблема:** как детерминированно преобразовать каждый допустимый `Chunk` в dense и sparse vectors, не изменяя его identity и не создавая embedding для неполного или невалидного текста.

---

## Факторы решения

* Совместимость с Qdrant dense+sparse schema.
* Соблюдение `embedding_hard_limit`.
* Сохранение исходного `text` без усечения.
* Детерминированное формирование `embed_text`.
* Идемпотентность повторной генерации vector для того же `chunk_id`.
* Отсутствие влияния embedding на `entity_uid`, `entity_version_id` и `content_hash`.
* Возможность обрабатывать разные `content_type` одним embedding-контрактом.
* Контролируемая ошибка при превышении лимита или недоступности embedding server.
* Embeddings используются для поиска semantic candidate relations между уже идентифицированными entities на этапе работы с графом, а не для identity resolution.**

---

## Решение

Мы используем BGE-M3 для генерации двух представлений каждого допустимого Chunk:

- dense vector размерности $1024$;
- sparse vector для лексического поиска.

Embedding выполняется только для Chunk, который:

```text
quality_verdict = PASS
entity_uid определён
entity_version_id определён
chunk_id сформирован
text доступен
```

### 1. Входные данные

Embedding получает:

```text
chunk_id
entity_uid
entity_version_id
content_hash
content_type
source_system
source_scope
source_object_id
source_revision_id
unit_source_anchor
text
section_hierarchy
is_atomic
is_oversized
```

Embedding не изменяет входные поля и не выполняет:

- Quality Gate;
- Identity Resolution;
- Chunking;
- Graph Extraction;
- пересчёт `content_hash`;
- пересчёт `entity_version_id`.

### 2. Формирование `embed_text`

Embedding Server формирует временное embedding-представление:

```text
embed_text =
  section_hierarchy.join(" > ") + "\n" + text
```

Если `section_hierarchy` отсутствует или пуста:

```text
embed_text = text
```

`embed_text`:

- вычисляется только на этапе эмбеддинга;
- не сохраняется в Qdrant payload;
- не заменяет `text`;
- не является LLM-based contextual enrichment;
- не влияет на `entity_uid`, `entity_version_id` или `content_hash`.

LLM-based contextual enrichment регулируется отдельным retrieval/enrichment ADR.

### 3. Проверка размера

Размер проверяется тем же tokenizer, который используется embedding server.

Embedding Server не принимает `embed_text`, если количество токенов превышает:

```text
embedding_hard_limit
```

Перед вызовом embedding server Chunker/Embedding stage:

1. проверяет размер `embed_text`;
2. удаляет `section_hierarchy`, если её длина приводит к превышению лимита;
3. повторно проверяет размер;
4. не изменяет `text`;
5. не создаёт embedding из произвольной части `text`.

Если после удаления hierarchy `embed_text` всё ещё превышает `embedding_hard_limit`:

- embedding для текущего Chunk не создаётся;
- исходный Chunk не изменяется;
- результат передаётся на этап материализации записей как неуспешный embedding result;
- дальнейшая публикация определяется `required_materialization_targets` и materialization policy.

Для atomic units применяется oversized policy. Embedding не выполняет самостоятельное произвольное разбиение.

### 4. Генерация dense vector

Для каждого допустимого `embed_text` Embedding Server формирует dense vector:

```text
dense_vector:
  dimension = 1024
  model = BGE-M3
  similarity = cosine
```

Dense vector нормализуется для cosine similarity согласно embedding server policy.

Если embedding server возвращает vector другой размерности или некорректное значение, результат считается неуспешным и не передаётся в Qdrant Writer.

### 5. Генерация sparse vector

Для того же `embed_text` Embedding Server формирует sparse vector BGE-M3.

Sparse vector:

- соответствует тому же `chunk_id`;
- формируется из того же `embed_text`;
- передаётся в Qdrant Writer вместе с dense vector;
- не используется как самостоятельная identity;
- не изменяет `content_hash`.

Dense и sparse vectors одного Chunk должны относиться к одной операции embedding. Нельзя смешивать dense vector одного `embed_text` со sparse vector другого.

### 6. Embedding operation

Для каждого Chunk выполняется одна embedding operation:

```text
Chunk
→ embed_text
→ dense_vector + sparse_vector
→ EmbeddingResult
```

Повторная операция для того же:

```text
chunk_id
+ embedding_model
+ embedding_model_version
```

должна давать совместимые vectors. Повторная генерация не создаёт новый `chunk_id` и не изменяет identity.

### 7. Результат Embedding

Успешный результат имеет формат:

```text
EmbeddingResult:
  chunk_id
  entity_uid
  entity_version_id
  content_hash
  dense_vector
  sparse_vector
  embedding_model
  embedding_model_version
  token_count
  embedding_status
```

`EmbeddingResult` передаётся на этап материализации записей вместе с исходным `Chunk`.

Embedding не сохраняет `embed_text` в payload. Qdrant Writer сохраняет `text` и metadata Chunk, а vectors записывает как vector fields Qdrant point.

### 8. Использование embeddings для semantic candidate generation

BGE-M3 embeddings используются не для identity resolution, а для поиска semantic candidate relations между уже идентифицированными entities.

Semantic candidate generation выполняется после Qdrant Materialization на этапе Semantic Relation Apply.

Процесс:

1. Embedding модуль генерирует dense и sparse vectors для всех Chunk.
2. Материализация векторов в Qdrant.
3. Semantic Relation Apply выполняет semantic candidate search:
   - Для каждого narrative/текстового unit (Confluence, Jira summary/description, code docstring) выполняется поиск по Qdrant.
   - Используются dense и sparse vectors (BGE-M3).
   - Поиск выполняется среди сущностей с финализированными `entity_uid` и `entity_version_id`.
   - Результат: `similarity_score` для каждой пары.

Embedding не выполняет semantic matching самостоятельно. Он только генерирует vectors, которые затем используются Semantic Relation Apply для candidate generation.

### 9. Статусы Embedding

```text
embedding_status:
  SUCCEEDED
  FAILED
  SKIPPED
```

- `SUCCEEDED` — dense и sparse vectors успешно созданы.
- `FAILED` — embedding operation завершилась ошибкой.
- `SKIPPED` — embedding не выполнялся по policy, например для недоступного или запрещённого target.

`embedding_status` относится к результату конкретной embedding operation и не является:

- `quality_verdict`;
- `canonical_status`;
- `processing_status`;
- свойством `entity_version_id`.

Если embedding завершился с `FAILED` из-за технической ошибки, это обрабатывается resilience policy и не превращается в Quality Gate rejection.

Если embedding невозможно создать из-за размера Chunk или отсутствия разрешённого fallback, результат передаётся в materialization policy.

### 10. Ошибки

Embedding не создаётся в следующих случаях:

- `Chunk` не прошёл входные условия;
- `embed_text` превышает `embedding_hard_limit`;
- embedding server недоступен;
- dense vector имеет неверную размерность;
- sparse vector некорректен;
- tokenizer не может обработать текст;
- модель не соответствует ожидаемой конфигурации.

Ошибка не изменяет:

```text
chunk_id
entity_uid
entity_version_id
content_hash
text
```

Детали технической ошибки передаются далее. `quality_reason` используется только для решений Quality Gate и не заменяет embedding operation result.

### 11. Совместимость с re-crawl и переиндексацией

Изменение embedding-модели или её параметров не меняет:

```text
entity_uid
entity_version_id
content_hash
chunk_id
```

При замене модели выполняется полная процедура re-crawl и vector replacement Старые vectors не должны смешиваться с vectors другой модели в одной актуальной Qdrant projection.

### 12. Границы ответственности

Векторизация отвечает за:

- формирование `embed_text`;
- token counting;
- проверку hard limit;
- генерацию dense и sparse vectors;
- проверку размерности и корректности результата;
- передачу `EmbeddingResult` в Qdrant materialization.

Векторизация не отвечает за:

- изменение `text`;
- изменение Chunk identity;
- выбор актуальной версии;
- Qdrant upsert;
- удаление старых points;
- current/historical projection;
- reconciliation;
- назначение `canonical_status`;
- semantic candidate generation;
- identity resolution.

---

## Инварианты

1. Embedding выполняется только для Chunk с `quality_verdict=PASS` и разрешённой identity.
2. Embedding не пересчитывает `entity_uid`, `entity_version_id`, `content_hash` или `chunk_id`.
3. `text` не изменяется при формировании `embed_text`.
4. `embed_text` не сохраняется в Qdrant payload.
5. Для dense и sparse vectors используется один и тот же `embed_text`.
6. Размер проверяется тем же tokenizer, который используется embedding server.
7. `embedding_hard_limit` не может быть превышен.
8. Embedding не выполняет произвольное разбиение Chunk.
9. Atomic units не разделяются на этапе Embedding.
10. Ошибка embedding не изменяет Chunk и его identity.
11. Dense vector имеет размерность $1024$.
12. Dense vector нормализуется для cosine similarity.
13. Повторная генерация vector для того же Chunk не создаёт новый `chunk_id`.
14. `embedding_status` не заменяет `quality_verdict`, `canonical_status` или `materialization_status`.
15. Embedding не записывает данные непосредственно в Qdrant.
16. Embedding failure не является автоматически Quality Gate rejection.
17. Vectors разных embedding-моделей не смешиваются в одной актуальной Qdrant projection.
18. `EmbeddingResult` передаётся в Qdrant Writer вместе с исходным Chunk.
19. Embeddings используются для semantic candidate generation на этапе Semantic Relation Apply, а не для identity resolution.
20. Semantic matching выполняется после Qdrant materialization.

---

## Новые переменные

| Переменная | Тип | Описание | Точка появления |
|---|---|---|---|
| `EmbeddingResult` | структура | Результат генерации dense и sparse vectors для Chunk | Embedding Server |
| `dense_vector` | vector[1024] | Плотное embedding-представление `embed_text` | Embedding Server |
| `sparse_vector` | sparse vector | Разреженное embedding-представление `embed_text` | Embedding Server |
| `embedding_model` | string | Идентификатор embedding-модели, например `BGE-M3` | Embedding configuration |
| `embedding_model_version` | string | Версия или immutable reference используемой embedding-модели | Embedding configuration |
| `token_count` | integer | Количество токенов итогового `embed_text` | Embedding stage |
| `embedding_status` | enum | Результат embedding operation: `SUCCEEDED`, `FAILED`, `SKIPPED` | Embedding stage |

---

## Последствия

### Положительные

* Один Chunk получает согласованные dense и sparse vectors.
* `text` сохраняется без усечения для цитирования и context assembly.
* Размер проверяется тем же tokenizer, что и на embedding server.
* Превышение hard limit не приводит к скрытому усечению текста.
* Dense и sparse представления совместимы с hybrid retrieval.
* Повторная генерация не создаёт новые Chunk IDs.
* Ошибки embedding не изменяют identity и content hash.
* Embedding отделён от Qdrant materialization и не записывает данные напрямую.
* Замена embedding-модели не требует изменения identity сущностей и chunks.


### Отрицательные

* Требуется доступный BGE-M3 embedding server и совместимый tokenizer.
* Для каждого Chunk создаются два vector representations.
* Большие atomic units могут остаться без Qdrant embedding при отсутствии безопасного fallback.
* Ошибка embedding server может задержать materialization.
* Замена embedding-модели требует полной vector replacement процедуры.
* Появляется дополнительный `EmbeddingResult` и набор технических статусов.
* Semantic candidate generation зависит от завершения Qdrant materialization, что увеличивает общую latency.

---

## Рассмотренные альтернативы

### Только dense vectors

Использовать только плотное embedding-представление без sparse vector.

**Плюсы:**

* проще Qdrant schema;
* меньше storage;
* ниже стоимость генерации и записи;
* проще эксплуатация embedding server.

**Минусы:**

* хуже обрабатываются точные идентификаторы, имена API, классов, функций и Jira keys;
* снижается качество лексического поиска;
* hybrid retrieval становится неполным.

### Только sparse vectors

Использовать только sparse representation без dense embedding.

**Плюсы:**

* хорошо сохраняются точные термины и идентификаторы;
* ниже требования к dense vector storage;
* проще искать по именам symbols и source identifiers.

**Минусы:**

* хуже работает семантический поиск;
* хуже обрабатываются переформулированные и narrative-запросы;
* relational и semantic QA получают меньший recall.

### Использовать `text` без `section_hierarchy`

Передавать в модель только исходный текст Chunk.

**Плюсы:**

* меньше токенов;
* проще вычисление embedding input;
* нет риска превышения hard limit из-за hierarchy.

**Минусы:**

* теряется контекст секции;
* одинаковые фрагменты из разных разделов хуже различаются;
* снижается качество retrieval для документированных API и Confluence pages.

### Сохранять `embed_text` в Qdrant payload

Сохранять в payload как исходный `text`, так и embedding input.

**Плюсы:**

* проще диагностика embedding;
* можно воспроизвести input без повторного формирования;
* виден hierarchy prefix.

**Минусы:**

* дублирование данных;
* увеличение размера Qdrant payload;
* повышенный риск рассогласования `text` и `embed_text`;
* `embed_text` не требуется для пользовательской цитаты.

### Молча усекать `embed_text` по hard limit

Передавать в embedding только допустимый префикс или суффикс текста.

**Плюсы:**

* почти всегда создаётся vector;
* меньше rejected oversized units;
* проще соблюсти hard limit.

**Минусы:**

* embedding представляет только часть Chunk;
* ухудшается объяснимость результата;
* важная информация может находиться в отброшенной части;
* embedding и отображаемый `text` семантически расходятся.

### Позволить Embedding Server выполнять произвольное разбиение

Передавать большой Chunk на embedding server, который самостоятельно делит его на несколько vectors.

**Плюсы:**

* часть oversized units сохраняет Qdrant-покрытие;
* Chunker проще;
* embedding server централизует token handling.

**Минусы:**

* нарушается контракт `chunk_id` одного Chunk;
* теряется контроль над chunk boundaries и provenance;
* разные серверы могут делить текст по-разному;
* Qdrant materialization получает неизвестный набор points;
* нарушается ответственность за формирование Qdrant representations.

### Записывать vectors непосредственно из Embedding stage

Embedding stage сразу выполняет upsert в Qdrant.

**Плюсы:**

* меньше промежуточных объектов;
* ниже задержка между генерацией и записью;
* проще короткий happy path.

**Минусы:**

* смешиваются генерация vectors и storage materialization;
* сложнее retry и partial failure;
* труднее согласовать Qdrant с Neo4j;
* сложнее выполнять expected-set cleanup и reconciliation.

### Выполнять semantic matching внутри embedding

Сразу после генерации vectors выполнять поиск semantic candidates.

**Плюсы:**

* потенциально более раннее обнаружение semantic relations;
* можно использовать свежесгенерированные vectors без ожидания Qdrant materialization.

**Минусы:**

* semantic matching требует доступа к vectors других entities, которые могут быть ещё не сгенерированы;
* нарушается разделение между embedding generation и relation discovery;
* сложнее обрабатывать partial materialization.

**Решение:** отклонено. Semantic matching выполняется после полной Qdrant materialization.