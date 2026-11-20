# Полный разбор: сборка RAG-стека для opencode

> Личный документ. Для пользователя и для анализа Claude'ом.
> Не коммитится в git (см. .gitignore). Дата: 2026-11-19.

---

## 0. TL;DR

Построен автономный RAG-репозиторий `/mnt/GameLinux/RAG`, прикреплённый к
существующему стеку `~/ai-compose` через внешнюю docker-сеть `ai-net`.
Три контейнера: CPU-эмбеддер (bge-m3 на том же pinned-образе llama.cpp,
`-ngl 0`), Qdrant, rag-core (FastAPI + MCP SDK v2). Агенты (opencode) сами
вызывают MCP-инструмент `search` — никакого LLM-прокси, виртуальных моделей
`+rag` и Chatbox в Phase 1 нет. Мониторинг — через существующий Prometheus
(ai-compose) по file_sd, без нового Prometheus/Grafana.

Итог: 159 чанков из 4 файлов, поиск работает, кросс-лингва (RU-запрос →
EN-корпус) работает, MCP handshake валиден, Prometheus скрейпит
`up{job="rag"}=1`, дашборд в Grafana живёт.

За время сборки поймано и решено **14 отдельных багов** + несколько
инструментальных проблем (aria2, pkill, docker CLI). Все — в разделе 5,
с точными текстами ошибок.

---

## 1. Исходные требования

Сформулированы пользователем, уточнялись в процессе:

1. **Отдельный репозиторий/каталог**, прикреплённый к существующему стеку —
   НЕ слить в ai-compose, НЕ создавать новый автономный стек (пользователь
   исправлял дважды, это главный архитектурный контур).
2. **МCP-first**: модель сама решает, когда искать. Не проксировать LLM, не
   делать виртуальную модель `+rag`, не тянуть Chatbox. Бинд только на
   `127.0.0.1:8090` (LAN — Phase 2).
3. **Без дублирования мониторинга**: метрики rag скрейпятся существующим
   Prometheus при желании (ai-net + file_sd). Не поднимать новый
   Prometheus/Grafana.
4. Путь: агент → MCP → rag-core → Qdrant → чанк+источник. **rag-core в MCP-пути
   никогда не звонит в llama.cpp** — только в эмбеддер.
5. In-memory очередь (без Redis). Инкрементальный реиндекс: delete-by-path +
   upsert. Полный рескан — страховка.
6. Образы пинованы по digest. Секреты в `secrets/` (gitignored) + `.env`.
7. Сообщить, когда появится **первый верифицированный** результат поиска.
8. Скачивания — aria2 (`/usr/bin/aria2c` 1.37.0).
9. Мета-ограничение: сессия режется на 20k токенов — держать ходы маленькими,
   дробить работу на куски.

---

## 2. Окружение (верифицировано до старта)

| Что | Значение |
|---|---|
| CPU | 12 cores |
| RAM | 31 GiB |
| GPU | RX 6800 XT (ROCm, gfx1030) — занята основной моделью, отсюда CPU-эмбеддер |
| Docker | 29.9.3 |
| aria2c | 1.37.0 |
| Основной стек | `~/ai-compose`, контейнеры `llama-qwen27b`, `llama-prometheus`, `llama-grafana`, `llama-node-exporter` |
| Образ llama.cpp | `ghcr.io/ggml-org/llama.cpp@sha256:6da04cde…` (ROCm, уже на хосте) |
| Модель эмбеддинга | `bge-m3-q8_0.gguf`, 634 553 760 байт (~605.1 MiB) |

---

## 3. Архитектура

```
opencode ──(MCP streamable-HTTP :8090/mcp, Bearer)──► rag-core
                                                         │  FastAPI + mcp v2 (MCPServer)
                                                         │  in-memory ingest queue
                              ┌──────────────────────────┤
                              ▼                          ▼
                     embedder :8081               qdrant :6333
                     llama.cpp --embeddings       (volume, keyed)
                     bge-m3 q8_0, -ngl 0 (CPU)
                     └──────────┬───────────────────┘
                                │ rag-net (bridge, internal: true)
                                ▼
                     [недостижимо с хоста и интернета]

ai-net (external bridge, создана `make init`):
  rag-core + llama-qwen27b (алиас llama) + llama-prometheus
```

