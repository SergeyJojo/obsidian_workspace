
Конечно! Ниже ты найдёшь **пошаговый гайд по работе с Redis в Go (Golang)**. В нём разберём:

- Подключение к Redis
- Основные операции: `SET`, `GET`, `DEL`, `EXPIRE`
- Работу с хэшами, списками, множествами
- Использование JSON и структур
- Примеры асинхронной работы и пайплайнов

---

# 🧠 Redis в Go: Полный Гайд

## 🛠 Технологии

- Язык: **Go (Golang)**
- Библиотека: **[go-redis](https://github.com/go-redis/redis/v8)** — одна из самых популярных библиотек для Redis на Go
- Версия: поддержка Redis 6+ и Go 1.18+

---

## ✅ Установка

```bash
go get github.com/go-redis/redis/v8
```

Также потребуется контекст (встроен в Go):

```go
import (
	"context"
	"fmt"
)
```

---

## 🔌 1. Подключение к Redis

### Простое подключение:

```go
package main

import (
	"context"
	"fmt"
	"github.com/go-redis/redis/v8"
)

var ctx = context.Background()

func main() {
	rdb := redis.NewClient(&redis.Options{
		Addr:     "localhost:6379", // host:port
		Password: "",               // no password
		DB:       0,                // default DB
	})

	// Проверка подключения
	_, err := rdb.Ping(ctx).Result()
	if err != nil {
		panic("Can't connect to Redis")
	}

	fmt.Println("Connected to Redis!")
}
```

---

## 📥 2. Основные операции: SET, GET, DEL, EXPIRE

```go
// Запись значения
err := rdb.Set(ctx, "key1", "value1", 0).Err()
if err != nil {
	panic(err)
}

// Чтение значения
val, err := rdb.Get(ctx, "key1").Result()
if err != nil {
	panic(err)
}
fmt.Println("key1:", val)

// Удаление ключа
rdb.Del(ctx, "key1")

// Установка TTL
rdb.Expire(ctx, "key2", 5*time.Second)
```

---

## 🗂 3. Работа с типами данных

### Хэши (`HSET`, `HGET`)

```go
rdb.HSet(ctx, "user:1000", map[string]interface{}{
	"name":  "Alice",
	"email": "alice@example.com",
})

name := rdb.HGet(ctx, "user:1000", "name").Val()
fmt.Println("User name:", name)
```

### Списки (`LPUSH`, `RPUSH`, `LRANGE`)

```go
rdb.LPush(ctx, "logs", "log1")
rdb.RPush(ctx, "logs", "log2")

logs := rdb.LRange(ctx, "logs", 0, -1).Val()
for _, log := range logs {
	fmt.Println(log)
}
```

### Множества (`SADD`, `SMEMBERS`)

```go
rdb.SAdd(ctx, "tags", "go", "redis", "go") // повторяющиеся игнорируются

	tags := rdb.SMembers(ctx, "tags").Val()
	fmt.Println("Tags:", tags)
```

---

## 🧱 4. Работа с JSON и структурами

Можно сохранять структуры как JSON.

```go
type User struct {
	ID    int    `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email"`
}

user := User{ID: 1, Name: "Bob", Email: "bob@example.com"}
jsonUser, _ := json.Marshal(user)

rdb.Set(ctx, "user:1", jsonUser, 0)

// Получаем и распаковываем
val, _ := rdb.Get(ctx, "user:1").Bytes()
var decoded User
json.Unmarshal(val, &decoded)

fmt.Printf("Decoded user: %+v\n", decoded)
```

---

## ⚡ 5. Асинхронная работа и пайплайны

### Пайплайн (Pipeline)

Позволяет выполнить несколько команд за один раундтрип:

```go
pipe := rdb.Pipeline()

cmd1 := pipe.Set(ctx, "key1", "value1", 0)
cmd2 := pipe.Expire(ctx, "key1", 10*time.Second)

_, err := pipe.Exec(ctx)
if err != nil {
	panic(err)
}
```

### TxPipeline (транзакции)

```go
tx := rdb.TxPipeline()

tx.Set(ctx, "keyA", "100", 0)
tx.Set(ctx, "keyB", "200", 0)

_, err := tx.Exec(ctx)
```

---

## 🧪 6. Проверка существования и TTL

```go
exists, _ := rdb.Exists(ctx, "key1").Result()
fmt.Println("Exists:", exists)

ttl, _ := rdb.TTL(ctx, "key1").Result()
fmt.Println("TTL:", ttl)
```

---

## 🧼 7. Удаление по маске (например, все ключи с префиксом)

```go
iter := rdb.Scan(ctx, 0, "user:*", 0).Iterator()
for iter.Next(ctx) {
	key := iter.Val()
	rdb.Del(ctx, key)
}
```

---

## 🔄 8. Публикация / Подписка (Pub/Sub)

### Publisher:

```go
err := rdb.Publish(ctx, "channel1", "Hello from Redis!").Err()
if err != nil {
	panic(err)
}
```

### Subscriber:

```go
sub := rdb.Subscribe(ctx, "channel1")
msgChannel := sub.Channel()

for msg := range msgChannel {
	fmt.Println("Received:", msg.Payload)
}
```

---

## 📊 9. Health Check и мониторинг

```go
info := rdb.Info(ctx, "memory").Val()
fmt.Println("Memory info:\n", info)

ping, _ := rdb.Ping(ctx).Result()
fmt.Println("Ping:", ping)
```

---

## 📌 Лучшие практики

| Что делать | Как делать |
|-----------|------------|
| Подключение | Используйте пулы, таймауты |
| Обработка ошибок | Проверяйте каждую операцию |
| Кеширование | Используйте `SET key value EX seconds` |
| Тестирование | Используйте тестовые ключи или in-memory mock |
| Производительность | Используйте пайплайны и Lua скрипты |

---

## 📚 Полезные ссылки

- [go-redis GitHub](https://github.com/go-redis/redis)
- [Redis Commands Reference](https://redis.io/commands/)
- [Redis CLI Tutorial](https://redis.io/topics/cli)
- [Go Redis Examples](https://github.com/go-redis/redis/tree/master/example)

---

Хочешь пример использования **Lua скриптов**, **контекста с таймаутом**, **интеграции с Gin** или **реализации Rate Limiter через Redis**? Напиши — покажу!