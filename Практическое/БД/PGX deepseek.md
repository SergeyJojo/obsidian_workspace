
Вот подробное руководство по работе с **pgx** — популярным драйвером PostgreSQL для Go, включая примеры запросов, транзакций и обработки результатов.

---

## **1. Установка pgx**
Добавьте пакет в проект:
```bash
go get github.com/jackc/pgx/v5
```

---

## **2. Подключение к БД**
### **Базовое подключение**
```go
package main

import (
	"context"
	"fmt"
	"github.com/jackc/pgx/v5"
)

func main() {
	// Формат строки подключения: postgres://user:password@host:port/database
	connStr := "postgres://user:password@localhost:5432/mydb"
	conn, err := pgx.Connect(context.Background(), connStr)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Unable to connect: %v\n", err)
		os.Exit(1)
	}
	defer conn.Close(context.Background())

	fmt.Println("Connected to PostgreSQL!")
}
```

### **Пулы соединений (pgxpool)**
Для высоконагруженных приложений:
```go
import "github.com/jackc/pgx/v5/pgxpool"

pool, err := pgxpool.New(context.Background(), connStr)
if err != nil {
    log.Fatal(err)
}
defer pool.Close()
```

---

## **3. Выполнение запросов**
### **SELECT (получение данных)**
```go
type User struct {
    ID    int
    Name  string
    Email string
}

rows, err := conn.Query(context.Background(), "SELECT id, name, email FROM users WHERE age > $1", 18)
if err != nil {
    return err
}
defer rows.Close()

var users []User
for rows.Next() {
    var u User
    if err := rows.Scan(&u.ID, &u.Name, &u.Email); err != nil {
        return err
    }
    users = append(users, u)
}

if err := rows.Err(); err != nil {
    return err
}

fmt.Printf("Users: %+v\n", users)
```

### **INSERT/UPDATE/DELETE**
```go
// INSERT с возвратом ID
var id int
err = conn.QueryRow(
    context.Background(),
    "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id",
    "Alice", "alice@example.com",
).Scan(&id)
if err != nil {
    return err
}
fmt.Println("Inserted ID:", id)

// UPDATE
_, err = conn.Exec(
    context.Background(),
    "UPDATE users SET email = $1 WHERE id = $2",
    "new_email@example.com", 1,
)

// DELETE
_, err = conn.Exec(context.Background(), "DELETE FROM users WHERE id = $1", 42)
```

---

## **4. Транзакции**
```go
tx, err := conn.Begin(context.Background())
if err != nil {
    return err
}
// Откат при ошибке (лучше использовать defer)
defer tx.Rollback(context.Background())

// Пример запросов в транзакции
_, err = tx.Exec(context.Background(), "UPDATE accounts SET balance = balance - $1 WHERE id = $2", 100, 1)
if err != nil {
    return err
}

_, err = tx.Exec(context.Background(), "UPDATE accounts SET balance = balance + $1 WHERE id = $2", 100, 2)
if err != nil {
    return err
}

// Фиксация транзакции
err = tx.Commit(context.Background())
if err != nil {
    return err
}
```

---

## **5. Работа с JSON**
```go
// Запись JSON
data := map[string]interface{}{"key": "value"}
_, err = conn.Exec(
    context.Background(),
    "INSERT INTO configs (data) VALUES ($1)",
    data,
)

// Чтение JSON
var result map[string]interface{}
err = conn.QueryRow(context.Background(), "SELECT data FROM configs WHERE id = $1", 1).Scan(&result)
```

---

## **6. Обработка ошибок**
- **Проверка на `pgx.ErrNoRows`**:
  ```go
  var name string
  err = conn.QueryRow(context.Background(), "SELECT name FROM users WHERE id = $1", 999).Scan(&name)
  if errors.Is(err, pgx.ErrNoRows) {
      fmt.Println("User not found")
  }
  ```

---

## **7. Подготовленные запросы (Prepared Statements)**
```go
// Создание prepared statement
stmt, err := conn.Prepare(context.Background(), "user_by_id", "SELECT name FROM users WHERE id = $1")
if err != nil {
    return err
}

// Использование
var name string
err = conn.QueryRow(context.Background(), "user_by_id", 1).Scan(&name)
```

---

## **8. Пример полного кода**
```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"

	"github.com/jackc/pgx/v5"
)

func main() {
	conn, err := pgx.Connect(context.Background(), "postgres://user:password@localhost:5432/mydb")
	if err != nil {
		fmt.Fprintf(os.Stderr, "Connection failed: %v\n", err)
		os.Exit(1)
	}
	defer conn.Close(context.Background())

	// SELECT
	row := conn.QueryRow(context.Background(), "SELECT name FROM users WHERE id = $1", 1)
	var name string
	if err := row.Scan(&name); err != nil {
		log.Fatal(err)
	}
	fmt.Println("User name:", name)

	// INSERT
	_, err = conn.Exec(context.Background(), "INSERT INTO users (name) VALUES ($1)", "Bob")
	if err != nil {
		log.Fatal(err)
	}
}
```

---

## **Дополнительно**
- **Логирование запросов**:  
  Подключите `pgx/v5/tracelog` для отладки SQL.
- **Конфигурация пула**:  
  ```go
  poolConfig, _ := pgxpool.ParseConfig(connStr)
  poolConfig.MaxConns = 10
  pool, _ := pgxpool.NewWithConfig(context.Background(), poolConfig)
  ```

---

### **Итог**
- Используйте `Query` для множественных строк, `QueryRow` для одной строки.
- Всегда закрывайте соединения (`defer conn.Close()`).
- Для сложных проектов выбирайте `pgxpool` вместо одного соединения.

[Официальная документация pgx](https://pkg.go.dev/github.com/jackc/pgx/v5) содержит больше примеров.