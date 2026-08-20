# ADR Стратегия chunking

**Статус:** Предложен

## Контекст

После Quality Gate в Chunking передаются units с `quality_verdict` = `PASS` и разрешённой identity. Chunking формирует представления для embedding и Qdrant, не изменяя identity.

## Решение

Алгоритм нарезки на chunk'и зависит от типа unit'а. Атомарные unit'ы полностью помещаются в размер одного chunk'а. Большой текст режется по структуре, заголовки начинают новые секции, параграфы группируются до ~500 токенов, допускается пересечение по контенту соседних chunk. Большие классы кода, превышающие допустимые размеры chunk, режутся по методам класса.

Каждый chunk получает свой идентификатор для последующей идемпотентной обработки.

Объектом работы Chunker является не Processing Units, а Qdrant Representations, которые:

- создаются Chunker'ом на основе processing units
- НЕ проходят Quality Gate повторно
- используют уже проверенные child processing units (fallback)
- наследуют identity processing unit

### Входные параметры Chunker для создания Qdrant Representations

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

Принимаются только units с `quality_verdict = PASS`, `entity_uid`, `entity_version_id`.

### Объект Chunker'а, являющийся выходным артефактом

```text
Chunk:
  chunk_id              — UUIDv5
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
  parent_unit_id        — для IR-навигации и связывания с родительским unit
  chunk_anchor
  text                  — исходный текст фрагмента С УЧЁТОМ overlap;
                          не изменяется после создания Chunk
  chunk_index           — порядковый номер
  is_atomic             — bool: не подлежит разбиению
  is_oversized          — bool: конкретный Chunk требует oversized policy
  hierarchy             — source-specific metadata
                          (для Confluence — page hierarchy;
                           для Git/Jira — null если неприменимо)
  section_hierarchy     — heading path внутри entity
                          (применимо к narrative content)
```

`embed_text` НЕ входит в выходной артефакт Chunk. Embedding вычисляет `embed_text` самостоятельно из `section_hierarchy + text` на этапе генерации векторов. Chunker передаёт оба поля в Embedding как вход, но не сохраняет `embed_text` в payload.

### `chunk_id` — UUIDv5

```text
chunk_name =
  entity_uid + "\0"
  + entity_version_id + "\0"
  + canonical_source_scope + "\0"
  + chunk_anchor

chunk_id = UUIDv5(FIXED_CHUNK_NAMESPACE, chunk_name)
```

**Правила экранирования для всех компонентов `chunk_name`:**

```text
1. NFC нормализация
2. UTF-8 сериализация
3. Экранирование:
   — ":" → "\:"
   — "\0" → "\\0"
   — "\" → "\\"
4. Пустые компоненты — пустая строка
5. Разделители не интерпретируются как часть компонента
```

### `canonical_source_scope` — сериализация `source_scope`

```text
Git: "git\0" + repository_id + "\0" + branch_or_release_reference
Confluence: "confluence\0" + space_key
Jira: "jira\0" + project_key
```

### `chunk_anchor` — с проверкой коллизий

```text
atomic:             canonical(unit_source_anchor) + ":atomic"
narrative:          canonical(unit_source_anchor) + ":section=" + structural_path + ":fragment=" + index
structural child:   canonical(child_unit_source_anchor) + ":structural"
table row group:    canonical(unit_source_anchor) + ":rows=" + N + "-" + M
jira field group:   canonical(unit_source_anchor) + ":fields=" + group_name
```

**Требование уникальности:**

child_unit_source_anchor должен быть уникальным среди child units
родительской entity.

Перед генерацией chunk_id Chunker проверяет уникальность chunk_anchor
в пределах entity_uid + entity_version_id + canonical_source_scope.

При коллизии:
  — обработка entity завершается ошибкой
  — chunk set не публикуется
  — фиксируется в ingestion metadata


### Rejected child — влияние на parent для `chunk_anchor` = `structural child`

Правило по умолчанию: если child processing unit входит в `canonical_content_payload` parent entity и имеет `REJECTED`/`QUARANTINED`, тогда **parent entity НЕ публикуется как полная `CANONICAL_PUBLISHED`**.

Исключение — Entity Policy. Entity Policy может явно определить, что child поля опциональны и их `REJECTED` не блокирует parent. В этом случае `content_hash` ВКЛЮЧАЕТ `REJECTED` unit'ы → parent публикуется с пометкой, что представление может быть неполным (но identity сохранён).

### `is_oversized` — свойство созданного Chunk

```text
true:  chunk требует oversized policy
false: chunk укладывается в нормальное представление

Если Chunk не создан (например, BINARY_ATTACHMENT > hard limit):
  — is_oversized отсутствует (нет Chunk)
  — факт превышения фиксируется в ingestion metadata
  — новая переменная не вводится
```

### Размерность Chunking'а

```text
target_chunk_size = 500 tokens
min_chunk_size = 100 tokens
max_chunk_size = 800 tokens
overlap_tokens = 50 tokens
embedding_hard_limit — embedding model/server policy
```

### Code entity chunk policy

**Нормальный режим:**

```text
Code entity верхнего уровня, укладывающийся в max_chunk_size:
  — один атомарный chunk с полным full_symbol_span_text
  — вложенные symbols включаются в parent span
  — дочерние chunks не создаются
```

**Fallback при oversized:**

