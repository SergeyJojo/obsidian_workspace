
**Гексагональная архитектура** (она же **Hexagonal Architecture**, она же **Порт-адаптер архитектура**, она же в контексте DDD — один из популярных **подходов организации бизнес-логики**) — это способ структурирования кода, который помогает отделить _внешние интерфейсы_ от _внутренней бизнес-логики_.

---

## 📐 Суть архитектуры

Основная идея: представить систему в виде **ядра (домена)**, окружённого **портами и адаптерами**.  
Внутреннее ядро ничего не знает о внешнем мире — оно работает с абстракциями.

```
                   +---------------------+
                   |  Адаптеры (внешние) |
                   +---------------------+
                            |         
          +---------------- v ----------------+
          |         Порты (интерфейсы)        |
          +---------------- v ----------------+
          |         Ядро (доменная логика)    |
          +-----------------------------------+
```

---

## 🧠 Компоненты

### 1. **Домен (ядро)**

Содержит бизнес-логику, сущности, агрегации, сервисы и т.д.  
Никаких зависимостей на базы данных, фреймворки, веб — только «чистый» бизнес.

### 2. **Порты**

Это **интерфейсы**, которые определяют, как внешний мир может взаимодействовать с доменом (и наоборот).

- **Входные порты** (Driving): вызываются извне (например, UseCases, ApplicationService)
    
- **Выходные порты** (Driven): то, что нужно самому ядру — например, `UserRepository`, `Notifier`, `PaymentProcessor`
    

### 3. **Адаптеры**

Реализации портов.

- Веб-адаптер (REST/gRPC)
    
- CLI-адаптер
    
- Адаптер к БД (реализация `UserRepository`)
    
- Адаптер к API внешнего сервиса
    

---

## 🔍 В контексте DDD

В DDD **гексагоналка** идеально сочетается с идеей, что бизнес-логика должна быть изолирована от инфраструктуры.  
Тебе важно:

- описать бизнес-правила (домен)
    
- выделить граничные интерфейсы (порты)
    
- реализовать детали доступа (адаптеры)
    

---

## 🆚 Против MVC, слоёв и прочего

- MVC и слоистая архитектура — это более **технические разбиения**
    
- Гексагональная — **архитектура, организующая зависимости**, делая систему _инвертированной_ (ядро — центр, всё остальное — периферия)
    

---

## 🧪 Пример (Go)

```go
// Внутренний порт
type UserRepository interface {
    FindUser(id string) (*User, error)
}

// UseCase
type UserService struct {
    repo UserRepository
}
func (s *UserService) GetUser(id string) (*User, error) {
    return s.repo.FindUser(id)
}
```

```go
// Адаптер: реализация порта
type PostgresUserRepo struct {
    db *sql.DB
}
func (r *PostgresUserRepo) FindUser(id string) (*User, error) {
    // SQL-запрос
}
```

---

## ✅ Зачем использовать?

- **Тестируемость** — можно мокать адаптеры
    
- **Гибкость** — можно легко менять реализацию адаптера (например, перейти с PostgreSQL на Redis)
    
- **Изоляция бизнес-логики** — не зависит от фреймворков, можно переиспользовать
    
- **Поддержка SRP и DIP (SOLID)**
    

---

Если хочешь — могу нарисовать схему, показать полный пример на Go/Kotlin/TS, или сравнить с Clean Architecture. Как тебе интереснее продолжить?

Отлично! Давай разберем **гексагональную архитектуру** (Hexagonal Architecture, или **порт-адаптер архитектуру**) **в глубину**, как это видят опытные архитекторы, практики DDD, и даже авторы популярных библиотек.

---

## 📚 История и мотивация

Концепцию предложил **Аластэйр Кокбёрн (Alistair Cockburn)** в середине 2000-х годов. Его цель была — создать архитектуру, в которой **основная бизнес-логика не зависит от среды выполнения**, будь то CLI, Web, база данных, или даже фреймворки. Такая архитектура решает проблему, когда твой бизнес-код буквально «тонет» в технологиях.

---

## 🔧 Гексагональная архитектура = Чистый вход и выход

В ней мы **разделяем систему на 3 главных уровня**:

### 1. **Application Core (домен)**

То, что содержит суть твоей системы: бизнес-правила, доменные сущности, агрегаты, use cases.  
Он **не знает**, как он вызывается (REST или CLI), и как он хранит данные (Postgres или Mongo).

