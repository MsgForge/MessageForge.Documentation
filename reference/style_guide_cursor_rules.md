# Cursor Rules - BC Wallet

## Архитектура
- Clean Architecture: слои не зависят от деталей реализации
- Один сервис = одна задача
- Интерфейсы определяются там, где используются

## Структура проекта
```
cmd/[service]/main.go    # Entry points
internal/
├── common/              # Общие компоненты
│   ├── errors/          # Кастомные ошибки проекта
│   ├── initial/         # Функции инициализации (LoadEnv, InitTracer, HealthJob)
│   ├── models/          # Общие модели (TON, блокчейн константы)
│   └── tags/            # Теги для логирования и трассировки
└── [service]/           # Каждый сервис
    ├── app.go           # NewApp() возвращает готовую сущность, Run() для squad
    ├── config/          # Конфиг сервиса с валидацией (отдельная папка)
    ├── grpc/server/     # gRPC handlers
    ├── services/        # Под-сервисы (calculator/, encrypt/, validator/)
    ├── models/          # Модели данных сервиса
    ├── client/          # Клиенты других сервисов
    ├── repository/      # БД операции
    └── proto/           # Proto + генерированный код
```

## Инициализация
```go
func main() {
    ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)
    defer cancel()
    
    log := logger.New()
    defer func() { _ = log.Sync() }()
    
    errs := squad.New(ctx).Init(func(ctx context.Context, runner *squad.Runner) error {
        return Init(ctx, log, runner)
    }).Wait()
    log.Info("shutdown")
    os.Exit(exitCode(log, errs))
}

// ВАЖНО: функция Init должна быть в том же файле main.go, а НЕ в отдельном init.go
// Это требование Go - функции должны быть в том же пакете main
func Init(ctx context.Context, log *zap.Logger, runner *squad.Runner) error {
    initial.LoadEnv(currency.AppName)
    initial.InitDecimalPrecision(ctx)
    err := initial.InitTracer(runner, currency.AppName)
    if err != nil {
        return errfmt.Error(err, "init tracer")
    }
    
    cfg := config{}
    err = libconf.Init(ctx, &cfg)
    if err != nil {
        return errfmt.Error(err, "init config")
    }
    
    // Приложение + health check
    currencyApp, err := currency.NewApp(ctx, &cfg.Currency, log)
    if err != nil {
        return errfmt.Error(err, "init currency app")
    }
    runner.RunGracefully(currencyApp.Run())
    runner.RunGracefully(initial.HealthJob(cfg.HealthCheckHost, cfg.HealthCheckPort))
    
    return nil
}
```

## Обязательные библиотеки
```go
// Логирование - go-logger/v2 ТОЛЬКО для инициализации
"github.com/cryptoboyio/go-logger/v2/logger"
// В остальном коде используй *zap.Logger
"go.uber.org/zap"

// Ошибки
errfmt "github.com/cryptoboyio/go-error-formatter"

// Валидация
"github.com/go-playground/validator/v10"

// Трассировка
"github.com/cryptoboyio/go-trace"

// Squad - ТОЛЬКО v2
"github.com/cryptoboyio/squad/v2"

// Health check
"github.com/cryptoboyio/health/v3"

// Конфигурация
"github.com/cryptoboyio/libconf"

// Env файлы
"github.com/joho/godotenv"

// Decimal для точных вычислений
"github.com/shopspring/decimal"

// gRPC конфигурация и middleware
"github.com/cryptoboyio/go-grpc/opts"

// Reflection для gRPC разработки
"google.golang.org/grpc/reflection"
```

## Паттерны кода

### app.go
```go
const AppName = "currency" // Обязательная константа в каждом сервисе

// New() - полностью готовая сущность
currencyApp, err := currency.NewApp(ctx, &cfg.Currency, log)
if err != nil {
    return errfmt.Error(err, "init currency app")
}
runner.RunGracefully(currencyApp.Run())

// Стандартный паттерн Run()
func (a *App) Run() (func(ctx context.Context) error, func(ctx context.Context) error) {
    return a.grpcServer.Run, a.grpcServer.Shutdown
}

// Health check обязателен
runner.RunGracefully(initial.HealthJob(cfg.HealthCheckHost, cfg.HealthCheckPort))
```

### service/service.go
```go
type Service struct {
    log    *zap.Logger  // Используй *zap.Logger в сервисах
    cfg    *config.Config
    
    // Под-сервисы
    encryptService encryptService
    signService    signService
}

// Интерфейсы зависимостей в том же файле
type encryptService interface {
    Encrypt(ctx context.Context, data []byte) ([]byte, error)
    Decrypt(ctx context.Context, data []byte) ([]byte, error)
}
```

### config/config.go
```go
type Config struct {
    GRPCHost string `env:"GRPC_HOST"`
    GRPCPort int    `env:"GRPC_PORT" validate:"required"`
    
    // TON specific config
    SupportedCurrencies []string `env:"SUPPORTED_CURRENCIES" validate:"required"`
    DefaultDecimals     int      `env:"DEFAULT_DECIMALS" default:"9"`
}
```

