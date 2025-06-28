
Отлично! Ты хочешь:

> **Загрузить `id` из CSV во временную таблицу в PostgreSQL, а потом обновлять данные по этим id батчами в Go**

Это очень распространённый сценарий:
- Импорт списка ID для обработки
- Обновление данных по этим ID порциями

---

## 🧠 Простыми словами

Ты:
1. Создаёшь временную таблицу: `tmp_ids (id INT)`
2. Загружаешь туда данные из CSV через `COPY FROM`
3. Делаешь `UPDATE ... FROM tmp_ids WHERE tmp_ids.id = main_table.id`
4. Удаляешь или очищаешь временную таблицу после обработки

---

# ✅ Шаг 1: Подготовка временной таблицы

```sql
CREATE TEMP TABLE tmp_ids (
    id INT PRIMARY KEY
);
```

📌 Временная таблица существует только в текущем сеансе.

---

# ✅ Шаг 2: Загрузка CSV во временную таблицу

В Go через `pgx`:

```go
func importIDsFromCSV(conn *pgx.Conn, filePath string) error {
    file, err := os.Open(filePath)
    if err != nil {
        return err
    }
    defer file.Close()

    // Очистим временную таблицу перед загрузкой
    _, err = conn.Exec(context.Background(), "TRUNCATE tmp_ids")
    if err != nil {
        return err
    }

    reader := csv.NewReader(file)

    _, err = reader.Read() // пропуск заголовка
    if err != nil && err != io.EOF {
        return err
    }

    copyRes, err := conn.CopyFrom(context.Background(),
        pgx.Identifier{"tmp_ids"},
        []string{"id"},
        pgx.QueryFuncReader(reader, func() ([]interface{}, error) {
            record, err := reader.Read()
            if err != nil {
                return nil, err
            }

            id, _ := strconv.Atoi(record[0])

            return []interface{}{id}, nil
        }),
    )

    fmt.Printf("Загружено %d записей\n", copyRes.RowsAffected())

    return err
}
```

---

# ✅ Шаг 3: Обновление основной таблицы по временной

После импорта CSV в `tmp_ids`, можно делать `UPDATE`:

```go
_, err = conn.Exec(context.Background(), `
    UPDATE users
    SET status = 'processed'
    FROM tmp_ids
    WHERE users.id = tmp_ids.id;
`)
if err != nil {
    panic(err)
}
```

📌 Это обновит всех пользователей из `tmp_ids`.

---

# ✅ Шаг 4: Обновление батчами (по частям)

Если у тебя много ID и ты не хочешь перегружать БД — делай по частям:

```go
batchSize := 1000
for {
    cmdTag, err := conn.Exec(context.Background(), `
        UPDATE users
        SET status = 'processed'
        FROM (
            SELECT id FROM tmp_ids
            ORDER BY id
            LIMIT $1
        ) AS batch
        WHERE users.id = batch.id;
        DELETE FROM tmp_ids
        WHERE id IN (
            SELECT id FROM tmp_ids
            ORDER BY id
            LIMIT $1
        );
    `, batchSize)

    if err != nil {
        panic(err)
    }

    if cmdTag.RowsAffected() == 0 {
        break
    }

    fmt.Printf("Обработано %d записей\n", cmdTag.RowsAffected())
}
```

---

## 📦 Полный пример: Go + pgx

