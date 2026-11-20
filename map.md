# Карта инфраструктуры: ai-compose + websearch + RAG

> Личный документ (gitignored). Дата: 2026-11-20.
> Все порты, сети и потоки проверены по живым compose-файлам и `docker network inspect`,
> а не по памяти.

---

## 0. Хост

| Что | Значение |
|---|---|
| ОС | CachyOS (Linux), Docker 29.9.3, Compose v2 |
| CPU / RAM | 12 cores / 31 GiB |
| GPU | RX 6800 XT, ROCm gfx1030 — занята основной моделью 27B |
| IP | LAN: `192.168.1.215` (в .env стеков как `LAN_IP`), localhost `127.0.0.1` |
| Модели | `/mnt/AI/models/bge-m3-q8_0.gguf` (~605 MiB), GGUF Qwen 3.8 27B (ai-compose) |

Три репозитория, три независимых compose-стека, один хост:

| Стек | Каталог | Git-remote | Назначение |
|---|---|---|---|
| **ai-compose** | `~/ai-compose` | `selfcortex/rocm-qwen27b-mtp-docker` | LLM-сервер (llama.cpp, GPU) + весь мониторинг (Prometheus, Grafana, node-exporter) |
| **RAG** | `/mnt/GameLinux/RAG` | `selfcortex/RAG` | Векторный поиск по докам: rag-core (MCP) + Qdrant + CPU-эмбеддер bge-m3 |
| **websearch** | `~/websearch` | `selfcortex/websearch` | Веб-поиск для агентов: SearXNG (метапоиск) + Crawl4AI (страница → markdown) |

---

## 1. Общая схема

```
                         ХОСТ 192.168.1.215 (CachyOS, 12c/31G, RX 6800 XT)

 ┌─────────────────────────── ПОЛЬЗОВАТЕЛЬ / СЕТИ ───────────────────────────┐
 │  Браузер LAN:  Grafana :3000 · LLM API :8080 (Chatbox и др.)              │
 │  Локально:     LLM API 127.0.0.1:8080 · RAG 127.0.0.1:8090                │
 │                SearXNG 127.0.0.1:8081 · Crawl4AI 127.0.0.1:11235          │
 └───────────────────────────────────────────────────────────────────────────┘

 ┌────────────── A. ОПЕНКОД (агент, на хосте) ───────────────────────────────┐
 │                                                                            │
 │   opencode ──MCP /mcp (Bearer, {file:secrets/rag_api_key})──┐              │
 │        │                                                    │              │
 │        └──shell curl (SearXNG JSON → Crawl4AI)──────────────┼──► websearch│
 └─────────────────────────────────────────────────────────────┼─────────────┘
                                                               │
        ┌──────────────────────────────────────────────────────┘
        ▼
 ╔══════════════════ ai-net (external bridge, общая точка стыка) ══════════╗
 ║                                                                          ║
 ║   ┌─────────────┐   DNS "rag-core"    ┌──────────────────────────────┐  ║
 ║   │ rag-core    │◄────────────────────│ llama-prometheus             │  ║
 ║   │ (RAG)       │  GET /metrics       │ (ai-compose)  :9090          │  ║
 ║   └──┬──────┬───┘  (анонимно)         └──────────────────────────────┘  ║
 ║      │      │      DNS "llama" (Phase 2, алиас жив)                       ║
 ║      │      └────────────────► llama-qwen27b (пока НЕ используется)       ║
 ║      │                                                       :8080        ║
 ╚══════┼═══════════════════════════════════════════════════════╪═══════════╝
        │                                                       │
        │ rag-net (internal: true — нет хоста, нет интернета)   │ ai-compose_default
        ▼                                                       ▼
 ┌──────────────────────┐                        ┌────────────────────────────┐
 │ rag-embedder :8081   │                        │ llama-qwen27b (llama.cpp)  │
 │ bge-m3 q8_0, CPU     │                        │ Qwen 3.8 27B, ROCm/GPU     │
 └──────────────────────┘                        │ 127.0.0.1:8080 + LAN:8080 │
 ┌──────────────────────┐                        └────────────────────────────┘
 │ rag-qdrant :6333     │   volume, Bearer-ключ внутри
 └──────────────────────┘
```

---

## 2. Сети Docker