### main.go config
```go
type config struct {
    HealthCheckHost string `envconfig:"HEALTH_CHECK_HOST" default:"0.0.0.0"`
    HealthCheckPort int    `envconfig:"HEALTH_CHECK_PORT" default:"8088"`
    
    Logger   *logger.Config
    Currency currency.Config // Встраивание конфига сервиса
}

// exitCode обязательна в main.go
func exitCode(log *zap.Logger, errs []squad.Error) int {
	code := 0
	for _, err := range errs {
		if err != nil {
			log.Error("application error", zap.Error(err))
			code = 1
		}
	}
	return code
}
```

### gRPC Server - ОБЯЗАТЕЛЬНЫЙ паттерн
```go
// internal/[service]/grpc/server/server.go
package server

import (
	"context"
	"net"

	"github.com/cryptoboyio/go-error-formatter/errfmt"
	"github.com/cryptoboyio/go-grpc/opts"
	logtag "github.com/cryptoboyio/go-logger/v2/tag"
	"go.uber.org/zap"
	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"

	"bc-wallet-ton/internal/common/tags"
)

const (
	DefaultServerMaxMessageSize = 1024 * 1024 * 24 // 24MB
)

type Server struct {
	log        *zap.Logger
	grpcAddr   string
	service    serviceInterface  // Интерфейс бизнес-логики
	grpcServer *grpc.Server
}

// Интерфейс для бизнес-логики - ОБЯЗАТЕЛЬНО
type serviceInterface interface {
	// Методы сервиса
}

func (s *Server) Run(ctx context.Context) error {
	listener, err := net.Listen("tcp", s.grpcAddr)
	if err != nil {
		return errfmt.ErrorTags(err, "listen", tags.GrpcAddr(s.grpcAddr))
	}

	s.log.With(logtag.Context(ctx)).Info("[service] grpc server starting")
	return s.grpcServer.Serve(listener)  // БЕЗ горутин и каналов! squad сам управляет
}

func (s *Server) Shutdown(_ context.Context) error {
	s.grpcServer.GracefulStop()
	return nil
}

func New(
	log *zap.Logger,
	grpcAddr string,
	grpcConfig *opts.GRPCConfig,
	service serviceInterface,
) (*Server, error) {
	options := opts.ServerOptions(grpcConfig)
	msgSizeOptions := []grpc.ServerOption{
		grpc.MaxRecvMsgSize(DefaultServerMaxMessageSize),
		grpc.MaxSendMsgSize(DefaultServerMaxMessageSize),
	}
	options = append(options, msgSizeOptions...)

	grpcServer := grpc.NewServer(options...)
	reflection.Register(grpcServer)
	// pb.RegisterAPIServiceServer(grpcServer, handlers)

	return &Server{
		log:        log,
		grpcAddr:   grpcAddr,
		service:    service,
		grpcServer: grpcServer,
	}, nil
}
```

### Сетевые адреса
```go
// ✅ ПРАВИЛЬНО - формирование адреса
addr := net.JoinHostPort(cfg.GRPCHost, strconv.Itoa(cfg.GRPCPort))
grpcServer, err := server.New(log, addr, grpcConfig, serviceSvc)
```

### Логирование - КРИТИЧНО
```go
// ❌ ЗАПРЕЩЕНО - логировать И возвращать ошибку
if err != nil {
    s.log.Error(ctx, "error")
    return err  // НЕ ДЕЛАЙ!
}

// ✅ ПРАВИЛЬНО - ТОЛЬКО возвращать
if err != nil {
    return errfmt.Error(err, "failed to process")
}
```

### Трассировка
```go
ctx, span, finish := tracer.Trace(ctx)
defer finish()
```

## Именование файлов
- handlers.go, service.go, errors.go, models.go
- НЕ handler.go, srv.go, err.go
- **ЗАПРЕЩЕНО interfaces.go** - интерфейсы только локальные в service.go

## Порядок функций в файле
- **Конструкторы (New) ВСЕГДА в конце файла**
- Сначала типы, интерфейсы, константы
- Затем методы
- В самом конце конструкторы

### ❗ КРИТИЧНО: Интерфейсы зависимостей
```go
// ✅ ПРАВИЛЬНО - локальный интерфейс в service.go
type repository interface {
    GetUser(ctx context.Context, id string) (*User, error)
}

// ❌ ЗАПРЕЩЕНО - глобальный интерфейс в interfaces.go
type Repository interface {
    GetUser(ctx context.Context, id string) (*User, error)
}
```
- Интерфейсы определяются там, где используются
- Интерфейсы должны быть локальными (с маленькой буквы)
- Никаких отдельных файлов interfaces.go