### 3.1 Контейнеры

| Сервис | Образ | Зачем |
|---|---|---|
| `embedder` | тот же pinned ROCm-образ llama.cpp, `--embeddings -ngl 0 -c 8192 -b 2048 -ub 4096` | bge-m3 на CPU: VRAM не трогаем (27B-модель живёт там). Без API-ключа: `rag-net` internal, снаружи невидим |
| `qdrant` | `qdrant/qdrant@sha256:d2775927…` | векторная БД, volume, **ключёный даже внутри** (`QDRANT__SERVICE__API_KEY_FILE`) |
| `rag-core` | `python:3.12-slim@sha256:5e2bdb1b…` + свой Dockerfile (pinned pip) | один процесс: FastAPI + MCP + qdrant-client; на обеих сетях |

Pinned pip (проверено 2026-11-19): fastapi 0.165.0, uvicorn 0.54.0,
**mcp 2.3.0**, qdrant-client 1.20.0, httpx 0.28.1.

### 3.2 Ключевые решения и почему

- **CPU-эмбеддер** (`-ngl 0`): 605 MiB GGUF грузится в RAM, 12 ядер CPU
  эмбедят чанк за доли секунды. GPU полностью отдаётся основной модели.
- **bge-m3**: 1024-dim, matryoshka, кросс-лингва — единственный кандидат
  под «EN-доки, RU-запросы». Запасной план на случай, если pinned ROCm-сборка
  не возьмёт XLM-RoBERTa (mRoPE): Qwen3-Embedding-0.6B с instruction-префиксом.
  Не понадобился — `--embeddings` заработал с первого раза.
- **MCP-first без LLM-прокси**: агент сам решает, что искать. Это убрало из
  стека прокси `/v1`, виртуальные модели, Chatbox. В Phase 1 rag-core вообще
  не знает про llama.
- **`rag-net` internal + `ai-net` external**: embedder/qdrant физически
  недоступны с хоста/интернета; `ai-net` — единственное окно в мир, и через
  него же Prometheus дотягивается до rag-core (Docker DNS по имени контейнера).
- **In-memory очередь инджеста**: для 4 файлов и десятков документов Redis —
  абсурд. Job-статус в словаре, worker — одна asyncio-задача.
- **Инкрементальный реиндекс**: `state.json` (path → mtime/hash), при `/ingest`
  файл перечанкивается, только если изменилось; старые точки удаляются по
  `path`, новые upsert'ятся. Полный рескан — `POST /ingest {}`.
- **Чанкинг**: заголовки → абзацы, накопление до ~2000 символов (`CHUNK_CHARS`),
  overlap ~1/3 цели (2–3 абзаца), code-fences и таблицы **никогда** не режутся,
  breadcrumb `path > Section > Subsection` в начале текста.
- **Point ID**: `UUID(sha256(path + "\0" + body)[:16])` — детерминированный,
  повторный инджест того же чанка бьёт в ту же точку. (Qdrant принимает
  только UUID или uint64 — см. баг 11.)
- **Секреты**: `secrets/rag_api_key` (opencode → rag-core), `secrets/qdrant_api_key`.
  Генерация в `make init`. **Делiberate**: `/metrics` без ключа (Prometheus
  скрейпит анонимно), MCP-обёртка авторизации пускает Bearer только на пути,
  начинающиеся с `/mcp` — остальное падает в саб-апп и 404'ится, а не 401'ится.
- **Мониторинг**: свой Prometheus/Grafana не поднимать. В ai-compose: prometheus
  прицеплен к `ai-net`, job `rag` по file_sd (`targets/rag.yml`), дашборд
  `rag.json` провиженится file-provider'ом.

### 3.3 Контракт env rag-core

```
EMBEDDER_URL=http://embedder:8081   EMBEDDER_MODEL=bge-m3
QDRANT_URL=http://qdrant:6333
COLLECTION=docs   EMBED_BATCH=16   CHUNK_CHARS=2000
STATE_DIR=/state
CORPUS_ROOTS=/corpus/ai-compose,/corpus/rag
```
`.env` обязан задать `EMBED_MODEL_FILE`, `CORPUS_AI_COMPOSE`, `CORPUS_RAG`
(compose использует `:?` — без них поднимется с ошибкой, а не с дефолтами).

