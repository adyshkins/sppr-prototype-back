# sppr-prototype

Веб-прототип интеллектуальной системы поддержки принятия решений (СППР) в контуре внутрифирменного планирования промышленного предприятия.

Первый этап содержит только базовую инфраструктуру monorepo: Go backend, PostgreSQL, Docker Compose, goose-миграции, sqlc-конфигурацию и healthcheck endpoints. Бизнес-логика СППР, CVaR, FSM, эксперименты, авторизация и frontend-экраны на этом этапе не реализуются.

## Стек

- Backend: Go + Gin
- Database: PostgreSQL
- DB access: pgx + sqlc
- Migrations: goose
- Frontend: React + Vite + TypeScript позднее
- Runtime: Docker Compose

## Запуск

```bash
docker compose up --build
```

## Проверка

```bash
curl http://localhost:8080/health
curl http://localhost:8080/ready
```

Ожидаемые ответы:

```json
{"service":"sppr-backend","status":"ok"}
```

```json
{"db":"up","status":"ok"}
```

## Миграции goose

Установка goose:

```bash
go install github.com/pressly/goose/v3/cmd/goose@latest
```

Применить миграции:

```bash
cd backend
goose -dir migrations postgres "$DATABASE_URL" up
```

Откатить последнюю миграцию:

```bash
cd backend
goose -dir migrations postgres "$DATABASE_URL" down
```

Для локального PostgreSQL из Docker Compose можно использовать:

```bash
cd backend
goose -dir migrations postgres "postgres://sppr_user:sppr_password@localhost:5433/sppr_db?sslmode=disable" up
```

## Генерация sqlc

Установка sqlc:

```bash
go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest
```

Если локальная версия Go ниже требований актуального `sqlc`, используйте совместимую версию:

```bash
go install github.com/sqlc-dev/sqlc/cmd/sqlc@v1.30.0
```

Генерация кода:

```bash
cd backend
sqlc generate
```

## Этап 3. FSM и протокол П

На третьем этапе добавлены чистая статусная машина управленческого случая, структуры протокола П и сервисная заготовка для смены статуса.

- `internal/fsm` хранит допустимые переходы между статусами `cases.status` и не зависит от БД, sqlc, repository или protocol.
- `internal/protocol` формирует JSON-структуру записи протокола П, включая trace-шаг `status_changed`.
- `internal/service.CaseService` связывает проверку перехода через FSM, обновление статуса `cases` и запись `decision_trace` через интерфейс репозитория.
- CVaR, диагностика, REST API для cases, frontend и экспериментальный runner будут реализованы позже.

## Этап 4. CVaR-модуль

`internal/cvar` реализует чистую сценарную оценку корректирующих воздействий для этапа A6 интеллектуального контура управления.

- Рассчитывается многокомпонентная функция потерь: отклонения KPI, нарушения ограничений и штраф устойчивости действия.
- По сценарным потерям рассчитываются `ExpectedLoss`, `VaR` и `CVaR`.
- Корректирующие действия ранжируются по `CVaR` с учётом критических нарушений ограничений.
- Модуль не зависит от БД, sqlc, API, Gin и frontend.

## Этап 5. Диагностика и генерация сценариев

На пятом этапе добавлены чистые подготовительные модули для будущей интеграции с CVaR-оценкой.

- `internal/diagnostics` реализует простую причинную диагностику отклонения и выбирает наиболее вероятную гипотезу.
- `internal/scenario` формирует сценарии, `KPIConfig` и корректирующие действия для `internal/cvar`.
- Оба модуля пока не зависят от БД, sqlc, Gin, REST API и frontend.
- Результаты будут интегрированы в сервисный слой на следующем этапе.

## Этап 6. Сквозной workflow service

`WorkflowService` связывает диагностику, генерацию сценариев, CVaR-оценку, FSM и протокол П в единый backend workflow.

- Процесс начинается с события отклонения и создания управленческого случая.
- Сервис записывает событие, диагностические гипотезы, сценарии, корректирующие действия и trace-записи протокола П.
- Статусы проходят цепочку до `action_selected`; экспертная валидация пока не выполняется.
- REST API, frontend, Swagger и experimental runner на этом этапе ещё не реализованы.
- Экспертная валидация будет следующим этапом.

## Этап 7. REST API

Добавлен минимальный REST API для запуска workflow и просмотра результатов управленческого случая.

### POST /api/workflow/process

Запускает `WorkflowService.ProcessEvent` и доводит кейс до статуса `action_selected`.

Пример body:

```json
{
  "event_id": "evt-hitl-011",
  "product": "A100",
  "plan_demand": 1000,
  "fact_demand": 1350,
  "capacity_load": 0.93,
  "forecast_error": 0.18,
  "campaign_flag": true,
  "alpha": 0.95
}
```

Пример curl:

```bash
curl -X POST http://localhost:18080/api/workflow/process \
  -H "Content-Type: application/json" \
  -d '{"event_id":"evt-hitl-011","product":"A100","plan_demand":1000,"fact_demand":1350,"capacity_load":0.93,"forecast_error":0.18,"campaign_flag":true,"alpha":0.95}'
```

### GET /api/cases

```bash
curl http://localhost:18080/api/cases
```

Также поддерживается фильтр по статусу:

```bash
curl "http://localhost:18080/api/cases?status=action_selected"
```

