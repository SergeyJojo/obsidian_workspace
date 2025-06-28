
Отличный вопрос!

Ты хочешь:

> **Выполнять SQL-запрос несколько раз подряд, чтобы результаты записывались в CSV, но строки не повторялись между запусками.**

В твоём случае:
```sql
COPY (
  WITH bad_rows AS (
    SELECT id, data
    FROM some_table
    WHERE is_broken = true
    LIMIT 1000
  )
  SELECT id, data
  FROM bad_rows
) TO '/tmp/broken_rows.csv' CSV;
```

➡️ Это выражение каждый раз выбирает одни и те же `id`, если данные в таблице не меняются → при нескольких запусках ты просто **перезапишешь или добавишь дубли**

---

## 🧠 Простыми словами

> Чтобы запрос **возвращал разные строки при каждом выполнении**, нужно:
1. Помечать уже выгруженные строки (например, флагом)
2. Или использовать временную таблицу / список ID для исключения

📌 А чтобы они **не дублировались в CSV** — нужно либо чистить файл перед каждым запуском,  
либо проверять, какие ID уже есть в файле, и исключать их.

---

# ✅ Решение 1: помечай выгруженные строки

Добавь поле `exported BOOLEAN DEFAULT FALSE` в свою таблицу:

```sql
ALTER TABLE some_table ADD COLUMN exported BOOLEAN DEFAULT FALSE;
```

### Тогда твой запрос будет таким:

```sql
COPY (
  WITH bad_rows AS (
    SELECT id, data
    FROM some_table
    WHERE is_broken = true AND exported = false
    LIMIT 1000
  ),
  update_flag AS (
    UPDATE some_table
    SET exported = true
    WHERE id IN (SELECT id FROM bad_rows)
  )
  SELECT id, data FROM bad_rows
) TO '/tmp/broken_rows.csv' CSV;
```

### Что происходит?
- Находим непомеченные записи
- Делаем COPY
- Обновляем флаг `exported = true`
- Следующий запуск этого запроса возьмёт другие строки

📌 Это работает, если у тебя есть возможность изменять таблицу

---

# ✅ Решение 2: используй временную таблицу

Создай временную таблицу, где будешь хранить уже выгруженные ID:

```sql
CREATE TEMP TABLE exported_ids (
    id INT PRIMARY KEY
);
```

### Затем обновляй её после каждого запроса:

```sql
COPY (
  WITH bad_rows AS (
    SELECT s.id, s.data
    FROM some_table s
    LEFT JOIN exported_ids e ON s.id = e.id
    WHERE s.is_broken = true AND e.id IS NULL
    LIMIT 1000
  ),
  remember_exported AS (
    INSERT INTO exported_ids (id)
    SELECT id FROM bad_rows
    ON CONFLICT (id) DO NOTHING
  )
  SELECT id, data FROM bad_rows
) TO '/tmp/broken_rows.csv' CSV;
```

### Что здесь происходит?
- Берём только те строки, которых ещё нет в `exported_ids`
- После экспорта сохраняем их в эту таблицу
- При следующих запусках эти ID больше не будут выбираться

📌 Это позволяет тебе **повторно выполнять один и тот же запрос**, и при этом он будет брать всё новые и новые строки

---

# ✅ Решение 3: читай CSV перед следующей выгрузкой

Если ты **не можешь менять структуру базы**, можно сделать так:

```sql
COPY (
  SELECT id, data
  FROM some_table
  WHERE is_broken = true
    AND id NOT IN (
      SELECT id FROM pg_read_csv_log('/tmp/broken_rows.csv')
    )
  LIMIT 1000
) TO '/tmp/broken_rows.csv' CSV HEADER;
```

Но такой синтаксис не существует напрямую. Нужно **сначала загрузить содержимое CSV во временную таблицу**, а потом исключить эти ID.

---

## 🔁 Как это сделать в реальности?

### Шаг 1: создай временную таблицу для хранения уже выгруженных ID

```sql
CREATE TEMP TABLE exported_ids (
    id INT PRIMARY KEY
);
```

### Шаг 2: загрузи CSV во временный буфер

```go
// В Go или любом другом языке читаешь CSV и делаешь INSERT в exported_ids
```

Или через `psql`:

```bash
\copy exported_ids(id) FROM '/tmp/broken_rows.csv' DELIMITER ',' CSV HEADER
```

### Шаг 3: выполняй запрос с исключением

```sql
COPY (
  SELECT id, data
  FROM some_table
  WHERE is_broken = true
    AND id NOT IN (SELECT id FROM exported_ids)
  ORDER BY id
  LIMIT 1000
) TO '/tmp/broken_rows.csv' CSV HEADER;
```

---

## ✅ Решение 4: объедини в одном запросе (без изменения основной таблицы)

Можно использовать `NOT EXISTS` и временную таблицу:

