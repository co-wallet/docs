# Development Guide

Как развернуть co-wallet локально, добавить новый домен в бэкенд и покрыть код тестами.

## Репозитории

co-wallet — это четыре отдельных git-репозитория, которые собраны в один VS Code workspace:

| Папка       | Назначение                    |
|-------------|-------------------------------|
| `docs/`     | Документация, спецификация    |
| `backend/`  | Go API (chi, pgx, goose)      |
| `frontend/` | React + Vite + TypeScript     |
| `docker/`   | docker-compose, Dockerfile-ы  |

У каждого — свой `.git`. Коммиты и ветки — по-репо, не на уровне workspace.

## Локальный запуск

### Полный стек в Docker

```bash
cd docker
cp .env.example .env    # заполнить JWT_SECRET, ADMIN_PASSWORD и т.д.
docker compose up -d
```

Фронтенд — http://localhost:3000, бэкенд проксируется через nginx на `/api/*`.

Режим разработки с hot-reload:

```bash
cd docker
docker compose -f docker-compose.yml -f docker-compose.dev.yml up
```

- бэкенд на `http://localhost:8080` (Air или `go run`)
- фронтенд на `http://localhost:3000` (Vite HMR)
- PostgreSQL на `localhost:5432`

### Только бэкенд

```bash
cd backend
cp .env.example .env    # проставить DATABASE_URL и JWT_SECRET
make run                # go run ./cmd/server
```

Миграции применяются автоматически на старте через goose.

### Только фронтенд

```bash
cd frontend
npm install
npm run dev
```

Ожидается, что бэкенд доступен на `http://localhost:8080`.

## Основные команды

### backend

| Команда          | Что делает                                        |
|------------------|---------------------------------------------------|
| `make run`       | Запустить сервер локально                         |
| `make build`     | Скомпилировать бинарь в `bin/server`              |
| `make test`      | `go test ./... -race`                             |
| `make lint`      | `golangci-lint run`                               |
| `make generate`  | Прогнать `go generate ./...` (моки mockgen)       |
| `make migrate`   | Накатить миграции                                 |

### frontend

| Команда          | Что делает                                  |
|------------------|---------------------------------------------|
| `npm run dev`    | Vite dev server                             |
| `npm run build`  | Сборка в `dist/`                            |
| `npm run lint`   | ESLint                                      |
| `npm run test`   | Vitest                                      |

## Git workflow

Для каждой задачи — отдельная feature-ветка и PR в `main`. В `main` напрямую не коммитить.

```bash
git checkout main
git pull origin main
git checkout -b feat/my-change
# ... правки, коммиты ...
git push -u origin feat/my-change
gh pr create
```

Перед следующей задачей дождаться мержа предыдущего PR.

## Как добавить новый домен в бэкенд

Пусть нужно добавить сущность **budgets**. Последовательность:

### 1. Миграция

`backend/migrations/000NN_create_budgets.sql` — формат goose:

```sql
-- +goose Up
CREATE TABLE budgets (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL REFERENCES users(id),
    category_id UUID NOT NULL REFERENCES categories(id),
    amount      NUMERIC(18,2) NOT NULL,
    currency    TEXT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at  TIMESTAMPTZ
);
CREATE INDEX idx_budgets_user ON budgets(user_id) WHERE deleted_at IS NULL;

-- +goose Down
DROP TABLE budgets;
```

### 2. Модель

`internal/model/budget.go` — только данные, **без JSON-тегов**:

```go
package model

import "time"

type Budget struct {
    ID         string
    UserID     string
    CategoryID string
    Amount     float64
    Currency   string
    CreatedAt  time.Time
}

type CreateBudgetReq struct {
    UserID     string
    CategoryID string
    Amount     float64
    Currency   string
}
```

### 3. Repository

`internal/repository/budget.go`:

```go
package repository

import (
    "context"
    "co-wallet/backend/internal/apperr"
    "co-wallet/backend/internal/db"
    "co-wallet/backend/internal/model"
    "github.com/jackc/pgx/v5"
)

type BudgetRepository struct {
    db db.DBTX
}

func NewBudgetRepository(dbx db.DBTX) *BudgetRepository {
    return &BudgetRepository{db: dbx}
}

func (r *BudgetRepository) WithTx(tx pgx.Tx) *BudgetRepository {
    return &BudgetRepository{db: tx}
}

func (r *BudgetRepository) Create(ctx context.Context, b model.Budget) (model.Budget, error) { ... }
func (r *BudgetRepository) GetByID(ctx context.Context, id string) (model.Budget, error) {
    // not-found оборачиваем в apperr.ErrNotFound
}
```

### 4. Service

`internal/service/budget.go`. Интерфейс зависимости объявлен в том же файле — это делает сервис мокаемым:

```go
//go:generate mockgen -source=budget.go -destination=mocks/mock_budget_repo.go -package=mocks
type budgetRepo interface {
    Create(ctx context.Context, b model.Budget) (model.Budget, error)
    GetByID(ctx context.Context, id string) (model.Budget, error)
    ListByUser(ctx context.Context, userID string) ([]model.Budget, error)
}

type BudgetService struct {
    repo budgetRepo
}

func NewBudgetService(repo budgetRepo) *BudgetService {
    return &BudgetService{repo: repo}
}

func (s *BudgetService) Create(ctx context.Context, req CreateBudgetReq) (model.Budget, error) {
    if req.Amount <= 0 {
        return model.Budget{}, fmt.Errorf("amount must be positive: %w", apperr.ErrValidation)
    }
    return s.repo.Create(ctx, model.Budget{...})
}
```