Пример:

```go
type User struct {
    ID   string
    Name string
}

type UserService struct {
    repo UserRepository
}
func (s *UserService) RegisterUser(name string) (*User, error) {
    id := uuid.New().String()
    user := &User{ID: id, Name: name}
    return user, s.repo.Save(user)
}
```

### 2. **Ports (интерфейсы)**

Это контракты. Они определяют, **что** нужно сделать, но не **как**.  
Делятся на:

- **Входные (driving ports)** — интерфейсы, через которые внешний мир вызывает ядро (например, `RegisterUser(name string)`).
    
- **Выходные (driven ports)** — то, что нужно ядру для работы с внешним миром (например, `UserRepository.Save()`).
    

Так мы **инвертируем зависимости** (знаешь [принцип DIP из SOLID](https://en.wikipedia.org/wiki/Dependency_inversion_principle)? Вот оно).

### 3. **Adapters (реализация портов)**

Это уже реализация интерфейсов:

- Web-адаптер (HTTP контроллер)
    
- CLI-адаптер (командная строка)
    
- DB-адаптер (реализация `UserRepository`)
    
- External API adapters
    
- Message Queue (Kafka, RabbitMQ)
    

Все они реализуют порты, **но не изменяют домен**.

---

## 🔄 Как это выглядит

В виде кругов (по аналогии с Clean Architecture):

```
               [ CLI / HTTP / gRPC / Tests ]
                        ↓ (входной адаптер)
     +-----------------------------------------------+
     |       Application Layer (Use Cases)           |
     |                                               |
     |   +-------------------------------------+     |
     |   |        Domain Model (Entities)      |     |
     |   +-------------------------------------+     |
     |                                               |
     +---------------------+-------------------------+
                           ↓ (выходной порт)
                   [ DB / API / Queue / Logger ]
```

---

## ⚙️ Что это даёт на практике?

### ✅ Преимущества:

- **Изоляция бизнес-логики**: можно переиспользовать, писать unit-тесты без БД.
    
- **Тестируемость**: легко мокать адаптеры (например, вместо PostgreSQL — in-memory).
    
- **Лёгкость поддержки и замены компонентов**: хочешь вместо REST — gRPC? Меняешь только адаптер.
    
- **Чистая архитектура**: реализация не зависит от внешнего окружения.
    
- **Сильное соблюдение DIP**: зависимости направлены _внутрь_.
    

---

## 🧠 Пример — Гексагональная структура на Go

```
/internal
  /user
    entity.go         // User Entity
    service.go        // Business Logic
    port.go           // Ports (interfaces)
  /db
    user_repo_pg.go   // Adapter для PostgreSQL
  /http
    handler.go        // Входной адаптер (HTTP)
main.go               // Wire up everything
```

---

### 📌 Пример входного и выходного порта

```go
// port.go
type UserRepository interface {
    Save(user *User) error
    Find(id string) (*User, error)
}

// service.go
type UserService struct {
    repo UserRepository
}
```

```go
// adapter: db/user_repo_pg.go
type PostgresUserRepo struct {
    db *sql.DB
}
func (r *PostgresUserRepo) Save(user *User) error {
    _, err := r.db.Exec("INSERT INTO users (id, name) VALUES ($1, $2)", user.ID, user.Name)
    return err
}
```

```go
// входной адаптер: http/handler.go
func RegisterUserHandler(w http.ResponseWriter, r *http.Request) {
    name := r.FormValue("name")
    user, err := service.RegisterUser(name)
    ...
}
```

---

## ❗️Распространённые ошибки

1. **Смешение слоёв** — когда use case напрямую обращается к SQL.
    
2. **Зависимость ядра от внешних пакетов** — например, использование `gorm.Model` внутри сущности.
    
3. **Нарушение SRP** — если один слой и бизнес-логику решает, и БД трогает, и HTTP знает.
    

---

## 🤔 Hexagonal vs Clean Architecture

Очень похожи: оба направлены на инверсию зависимостей и изоляцию ядра. Разница скорее в подаче:

|Характеристика|Hexagonal|Clean Architecture|
|---|---|---|
|Фокус|Интерфейсы и адаптеры|Слои и круги|
|Названия|Порты / Адаптеры|Entities / UseCases / Interface|
|Связь с DDD|Отлично сочетается|Отлично сочетается|
|Визуализация|Гексагональная|Круговая|

---

## 🚀 Заключение

Гексагональная архитектура — это **практичный, адаптируемый** способ организации приложения. Она идеально ложится в микросервисы, в проектирование на основе DDD, и в системы с высокими требованиями к поддержке и тестированию.

---

Хочешь, я покажу тебе пример полной гексагональной структуры на Go или на другом языке? Или разобрать её применение к реальному проекту, например, маркетплейсу или стриминговой платформе?

🧙🏾‍♂️ Превосходно! Погружаемся в **структуру проекта**, реализующего **гексагональную архитектуру на Go**, с примерами, пояснениями и логикой. Это поможет тебе уверенно проектировать архитектуру, расширять её и делать легко поддерживаемой.

---

## 🧱 Общая структура гексагонального проекта на Go

```
/cmd/
  app/                 # Точка входа (main.go), инициализация приложения

/internal/
  /core/               # Бизнес-логика (домен + use-cases)
    /user/
      entity.go        # Domain Entity (User)
      service.go       # Use Case (RegisterUser)
      port.go          # Интерфейсы (интерфейс UserRepository)
  
  /adapter/            # Адаптеры (входные и выходные)
    /http/             # Входной адаптер (REST/gRPC)
      handler.go
      router.go
    /db/               # Выходной адаптер (Postgres/Mongo и т.д.)
      user_repo_pg.go
    /mq/               # Выходной адаптер (Kafka/RabbitMQ)
    /cli/              # CLI интерфейс (опционально)
    /externalapi/      # Работа с внешними API

/pkg/                  # Утилиты, хелперы, общие библиотеки
/config/               # Конфиги YAML/ENV/JSON
/scripts/              # Миграции, генераторы, инициализация
/test/                 # Интеграционные тесты

/build/                # Dockerfile, Kubernetes манифесты и т.д.

go.mod
go.sum
README.md
```

---

## 🔎 Подробный разбор каждой части

### `/cmd/app/`

Точка входа приложения. Здесь:

- запускается HTTP сервер или CLI
    
- настраиваются зависимости
    
- связываются интерфейсы с реализациями (ручной wiring или с помощью [fx](https://github.com/uber-go/fx), [wire](https://github.com/google/wire))
    

```go
func main() {
  cfg := config.Load()
  db := postgres.NewConnection(cfg.DB)

  repo := dbadapter.NewPostgresUserRepo(db)
  service := user.NewUserService(repo)

  handler := httpadapter.NewHandler(service)
  httpadapter.StartServer(handler)
}
```

---

### `/internal/core/user/`

#### `entity.go`

```go
type User struct {
  ID   string
  Name string
}
```

#### `port.go` — **Порты**

```go
type UserRepository interface {
  Save(user *User) error
  FindByID(id string) (*User, error)
}
```

#### `service.go` — **Use Case**

```go
type Service struct {
  repo UserRepository
}

func (s *Service) Register(name string) (*User, error) {
  user := &User{ID: uuid.New().String(), Name: name}
  err := s.repo.Save(user)
  return user, err
}
```

---

### `/internal/adapter/db/user_repo_pg.go`

Реализация интерфейса `UserRepository`.

```go
type PostgresUserRepo struct {
  db *sqlx.DB
}

func (r *PostgresUserRepo) Save(user *User) error {
  _, err := r.db.Exec("INSERT INTO users (id, name) VALUES ($1, $2)", user.ID, user.Name)
  return err
}
```

---

### `/internal/adapter/http/handler.go`

Входной адаптер — вызывает use case через интерфейс.

```go
type Handler struct {
  service user.Service
}

func (h *Handler) RegisterUser(w http.ResponseWriter, r *http.Request) {
  name := r.FormValue("name")
  user, err := h.service.Register(name)
  ...
}
```

---

### `/pkg/`

Утилитарный код: логгеры, обёртки, middlewares, error-handling, валидаторы, etc.

---

### `/config/`

Работа с конфигурацией: YAML, JSON или ENV. Можно использовать `viper` или `envconfig`.

---

### `/scripts/`

Скрипты миграции (например, с `golang-migrate`), сидеры, генераторы кода (например, mockgen), pre-commit хуки.

---

### `/test/`

Интеграционные тесты, которые проверяют взаимодействие adapter → порт → core.

---

## 🧩 Диаграмма зависимостей

```
[ HTTP Handler ] ---> [ Use Case ] ---> [ Interface: UserRepository ] ---> [ Postgres Adapter ]
```

Важно: **стрелки зависимости всегда смотрят в сторону core**. Внешние адаптеры реализуют интерфейсы, но не влияют на бизнес-логику.

---

## 🧪 Тестирование

- Use cases: просто тестируются через интерфейсы, без БД
    
- Адаптеры: тестируются отдельно (можно с моками или тестовой БД)
    
- Интеграционные тесты: поднимают всё целиком и проверяют взаимодействие
    

---

## 🚀 Хочешь примеры генерации моков, либо как использовать `wire` для связывания адаптеров?

Или показать шаблон для твоего проекта в виде `go-hex-boilerplate`?

🧙🏾‍♂️ Отличный вопрос! Мокация адаптеров — важнейшая часть тестирования в **гексагональной архитектуре (DDD)**, ведь она позволяет:

- Тестировать бизнес-логику изолированно от инфраструктуры
    
- Обеспечить воспроизводимость и контроль тестов
    
- Повысить надёжность, не трогая реальную БД, API или файловую систему
    

---

## 🤖 Что такое мокация адаптера в контексте DDD?

В DDD/Hexagonal архитектуре **адаптеры** — это реализации интерфейсов (портов), через которые бизнес-логика общается с внешним миром. Чтобы протестировать **use case**, мы мокаем (подменяем) эти интерфейсы.

Например:

```go
type UserRepository interface {
    Save(user *User) error
    FindByID(id string) (*User, error)
}
```

В тесте мы не хотим использовать настоящий Postgres. Вместо этого — мок, реализующий тот же интерфейс.

---

## 🛠 Как мокаются адаптеры?

### ✅ Вариант 1: Ручной мок (вручную пишем struct с методами)

```go
type MockUserRepo struct {
    SavedUsers []*User
}

func (m *MockUserRepo) Save(user *User) error {
    m.SavedUsers = append(m.SavedUsers, user)
    return nil
}

func (m *MockUserRepo) FindByID(id string) (*User, error) {
    for _, u := range m.SavedUsers {
        if u.ID == id {
            return u, nil
        }
    }
    return nil, errors.New("not found")
}
```

Тестируем use case:

```go
func TestRegisterUser(t *testing.T) {
    mockRepo := &MockUserRepo{}
    svc := user.NewUserService(mockRepo)

    u, err := svc.Register("Alice")
    assert.NoError(t, err)
    assert.Equal(t, "Alice", u.Name)
    assert.Len(t, mockRepo.SavedUsers, 1)
}
```

---

### ✅ Вариант 2: Генерация моков с помощью `mockgen` (GoMock)

```bash
go install github.com/golang/mock/mockgen@latest
mockgen -source=internal/core/user/port.go -destination=mocks/mock_user_repo.go -package=mocks
```

В коде:

```go
mockRepo := mocks.NewMockUserRepository(ctrl)
mockRepo.EXPECT().
    Save(gomock.Any()).
    Return(nil)
```

---

## 🧪 Что мокать?

|Что|Как мокать|Где использовать|
|---|---|---|
|БД|Мок интерфейса репозитория|Юнит-тесты use case'ов|
|Kafka, MQ|Мок pub/sub интерфейса|Проверка взаимодействий|
|HTTP API|Мок внешнего клиента|Поведение при ошибках API|
|Кэш (Redis)|Мок кэш-адаптера|Проверка стратегии кэширования|

---

## 📦 Полезные библиотеки

- [`golang/mock`](https://github.com/golang/mock) — промышленный стандарт
    
- [`testify/mock`](https://pkg.go.dev/github.com/stretchr/testify/mock) — проще для быстрого старта
    
- [`moq`](https://github.com/matryer/moq) — минимализм и чистота
    
- [`counterfeiter`](https://github.com/maxbrunsfeld/counterfeiter) — идеален для строгих контрактов
    

---

## 🚨 Ошибки при мокации

- Мок повторяет баги настоящего адаптера — не выявляются настоящие проблемы
    
- Забыли проверить вызовы моков (`WasCalled`, `CalledWith`)
    
- Перетестировали адаптер — нужно тестировать **use case**, а не поведение мока
    

---

Хочешь я покажу **конкретный пример** с генерацией мока для Kafka-публикатора или клиента внешнего API?