---

## 4. Ход работ (по шагам)

1. **Реконструкция окружения**: ядра/RAM/Docker/aria2; подтверждено, что
   llama.cpp-образ уже локален; qdrant и python:3.12-slim стянуты.
2. **Скачивание модели** aria2 (2 части, resume). После скача — **обязательная
   сверка**: размер 634 553 760 B и sha256 `aa473d51…` сверены с HF LFS `oid`
   через API (`huggingface.co/api/models/ggml-org/bge-m3-Q8_0-GGUF/tree/main`) —
   совпали ровно. (См. урок 9 про aria2 resume.)
3. **Написание репо**: `docker-compose.yml`, `rag-core/Dockerfile`,
   `rag-core/app/{main,store,chunker,embedder}.py`, `.env`/`.env.example`,
   `.gitignore`, `Makefile`, `README.md`.
4. **`make init`**: генерация `.env`, обоих ключей, сети `ai-net`;
   `docker compose config -q` — OK.
5. **Локальное тестирование chunker'а** на реальном `~/ai-compose/README.md`
   (вне контейнера, `python3 -c`): нашлись баги 1 и 2, фикс, перепроверка:
   23 чанка, 23 уникальных ID, абсолютные номера строк. Смоук на всём корпусе:
   3 файла, 152 чанка, без крашей.
6. **`docker compose build`** (фоновый). Пока билдился — **разведка mcp-SDK**:
   выяснилось, что pinned `mcp==2.3.0` — это **v2**, где `FastMCP` удалён
   (баг 4). Чтение миграционного гайда, переработка `main.py` под v2 API.
7. **Итерации build→run→logs** (основной цикл, ~8 циклов): каждый цикл
   ловил один-два запуска-бага (раздел 5). Промежуточная тактика:
   `docker run --rm --entrypoint python3 rag-core:local -c "import main"` —
   ловить ImportError'ы ДО `up`, без ожидания healthcheck.
8. **Первый успешный ап**: все три healthy. `make ingest` → первые 2 ошибки
   (баги 9 и 10): embedder 500 по батчу + `tuple index out of range`.
9. **Фикс батча** (`-b 2048 -ub 4096`) и upsert-индекса → реинджест: 159 чанков,
   0 ошибок.
10. **Первый поиск**: работает, но «KV cache» выдаёт README самого RAG-репо
    вместо ai-compose. Диагностика: в qdrant'е 135 точек, а не 159, и всего
    3 пути — `README.md` из ai-compose **вырезан** коллизией путей (баг 12).
    Фикс: префикс имени корня в `rel_path`. Дроп коллекции, чистый реинджест:
    158 точек, 4 пути, оба README различимы.
11. **Верификация поиска** (3 запроса, EN и RU) — раздел 6.
12. **Верификация MCP-эндпоинта** честным handshake'ом:
    `initialize` → заголовок `Mcp-Session-Id` → `notifications/initialized` →
    `tools/list` (3 инструмента: search, list_collections, get_document) →
    `tools/call search`. Всё валидно.
13. **Интеграция в ai-compose** (минимальные правки): `ai-net` external +
    алиас `llama` на `llama-server`; `docker network connect --alias llama
    ai-net llama-qwen27b` (без рестарта LLM); prometheus на `ai-net` +
    job `rag` + `targets/rag.yml`; дашборд `rag.json`.
14. **Проверка мониторинга**: таргет `rag-core:8090 → up`, `up{rag}=1`,
    `search_total` растёт после живого поиска, метрики `ingest_chunks_total`
    и пр. видны. Дашборд подхвачен file-provider'ом без рестарта Grafana.
15. **Репозитории**: `git init` RAG (10 файлов, 1099 строк; секреты вне
    коммита; чужой черновик `Набросок плана.txt` не коммитился), пуш на
    `selfcortex/RAG`. ai-compose: 2 отдельных коммита (RAG-интеграция +
    чужая правка run.sh), пуш. README обновлён (слоган, реальный формат
    opencode-конфига, UUID id, мониторинг).

---

## 5. Журнал проблем

