
Отличный вопрос! Ты хочешь:

> **Создать CSV из таблицы с `jsonb`-полем в PostgreSQL, а потом загрузить его обратно в другую таблицу с заменами или преобразованиями**

Это очень распространённая задача:
- Экспорт данных
- Перенос между базами
- Миграции
- Обработка логов
- Преобразование структур

---

## 🧠 Простыми словами

1. У тебя есть таблица с полем `JSONB`, например:

```sql
CREATE TABLE logs (
    id SERIAL PRIMARY KEY,
    data JSONB
);
```

2. Ты хочешь выгрузить её содержимое в CSV, чтобы было так:

```csv
id,name,email,timestamp
1,Alice,alice@example.com,"2025-04-05 12:00:00"
2,Bob,bob@example.com,"2025-04-05 12:01:00"
```

3. Потом ты хочешь загрузить этот CSV в другую таблицу, возможно с изменением структуры:

```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    name TEXT,
    email TEXT,
    created_at TIMESTAMP
);
```

---

# ✅ Шаг 1: Выгрузка в CSV с разбором `JSONB`

Допустим, у тебя есть данные вроде:

```sql
INSERT INTO logs (data) VALUES
('{"name": "Alice", "email": "alice@example.com", "timestamp": "2025-04-05T12:00:00Z"}'),
('{"name": "Bob", "email": "bob@example.com", "timestamp": "2025-04-05T12:01:00Z"}');
```

Тогда можно сделать выгрузку так:

```sql
COPY (
    SELECT 
        id,
        data->>'name' AS name,
        data->>'email' AS email,
        data->>'timestamp' AS timestamp
    FROM logs
) TO '/tmp/logs_export.csv' WITH CSV HEADER;
```

📌 Это создаст файл `/tmp/logs_export.csv` с полями `id`, `name`, `email`, `timestamp`

---

# ✅ Шаг 2: Загрузка из CSV в другую таблицу

Предположим, ты хочешь загрузить эти данные в таблицу `users`.

```sql
TRUNCATE users;

COPY users (id, name, email, created_at)
FROM '/tmp/logs_export.csv'
WITH CSV HEADER;
```

📌 Если типы совпадают — всё окей  
Если нет — нужно сделать промежуточную обработку

---

# 🧩 Что делать, если нужно преобразовать поля?

Например:
- `timestamp` → `created_at` типа `TIMESTAMP`
- `name` → преобразование к верхнему регистру
- `email` → добавить домен, если отсутствует

### Вариант A: через временную таблицу

```sql
CREATE TEMP TABLE tmp_users (
    id INT,
    name TEXT,
    email TEXT,
    timestamp TEXT
);

COPY tmp_users FROM '/tmp/logs_export.csv' WITH CSV HEADER;

-- Делаем преобразования
INSERT INTO users (id, name, email, created_at)
SELECT 
    id,
    UPPER(name),
    COALESCE(email, 'no-email@example.com'),
    to_timestamp(timestamp, 'YYYY-MM-DD HH:MI:SS') -- или from ISO
FROM tmp_users;
```

---

### Вариант B: сразу в запросе `COPY ... TO`

Если тебе нужно сохранить преобразования при экспорте:

```sql
COPY (
    SELECT 
        id,
        UPPER(data->>'name') AS name,
        COALESCE(data->>'email', 'no-email@example.com') AS email,
        (data->>'timestamp')::timestamptz AS created_at
    FROM logs
) TO '/tmp/logs_processed.csv' WITH CSV HEADER;
```

---

# 📦 Полезные функции для работы с `JSONB`

| Операция | SQL |
|---------|-----|
| Извлечь значение по ключу | `data->>'key'` |
| Парсинг как дата/время | `to_timestamp(data->>'timestamp', 'YYYY-MM-DD...')` |
| Парсинг как число | `CAST((data->>'age') AS INTEGER)` |
| Парсинг как boolean | `(data->>'active')::BOOLEAN` |
| Поиск вложенных полей | `data->'address'->>'city'` |
| Проверка наличия ключа | `data ? 'required_key'` |
| Проверка значения | `data @> '{"status":"active"}'` |

---

## 🧪 Пример: сложное преобразование перед выгрузкой

```sql
COPY (
    SELECT 
        id,
        (data->>'name') AS name,
        (data->>'email') AS email,
        COALESCE(NULLIF(data->>'age', ''), '0')::INT AS age,
        (data->>'is_active')::BOOLEAN AS active,
        to_timestamp(data->>'timestamp', 'YYYY-MM-DD HH:MI:SS') AS created_at
    FROM logs
) TO '/tmp/users_transformed.csv' WITH CSV HEADER;
```

---

## 📌 Какие проблемы могут возникнуть?

| Проблема | Решение |
|----------|----------|
| Несоответствие типов | Явное преобразование (`::int`, `to_timestamp`) |
| Отсутствующие ключи | Используй `COALESCE(data->>'key', 'default')` |
| Плохой формат даты | `to_timestamp(..., 'format')` |
| Плохой CSV при наличии запятых / кавычек | `CSV HEADER` автоматически экранирует |
| Не хватает прав на файл | Убедись, что Postgres может читать/писать в `/tmp` |
| Ошибки парсинга JSON | Используй `jsonb_each_text()` или `jsonb_to_record()` |

---

# 🛠️ Альтернативы: без использования CSV

Если ты не обязан использовать CSV, то можешь просто скопировать данные через SQL:

```sql
INSERT INTO users (id, name, email, created_at)
SELECT 
    id,
    data->>'name',
    data->>'email',
    (data->>'timestamp')::timestamptz
FROM logs;
```

---

## 🧰 Как проверить, что CSV создан правильно?