| Сеть | Тип | Кто создал | Участники | Зачем |
|---|---|---|---|---|
| `ai-net` | external bridge | `make init` в RAG (`docker network create ai-net`), обе стороны объявляют её `external: true` | `rag-core`, `llama-qwen27b` (алиас **`llama`**, прицеплен через `docker network connect --alias llama`, в compose ai-compose записано — подхватится после следующего `up`), `llama-prometheus` | Единственная общая точка стыка: Prometheus → rag-core (скрейп) и rag-core → llama (Phase 2) |
| `ai-compose_default` | bridge | compose ai-compose | llama-qwen27b, llama-node-exporter, llama-prometheus(-init), llama-grafana | Внутренняя жизнь стека: prometheus → node-exporter/llama-server по имени сервиса; grafana → prometheus:9090 |
| `rag_rag-net` | bridge, **`internal: true`** | compose RAG | rag-embedder, rag-qdrant, rag-core | `internal: true` = **нет маршрута наружу**: embedder и qdrant физически недоступны с хоста и из интернета, не могут сами выйти в интернет. rag-core — мост (он же на ai-net) |
| `websearch_search-net` | bridge | compose websearch | searxng, crawl4ai | Изолированный стек; наружу торчат только 127.0.0.1-порты |

Доступность по сетям:

```
                 ai-net    rag-net    ai-compose_def    search-net
rag-core           ✔          ✔           —                —
rag-embedder       —          ✔           —                —
rag-qdrant         —          ✔           —                —
llama-qwen27b      ✔ (llama)  —           ✔                —
llama-prometheus   ✔          —           ✔                —
llama-grafana      —          —           ✔                —
llama-node-exp.    —          —           ✔                —
searxng            —          —           —                ✔
crawl4ai           —          —           —                ✔
```

Вывод: websearch полностью обособлен (агент достает его только через localhost-порты).
RAG-векторная часть (embedder/qdrant) закрыта internal-сетью. Общие окна наружу:
`8080` (llama), `3000` (Grafana, только LAN), `8090` (RAG, только localhost).

---

## 3. Контейнеры, порты, образы

### 3.1 ai-compose (`~/ai-compose`)

| Контейнер | Образ (pinned) | Порты | Аутентификация | Роль |
|---|---|---|---|---|
| `llama-qwen27b` | `ghcr.io/ggml-org/llama.cpp@sha256:6da04cde…` (ROCm) | `127.0.0.1:8080` и `192.168.1.215:8080` → 8080 | API-ключ **на всех роутах**, кроме `/health` и UI-ассетов | LLM: Qwen 3.8 27B, GPU, MTP, KV-квант; OpenAI-совместимый API |
| `llama-node-exporter` | `prom/node-exporter:v1.12.1@sha256:1b4e44…` | нет (только сеть стека, 9100) | нет | Метрики хоста: CPU/RAM/диски/сеть |
| `llama-prometheus` | `prom/prometheus:v3.16.0@sha256:e04987…` | нет (только сеть стека, 9090; снаружи — через Grafana proxy или docker) | сам без; скрейпит с ключами (см. §4) | TSDB: retention 2d + 2GB, read_only rootfs, cap_drop ALL |
| `llama-prometheus-init` | тот же prometheus | — | — | one-shot: `chown 65534:65534 /data` |
| `llama-grafana` | `grafana/grafana:13.3.1@sha256:0615ec…` | `192.168.1.215:3000` → 3000 (только LAN, не 0.0.0.0) | admin из `GRAFANA_USER/PASSWORD` (.env) | UI: дашборды из `grafana/dashboards/*.json`, datasource `prometheus` (uid `prometheus`) → `http://prometheus:9090` |

Volumes: `prom-data`, `grafana-data`. Секреты: `~/ai-compose/secrets/llama_api_key`
(монтируется и в llama-server, и в prometheus — один ключ на оба конца скрейпа).

### 3.2 RAG (`/mnt/GameLinux/RAG`)

| Контейнер | Образ (pinned) | Порты | Аутентификация | Роль |
|---|---|---|---|---|
| `rag-embedder` | тот же ROCm-образ llama.cpp, `--embeddings -ngl 0 -c 8192 -b 2048 -ub 4096` | только rag-net: 8081 | **нет** (internal-сеть = недосягаем снаружи) | bge-m3 q8_0 на CPU: 1024-dim, кросс-лингва |
| `rag-qdrant` | `qdrant/qdrant@sha256:d2775927…` | только rag-net: 6333 | Bearer-ключ даже внутри (`QDRANT__SERVICE__API_KEY_FILE` → `secrets/qdrant_api_key`) | Векторная БД, collection `docs`, volume |
| `rag-core` | `rag-core:local` (собственный Dockerfile: `python:3.12-slim@sha256:5e2bdb…`, pinned pip) | `127.0.0.1:8090` → 8090 | Bearer `secrets/rag_api_key` — **только на `/mcp*`**; `/metrics` анонимный; `/search`, `/ingest` — с ключом | Оркестратор: FastAPI + MCP v2 + очередь + chunker |

