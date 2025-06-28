
Вот краткий и понятный гайд по созданию таблиц в PostgreSQL — с примерами, пояснениями и полезными советами:

---

### 📦 1. Базовый синтаксис

```sql
CREATE TABLE имя_таблицы (
    имя_столбца1 тип_данных [ограничения],
    имя_столбца2 тип_данных [ограничения],
    ...
);
```

Пример:

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

### 🧱 2. Основные типы данных

- `INTEGER`, `BIGINT` — целые числа
- `SERIAL` — автоинкрементное целое (удобно для `id`)
- `VARCHAR(n)` — строка длиной до `n` символов
- `TEXT` — строка произвольной длины
- `BOOLEAN` — `true` / `false`
- `DATE`, `TIMESTAMP` — дата и дата-время

---

### 🔒 3. Ограничения (constraints)

- `PRIMARY KEY` — уникальный идентификатор строки
- `UNIQUE` — значение должно быть уникальным
- `NOT NULL` — значение обязательно
- `DEFAULT` — значение по умолчанию
- `CHECK` — условие (например, `CHECK (age >= 18)`)

---

### 🔗 4. Внешние ключи (foreign keys)

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    total NUMERIC(10, 2) NOT NULL
);
```

---

### 🧪 5. Проверка и удаление

- Проверить структуру:  
  ```sql
  \d имя_таблицы
  ```
- Удалить таблицу:  
  ```sql
  DROP TABLE имя_таблицы;
  ```

---

