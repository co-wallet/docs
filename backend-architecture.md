# Архитектура бэкенда

Бэкенд co-wallet написан на Go 1.25 и придерживается трёхслойной архитектуры: **handler → service → repository**. Каждый слой знает только о соседнем снизу и общается через узкие интерфейсы, чтобы все зависимости можно было подменить моками в тестах.

## Структура пакетов

```
backend/
├── cmd/server/              # main.go, router.go — entrypoint и сборка зависимостей
├── internal/
│   ├── config/              # загрузка .env в Config-структуру
│   ├── db/                  # pgx-pool, DBTX интерфейс, WithTx хелпер
│   ├── model/               # доменные сущности и request/response DTO модели
│   ├── repository/          # SQL-слой, по одному файлу на домен
│   ├── service/             # бизнес-логика, зависит только от интерфейсов
│   ├── handler/<domain>/    # HTTP-ручки, request/response DTO, валидация
│   ├── middleware/          # Auth, Admin, AccountMember, RateLimit
│   ├── apperr/              # типизированные sentinel-ошибки
│   ├── httputil/            # JSON helpers и маппинг ошибок в HTTP-коды
│   └── ptr/                 # generic helper ptr.To[T] для тестов
└── migrations/              # goose SQL миграции
```

**Пример одного домена — transaction:**

| Слой       | Файл                                       |
|------------|--------------------------------------------|
| Модель     | `internal/model/transaction.go`            |
| Repository | `internal/repository/transaction.go`       |
| Service    | `internal/service/transaction.go`          |
| Handler    | `internal/handler/transaction/transaction.go` |
| Response   | `internal/handler/transaction/response.go` |
| Validate   | `internal/handler/transaction/validate.go` |

Handler никогда не обращается к repository напрямую — только через service. Service не знает про HTTP и не импортирует handler.

## DBTX и транзакции

`internal/db/db.go` экспортирует интерфейс, которому удовлетворяет и пул, и `pgx.Tx`:

```go
type DBTX interface {
    Exec(ctx context.Context, sql string, args ...any) (pgconn.CommandTag, error)
    Query(ctx context.Context, sql string, args ...any) (pgx.Rows, error)
    QueryRow(ctx context.Context, sql string, args ...any) pgx.Row
}

func WithTx(ctx context.Context, pool TxBeginner, fn func(pgx.Tx) error) error
```

Каждый репозиторий держит `DBTX` в поле `db` и имеет метод `WithTx(tx pgx.Tx) *XxxRepository`, который возвращает копию с заменённым `db` на транзакцию. Сервис, когда нужна атомарность нескольких операций, оборачивает их в `db.WithTx`:

```go
// service/invite.go — создание пользователя и пометка инвайта одной транзакцией
err := s.withTx(ctx, func(invRepo inviteRepo, userRepo inviteUserRepo) error {
    u, err := userRepo.Create(ctx, model.User{...})
    if err != nil {
        return fmt.Errorf("create user: %w", err)
    }
    created = u
    return invRepo.MarkUsed(ctx, req.Token)
})
```

Места, где применяется WithTx:

- `repository/transaction.go` — `Create`/`Update` + upsert долей
- `repository/tag.go` — `UpsertForTransaction` (DELETE links + INSERT tags+links)
- `service/account.go` — `CreateAccount` + `AddMember`
- `service/invite.go` — `AcceptInvite` (создание пользователя + `MarkUsed`)

## Обработка ошибок

Все доменные ошибки выражаются через sentinel-значения в `internal/apperr/errors.go`:

```go
var (
    ErrNotFound     = errors.New("not found")
    ErrForbidden    = errors.New("forbidden")
    ErrValidation   = errors.New("validation error")
    ErrConflict     = errors.New("conflict")
    ErrUnauthorized = errors.New("unauthorized")
)
```

Сервисы оборачивают контекст через `%w`:

```go
if a.OwnerID != requesterID {
    return fmt.Errorf("only the owner can delete an account: %w", apperr.ErrForbidden)
}
```

`internal/httputil/respond.go` маппит их в HTTP-коды:

| Sentinel          | HTTP |
|-------------------|------|
| `ErrValidation`   | 400  |
| `ErrUnauthorized` | 401  |
| `ErrForbidden`    | 403  |
| `ErrNotFound`     | 404  |
| `ErrConflict`     | 409  |
| _любая другая_    | 500  |