Volumes: `rag_state` (state.json: path → mtime/hash), volume qdrant.
Korpus-маунты (read-only): `/corpus/ai-compose` ← `~/ai-compose`, `/corpus/rag` ← `/mnt/GameLinux/RAG`.
Pinned pip: fastapi 0.165.0, uvicorn 0.54.0, **mcp 2.3.0**, qdrant-client 1.20.0, httpx 0.28.1.

### 3.3 websearch (`~/websearch`)

| Контейнер | Образ | Порты | Аутентификация | Роль |
|---|---|---|---|---|
| `searxng` | `searxng/searxng:latest` | `127.0.0.1:8081` → 8080 | `SEARXNG_SECRET` (.env); JSON-формат включён (`formats: html, json`) | Метапоиск: Google, Bing, DuckDuckGo, Startpage, Qwant + GitHub, StackOverflow |
| `crawl4ai` | `unclecode/crawl4ai:latest` | `127.0.0.1:11235` | `Authorization: Bearer $CRAWL4AI_API_TOKEN` (.env); mem_limit 3g, shm 1g | Headless-браузер: URL → markdown для LLM |

Свой `.env`: `SEARXNG_SECRET`, `CRAWL4AI_API_TOKEN`.

### 3.4 Сводная таблица портов

| Порт (хост) | Бинд | Сервис | Протокол | Кто ходит |
|---|---|---|---|---|
| `8080` | 127.0.0.1 + `192.168.1.215` | llama-server | OpenAI-совместимый HTTP API, Bearer | Chatbox/клиенты с LAN; агент локально |
| `3000` | `192.168.1.215` | Grafana | HTTP, basic auth | браузер с LAN |
| `8090` | **только 127.0.0.1** | rag-core | MCP streamable-HTTP `/mcp` + REST `/search` `/ingest` `/health` `/metrics` | opencode (MCP); человек (curl/Makefile); Prometheus (через ai-net, без хост-порта) |
| `8081` | **только 127.0.0.1** | SearXNG | HTTP, JSON API | агент (curl), человек |
| `11235` | **только 127.0.0.1** | Crawl4AI | HTTP, Bearer | агент (curl), человек |
| `9090` | не опубликован | Prometheus | HTTP | только grafana (proxy) и docker exec |
| `9100` | не опубликован | node-exporter | Prometheus text | только prometheus |
| `8081` (в rag-net) | не опубликован | rag-embedder | llama.cpp embeddings API | только rag-core |
| `6333` (в rag-net) | не опубликован | rag-qdrant | gRPC/REST | только rag-core |

---

## 4. Поток A: агент → RAG (MCP) — «запомни/найди в наших доках»

```
opencode (агент)
   │  HTTP POST http://127.0.0.1:8090/mcp
   │  Headers: Authorization: Bearer <secrets/rag_api_key>   (опencode подставляет из файла {file:...})
   │  Accept: application/json, text/event-stream
   ▼
rag-core (FastAPI)
   │  1) handshake: initialize → заголовок Mcp-Session-Id → notifications/initialized
   │  2) tools/call  search(query, collection="docs", k, path?)
   │
   │  а) embed: POST http://embedder:8081 (rag-net, без ключа)
   │       llama.cpp --embeddings, bge-m3, CPU → вектор 1024-dim (~10-30 мс на чанк)
   │  б) search: POST http://qdrant:6333/collections/docs/points/search
   │       Bearer <secrets/qdrant_api_key> → топ-k точек (score, path, section, text, lines)
   ▼
ответ агенту: [ {score, path, section, text, start_line, end_line}, … ]
```

Тайминги (живые замеры): embed ~10-20 мс, поиск ~5 мс, **итог ~25 мс/запрос**
(`search_seconds_total = 0.56с` на 22 запроса).

Инструменты MCP: `search`, `list_collections`, `get_document`
(`get_document(path)` — полный текст файла; кириллические имена работают).
REST-дубли для человека: `POST /search`, `POST /ingest` (Makefile `make search`).

**Индексация** (отдельный поток, по требованию, не в MCP-пути):
```
make ingest → POST /ingest → rag-core:
  для каждого .md в /corpus/*:
    state.json: mtime/hash не изменился → skip
    иначе: chunker (блоки по заголовкам/абзацам, ~2000 символов, overlap 1/3,
            fence/таблицы не режутся, breadcrumb «path > Section»)
    → embedder (батчами EMBED_BATCH=16) → qdrant: delete_by_path + upsert
    → point id = UUID(sha256(path+'\0'+body)[:16]) — детерминированный, idempotent
```
Корпус: `ai-compose/` (3 файла) + `rag/README.md` = 4 файла → 158 точек (159 чанков,
один дедуп — два идентичных трёхстрочных блока в AUDIT…md).