Формат: симптом → причина → фикс. Тексты ошибок — дословные (обрезка — моя).

### Баг 1. Chunker: блоки против плоского списка
```
ValueError: not enough values to unpack (expected 2, got 1)
```
На `~/ai-compose/README.md`. `cur` аккумулировал **блоки** (списки кортежей),
а `emit()` распаковывал аргумент как плоский список кортежей `(line, text)` —
первый элемент «плоского» списка оказался самим блоком.
**Фикс**: `emit(block_list: list[list[tuple[int, str]]])` с уплощением
`flat = [t for blk in block_list for t in blk]`; `start_line=flat[0][0]`.
Типы аккумулирования и эмита приведены к одному: `list[list[tuple[int,str]]]`.

### Баг 2. Chunker: номера строк относительно секции
Симптом в тесте: второй чанк `[1-18]` — строки начинаются заново с 1, потому
что `_split_blocks` нумеровал срез секции, а не файл.
**Фикс**: параметр `offset` в `_split_blocks(lines, offset)`,
`enumerate(lines, start=offset + 1)`, вызов с `offset=s - 1`. После:
`[1-14] [15-32] [33-47]` — абсолютные.

### Баг 3. Лишний импорт
```
ImportError: cannot import name 'EMBED_BATCH' from 'embedder'
```
`main.py` импортировал `EMBED_BATCH` из `embedder.py`, где его нет (модуль
читает его из env сам). **Фикс**: убрать из списка импортов. Классика:
импорт, написанный «на будущее» до того, как функция появилась.

### Баг 4. mcp 2.x: FastMCP удалён
```
No module named 'mcp.server.fastmcp'. This is mcp 2.x, where FastMCP was
renamed to MCPServer (from mcp.server.mcpserver import MCPServer) ...
pin 'mcp<2' to keep running v1 code.
```
Обнаружено **до первого запуска** — по чтению исходников SDK (raw fetch
`mcp/server/fastmcp.py` вернул заглушку с этим сообщением). Пин `mcp==2.3.0`
был выбран по свежести, а не по API. **Фикс** (4 пункта):
1. `from mcp.server.mcpserver import MCPServer`; `mcp = MCPServer("rag", …)`.
   Декораторы `@mcp.tool()` — без изменений.
2. Монтирование: `streamable_http_app()` отдаёт путь по умолчанию `/mcp`,
   поэтому саб-апп монтируется на `"/"`, а не на `"/mcp"` (иначе `/mcp/mcp`).
   REST-роуты, зарегистрированные до mount'а, имеют приоритет.
3. **Критично**: lifespan *хостового* приложения обязан войти в
   `async with mcp.session_manager.run():` — lifespan смонтированного саб-аппа
   **никогда не исполняется**. Без этого каждый запрос падает
   «Task group is not initialized».
4. Auth-обёртка пускает Bearer-проверку только на `path.startswith("/mcp")` —
   иначе 401-ки вместо 404 на несуществующих MCP-путях.

### Баг 5. `ASGI` нет в starlette
```
ImportError: cannot import name 'ASGI' from 'starlette.types'
```
Тип `ASGI` не экспортируется в установленной версии. **Фикс**: убрать
аннотацию (`def __init__(self, app) -> None`).

### Баг 6. `_read_key` использовался до определения
```
NameError: name '_read_key' is not defined   (store.py:19)
```
Модульный уровень: `QDRANT_API_KEY = _read_key(...)` стоял **выше**
`def _read_key`. **Фикс**: переставить функцию вверх. (py_compile это не
ловит — это время исполнения.)

### Баг 7. Healthcheck Qdrant: нет ни curl, ни wget, ни /dev/tcp
Первая версия: `sh -c "exec 3<>/dev/tcp/127.0.0.1/6333"` — не сработало:
```
sh: 1: cannot create /dev/tcp/127.0.0.1/6333: Directory nonexistent
```
Образ — **Debian-based** (не Alpine, как ожидалось): в `/bin` есть bash, perl,
dash. Но `sh` → dash, без bash-измов. `nc`/`nc-telnet` отсутствуют. Perl'овский
`HTTP::Tiny` — не в perl-base:
```
Can't locate HTTP/Tiny.pm in @INC
```
**Фикс**: `IO::Socket::INET` — в perl-base, проверено живьём (на закрытом
порту корректно `exit 1`):
```yaml
test:
  - CMD
  - perl
  - -MIO::Socket::INET
  - -e
  - 'exit(!IO::Socket::INET->new(PeerAddr => "127.0.0.1:6333", Timeout => 3));'
```
Методика: не гадать, а `docker run --rm --entrypoint sh <image> -c 'ls /bin …'`
и пробные запуски каждого кандидата.