Handler вызывает один helper `httputil.HandleServiceError(w, err)` и не занимается ручным мэппингом.

## Аутентификация

Аутентификация — JWT (HS256) с двумя токенами:

- **access** — 15 минут, кладётся в `Authorization: Bearer <token>`
- **refresh** — 30 дней, обменивается на пару через `POST /api/auth/refresh`

Claims содержат `userId`, `isAdmin` и `tokenType` (`access` или `refresh`). Тип проверяется на каждом auth-пути, поэтому refresh-токен нельзя использовать для доступа к API. При обновлении токенов сервис повторно загружает пользователя и отклоняет деактивированные аккаунты. Middleware `internal/middleware/auth.go` разбирает заголовок, валидирует подпись через `AuthService.ValidateAccessToken` и кладёт значения в контекст:

```go
const (
    ContextUserID  contextKey = "userID"
    ContextIsAdmin contextKey = "isAdmin"
)

func UserIDFromCtx(ctx context.Context) string { ... }
```

Хэндлеры берут id пользователя как `middleware.UserIDFromCtx(r.Context())`.

Дополнительные middleware:

- `Admin` — проверяет `isAdmin=true` в контексте
- `AccountMember` — делает `IsMember(accountID, userID)` через интерфейс `memberChecker`
- `RateLimit` — token bucket (`golang.org/x/time/rate`) на auth-эндпоинты

## Router

`cmd/server/router.go` собирает chi-router. Middleware применяются группами:

```go
r := chi.NewRouter()
r.Use(chimw.RequestID, chimw.Logger, chimw.Recoverer, cors.Handler(...))

authLimiter := middleware.RateLimit(rate.Limit(5), 10, 5*time.Minute)

r.Route("/api", func(r chi.Router) {
    r.Get("/health", handler.Health)

    r.Route("/auth", func(r chi.Router) {
        r.Use(authLimiter)
        r.Post("/login", authHandler.Login)
        r.Post("/refresh", authHandler.Refresh)
    })

    r.Group(func(r chi.Router) {
        r.Use(middleware.Auth(authSvc))
        // protected routes...
        r.Route("/admin", func(r chi.Router) {
            r.Use(middleware.Admin)
            // admin-only routes...
        })
    })
})
```

## Тестирование

**Инструменты:**

- `github.com/stretchr/testify` (`suite`, `require`)
- `go.uber.org/mock/gomock` для моков
- `internal/ptr` для pointer-хелперов в тестовых данных

**Генерация моков** через директиву в исходнике:

```go
//go:generate mockgen -source=account.go -destination=mocks/mock_account_repo.go -package=mocks
type accountRepo interface { ... }
```

`make generate` обходит все `go:generate` директивы.

**Шаблон тестового suite:**

```go
type TransactionServiceSuite struct {
    suite.Suite
    ctrl *gomock.Controller
    repo *mocks.MockTransactionRepo
    svc  *TransactionService
}

func (s *TransactionServiceSuite) SetupTest() {
    s.ctrl = gomock.NewController(s.T())
    s.repo = mocks.NewMockTransactionRepo(s.ctrl)
    s.svc = &TransactionService{repo: s.repo}
}

func TestTransactionServiceSuite(t *testing.T) {
    suite.Run(t, new(TransactionServiceSuite))
}
```

Для чистой логики (валидации, расчёт долей) — table-driven тесты без моков.

## Конвенции

- **Модели** (`internal/model/`) не содержат JSON-тегов. Теги — только на Response DTO внутри `handler/<domain>/response.go`.
- **Request DTO** делятся на два уровня: неэкспортируемые handler-level (`createTransactionReq` в `handler/.../validate.go`) для разбора HTTP и экспортируемые model-level (`model.CreateTransactionReq`) для передачи в сервис.
- **Repositories** возвращают значения, а не указатели (`(model.User, error)`, не `(*model.User, error)`).
- **Services** зависят только от интерфейсов, объявленных в том же пакете. Интерфейсы — узкие, только необходимые методы.
- **Soft-delete** для счетов и категорий (поле `deleted_at`), транзакции удаляются жёстко с каскадом на `transaction_shares` и `transaction_tags`.