```text
Parser/IR заранее выделяет class unit и method units.
Quality Gate проверяет все units.

Chunker (fallback):
  — class не получает свой chunk (oversized)
  — methods с PASS → Qdrant representations

Каждый fallback method chunk:
  entity_uid          = parent class
  entity_version_id   = parent class
  content_hash        = parent class
  unit_source_anchor  = method (child) — для локализации
  chunk_anchor        = canonical(child_unit_source_anchor) + ":structural"
  is_atomic           = true
  is_oversized        = false (если embed_text <= hard limit)
```

**Важно:** Fallback использует **уже проверенные** child processing units, не создаёт новые в обход Quality Gate.

### Narrative chunking

Правила разделения narrative text:

1. Heading не образует отдельный chunk, если нет собственного текста.

2. Heading включается в `section_hierarchy`.

3. Heading-only fragment всегда присоединяется к следующему содержательному chunk, не создаёт отдельный chunk.

4. Новый heading закрывает текущий chunk.

5. Paragraphs группируются до `target_chunk_size`.

6. Paragraph > `max_chunk_size` разбивается по предложениям.

7. Предложение разбивается по безопасным tokenizer boundaries.

8. Неразбиваемое предложение → `is_oversized = true`.

9. Overlap только между narrative chunks:

```text
— в начало следующего chunk добавляются последние overlap_tokens
  токенов предыдущего chunk
— если нарушает структурную границу: overlap уменьшается до границы
— если overlap невозможно в пределах max_chunk_size:
  overlap сокращается детерминированно (или 0)
— не применяется к atomic units
— не пересекает code/table/diagram blocks
— overlap входит в text фрагмента
```

10. Fragment < `min_chunk_size`:

```text
— присоединяется к следующему содержательному chunk,
  если результат <= max_chunk_size
— иначе к предыдущему
— если невозможно: остаётся отдельным
```

11. Последний chunk может быть < `min_chunk_size`.

12. Atomic units не объединяются.

### `text` vs `embed_text`

```text
text:
  — исходный текст фрагмента С УЧЁТОМ overlap
  — НИКОГДА не изменяется после создания Chunk
  — сохраняется в Qdrant payload

embed_text:
  — вычисляется на этапе Embedding (ADR-014)
  — = section_hierarchy.join(" > ") + "\n" + text
  — НЕ сохраняется в Qdrant payload
  — используется ТОЛЬКО для генерации векторов
```

Если `embed_text` превышает `embedding_hard_limit`:

```text
1. Сокращается section_hierarchy
2. Если всё ещё превышает — Qdrant representation НЕ создаётся
3. Применяется oversized policy / Entity Policy
4. text остаётся неизменным
```

Запрещено создавать embedding, представляющий часть text без явного chunk boundary.

### Oversized atomic policy

```text
CODE:
  — structural fallback: methods как Qdrant representations
  — rejected method не блокирует parent (если Entity Policy разрешает)

TABLE:
  — Qdrant representations (не processing units)
  — наследуют parent identity
  — строки в исходном порядке, headers повторяются, spans не разрываются

DIAGRAM:
  — structural description (ADR-008)
  — Graph отдельно

STRUCTURED_RECORD (Jira):
  — field-group representation
  — chunk_anchor: unit_source_anchor + ":fields=group_name"
  — наследует identity родительской Jira issue
  — НЕ отдельная entity
  — rejected field group блокирует parent только если
    поле входит в canonical_content_payload

BINARY_ATTACHMENT:
  — не режется
  — Qdrant target недоступен
  — metadata сохраняется в ingestion metadata
    или связывается с parent по Entity Policy
  — запись в Neo4j ТОЛЬКО если Entity Policy разрешает
    attachment как Graph entity
```

### Инварианты

1. Chunking не пересчитывает identity-поля
2. Chunker не создаёт processing units в обход Quality Gate
3. Structural fallback использует проверенные child units
4. Fallback chunks наследуют parent identity
5. Rejected child в content_hash может блокировать parent (по Entity Policy)
6. `chunk_anchor` детерминирован, проверка коллизий обязательна
7. `chunk_id` — UUIDv5 с разделителями и экранированием
8. `snapshot_id` не входит в `chunk_id`
9. `text` не изменяется после создания Chunk
10. `is_oversized` относится к созданному Chunk
11. Normal/fallback не меняет identity
12. Stale chunks удаляются по expected set
13. Overlap — детерминированный алгоритм
14. Attachment в Neo4j только по Entity Policy
15. Механизм replacement — ADR-018/ADR-019 (незавершённая dependency)

## Последствия

### Положительные

- `parent_unit_id` сохраняется в выходном артефакте для IR-навигации
- `text` определён как «с учётом overlap, не изменяется после создания»
- Fallback явно использует проверенные child units
- Экранnирование и коллизии формализованы

### Отрицательные

- Требуется Entity Policy для code child rejection
- Требуется проверка коллизий chunk_anchor
- Требуется фактическое обновление реестра
- Механизм replacement — незавершённая dependency

## Рассмотренные альтернативы

### Rejected child всегда блокирует parent
Минусы: массовое отклонение классов из-за LSP-ошибок. Отклонено — Entity Policy.

### Overlap без алгоритма
Минусы: недетерминированность. Отклонено.

### `chunk_anchor` без проверки коллизий
Минусы: перезапись Qdrant points. Отклонено.

### Attachment metadata всегда в Neo4j
Минусы: загрязнение domain graph. Отклонено.

### `is_oversized` для несозданного Chunk
Минусы: нет места хранения. Отклонено — ingestion metadata.