### Баг 8. Compose интерполирует `$` в healthcheck
Первая perl-версия содержала `my $s = …` → compose:
```
warning msg="The \"s\" variable is not set. Defaulting to a blank string."
```
и perl получал мусор. **Фикс**: переписать однострочник **без единого `$`**
(см. баг 7). Общий урок: в compose-файлах `$` — валюта интерполяции.

### Баг 9. Embedder: чанк не влезает в physical batch
```
embedder HTTP 500: {"error":{"message":"input (575 tokens) is too large to
process. increase the physical batch size (current batch size: 512)"}}
```
В режиме `--embeddings` llama.cpp требует, чтобы **каждый инпут целиком**
влез в один physical batch. Чанки ~2000 символов ≈ 550–620 токенов > `-b 512`.
**Фикс**: `-b 2048 -ub 4096` (запас на `CHUNK_CHARS` вдвое больше).
Это поведение специфично для embeddings-режима — в chat-режиме батч делится,
здесь нет.

### Баг 10. `tuple index out of range` в upsert
Кортеж точки — 4 элемента `(pid, vector, text, payload)`, а код брал `p[4]`.
**Фикс**: `p[3]`. Простой, но проявился только на живом инджесте (на
README RAG-репо; три других файла упали раньше на баге 9 и не дошли).

### Баг 11. Qdrant point ID: hex-строка невалидна
```
400 … "value 1ade9e05809105a473c90bf835319a62c4b5b08e6238960109dee5d8350e989a
is not a valid point ID, valid values are either an unsigned integer or a UUID"
```
Дизайн-документ предполагал hex sha256. Qdrant принимает **только UUID или
uint64**. **Фикс**: `chunk_id` возвращает `uuid.UUID(hex=sha256digest[:32])` —
первые 128 бит, детерминированно, коллизий практически нет.

### Баг 12. Коллизия путей: два README.md
Самый коварный — **тихий** (без ошибок): первый поиск по «KV cache» давал
README RAG-репо, а не ai-compose. Диагностика:
```
points: 135        # ожидалось 159
paths: ['AUDIT_AND_IMPLEMENTATION_GUIDE.md', 'README.md', 'ОТЧЁТ.md']   # 3, не 4
```
`rel_path` относил путь к корню → оба `README.md` сохранялись как один путь. При
инджесте RAG README `delete_by_path("README.md")` **вырезал чанки ai-compose**
(159 − 24 = 135). **Фикс**: `rel_path` = `<имя-корня>/<внутренний-путь>`:
`ai-compose/README.md` и `rag/README.md`. `get_document` подправлен
соответственно (префикс корня опознаётся, снимается, файл ищется по всем
корням). Последствия фикса: дроп коллекции, сброс state (см. баг 14), полный
реинджест. Урок: **unique key = путь, уникальный по всему корпусу**, а не
«относительно чего-то».

### Баг 13. `ensure_collection`: 404 на пустой коллекции
```
RuntimeError: dependencies not ready: Unexpected Response: 404 (Not Found)
… Collection `docs` doesn't exist!
```
`get_collection` на отсутствующую коллекцию бросает `ResponseHandlingException`
(обёртка 404), а класс `CollectionDoesNotExistError` в qdrant-client 1.20
**не экспортируется** в ожидаемых местах:
```
ModuleNotFoundError: No module named 'qdrant_client.exceptions'
dir(qdrant_client) → []  # ни одного *Error* на верхнем уровне
```
**Фикс**: не ловить исключение, а спрашивать:
`if not self.client.collection_exists(COLLECTION): create_collection(...)`.
Плюс (баг 13b, ниже) — schema 1.20.