```bash
head -n 5 /tmp/logs_export.csv
```

Или в самом PostgreSQL:

```sql
SELECT * FROM pg_read_csv_log('/tmp/logs_export.csv');
```

(при условии, что у тебя установлен расширения вроде `file_fdw`)

---

# 📋 Полезные советы

| Что делать | Как делать |
|------------|-------------|
| Проверить структуру JSON | `SELECT jsonb_object_keys(data) FROM logs LIMIT 1;` |
| Сделать CSV с определёнными полями | Через `SELECT id, data->>'name' ...` |
| Убедиться, что типы совпадают | `INSERT INTO ... SELECT ...` |
| Логировать ошибки импорта | Используй `ON CONFLICT DO NOTHING` |
| Добавить прогрессбар | В Go или Python используй `tqdm` |
| Автоматизировать процесс | Скрипты на Bash / Go / Python |
| Использовать `psql` вместо COPY | Для маленьких объёмов |

---

## 🧱 Как это сделать в Go?

Можно выгрузить и загрузить через `pgx`:

```go
rows, _ := conn.Query(ctx, `
    SELECT 
        id,
        data->>'name' AS name,
        data->>'email' AS email,
        data->>'timestamp' AS ts
    FROM logs;
`)

file, _ := os.Create("logs.csv")
writer := csv.NewWriter(file)

for rows.Next() {
    var id int
    var name string
    var email string
    var ts string
    rows.Scan(&id, &name, &email, &ts)

    writer.Write([]string{
        strconv.Itoa(id),
        name,
        email,
        ts,
    })
}

writer.Flush()
file.Close()
```

Затем загрузить в другую таблицу:

```go
file, _ := os.Open("logs.csv")
reader := csv.NewReader(file)

for {
    record, err := reader.Read()
    if err != nil {
        break
    }

    _, _ = conn.Exec(ctx, `
        INSERT INTO users (id, name, email, created_at)
        VALUES ($1, $2, $3, $4)
        ON CONFLICT (id) DO UPDATE SET
            name = EXCLUDED.name,
            email = EXCLUDED.email
    `, record[0], record[1], record[2], record[3])
}
```

---

## 🧠 Как работает `COPY`?

| Этап                              | Что происходит                             |
| --------------------------------- | ------------------------------------------ |
| `COPY (SELECT ...) TO 'file.csv'` | Выгружает результат в файл                 |
| `COPY table FROM 'file.csv'`      | Читает строки из файла                     |
| `WITH CSV HEADER`                 | Первую строку игнорирует                   |
| `DELIMITER ','`                   | Можно указать разделитель                  |
| `FORMAT CSV`                      | Форматирование с экранированием            |
| `FORCE_QUOTE`, `FORCE_NOT_NULL`   | Контроль за кавычками и пустыми значениями |

---

## 📊 Когда использовать `COPY`, когда — `INSERT INTO ... SELECT`

| Сценарий                        | Лучше использовать                          |
| ------------------------------- | ------------------------------------------- |
| Большие объемы данных           | `COPY`                                      |
| Маленькие объемы                | `INSERT INTO ... SELECT`                    |
| Нужна обработка на лету         | `COPY ... TO` + обработка + `COPY ... FROM` |
| Нужны замены                    | Создай временную таблицу                    |
| Нужны индексы                   | После импорта сделай `ANALYZE users`        |
| Нужно переписать поля           | `SELECT ... FROM ...` с трансформацией      |
| Нужно работать с JSONB напрямую | `jsonb_to_recordset`, `jsonb_each` и т.д.   |

---

## 🧩 А если JSONB сложный?

У тебя может быть:

```json
{
  "name": "Alice",
  "emails": ["a@example.com", "a@work.com"],
  "tags": {"admin": true, "verified": false},
  "meta": {"score": 98.76, "rank": 5}
}
```

Тогда:

```sql
COPY (
    SELECT 
        id,
        data->>'name' AS name,
        jsonb_path_query_first(data, '$.emails[*] ? (@ like_regex "@work.com")') #>> '{}' AS work_email,
        (data->'tags'->'admin')::BOOLEAN AS is_admin,
        (data->'meta'->'score')::FLOAT AS score
    FROM logs
) TO '/tmp/processed.csv' WITH CSV HEADER;
```

---

## 🧰 Какие ещё есть методы?

| Подход | Когда использовать |
|--------|---------------------|
| `jsonb_to_record` | Для преобразования JSONB в набор полей |
| `jsonb_populate_record` | То же самое, но с типами |
| `jsonb_array_elements` | Для массивов внутри JSONB |
| `jsonb_path_query` | Для сложных JSONB-запросов |
| `pg_read_csv` | Чтобы прочитать CSV как таблицу |
| `file_fdw` | Для внешнего доступа к CSV |
| `psql \copy` | Для ручного импорта |

---

## 📌 Вывод

> ✅ Чтобы создать CSV из `jsonb` и вставить его в другую таблицу:
- Используй `SELECT id, data->>'name' ...`
- Сохрани в CSV через `COPY (...) TO ...`
- Затем загрузи через `COPY ... FROM` или `INSERT INTO ... SELECT`

> 🧠 Но если ты хочешь делать замены:
- Используй временные таблицы
- Или делай `SELECT` с преобразованиями

> 🚀 Также можно:
- Сделать это в Go, Python, Rust
- Добавить прогрессбар
- Использовать `ON CONFLICT` для UPSERT
- Проверять типы и форматы заранее

---

Хочешь — могу показать:
- Как автоматизировать такие миграции
- Как использовать `file_fdw` для чтения CSV
- Как применять `jsonb_to_recordset`
- Как работать с большими JSONB
- Как тестировать производительность таких операций

Просто скажи 😊
