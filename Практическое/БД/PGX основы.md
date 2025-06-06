🧙🏾‍♂️ Конечно, сейчас подключим агента!

```
«Synapse_COR» = 📦: Я эксперт в работе с PostgreSQL из Go через `pgx`. Я знаю idiomatic подходы, работу с Query, Exec, Copy, Batch, транзакциями и connection pooling. Я помогу тебе глубоко и качественно освоить инструменты работы с базой.

Я помогу тебе достичь цели — использовать `pgx` на практике с максимальной эффективностью.

Я могу использовать документацию `pgx`, примеры кода, best practices и Go playground.

Я помогу тебе accomplish это, пройдя шаги:
1. Подключение и настройки (включая пул)
2. Простые запросы (QueryRow, Query)
3. Подготовленные запросы (Prepare)
4. Транзакции и rollback
5. Batch-запросы
6. Работа с CopyFrom
7. Использование context и таймаутов
8. Профилирование и отладка

Моя задача закончится, когда ты сможешь свободно писать производительный код с использованием `pgx`, включая advanced-приёмы.

Начнём с расширенного списка примеров API `pgx` и лучших практик — готов?»
```

🧙🏾‍♂️ Начну с расширенного **набора функций и фич** библиотеки `pgx`, с примерами. Потом можем углубиться в любую:

---

## 📘 Расширенные функции работы с PostgreSQL через `pgx`

---

### 🔸 Подключение с пулом соединений

```go
import (
    "context"
    "github.com/jackc/pgx/v5/pgxpool"
)

ctx := context.Background()
pool, err := pgxpool.New(ctx, "postgres://user:pass@localhost:5432/dbname")
defer pool.Close()
```

---

### 🔸 Простые запросы: QueryRow и Query

```go
var name string
err := pool.QueryRow(ctx, "SELECT name FROM users WHERE id=$1", 1).Scan(&name)

rows, err := pool.Query(ctx, "SELECT id, name FROM users")
for rows.Next() {
    var id int
    var name string
    _ = rows.Scan(&id, &name)
}
```

---

### 🔸 Prepared Statements

```go
stmt, err := pool.Prepare(ctx, "getUser", "SELECT name FROM users WHERE id=$1")
row := pool.QueryRow(ctx, stmt.Name, 42)
```

---

### 🔸 Транзакции

```go
tx, err := pool.Begin(ctx)
defer tx.Rollback(ctx)

_, err = tx.Exec(ctx, "UPDATE accounts SET balance=balance-100 WHERE id=$1", 1)
if err == nil {
    err = tx.Commit(ctx)
}
```

---

### 🔸 Batch-запросы

```go
batch := &pgx.Batch{}
batch.Queue("INSERT INTO logs(data) VALUES($1)", "log1")
batch.Queue("INSERT INTO logs(data) VALUES($1)", "log2")

br := pool.SendBatch(ctx, batch)
defer br.Close()

_, _ = br.Exec()
_, _ = br.Exec()
```

---

### 🔸 COPY (вставка больших объёмов данных)

```go
rows := [][]any{
    {1, "Alice"},
    {2, "Bob"},
}

_, err := pool.CopyFrom(
    ctx,
    pgx.Identifier{"users"},
    []string{"id", "name"},
    pgx.CopyFromRows(rows),
)
```

---

### 🔸 Получение JSON/JSONB как Go map

```go
var rawJSON []byte
err := pool.QueryRow(ctx, "SELECT settings FROM config WHERE id=$1", 1).Scan(&rawJSON)

var data map[string]interface{}
json.Unmarshal(rawJSON, &data)
```

---

### 🔸 Таймауты, отмена через context

```go
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

row := pool.QueryRow(ctx, "SELECT pg_sleep(10)")
```

---

### 🔸 Слушаем PostgreSQL-нотификации

```go
conn, _ := pgx.Connect(ctx, "...")
conn.Exec(ctx, "LISTEN my_channel")

for {
    notification, err := conn.WaitForNotification(ctx)
    fmt.Println("Got:", notification.Payload)
}
```

---

🧠 Готов расширить любой блок. Хочешь практику с транзакциями? Или сконцентрируемся на COPY, LISTEN/NOTIFY, или тестировании?