Если операция многошаговая — оборачивать в `db.WithTx` и внутри вызывать `repo.WithTx(tx)`.

### 5. Handler

`internal/handler/budget/`:

- `handler.go` — структура `Handler` с интерфейсом `budgetService` (узкий, только нужные методы)
- `validate.go` — handler-local request структуры и `.validate()`
- `response.go` — DTO с JSON-тегами + `toBudgetResponse()`
- `budget.go` — методы `Create`, `Get`, `List`, `Update`, `Delete`

Шаблон метода:

```go
func (h *Handler) Create(w http.ResponseWriter, r *http.Request) {
    var req createBudgetReq
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        httputil.JSONError(w, "invalid body", http.StatusBadRequest)
        return
    }
    if err := req.validate(); err != nil {
        httputil.HandleServiceError(w, err)
        return
    }
    userID := middleware.UserIDFromCtx(r.Context())
    b, err := h.svc.Create(r.Context(), req.toModelReq(userID))
    if err != nil {
        httputil.HandleServiceError(w, err)
        return
    }
    httputil.JSON(w, http.StatusCreated, toBudgetResponse(b))
}
```

### 6. Проводка в router/main

`cmd/server/main.go`:

```go
budgetRepo := repository.NewBudgetRepository(pool)
budgetSvc  := service.NewBudgetService(budgetRepo)
budgetHdr  := budget.NewHandler(budgetSvc)
```

`cmd/server/router.go` внутри защищённой группы:

```go
r.Route("/budgets", func(r chi.Router) {
    r.Get("/", budgetHdr.List)
    r.Post("/", budgetHdr.Create)
    r.Route("/{budgetID}", func(r chi.Router) {
        r.Get("/", budgetHdr.Get)
        r.Patch("/", budgetHdr.Update)
        r.Delete("/", budgetHdr.Delete)
    })
})
```

### 7. Тесты

- `internal/service/budget_test.go` — suite + моки из `mocks/`
- `internal/handler/budget/validate_test.go` — table-driven на валидацию
- Прогнать `make generate` после изменения интерфейсов
- `make test` должен пройти зелёным перед PR

## Как писать тесты

### Unit-тесты сервисов

```go
type BudgetServiceSuite struct {
    suite.Suite
    ctrl *gomock.Controller
    repo *mocks.MockBudgetRepo
    svc  *BudgetService
}

func (s *BudgetServiceSuite) SetupTest() {
    s.ctrl = gomock.NewController(s.T())
    s.repo = mocks.NewMockBudgetRepo(s.ctrl)
    s.svc  = NewBudgetService(s.repo)
}

func (s *BudgetServiceSuite) TearDownTest() { s.ctrl.Finish() }

func (s *BudgetServiceSuite) TestCreate_NegativeAmount_Validation() {
    _, err := s.svc.Create(context.Background(), CreateBudgetReq{Amount: -1})
    s.Require().ErrorIs(err, apperr.ErrValidation)
}

func (s *BudgetServiceSuite) TestCreate_HappyPath() {
    s.repo.EXPECT().
        Create(gomock.Any(), gomock.Any()).
        Return(model.Budget{ID: "b1", Amount: 100}, nil)

    b, err := s.svc.Create(context.Background(), CreateBudgetReq{
        UserID: "u1", CategoryID: "c1", Amount: 100, Currency: "USD",
    })
    s.Require().NoError(err)
    s.Equal("b1", b.ID)
}

func TestBudgetServiceSuite(t *testing.T) { suite.Run(t, new(BudgetServiceSuite)) }
```

### Что тестировать обязательно

- Все ветки валидации (каждый `apperr.ErrValidation`-return)
- Каждая проверка доступа (`apperr.ErrForbidden`)
- not-found пути (`apperr.ErrNotFound`)
- Happy path с проверкой, что в repo передаются ожидаемые поля
- Для WithTx-операций — что при ошибке одного шага ошибка пробрасывается вверх

### Чистая логика

Валидаторы (`validate.go`), калькуляторы (`ShareCalculator`) — table-driven без моков:

```go
func TestValidateCreateReq(t *testing.T) {
    cases := []struct {
        name    string
        req     createBudgetReq
        wantErr error
    }{
        {"ok", createBudgetReq{Amount: 10, Currency: "USD"}, nil},
        {"zero amount", createBudgetReq{Amount: 0, Currency: "USD"}, apperr.ErrValidation},
        {"empty currency", createBudgetReq{Amount: 10}, apperr.ErrValidation},
    }
    for _, tc := range cases {
        t.Run(tc.name, func(t *testing.T) {
            err := tc.req.validate()
            if tc.wantErr == nil {
                require.NoError(t, err)
            } else {
                require.ErrorIs(t, err, tc.wantErr)
            }
        })
    }
}
```

### Моки в тестах

- Моки генерируются `go generate` — правь `go:generate` директиву, не мок-файл
- Используй `ptr.To[T]()` из `internal/ptr` вместо локальных helper-ов
- Для временных меток в ассершенах — `gomock.Any()` или `matchers`, не фиксированные значения

## Дебаг

- Логи сервера: `docker compose logs -f backend`
- Прямой коннект к БД: `docker exec -it docker-db-1 psql -U postgres cowallet`
- Swagger нет — используй `api-reference.md` и `curl`/Postman
- JWT можно разобрать на jwt.io чтобы увидеть claims

## Дальше

- Архитектура подробно — [backend-architecture.md](backend-architecture.md)
- Полный список эндпоинтов — [api-reference.md](api-reference.md)
- Деплой, резервные копии, переменные окружения — [admin-guide.md](admin-guide.md)
