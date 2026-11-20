# Runbook — эксплуатация

> L3. Операции: запуск, ротация, реинджест, «что делать если X».
> Команды проверяются на том, что реально лежит в стеках.

---

## 1. Запуск с нуля (порядок важен)

```bash
# 1. LLM + мониторинг (самодостаточный)
cd ~/ai-compose
make init            # .env, секреты, volumes
docker compose up -d # llama-qwen27b (GPU, ~минута), prometheus, grafana, node-exporter

# 2. RAG (требует ai-net)
cd /mnt/GameLinux/RAG
make init            # .env, secrets/{rag_api_key,qdrant_api_key}, docker network create ai-net
make up              # build rag-core; embedder+qdrant; ждёт healthy
make ingest          # индексация корпуса (idempotent, patch-режим)

# 3. websearch (самодостаточный, независим)
cd ~/websearch
docker compose up -d

# 4. Агент
#    mcp.rag уже в ~/.config/opencode/opencode.json — opencode готов
```

Зависимости: RAG → `ai-net` (создаётся в `make init` RAG; ai-compose объявляет
сеть `external` и не создаёт её сам). ai-compose и websearch не зависят от RAG.

## 2. Проверка здоровья

```bash
docker compose ps                 # во всех трёх: все (healthy)
curl -s http://127.0.0.1:8090/health          # rag-core: {"status":"ok",...}
curl -s http://192.168.1.215:3000/api/health  # grafana
# Prometheus: job'ы up (через Grafana proxy или docker exec)
docker exec llama-prometheus wget -qO- http://localhost:9090/api/v1/query?query=up
```

## 3. Повседневные операции

### Поиск (человек, без MCP)
```bash
cd /mnt/GameLinux/RAG
make search Q="как работает KV cache"     # REST /search, Bearer из secrets
```

### Переиндексация
```bash
cd /mnt/GameLinux/RAG
make ingest                # patch-режим: только изменённые файлы (state.json)
# полный рескан всего корпуса:
curl -s -X POST http://127.0.0.1:8090/ingest \
  -H "Authorization: Bearer $(cat secrets/rag_api_key)" -H 'Content-Type: application/json' -d '{}'
```

### После инджеста — ОБЯЗАТЕЛЬНО count check
```bash
docker compose run --rm --entrypoint python3 rag-core -c "
from store import Store
s = Store()
pts, _ = s.client.scroll('docs', limit=500, with_payload=True, with_vectors=False)
from collections import Counter
print('points:', len(pts), 'paths:', dict(Counter(p.payload['path'] for p in pts)))
"
```
Сверить: точки = чанки − дедупы; пути = все ожидаемые файлы (принцип 2.2).

### Веб-поиск (агентный пайплайн)
```bash
curl "http://localhost:8081/search?q=<query>&format=json"       # SearXNG → URL
curl -X POST http://localhost:11235/crawl \
  -H "Authorization: Bearer $CRAWL4AI_API_TOKEN" -H 'Content-Type: application/json' \
  -d '{"urls":["https://..."]}'                                 # Crawl4AI → markdown
```

## 4. Ротация ключей

| Ключ | Где | Как |
|---|---|---|
| `rag_api_key` | `secrets/rag_api_key` | перезаписать файл; rag-core читает при старте → `docker compose restart rag-core`; opencode подхватит при следующем старте (ключ из `{file:...}`) |
| `qdrant_api_key` | `secrets/qdrant_api_key` | перезаписать + `docker compose restart rag-core rag-qdrant` (qdrant читает из env при старте) |
| `llama_api_key` | `~/ai-compose/secrets/llama_api_key` | перезаписать; prometheus перечитывает на каждый скрейп (credentials_file) — без рестарта; клиенты (Chatbox) — обновить сами |

## 5. Что делать если X

| Симптом | Диагностика | Действие |
|---|---|---|
| `up{job="rag"}=0` | `docker network inspect ai-net` — rag-core там? | `docker network connect ai-net rag-core` (или `docker compose up -d rag-core`) |
| Target rag в Prometheus, но ошибки auth | job `rag` не должен иметь credentials | проверить `prometheus.yml`: job rag без `authorization` |
| MCP: `Session not found` | сессия истекла/не создана | повторить handshake: initialize → забрать `mcp-session-id` из заголовка → initialized → call |
| MCP: `Task group is not initialized` | lifespan хоста не вошёл в `session_manager.run()` | проверить main.py (ADR-005); это баг, не эксплуатация |
| Инджест: embedder 500 «input too large» | чанк > physical batch | поднять `-b` (сейчас 2048) или `CHUNK_CHARS`; не `-ub` |
| Инджест: точки не появились | `docker compose logs rag-core` | чаще всего: qdrant 401 (ключ), embedder down (healthcheck) |
| Точек меньше ожидаемого | count check (§3) | коллизия путей (ADR-007)? Дедуп идентичных тел (норма)? |
| Volume `in use` при удалении | `docker volume ls` + `docker ps` | не бороться: state-схема расширяется монотонно (postmortem баг №14); просто реинджест |
| Qdrant healthcheck падает | `docker exec rag-qdrant ls /bin` | healthcheck на `perl -MIO::Socket::INET` (в образе нет curl/wget; без `$` в compose-строке) |
| Grafana не видит новый дашборд | `curl http://192.168.1.215:3000/api/search?query=rag` | file-provider смотрит в `grafana/dashboards/`; файл валидный JSON? uid уникален? |
| llama не отвечает, RAG работает | раздельные проблемы: RAG не зависит от llama (Phase 1) | `docker compose -f ~/ai-compose/docker-compose.yml ps` |
| aria2 «OK», файл битый | размер + sha256 | скачать заново; сверять с LFS oid через HF API (принцип 2.3) |
| `pkill` убил shell (exit 143) | паттерн совпал с cmdline shell'а | не pkill по таким паттернам; `docker stop`/`kill` по PID |

## 6. Резервные копии

| Что | Где | Как восстановить |
|---|---|---|
| Qdrant-данные | volume rag (qdrant) | `docker run` qdrant с тем же volume; или просто `make ingest` (полный рескан идемпотентен) |
| State (mtime/hash) | volume `rag_state` | потерять безопасно — полный рескан |
| Модели | `/mnt/AI/models/` | скачати заново (sha256 известен) |
| Секреты | `secrets/` в обоих репо | `make init` перегенерирует (MCP-ключ потребует перезапуска rag-core) |
| Мониторинг | volume `prom-data` (retention 2d) | не критично — история 2 дня |

**Главное**: RAG полностью воспроизводим — `make init && make up && make ingest`
даёт тот же результат (детерминированные point id, ADR-006).

## 7. Обновление документации

Любое изменение инфраструктуры (compose, порты, сети, сервисы, ключи):
1. правка compose в соответствующем репо → коммит;
2. правка `map.md` (факты) и, если изменилось *почему* — `decisions.md` (новый ADR
   или `superseded`);
3. статусы в `tasks.md`;
4. коммит в `infra-spec`.

Документ без привязки к коммиту = устаревший (правило README).