```go
package main

import (
	"bufio"
	"context"
	"fmt"
	"io"
	"log"
	"os"
	"strconv"

	"github.com/jackc/pgx/v4"
	"encoding/csv"
)

func main() {
	conn, err := pgx.Connect(context.Background(), "postgres://postgres:postgres@localhost/demo?sslmode=disable")
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close(context.Background())

	// 1. Создаём временную таблицу
	_, err = conn.Exec(context.Background(), `
		CREATE TEMP TABLE IF NOT EXISTS tmp_ids (
			id INT PRIMARY KEY
		);`,
	)
	if err != nil {
		log.Fatal(err)
	}

	// 2. Чистим старые данные
	_, err = conn.Exec(context.Background(), "TRUNCATE tmp_ids;")
	if err != nil {
		log.Fatal(err)
	}

	// 3. Загружаем CSV
	err = importIDsFromCSV(conn, "ids.csv")
	if err != nil {
		log.Fatal("Ошибка импорта:", err)
	}

	// 4. Обновляем основную таблицу по частям
	batchSize := 1000
	for {
		cmdTag, err := conn.Exec(context.Background(), `
			WITH batch AS (
				SELECT id FROM tmp_ids
				LIMIT $1
			)
			UPDATE users
			SET status = 'processed'
			FROM batch
			WHERE users.id = batch.id;

			DELETE FROM tmp_ids
			WHERE id IN (
				SELECT id FROM tmp_ids
				LIMIT $1
			);`,
			batchSize,
		)

		if err != nil {
			log.Fatal("Ошибка обновления:", err)
		}

		if cmdTag.RowsAffected() == 0 {
			break
		}

		fmt.Printf("Обработано %d записей\n", cmdTag.RowsAffected())
	}
}

func importIDsFromCSV(conn *pgx.Conn, filePath string) error {
	file, err := os.Open(filePath)
	if err != nil {
		return err
	}
	defer file.Close()

	reader := csv.NewReader(bufio.NewReader(file))
	header, err := reader.Read()
	if err != nil || header[0] != "id" {
		log.Println("Нет заголовка или файл повреждён")
	}

	copyRes, err := conn.CopyFrom(context.Background(),
		pgx.Identifier{"tmp_ids"},
		[]string{"id"},
		pgx.QueryFuncReader(reader, func() ([]interface{}, error) {
			record, err := reader.Read()
			if err == io.EOF {
				return nil, err
			}

			id, _ := strconv.Atoi(strings.TrimSpace(record[0]))

			return []interface{}{id}, nil
		}),
	)

	if err != nil {
		return err
	}

	fmt.Printf("Загружено %d записей\n", copyRes.RowsAffected())
	return nil
}
```

---

## 📁 Содержимое `ids.csv`

```
id
1
2
3
...
```

---

## 🔁 Как работает обновление батчами?

| Что происходит | SQL |
|----------------|-----|
| Выбирается часть ID из `tmp_ids` | `SELECT id FROM tmp_ids LIMIT 1000` |
| Обновляются соответствующие записи | `UPDATE users SET ... WHERE id IN (...)` |
| Обработанные ID удаляются | `DELETE FROM tmp_ids WHERE id IN (...)` |

📌 Такой подход:
- Нагружает БД равномерно
- Позволяет отслеживать прогресс
- Помогает избежать lock'ов

---

## 🛠️ Как проверить, что всё прошло нормально?

```go
var count int
_ = conn.QueryRow(context.Background(), "SELECT COUNT(*) FROM tmp_ids").Scan(&count)
fmt.Println("Осталось обработать:", count)
```

---

## 🧩 Альтернативные стратегии

| Метод | Когда использовать |
|------|--------------------|
| `IN (SELECT id FROM tmp_ids)` | Если есть индекс на `id` |
| `JOIN` | Для сложных условий |
| `CTE + DELETE RETURNING` | Для работы с очередями |
| `UNLOGGED` временная таблица | Для ускорения импорта |
| `TRUNCATE` после обработки | Чтобы освободить место |

---

## 🧪 Как ещё можно читать CSV?

Можно использовать `encoding/csv`, `segmentio/encoding`, или `gocarina/carina`.

Пример:

```go
reader := csv.NewReader(file)
record, err := reader.Read()
```

---

## 📌 Советы по производительности

| Что сделать | Как сделать |
|------------|-------------|
| Использовать COPY вместо INSERT | `COPY FROM` намного быстрее |
| Делать `VACUUM ANALYZE` после массового обновления | Чтобы обновились статистики |
| Не забывать про индексы | На поле `id` в `users` и `tmp_ids` |
| Логировать ошибки | В функции `QueryFuncReader` |
| Разделить на транзакции | Для rollback при ошибках |
| Использовать `EXPLAIN ANALYZE` | Перед запуском |

---

## 📈 Как проверить план выполнения

```sql
EXPLAIN ANALYZE
UPDATE users
SET status = 'processed'
FROM tmp_ids
WHERE users.id = tmp_ids.id;
```

---

## 🧰 Как автоматизировать этот процесс?

Можно:
- Добавить флаги через `flag` или `cobra`
- Сделать Makefile
- Скрипт на Go → CLI-утилита

---

## 📌 Вывод

> ✅ Ты можешь грузить ID из CSV во временную таблицу  
> ✅ И обновлять данные по ним батчами через `LIMIT / DELETE RETURNING`  
> ✅ Это позволяет тебе работать с большими объёмами безопасно и быстро

---

Хочешь — могу показать:
- Как экспортировать ID обратно в CSV
- Как использовать `COPY TO` для логирования
- Как добавить прогрессбар
- Как сделать это через ORM (GORM, upper/db и т.д.)

Просто скажи 😊