### Баг 13b. Schema qdrant-client 1.20: поля ушли под `.config`
```
AttributeError: 'CollectionInfo' object has no attribute 'vectors'
```
В 1.20 `CollectionInfo` = `{status, config, points_count, …}`; векторный конфиг
и метаданные — в `info.config.params.vectors` и `info.config.metadata`.
**Фикс**: читать через `.config`; `params.vectors` может быть
`VectorParams` **или** dict (named vectors) — `hasattr`-фолбэк.
Урок: версионные миграции клиентских SDK — всегда читать `model_fields`
живого объекта, а не документацию.

### Баг 14. State-volume «in use» и старые ключи
После бага 12 захотелось сбросить `rag_state` (state.json с ключами **без**
префикса корня):
```
Error response from daemon: remove rag_state: volume is in use - [container-id]
```
даже после `docker compose stop rag-core` (volume держался оставшимся
объектом). Решение **не бороться**: старые ключи никогда не совпадут с
новыми `rel_path` → каждый файл просто перечанкивается один раз. Коллекция
всё равно была сброшена. Урок: state-файл с устаревшей схемой ключей —
не катастрофа, если схема ключей монотонно расширяется.

### Инструментальные проблемы (не код)

- **aria2 resume ненадёжен**: ранее «OK»-статус на обрубленном файле. Правило:
  после любого скача — размер + sha256, сверка с LFS `oid` через HF API.
- **pkill самоубийство**: `pkill -f "bge-m3-q8_0"` убил **собственную оболочку**
  (exit 143, SIGTERM), т.к. паттерн совпал с командной строкой shell'а,
  запускающего pkill. Правило: не pkill по паттернам, вхождением в которые
  служит сама команда.
- **docker run + grep -c**: `docker run --rm --entrypoint grep -c "pat" img
  /file` → `-c` парсится как cpu-shares:
  ```
  invalid argument "MCPServer\|streamable_http_app" for "-c, --cpu-shares" flag
  ```
  **Фикс**: `--entrypoint sh img -c 'grep -c …'`.
- **MCP one-shot не работает**: Makefile-цель `search` первоначально делала
  одиночный JSON-RPC `tools/call` в `/mcp` — но streamable-HTTP требует
  handshake (`initialize` → session id → `notifications/initialized`).
  **Решение**: отдельный REST `POST /search` (тот же путь: embed → qdrant →
  hits) для CLI и верификации; MCP — только для opencode.
- **Дубликат job**: `make ingest` внутри команды, подставляющей job_id,
  создал второй job («queued»); очередь сериализует, второй ушёл за первым.
  Не баг, но в Makefile цель не должна запускать саму себя.
- **20k-лимит сессии**: несколько раз ходы обрезались на полуприписанном
  edit'е (первый фикс chunker'а не применился и пришлось перечитывать файл
  и переписывать). Правило: один edit = один ход, после edit сразу
  верификация.

### Бесполезные, но замеченные вещи

- Qdrant пишет `ERROR telemetry_reporting … (https://telemetry.qdrant.io/)` —
  internal-сеть не выходит в интернет; телеметрия не критична, лог-шум.
- `UserWarning: Api key is used with an insecure connection` — ожидаемо
  (http, не https, внутри internal-сети).

---

## 6. Верификация (что реально проверено)

### Поиск (REST `/search`, живые данные)
| Запрос | Топ-хит | Ожидалось |
|---|---|---|
| `how does the KV cache restore work` | `ai-compose/README.md` [15-32] (секция KV Cache) | ✅ |
| `snapshot restore slot action save restore VRAM` | `ai-compose/README.md` [107] | ✅ |
| `как увеличить контекст и что происходит при переполнении` | `AUDIT_AND_IMPLEMENTATION_GUIDE.md` [549-588] | ✅ кросс-лингва |

### MCP-эндпоинт (полный handshake, curl)
`initialize` → заголовок `Mcp-Session-Id` получен → `notifications/initialized`
→ `tools/list`: 3 инструмента со схемами (`search(query, collection, k, path)`,
`list_collections`, `get_document`). Сессия живая, авторизация Bearer работает.

