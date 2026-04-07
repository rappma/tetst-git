# Corporate KB MVP v2 — пошаговый how-to для новичка

## 1. Что это за версия
Это уже не пустой каркас. В этой версии есть:
- PostgreSQL как база метаданных и источник правды;
- OpenSearch как поисковый слой;
- worker, который обрабатывает PDF по заданиям ingestion;
- индекс chunk-ов в OpenSearch;
- API поиска;
- API чата с цитатами;
- минимальный web UI.

Но есть важное ограничение:
- полноценный OCR для сканов в этой версии еще не доведен;
- если PDF без извлекаемого текста, worker пометит проблему как `OCR_REQUIRED`.

## 2. Как назвать тестовые PDF правильно
Лучший вариант имени файла:

```text
DOC-001__v1__Положение о резервном копировании.pdf
```

Где:
- `DOC-001` — код документа;
- `v1` — версия;
- `Положение о резервном копировании` — заголовок.

Если файл назвать как попало, система тоже проглотит его, но код и название документа будут выведены автоматически, хуже и грязнее.

## 3. Подготовка сервера
Минимум:
- Ubuntu 22.04 LTS
- 4 vCPU
- 8 GB RAM
- 50 GB SSD

## 4. Установка Docker
Выполняйте в терминале сервера.

```bash
sudo apt remove docker.io docker-compose docker-compose-v2 podman-docker containerd runc
sudo apt update
sudo apt install -y ca-certificates curl gnupg jq
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu   $(. /etc/os-release && echo "$VERSION_CODENAME") stable" |   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Проверка:
```bash
docker --version
docker compose version
```

## 5. Разверните проект
Распакуйте архив и перейдите в папку проекта:

```bash
cd /opt
sudo mkdir -p corp-kb
sudo chown $USER:$USER corp-kb
cd corp-kb
unzip /path/to/kb_mvp_v2.zip
cd kb_mvp_v2
```

Создайте `.env`:
```bash
cp .env.example .env
nano .env
```

Поменяйте минимум:
- `POSTGRES_PASSWORD`
- `OPENSEARCH_PASSWORD`
- `ADMIN_API_KEY`

## 6. Запуск контейнеров
```bash
docker compose up -d --build
```

Проверка:
```bash
docker compose ps
```

Нужно увидеть контейнеры `postgres`, `opensearch`, `dashboards`, `app`, `worker`, `nginx`.

## 7. Проверка здоровья
```bash
curl http://localhost/api/v1/health
```

Ожидаемый ответ:
```json
{"status":"ok","postgres":true,"opensearch":true}
```

## 8. Положите тестовый PDF
```bash
cp "/path/to/DOC-001__v1__Положение о резервном копировании.pdf" storage/pdfs/
ls -lah storage/pdfs
```

## 9. Поставьте ingestion в очередь
```bash
curl -X POST http://localhost/api/v1/admin/ingest   -H "X-API-Key: change_me_admin_api_key"
```

Если вы уже поменяли ключ в `.env`, используйте новый ключ.

Ожидаемый ответ:
```json
{"queued":1,"files":["DOC-001__v1__Положение о резервном копировании.pdf"]}
```

## 10. Смотрите логи worker
```bash
docker compose logs -f worker
```

Нормальный сценарий:
- job picked
- indexed N chunks
- job done

Плохой сценарий:
- `OCR_REQUIRED`
- `No chunks produced`
- `PDF not found`

## 11. Проверьте список документов
```bash
curl http://localhost/api/v1/documents | jq .
```

## 12. Проверьте поиск
```bash
curl -X POST http://localhost/api/v1/search   -H "Content-Type: application/json"   -d '{"query":"резервное копирование сроки хранения","top_k":5}' | jq .
```

Что должно быть:
- в `items` есть chunk-ы;
- есть `doc_code`, `title`, `version_label`, `page_from`, `page_to`;
- есть `file_url`.

## 13. Проверьте чат с цитатами
```bash
curl -X POST http://localhost/api/v1/chat/ask   -H "Content-Type: application/json"   -d '{"question":"Что сказано про сроки хранения резервных копий?","top_k":5}' | jq .
```

Что должно быть:
- `answer` не пустой;
- есть массив `citations`;
- в каждой цитате есть документ, версия и страницы.

## 14. Откройте UI
Откройте в браузере:

```text
http://IP_СЕРВЕРА/
```

Там можно:
- увидеть список документов;
- сделать поиск;
- задать вопрос в чат.

## 15. Если ingestion не сработал
### Случай 1. PDF — скан
Если worker пишет `OCR_REQUIRED`, значит текст из PDF не извлекся. Это нормально для сканов. Эта версия не делает полноценный OCR автоматически.

### Случай 2. OpenSearch не стартует
Проверьте:
```bash
docker compose logs opensearch
free -h
```
Часто причина — мало памяти.

### Случай 3. Документ в БД есть, а поиск пустой
Проверьте:
```bash
curl http://localhost:9200/_cat/indices?v
curl http://localhost:9200/${OPENSEARCH_INDEX}/_count
```

### Случай 4. UI открывается, API нет
Проверьте:
```bash
docker compose logs nginx
docker compose logs app
```

## 16. Что делать дальше после этой версии
Следующий обязательный слой:
1. добавить OCR контейнером или внешним локальным сервисом;
2. заменить `hash_stub` embeddings на нормальную локальную модель;
3. заменить `extractive_stub` LLM на локальную модель;
4. встроить AD/LDAP/SSO;
5. сделать архивные версии и явное переключение active/archive;
6. добавить upload endpoint и UI-загрузку PDF;
7. улучшить chunking по структуре документов.