```sql
-- Выгружаем и запоминаем ID
COPY (
  WITH new_rows AS (
    SELECT id, data
    FROM some_table
    WHERE is_broken = true
      AND id NOT IN (SELECT id FROM exported_ids)
    LIMIT 1000
  ),
  remember AS (
    INSERT INTO exported_ids
    SELECT id FROM new_rows
    ON CONFLICT (id) DO NOTHING
  )
  SELECT * FROM new_rows
) TO '/tmp/broken_rows.csv' CSV HEADER;
```

---

## 📌 Почему важно избегать дублей?

| Проблема | Описание |
|---------|----------|
| Дубликаты в CSV | Можно потерять данные или сломать бизнес-логику |
| Одна и та же строка обрабатывается дважды | Может привести к ошибкам |
| Логика "обработано" не отслеживается | Нет гарантии, что ты не перезапишешь старые данные |

---

## 🛠️ Как автоматизировать это в Go

```go
package main

import (
	"database/sql"
	"fmt"
	"log"
	"os"

	_ "github.com/lib/pq"
)

func main() {
	db, err := sql.Open("postgres", "user=postgres dbname=mydb sslmode=disable")
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	// 1. Создаем временную таблицу exported_ids
	_, _ = db.Exec(`CREATE TEMP TABLE IF NOT EXISTS exported_ids (id INT PRIMARY KEY)`)

	// 2. Выполняем COPY + запоминаем ID
	query := `
	WITH new_rows AS (
		SELECT id, data
		FROM some_table
		WHERE is_broken = true
			AND id NOT IN (SELECT id FROM exported_ids)
		LIMIT 1000
	),
	remember AS (
		INSERT INTO exported_ids (id)
		SELECT id FROM new_rows
		ON CONFLICT (id) DO NOTHING
	)
	SELECT id, data FROM new_rows;`

	// 3. Экспорт в CSV
	cmd := fmt.Sprintf(`COPY (%s) TO '/tmp/broken_rows.csv' CSV HEADER;`, query)
	_, err = db.Exec(cmd)
	if err != nil {
		log.Fatal("Ошибка экспорта:", err)
	}

	fmt.Println("Экспорт завершён")
}
```

---

## 📋 Советы по работе с CSV

| Что делать | Как делать |
|------------|-------------|
| Читать CSV в Go | Используй `encoding/csv` или `os.File` |
| Убедиться, что файл существует | `os.Stat()` |
| Проверять, есть ли уже выгруженные ID | `SELECT id FROM exported_ids` |
| Логировать ошибки | `log.Printf(...)` |
| Использовать transactions | Для атомарности операций |
| Очищать temp-таблицу | Если нужно начать заново |

---

## 🧩 Альтернатива без изменения таблицы

Если ты **не можешь ничего менять в БД**, тогда:

### Шаг 1: выгрузи все ID из CSV в отдельную таблицу:

```bash
\copy exported_ids(id) FROM '/tmp/broken_rows.csv' CSV HEADER
```

### Шаг 2: сделай запрос с исключением:

```sql
COPY (
  SELECT id, data
  FROM some_table
  WHERE is_broken = true
    AND id NOT IN (SELECT id FROM exported_ids)
  LIMIT 1000
) TO '/tmp/broken_rows.csv' CSV HEADER;
```

---

## 📊 Когда какое решение лучше?

| Сценарий | Лучше использовать |
|---------|--------------------|
| Можно менять таблицу | Добавь `exported` столбец |
| Нельзя менять таблицу | Используй временную таблицу `exported_ids` |
| Нет доступа к БД | Читай CSV в памяти и исключай дубли в коде |
| Много ядер / большие данные | Green Tea GC поможет с параллелизмом (если ты работаешь с Go) |
| Нужно повторное выполнение | Через временную таблицу или CSV-файл |
| Нужно восстановление после сбоя | Храни exported_ids в БД или файле |

---

## 🧰 Полезные команды PostgreSQL

| Что сделать | Команда |
|-------------|----------|
| Создать временную таблицу | `CREATE TEMP TABLE ...` |
| Добавить данные в неё | `INSERT INTO exported_ids (...)` |
| Удалить дубли | `DELETE FROM exported_ids WHERE id IN (...)` |
| Проверить, есть ли ID в списке | `SELECT id FROM exported_ids WHERE id = ?` |
| Объединить с CSV | `\copy exported_ids FROM ...` |
| Сделать CSV уникальным | Проверяй перед выгрузкой |

---

## 📌 Вывод

> ✅ Чтобы запрос не выводил одни и те же строки, тебе нужно:
- Отметить уже выгруженные строки
- Или хранить список выгруженных ID вне запроса
- Или читать CSV перед выгрузкой и исключать его содержимое

> 💡 Лучший способ:
- Создай временную таблицу `exported_ids`
- Перед каждым `COPY` делай `WHERE id NOT IN (SELECT id FROM exported_ids)`
- После экспорта вставляй эти ID в `exported_ids`

---

Хочешь — могу показать:
- Как автоматически обновлять exported_ids через Go
- Как тестировать такие запросы
- Как интегрировать в CLI-утилиту
- Как работать с большими CSV

Просто скажи 😊