### Инджест
`{"status":"done","chunks":159,"errors":0}` из 4 файлов; в qdrant 158 точек.
**Разгадано (2026-11-19)**: пересчёт корпуса ai-compose дал 152 чанка →
151 уникальный ID → 151 в qdrant (0 пропавших, 0 лишних). Два трёхстрочных
блока в `AUDIT…md` (строки 139–141 и 302–304) имеют идентичное тело →
один UUID → upsert-дедупликация. 159 − 1 = 158. Не баг.
Доп. сверка: ID всех 158 точек пересчитаны как `UUID(sha256(path+'\0'+body))`
от сохранённого текста — 158/158 совпали.

### Состояние коллекции
4 пути: `ai-compose/{AUDIT…md, README.md, ОТЧЁТ.md}`, `rag/README.md`.
Два README различимы (после фикса бага 12).

### Prometheus
- Таргет `http://rag-core:8090/metrics → up`.
- `up{job="rag"} = 1`.
- `search_total{job="rag"}` = 4 после четырёх живых поисков (растёт).
- `sum(ingest_chunks_total{job="rag"}) = 159`.
- `rate(search_total[5m]) > 0` в окне после поисков.

### Grafana
Дашборд `rag` (uid `rag`, папка llama.cpp) провиженится file-provider'ом
без рестарта: `/d/rag` — статус (up, queue, chunks, errors), searches/sec,
латентность, ingest-счётчики. Все выражения проверены через
`/api/v1/query` до открытия UI.

### Конфиги
`docker compose config -q` — OK в обоих репо. `git push` — оба репо на `main`.

---

## 7. Недоделки / открытые вопросы

1. ~~**158 точек vs 159 чанков**~~ — **разгадано**: два трёхстрочных блока
   в `AUDIT…md` (строки 139–141 и 302–304) имеют идентичное тело → один
   UUID → upsert-дедупликация. 159 − 1 = 158. ID всех 158 точек пересчитаны
   и сверены с сохранённым текстом: 158/158 совпали. Не баг, закрыто.
2. **`make search`** использует REST `/search`, не MCP — для CLI это верно,
   но «официальный» путь (MCP) проверялся только curl-хэндшейком, не из
   opencode. Следующий шаг: реальный вопрос через opencode с включённым MCP.
3. **Phase 2** не построена: `/v1` OpenAI-прокси + виртуальная `+rag` для
   Chatbox (rag-core → llama через ai-net, алиас `llama`; алиас уже активен
   после следующего `up` ai-compose), LAN-биндинг 8090.
4. **Алиас `llama`**: контейнер прицеплен к ai-net через `docker network
   connect` (без рестарта), но после рестарта llama-server compose подхватит
   `networks:` из файла — проверить после следующего `docker compose up`.
5. **Описание GitHub-репо** (`selfcortex/RAG`) не поставлено — нужен UI или
   PAT; текст готов в разделе 8.
6. **bge-m3 vs Qwen3-Embedding**: запасной план не понадобился; если в Phase 2
   появится деградация кросс-лингвы, вернуться.
7. ~~`ОТЧЁТ.md` с кириллицей в имени~~ — **проверено**: инджест, поиск и
   `get_document` (полный MCP handshake) по нему работают; документ
   возвращается целиком. Закрыто.

---

## 8. Уроки (для будущего / для Claude)

### По стеку
1. **Пины — это API-контракт**. `mcp==2.3.0` «по свежести» означал v2 с
   удалённым `FastMCP`. Пинить нужно после чтения migration guide, а до.
   Аналогично qdrant-client 1.20: schema `CollectionInfo` переехала.
   Правило: перед пингом — `docker run … python3 -c "import …; print(api)"`
   на реальном образе.
2. **Healthcheck'и — по содержимому образа, не по привычке**: qdrant без
   curl/wget, dash без `/dev/tcp`; спас `perl -MIO::Socket::INET` из
   perl-base. Методика: `ls /bin /usr/bin` + пробные однострочники.
3. **`$` в compose = интерполяция**: любой healthcheck-скрипт писать без
   `$` (или экранировать `$$`).
4. **Qdrant**: point ID — только UUID/uint64; `collection_exists()` вместо
   ловли 404; `config.params.vectors` может быть dict.
5. **llama.cpp `--embeddings`**: инпут обязан влезть в **один** physical
   batch → `-b` ≥ макс. чанка в токенах, а не 512 по умолчанию.
