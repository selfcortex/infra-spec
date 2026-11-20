# Задачи

> L3. Статусы: `done` / `open` / `deferred` / `dropped`.
> Каждая задача — дата + ссылка (коммит, файл, ADR).

---

## Done (закрыто)

### RAG Phase 1 — MCP-поиск (2026-11-19, `selfcortex/RAG` main)
- [x] Стек из 3 контейнеров: rag-core / rag-embedder (bge-m3, CPU) / rag-qdrant — healthy
- [x] Индексация корпуса: 4 файла → 159 чанков → 158 точек, 0 ошибок
- [x] Разгадано 158 vs 159: два идентичных трёхстрочных блока в `AUDIT…md` (строки 139–141, 302–304) → один UUID → upsert-дедуп
- [x] Верификация ID: пересчёт `UUID(sha256(path+'\0'+body))` из сохранённого текста — 158/158 совпали
- [x] Поиск: EN, RU, смешанные — топ-хиты корректны; кросс-лингва RU→EN работает
- [x] MCP handshake: initialize → session id → initialized → tools/list (3 инструмента) → tools/call
- [x] `get_document` по кириллическому имени (`ОТЧЁТ.md`) — полный документ через MCP
- [x] REST `/search` для CLI (Makefile `make search`)
- [x] 14 багов найдено и исправлено — полный список в `postmortem-rag-build.md` §5

### Интеграция в ai-compose (2026-11-19, `selfcortex/rocm-qwen27b-mtp-docker` main)
- [x] Коммит `2e52147` «rag: scrape rag-core + dashboard (external ai-net)»:
  llama-server на ai-net (алиас `llama`), prometheus на ai-net, job `rag`,
  `targets/rag.yml`, `dashboards/rag.json`
- [x] Коммит `bae61eb` «run.sh: mark started=1 before compose up» (чужая правка, отдельно)
- [x] Runtime: `docker network connect --alias llama ai-net llama-qwen27b` (без рестарта LLM)
- [x] Prometheus: `up{job="rag"}=1`, `search_total` растёт (4 → 22 за проверки),
  `sum(ingest_chunks_total)=159`
- [x] Grafana: дашборд uid `rag`, папка llama.cpp, `/d/rag`, подхвачен file-provider'ом без рестарта

### Репозитории (2026-11-19)
- [x] `selfcortex/RAG` — создан, 3 коммита (код, docs-фиксы, gitignore REVIEW.md)
- [x] `selfcortex/rocm-qwen27b-mtp-docker` — запушены коммиты RAG-интеграции
- [x] Модель bge-m3: скачана, sha256 сверен с HF LFS oid (`aa473d51…`)

### Документация (2026-11-20)
- [x] `postmortem-rag-build.md` (566 строк: 14 багов, уроки, верификация)
- [x] `map.md` — карта трёх стеков (сети, порты, потоки A–D, секреты)
- [x] `principles.md`, `decisions.md`, `tasks.md`, `runbook.md`, `README.md` — этот репо

---

## Open (открыто)

| # | Задача | Где | Примечание |
|---|---|---|---|
| 1 | Живой тест RAG из opencode (реальный вопрос с включённым MCP, не curl) | RAG | MCP-эндпоинт проверен handshake'ом; из агента — ещё нет |
| 2 | Description GitHub-репо `selfcortex/RAG` — вставить текст | UI GitHub | Текст в `map.md` §11 / был в postmortem §9; без PAT только через UI |
| 3 | Проверить алиас `llama` после следующего `docker compose up` ai-compose | ai-compose | Сейчас активен только runtime-командой |
| 4 | Push этого репо (`infra-spec`) на GitHub | здесь | Репозиторий создан; remote + push |

---

## Deferred (отложено)

### Phase 2: LLM-интеграция RAG
| # | Задача | Блокер/зависимость |
|---|---|---|
| 5 | OpenAI-прокси `/v1` в rag-core: агент ходит в одну точку, rag-core сам решает search → inject → llama | алиас `llama` (активен), `secrets/llama_api_key` доступный rag-core |
| 6 | Виртуальная модель `+rag` для Chatbox | #5 |
| 7 | LAN-биндинг rag-core:8090 (сейчас только 127.0.0.1) | решение по auth (Bearer уже есть) |
| 8 | Qwen3-Embedding-0.6B (instruction-префикс) — запасной эмбеддер | деградация кросс-лингвы bge-m3 (пока не наблюдается) |

### Наблюдаемость
| # | Задача | Блокер |
|---|---|---|
| 9 | Alertmanager (rules/ в prometheus пуст) | приоритет; job'а уже есть |
| 10 | Мониторинг websearch (SearXNG/Crawl4AI не скрейпятся) | осознанно: не LLM-нагрузка; при росте — те же паттерны file_sd |

---

## Dropped (отброшено)

| # | Что | Почему | Дата |
|---|---|---|---|
| 1 | LLM-прокси + виртуальная модель `+rag` в Phase 1 | MCP-first: агент сам решает; прокси = лишняя точка отказа (ADR-002) | 2026-11 |
| 2 | Chatbox-обёртка над RAG | то же; Chatbox остаётся на прямом `llama:8080` | 2026-11 |
| 3 | Отдельный Prometheus/Grafana для RAG | дублирование мониторинга запрещено (принцип 1.4) | 2026-11 |
| 4 | Redis для очереди инджеста | корпус = десятки файлов; in-memory достаточно (ADR-008) | 2026-11 |
| 5 | XLM-RoBERTa / mRoPE-путь в llama.cpp | bge-m3 заработал с первого раза, запасной план не понадобился (ADR-001) | 2026-11 |
| 6 | FastMCP (mcp v1) | v2 принят как есть (ADR-005); откат на `mcp<2` не рассматривался | 2026-11 |
| 7 | `0.0.0.0`-биндинг RAG | безопасность: Phase 1 только localhost (принцип 1.7) | 2026-11 |