### GET /api/cases/:id

```bash
curl http://localhost:18080/api/cases/<case_id>
```

### GET /api/cases/:id/trace

```bash
curl http://localhost:18080/api/cases/<case_id>/trace
```

На этапе 7 frontend, Swagger, experiments runner и экспертная валидация ещё не реализованы.

## Этап 8. Экспертная валидация human-in-the-loop

Добавлена backend-часть экспертной валидации. Все смены статусов выполняются через FSM и `CaseService.AdvanceStatus`, а экспертные решения пишутся в `decision_trace` со step `expert_decision_recorded`.

Endpoints:

- `POST /api/cases/:id/submit-validation`
- `POST /api/cases/:id/expert-decision`
- `POST /api/cases/:id/execute`
- `POST /api/cases/:id/archive`

Body для экспертного решения:

```json
{
  "outcome": "approved",
  "comment": "Решение подтверждено экспертом",
  "expert_id": "demo-expert"
}
```

Approved-сценарий:

```bash
curl -X POST http://localhost:18080/api/workflow/process \
  -H "Content-Type: application/json" \
  -d '{"event_id":"evt-hitl-012","product":"A100","plan_demand":1000,"fact_demand":1350,"capacity_load":0.93,"forecast_error":0.18,"campaign_flag":true,"alpha":0.95}'

curl -X POST http://localhost:18080/api/cases/<case_id>/submit-validation

curl -X POST http://localhost:18080/api/cases/<case_id>/expert-decision \
  -H "Content-Type: application/json" \
  -d '{"outcome":"approved","comment":"Решение подтверждено экспертом","expert_id":"demo-expert"}'

curl -X POST http://localhost:18080/api/cases/<case_id>/execute

curl -X POST http://localhost:18080/api/cases/<case_id>/archive

curl http://localhost:18080/api/cases/<case_id>/trace
```

Rejected-сценарий:

```bash
curl -X POST http://localhost:18080/api/workflow/process \
  -H "Content-Type: application/json" \
  -d '{"event_id":"evt-hitl-013","product":"A100","plan_demand":1000,"fact_demand":1350,"capacity_load":0.93,"forecast_error":0.18,"campaign_flag":true,"alpha":0.95}'

curl -X POST http://localhost:18080/api/cases/<case_id>/submit-validation

curl -X POST http://localhost:18080/api/cases/<case_id>/expert-decision \
  -H "Content-Type: application/json" \
  -d '{"outcome":"rejected","comment":"Решение отклонено экспертом","expert_id":"demo-expert"}'

curl -X POST http://localhost:18080/api/cases/<case_id>/archive

curl http://localhost:18080/api/cases/<case_id>/trace
```

Frontend, Swagger, auth и experiments runner на этом этапе не реализованы.

## Этап 9A. Подготовка backend API для frontend

Backend подготовлен для будущего отдельного frontend-проекта.

### CORS

Frontend origin задаётся переменной окружения:

```bash
FRONTEND_ORIGIN=http://localhost:5173
```

Если переменная не задана, backend использует `http://localhost:5173`.

### Backend Base URL

```text
http://localhost:18080
```

### API Endpoints

Workflow:

- `POST /api/workflow/process`

Cases:

- `GET /api/cases`
- `GET /api/cases/:id`
- `GET /api/cases/:id/trace`
- `GET /api/cases/:id/diagnostics`
- `GET /api/cases/:id/scenarios`
- `GET /api/cases/:id/actions`

Human-in-the-loop:

- `POST /api/cases/:id/submit-validation`
- `POST /api/cases/:id/expert-decision`
- `POST /api/cases/:id/execute`
- `POST /api/cases/:id/archive`

### Минимальная Проверка Через Curl

Создать кейс:

```bash
curl -X POST http://localhost:18080/api/workflow/process \
  -H "Content-Type: application/json" \
  -d '{"event_id":"evt-ui-ready-001","product":"A100","plan_demand":1000,"fact_demand":1350,"capacity_load":0.93,"forecast_error":0.18,"campaign_flag":true,"alpha":0.95}'
```

Получить данные карточки кейса:

```bash
curl http://localhost:18080/api/cases
curl http://localhost:18080/api/cases/<case_id>
curl http://localhost:18080/api/cases/<case_id>/trace
curl http://localhost:18080/api/cases/<case_id>/diagnostics
curl http://localhost:18080/api/cases/<case_id>/scenarios
curl http://localhost:18080/api/cases/<case_id>/actions
```

Пройти HITL-сценарий:

```bash
curl -X POST http://localhost:18080/api/cases/<case_id>/submit-validation

curl -X POST http://localhost:18080/api/cases/<case_id>/expert-decision \
  -H "Content-Type: application/json" \
  -d '{"outcome":"approved","comment":"Решение подтверждено экспертом","expert_id":"demo-expert"}'

curl -X POST http://localhost:18080/api/cases/<case_id>/execute

curl -X POST http://localhost:18080/api/cases/<case_id>/archive
```

Проверить CORS preflight:

```bash
curl -i -X OPTIONS http://localhost:18080/api/cases \
  -H "Origin: http://localhost:5173" \
  -H "Access-Control-Request-Method: GET"
```

Frontend, Swagger, auth и experiments runner на этом этапе не реализованы.
