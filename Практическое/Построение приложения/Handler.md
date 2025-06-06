
Шаблон для проектирования и написания HTTP-обработчиков (handlers) в Go может варьироваться в зависимости от сложности приложения, но я предложу универсальный подход, который подойдет для большинства случаев. Этот шаблон включает в себя:

1. **Структуру проекта**.
2. **Маршрутизацию (routing)**.
3. **Обработку запросов (handlers)**.
4. **Промежуточное ПО (middleware)**.
5. **Обработку ошибок**.
6. **Тестирование**.

---

### 1. Структура проекта
Рекомендуемая структура проекта:
```
/myapp
  /handlers       # Обработчики запросов
  /middleware     # Промежуточное ПО
  /models         # Модели данных
  /services       # Бизнес-логика
  /repositories   # Работа с данными (база данных, кэш и т.д.)
  /config         # Конфигурация приложения
  /utils          # Вспомогательные функции
  main.go         # Точка входа
```

---

### 2. Маршрутизация
Используйте `net/http` или сторонние библиотеки, такие как `gorilla/mux`, для маршрутизации.

Пример с `net/http`:
```go
package main

import (
    "net/http"
)

func main() {
    http.HandleFunc("/health", healthHandler)
    http.HandleFunc("/users", usersHandler)
    http.ListenAndServe(":8080", nil)
}
```

Пример с `gorilla/mux`:
```go
package main

import (
    "github.com/gorilla/mux"
    "net/http"
)

func main() {
    r := mux.NewRouter()
    r.HandleFunc("/health", healthHandler).Methods("GET")
    r.HandleFunc("/users", usersHandler).Methods("GET")
    http.ListenAndServe(":8080", r)
}
```

---

### 3. Обработчики запросов (Handlers)
Обработчики должны быть простыми и сосредоточены только на обработке HTTP-запросов. Бизнес-логику выносите в отдельные пакеты (например, `services`).

Пример обработчика:
```go
package handlers

import (
    "encoding/json"
    "net/http"
    "myapp/models"
    "myapp/services"
)

func usersHandler(w http.ResponseWriter, r *http.Request) {
    // Получение данных из запроса (например, query parameters)
    query := r.URL.Query().Get("query")

    // Вызов сервиса для получения данных
    users, err := services.GetUsers(query)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    // Отправка ответа
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}
```

---

### 4. Промежуточное ПО (Middleware)
Middleware используется для обработки запросов до или после основного обработчика. Например, для логирования, аутентификации или сжатия данных.

Пример middleware для логирования:
```go
package middleware

import (
    "log"
    "net/http"
    "time"
)

func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        log.Printf("Started %s %s", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
        log.Printf("Completed %s in %v", r.URL.Path, time.Since(start))
    })
}
```

Использование middleware:
```go
r := mux.NewRouter()
r.Use(middleware.LoggingMiddleware)
```

---

### 5. Обработка ошибок
Обработка ошибок должна быть централизованной. Можно использовать кастомные ошибки и middleware для их обработки.

Пример:
```go
package handlers

import (
    "errors"
    "net/http"
)

var (
    ErrInvalidInput = errors.New("invalid input")
)

func someHandler(w http.ResponseWriter, r *http.Request) {
    if err := validateInput(r); err != nil {
        http.Error(w, ErrInvalidInput.Error(), http.StatusBadRequest)
        return
    }
    // Обработка запроса
}
```

---

### 6. Тестирование
Используйте встроенный пакет `testing` для написания unit-тестов.

Пример теста для обработчика:
```go
package handlers

import (
    "net/http"
    "net/http/httptest"
    "testing"
)

func TestUsersHandler(t *testing.T) {
    req, err := http.NewRequest("GET", "/users", nil)
    if err != nil {
        t.Fatal(err)
    }

    rr := httptest.NewRecorder()
    handler := http.HandlerFunc(usersHandler)

    handler.ServeHTTP(rr, req)

    if status := rr.Code; status != http.StatusOK {
        t.Errorf("handler returned wrong status code: got %v want %v", status, http.StatusOK)
    }

    // Проверка тела ответа
    expected := `[{"id":1,"name":"John"}]`
    if rr.Body.String() != expected {
        t.Errorf("handler returned unexpected body: got %v want %v", rr.Body.String(), expected)
    }
}
```

---

### Полный пример

#### `main.go`
```go
package main

import (
    "log"
    "net/http"
    "myapp/handlers"
    "myapp/middleware"
    "github.com/gorilla/mux"
)

func main() {
    r := mux.NewRouter()
    r.Use(middleware.LoggingMiddleware)

    r.HandleFunc("/health", handlers.HealthHandler).Methods("GET")
    r.HandleFunc("/users", handlers.UsersHandler).Methods("GET")

    log.Println("Server started on :8080")
    http.ListenAndServe(":8080", r)
}
```

#### `handlers/users.go`
```go
package handlers

import (
    "encoding/json"
    "net/http"
    "myapp/models"
    "myapp/services"
)

func UsersHandler(w http.ResponseWriter, r *http.Request) {
    query := r.URL.Query().Get("query")

    users, err := services.GetUsers(query)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}
```

#### `middleware/logging.go`
```go
package middleware

import (
    "log"
    "net/http"
    "time"
)

func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        log.Printf("Started %s %s", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
        log.Printf("Completed %s in %v", r.URL.Path, time.Since(start))
    })
}
```

---

### Итог
- Используйте четкую структуру проекта.
- Разделяйте ответственность: handlers — для обработки запросов, services — для бизнес-логики.
- Добавляйте middleware для обработки общих задач (логирование, аутентификация).
- Централизуйте обработку ошибок.
- Пишите тесты для обработчиков.

Этот шаблон поможет вам создавать масштабируемые и поддерживаемые HTTP-обработчики в Go.