### ❗ КРИТИЧНО: Запрет на комментарии
```go
// ❌ ЗАПРЕЩЕНО - русские комментарии
// получает пользователя по ID
func GetUser(id string) *User {}

// ❌ ЗАПРЕЩЕНО - очевидные комментарии
func (s *Service) GetUser(ctx context.Context, id string) (*User, error) {
    // Получаем пользователя из базы данных
    return s.repository.GetUser(ctx, id)
}

// ✅ ПРАВИЛЬНО - самодокументированный код
func (s *Service) GetUser(ctx context.Context, id string) (*User, error) {
    return s.repository.GetUser(ctx, id)
}

// ✅ ПРАВИЛЬНО - комментарий для сложной логики
func (s *Service) CalculateDiscount(user *User) decimal.Decimal {
    // Apply 5% discount for orders > $100, 10% for VIP, max 15%
    return s.applyTieredDiscount(user)
}
```
- **ЗАПРЕЩЕНЫ комментарии на русском** - только английский
- **ЗАПРЕЩЕНЫ очевидные комментарии** - код должен быть понятным
- Комментарии только для сложной бизнес-логики

### app.go - ОБЯЗАТЕЛЬНЫЕ требования
```go
// internal/[service]/app.go
func New(_ context.Context, cfg *Config, log *zap.Logger) (*App, error) {
	// 1. Создание бизнес-сервисов
	validatorSvc := validator.New(log)
	
	// 2. Создание gRPC сервера с opts.GRPCConfig
	grpcServer, err := server.New(log, net.JoinHostPort(cfg.GRPCHost, strconv.Itoa(cfg.GRPCPort)), nil, validatorSvc)
	if err != nil {
		return nil, errfmt.Error(err, "init grpc server")
	}

	return &App{
		grpcServer: grpcServer,
	}, nil
}
```

### Обязательные библиотеки для gRPC
```go
// gRPC конфигурация и опции
"github.com/cryptoboyio/go-grpc/opts"

// Теги для ошибок
"bc-wallet-ton/internal/common/tags"

// Reflection для gRPC
"google.golang.org/grpc/reflection"
```

### ErrorTags - ОБЯЗАТЕЛЬНО для gRPC ошибок
```go
// ✅ ПРАВИЛЬНО - использование ErrorTags с тегами
return errfmt.ErrorTags(err, "listen", tags.GrpcAddr(s.grpcAddr))

// ❌ НЕПРАВИЛЬНО - обычный errfmt.Error
return errfmt.Error(err, "listen grpc server")
```

## Запреты
- Логировать ошибку И возвращать её
- Старые библиотеки (go-logger без v2, squad без v2)
- Горутины без squad для критичных сервисов
- **САМОДЕЛЬНЫЕ РЕШЕНИЯ С КАНАЛАМИ И ГОРУТИНАМИ** - для этого есть squad!
- Сущности не готовые к работе после New()
- Глобальные интерфейсы в repository
- **ОТДЕЛЬНЫЕ ФАЙЛЫ interfaces.go** - интерфейсы только локальные в service.go
- **ГЛОБАЛЬНЫЕ ИНТЕРФЕЙСЫ** (с большой буквы) - только локальные (с маленькой)
- **КОММЕНТАРИИ НА РУССКОМ ЯЗЫКЕ** - только английский
- **ОЧЕВИДНЫЕ КОММЕНТАРИИ** - код должен быть самодокументированным
- Функция Init в отдельном файле (должна быть в main.go)
- Забывать константу AppName в сервисах
- Забывать exitCode функцию в main.go
- gRPC сервер без opts.GRPCConfig
- gRPC сервер без reflection.Register()
- gRPC сервер без DefaultServerMaxMessageSize
- Ошибки без ErrorTags в gRPC layer
- gRPC сервер без интерфейса для бизнес-логики

### ❌ ЗАПРЕЩЕННЫЙ паттерн - самодельные каналы
```go
// ❌ НЕ ДЕЛАЙ ТАК! Для этого есть squad
func (s *Server) Run(ctx context.Context) error {
    errCh := make(chan error, 1)
    go func() {
        errCh <- s.server.Serve(listener)
    }()
    
    select {
    case <-ctx.Done():
        return nil
    case err := <-errCh:
        return err
    }
}
```

### ✅ ПРАВИЛЬНЫЙ паттерн - squad управляет горутинами
```go
// ✅ ПРАВИЛЬНО - squad сам запускает в горутине и обрабатывает контекст
func (s *Server) Run(ctx context.Context) error {
    listener, err := net.Listen("tcp", s.grpcAddr)
    if err != nil {
        return errfmt.ErrorTags(err, "listen", tags.GrpcAddr(s.grpcAddr))
    }
    
    return s.grpcServer.Serve(listener) // squad сам управляет
}
```

## Обязательные файлы в каждом сервисе
```
cmd/[service]/main.go   # main(), Init(), exitCode(), config struct
internal/[service]/
├── app.go              # const AppName, New(), Run()
├── config/             # Config папка
│   └── config.go       # Config struct для всей App с валидацией
└── ...
```