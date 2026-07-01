# Ревью деплоя (2026-07-01)

> Черновик находок, не закоммичен. Разбор `.github/workflows/ci-cd.yml`, `backend/Dockerfile`, `services/auth/Dockerfile`, `docker-compose.prod.yml`, `scripts/setup-server.sh`.

## Общий вывод

Крепкая база для demo/pilot-стенда: пайплайн честно гоняет unit → integration → e2e перед сборкой, секреты деплоя изолированы в GitHub `environment: production`, бутстрап нового сервера идемпотентен. Но есть дыры, из-за которых зелёный CI не гарантирует ни воспроизводимость собранного образа, ни то, что задеплоенный стек реально жив.

## Хорошо продумано

- Последовательность test → integration-test (реальный docker-стек) → e2e (Playwright) → build → deploy, каждый шаг блокирует следующий.
- `deploy` job в GitHub `environment: production` — секреты (`DEPLOY_HOST/USER/SSH_KEY/PATH`) не хардкожены, можно повесить required reviewers.
- `scripts/setup-server.sh` идемпотентен (`[ -f "$CERT" ]`/`[ -f "$ENV_FILE" ]` перед генерацией) — повторный запуск не перетирает ключи.
- `docker-compose.prod.yml` требует критичные секреты через `${VAR:?message}` — без `POSTGRES_PASSWORD`/`ADMIN_USERS`/`JWT_PRIVATE_KEY` compose не поднимется с пустыми значениями.
- `migrate()` в обоих сервисах — чисто аддитивные миграции (`CREATE TABLE IF NOT EXISTS`, `ADD COLUMN IF NOT EXISTS`), безопасны при перезапуске поверх существующей БД.

## Проблемы (по приоритету)

### 1. `backend/Dockerfile` отключает верификацию модулей

`Dockerfile:4-5`:
```
ENV GONOSUMDB='*'
ENV GOFLAGS='-mod=mod'
```
Образ, уходящий в прод, собирается без проверки `go.sum` и с разрешением менять резолв зависимостей прямо во время сборки. `test` job в CI гоняет `go test` без этих флагов (с нормальной проверкой сумм) — то есть протестированный граф зависимостей не гарантированно совпадает с задеплоенным. Подрывает смысл пайплайна test → build → deploy.

`services/auth/Dockerfile` собран правильно (сначала `go.mod`/`go.sum` + `go mod download` отдельным слоем, без оверрайдов) — несоответствие между двумя Dockerfile, не осознанное решение.

**Фикс:** убрать `GONOSUMDB`/`GOFLAGS` из `backend/Dockerfile`, привести к паттерну `services/auth/Dockerfile` (copy go.mod/go.sum → `go mod download` → copy остального → build).

### 2. Деплой не проверяет, что стек реально поднялся

`ci-cd.yml`, шаг Deploy: `docker compose -f docker-compose.prod.yml up -d --remove-orphans` — без `--wait`. Для сравнения, `integration-test` job для тестового стека корректно использует `up -d --wait --wait-timeout 120`.

Если новый образ падает на старте (например, из-за бракованной переменной окружения) — SSH-команда всё равно завершится с exit 0, Actions покажет зелёное, а прод при этом в краш-лупе. Post-deploy smoke-теста тоже нет.

**Фикс:** добавить `--wait --wait-timeout <N>` в деплой-шаг (как в integration-test), и/или curl на публичный health-эндпоинт после деплоя с падением job при неуспехе.

### 3. SHA-теги образов собираются, но никогда не используются

`build` job считает `meta-gateway`/`meta-auth` (short-sha + `latest`) и выставляет их в `outputs`, но `deploy` job их не читает — всегда тянет жёстко прописанный `:latest`.

Следствия:
- `outputs.gateway_tag`/`auth_tag` — мёртвый код в workflow.
- Нет способа откатиться на конкретную версию иначе, чем руками зайти по SSH и подставить тег — по факту rollback-механизма нет.

**Фикс:** передавать `needs.build.outputs.gateway_tag`/`auth_tag` в деплой-шаг вместо хардкода `:latest`, чтобы деплоился именно тот образ, который прошёл тесты в этом прогоне; это же открывает путь к ручному/автоматическому rollback на предыдущий sha.

### 4. TLS — self-signed сертификат

`scripts/setup-server.sh` генерирует self-signed через `openssl req -x509 ... -subj "/CN=$SERVER_IP"`. Осознанно и ожидаемо для demo-стенда (сообщение скрипта прямо предупреждает про warning в браузере). Но для реального B2B-канала с партнёрами не годится: серверные HTTP-клиенты партнёров по умолчанию отклонят self-signed сертификат.

**Фикс (на будущее, не сейчас):** Let's Encrypt / реальный CA, когда проект выходит за рамки demo-стенда.

### 5. Мелочь: разные версии Go builder-образа

`backend/Dockerfile` — `golang:1.22-alpine`, `services/auth/Dockerfile` — `golang:1.24-alpine`. Соответствуют версиям в своих `go.mod`, не баг, но стоило бы синхронизировать при случае.

## Статус (обновлено 2026-07-02)

Реализовано в `src/`:
- **#1** — `Dockerfile` приведён к паттерну `services/auth/Dockerfile` (`go.mod`/`go.sum` → `go mod download` → build), `GONOSUMDB`/`GOFLAGS` убраны.
- **#2** — `docker-compose.prod.yml`: у `gateway` появился `healthcheck` (аналогично test-стеку), `nginx` теперь ждёт `condition: service_healthy`. В `ci-cd.yml` деплой-шаг использует `up -d --wait --wait-timeout 120`, добавлен отдельный шаг `Smoke test` (`curl` на `/.well-known/gateway` через nginx).
- **#3** — деплой-шаг больше не хардкодит `:latest`: `build` job вычисляет `sha_tag` (короткий `GITHUB_SHA`) и передаёт его в `deploy` через `needs.build.outputs`; и `pull`, и `up -d` используют именно этот тег. Откат на предыдущую версию — задеплоить с предыдущим sha вручную.

Не тронуто (осознанно отложено):
- **#4** — TLS self-signed, ждёт выхода за рамки demo-стенда.
- **#5** — версии Go builder-образов (1.22/1.24) соответствуют `go.mod` каждого модуля, синхронизировать не нужно — снято как невалидная находка.