**Важно:** в MCP-пути rag-core **никогда не звонит в llama-server** — только в embedder.
Модель (агент) сама решает, когда искать; генерация ответа — дело LLM.

---

## 5. Поток B: агент → websearch — «найди в интернете»

```
opencode (агент, через shell/curl — MCP-сервера у websearch нет)
   │
   ├─1─► GET/POST http://127.0.0.1:8081/search?q=...&format=json
   │      SearXNG → агрегирует Google/Bing/DDG/Startpage/Qwant (+GitHub, SO)
   │      ◄── JSON: список результатов с URL
   │
   ├─2─► POST http://127.0.0.1:11235/crawl
   │      Authorization: Bearer $CRAWL4AI_API_TOKEN
   │      {"urls": ["https://…"]} → headless-браузер рендерит страницу
   │      ◄── markdown страницы
   │
   └─3─► markdown в контекст LLM → агент отвечает
```

Типовой пайплайн: `SearXNG /search` → URL → `Crawl4AI /crawl` → markdown.
Стек обособлен: не на ai-net, не мониторится, наружу только localhost.

---

## 6. Поток C: мониторинг — кто кого скрейпит

```
llama-prometheus (:9090, в ai-compose_default И на ai-net)
   │  scrape_interval 15s, retention 2d/2GB
   │
   ├─ job "node" ──► node-exporter:9100        (ai-compose_default, без auth)
   │                 CPU/RAM/диски/сеть хоста
   │
   ├─ job "llama" ──► llama-server:8080        (ai-compose_default, file_sd targets/llama.yml)
   │                 Bearer <secrets/llama_api_key> (credentials_file, перечитывается
   │                 на каждый скрейп — ротация ключа без reload)
   │                 токены/s, VRAM, KV-cache, слоты
   │
   └─ job "rag" ────► rag-core:8090            (через ai-net! file_sd targets/rag.yml)
                     без auth (/metrics анонимный, ключ только на /mcp)
                     search_total, embed_seconds, ingest_*, queue_size
   ▼
llama-grafana (:3000, LAN)
   datasource "prometheus" → http://prometheus:9090 (proxy, uid prometheus)
   дашборды из grafana/dashboards/: основной llama + rag.json (uid rag, папка llama.cpp)
```

Ключевое: Prometheus прицеплен к `ai-net` **только ради** job `rag` — до rag-core
из своего стека ему не добраться. Это и есть цена интеграции RAG в чужой стек:
одна внешняя сеть + один file_sd-файл + один дашборд.

---

## 7. Поток D: пользователь → LLM (Chatbox и т.п.)

```
Браузер/Chatbox на LAN ──► 192.168.1.215:8080 ──► llama-qwen27b
   Bearer <llama_api_key>; OpenAI-совместимый API:
   /v1/chat/completions, /v1/completions, /v1/models, /metrics
```

LLM-модель: Qwen 3.8 27B, MTP-слои, KV q4_0, контекст ~81k (см. ai-compose README).
GPU полностью отдана этой модели; RAG-эмбеддер намеренно на CPU.

---

## 8. Планируемое (Phase 2) — как всё свяжется дальше

```
Phase 2 (не построено):
  rag-core ──(ai-net, DNS "llama")──► llama-qwen27b:8080
    — OpenAI-прокси /v1 в rag-core: агент ходит в ОДНУ точку,
      rag-core сам решает: поиск → inject в контекст → запрос к LLM
    — виртуальная модель "+rag" для Chatbox
    — LAN-биндинг 8090 (сейчас только 127.0.0.1)
  Алиас llama уже активен (docker network connect); после следующего
  `docker compose up` в ai-compose compose подхватит его из файла.
```

---

## 9. Секреты и где они лежат

| Секрет | Файл | Кто использует | Как передается |
|---|---|---|---|
| `rag_api_key` | `/mnt/GameLinux/RAG/secrets/rag_api_key` | opencode → rag-core; человек в curl | Bearer; в opencode через `{file:...}` (ключ не в конфиге) |
| `qdrant_api_key` | `/mnt/GameLinux/RAG/secrets/qdrant_api_key` | rag-core → qdrant | Bearer; монтируется в контейнер файлом |
| `llama_api_key` | `~/ai-compose/secrets/llama_api_key` | клиенты LLM; prometheus → llama-server | Bearer; один файл, два маунта |
| `SEARXNG_SECRET` | `~/websearch/.env` | searxng (session) | env |
| `CRAWL4AI_API_TOKEN` | `~/websearch/.env` | агент → crawl4ai | Bearer |
| `GRAFANA_USER/PASSWORD`, `LAN_IP` | `~/ai-compose/.env` | grafana, порты | env |