6. **Unique key индекса = глобально уникальный путь**. «Путь относительно
   корня» при нескольких корнях = молчаливое стирание данных (баг 12 —
   худший баг проекта: без единой ошибки).
7. **MCP streamable-HTTP**: handshake обязателен (один POST в `/mcp` не
   работает); host-app lifespan обязан держать `session_manager.run()`;
   монтировать на `/` с дефолтным `streamable_http_path=/mcp`.
8. **Мониторинг чужого стека**: внешняя сеть + file_sd + анонимный
   `/metrics` (ключ — только на `/mcp`) — минимум изменений в чужом репо.

### По процессу
9. **aria2**: после любого скача — размер + sha256, сверка с LFS oid.
10. **pkill по паттерну = пуля в потолок**: паттерн совпадает с собственной
    командной строкой.
11. **Импорти/схему проверять живьём**, не по документации:
    `model_fields`, `dir()`, `inspect.signature` в целевом образе.
12. **build → `docker run … import main` → up**: ловить ImportError до
    healthcheck-цикла — экономит ~8 циклов.
13. **Один edit = один ход + верификация**: 20k-лимит режет полуприписанные
    правки; перечитывать и переписывать дороже, чем дробить.
14. **Тихие баги опаснее громких**: баг 12 не дал ни одной ошибки, только
    «странное» топ-1 в поиске. Правило: после инджеста сверять
    `count` точек против ожидаемого числа чанков **и** набор путей.
15. **Минимальные правки чужого репо**: в ai-compose — 3 файла + 2 новых
    (targets/rag.yml, rag.json) + 2 коммита; ничего лишнего, мусор
    (`a.out`, `dod3`) не тронут.

---

## 9. Файлы и коммиты

### `/mnt/GameLinux/RAG` → `selfcortex/RAG` (main, 2 коммита)
```
9493efb rag: MCP search over markdown corpus (qdrant + bge-m3 CPU embedder)
<hash>  docs: tagline, real opencode MCP config, UUID point id, monitoring status
```
| Файл | Суть |
|---|---|
| `docker-compose.yml` | 3 сервиса, rag-net internal, ai-net external, healthcheck'и |
| `rag-core/Dockerfile` | python:3.12-slim + pinned pip |
| `rag-core/app/main.py` | FastAPI + MCPServer + очередь + REST (/search /ingest /health /metrics) |
| `rag-core/app/store.py` | qdrant-client, ensure/search/scroll, schema 1.20 |
| `rag-core/app/chunker.py` | блоки/фences/overlap, UUID id, абсолютные строки |
| `rag-core/app/embedder.py` | httpx → embedder, METRICS, batch |
| `Makefile` | init/up/ingest/search/check |
| `README.md`, `.env.example`, `.gitignore` | — |
| **вне git**: `secrets/`, `.env`, `REVIEW.md` (этот файл) | — |

### `~/ai-compose` → `selfcortex/rocm-qwen27b-mtp-docker` (2 коммита)
```
2e52147 rag: scrape rag-core + dashboard (external ai-net)
bae61eb run.sh: mark started=1 before compose up (Ctrl+C during startup counts)
```
Изменено: `docker-compose.yml` (llama-server: ai-net+алиас; prometheus: ai-net),
`prometheus/prometheus.yml` (job rag), **новое**: `prometheus/targets/rag.yml`,
`grafana/dashboards/rag.json`.

### Вне репо
- `~/.config/opencode/opencode.json` — `mcp.rag` (Bearer через `{file:…}`).
- `/mnt/AI/models/bge-m3-q8_0.gguf` — модель, sha256 верифицирован.

### Описание для GitHub (вставить в Settings → Description)
```
MCP vector search for opencode: Qdrant + CPU bge-m3 embedder, plugs into the llama.cpp stack via shared ai-net
```

---

## 10. Если читать только один раздел

**Раздел 5** (журнал проблем) — там вся правда: 14 багов с точными ошибками.
Для Claude-анализа наиболее показательные: **баг 4** (пин без чтения
migration guide), **баг 12** (молчаливое стирание данных через не-уникальный
ключ) и **баг 9** (специфика `--embeddings` llama.cpp).