Все `secrets/` и `.env` — gitignored во всех трёх репо.

---

## 10. Жизненный цикл и порядок запуска

```
1. ai-compose:  docker compose up -d     (или run.sh — GPU-модель + мониторинг)
                llama-qwen27b → GPU, ~минута на загрузку 27B
2. RAG:         make init    (one-shot: .env, оба ключа, docker network create ai-net)
                make up      (build rag-core; embedder+qdrant+core; healthcheck'и)
                make ingest  (индексация корпуса)
3. websearch:   docker compose up -d     (самодостаточный)
4. opencode:    mcp.rag в ~/.config/opencode/opencode.json → готово
```

Зависимости: RAG требует `ai-net` (создается `make init`; ai-compose объявляет её
`external` и не падает, если сеть есть). ai-compose и websearch друг от друга не
зависят. Единственная «мягкая» связь: алиас `llama` на ai-net (Phase 2).

---

## 11. Быстрая навигация: «а где это?»

| Вопрос | Ответ |
|---|---|
| Где LLM отвечает? | `llama-qwen27b:8080` (LAN + localhost), Bearer |
| Где агент ищет в доках? | opencode → MCP → `rag-core:8090/mcp` |
| Где вектора? | `rag-qdrant:6333`, collection `docs`, 1024-dim, 158 точек |
| Где эмбеддинги считаются? | `rag-embedder:8081`, bge-m3 q8_0, CPU, тот же образ llama.cpp |
| Где веб-поиск? | `searxng:8081` + `crawl4ai:11235` (localhost only) |
| Где графики? | Grafana `http://192.168.1.215:3000`, дашборд RAG: `/d/rag` |
| Где сырые метрики? | Prometheus:9090 (не опубликован — через Grafana proxy) |
| Как rag-core видит llama? | ai-net, DNS-имя `llama` (Phase 2) |
| Как prometheus видит rag-core? | ai-net, `targets/rag.yml` → `rag-core:8090` |
| Что снаружи точно недоступно? | embedder, qdrant (internal-сеть), prometheus, node-exporter, llama-ключ |
| Разбор RAG-сборки | `REVIEW.md` (рядом, тоже gitignored) |

---

## 12. Стек Dify (добавлен 2026-11-20, ADR-013)

GUI-стек для индексации документов (в т.ч. PDF): `/mnt/GameLinux/dify`,
compose Dify 1.20.0 (релиз 2026-11-18). **Отдельный самодостаточный стек**;
RAG-стек не тронут.

| Контейнер | Роль |
|---|---|
| `dify-api-1` (healthy) | backend, :5001 внутри |
| `dify-web-1` | console UI |
| `dify-worker-1`, `dify-worker_beat-1` | асинхронная обработка документов |
| `dify-plugin_daemon-1` | плагины (5003 наружу **не** опубликован) |
| `dify-sandbox-1`, `dify-local_sandbox-1`, `dify-agent_backend-1`, `dify-agent_ssrf_proxy-1` | sandbox/агенты |
| `dify-db_postgres-1` (healthy) | метаданные (вместо MySQL из RAGFlow) |
| `dify-weaviate-1` | вектор-стор (свой, изолирован от Qdrant) |
| `dify-redis-1` (healthy), `dify-ssrf_proxy-1`, `dify-nginx-1` | кэш, прокси, GUI |

Порты наружу: **только** `192.168.1.215:8888` (LAN, nginx → GUI; не 0.0.0.0,
не localhost — переведено с localhost 2026-11-20). 443 и 5003 сняты.
Сеть `dify_default`: к ней подключён `rag-embedder` с алиасом **`embedder`**
(`docker network connect --alias embedder dify_default rag-embedder`) —
проверено: POST `embedder:8081/v1/embeddings` → 1024-dim (2026-11-20).

Провайдеры (ввести в Dify Console → Model Providers → OpenAI-API-compat):
- **Embedding**: `http://embedder:8081/v1`, модель `bge-m3`, dim 1024, ключ — любая непустая строка.
- **LLM**: `http://192.168.1.215:8080/v1`, модель `qwen-27b`, ключ — `~/ai-compose/secrets/llama_api_key`.

Два векторных хранилища осознанно (ADR-013): Qdrant (агент/MCP) и Weaviate
(человек/GUI) — изолированы стеками.
