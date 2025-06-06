# **Типовые PostgreSQL-запросы для повторения**

Эти запросы охватывают все основные операции: создание таблиц, индексы, CRUD-операции, JOIN, агрегацию и управление транзакциями. Сохраните их в шпаргалку для быстрого доступа.

---

## **1. Управление таблицами**
### **Создание таблицы**
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE,
    age INT CHECK (age >= 18),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### **Удаление таблицы**
```sql
DROP TABLE IF EXISTS users;
```

### **Изменение таблицы**
```sql
-- Добавление столбца
ALTER TABLE users ADD COLUMN is_active BOOLEAN DEFAULT TRUE;

-- Удаление столбца
ALTER TABLE users DROP COLUMN age;

-- Переименование столбца
ALTER TABLE users RENAME COLUMN email TO user_email;
```

---

## **2. Индексы**
### **Создание индекса**
```sql
-- Обычный индекс
CREATE INDEX idx_users_username ON users(username);

-- Уникальный индекс
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- Составной индекс
CREATE INDEX idx_users_created_at_active ON users(created_at, is_active);
```

### **Удаление индекса**
```sql
DROP INDEX IF EXISTS idx_users_username;
```

---

## **3. Вставка данных (INSERT)**
```sql
-- Вставка одной строки
INSERT INTO users (username, email, age) 
VALUES ('alice', 'alice@example.com', 25);

-- Вставка нескольких строк
INSERT INTO users (username, email, age) 
VALUES 
    ('bob', 'bob@example.com', 30),
    ('charlie', 'charlie@example.com', 22);

-- Вставка с возвратом ID
INSERT INTO users (username, email) 
VALUES ('dave', 'dave@example.com') 
RETURNING id;
```

---

## **4. Выборка данных (SELECT)**
### **Базовые запросы**
```sql
-- Выбор всех данных
SELECT * FROM users;

-- Выбор конкретных столбцов
SELECT username, email FROM users;

-- Сортировка
SELECT * FROM users ORDER BY created_at DESC;

-- Лимит и оффсет (пагинация)
SELECT * FROM users LIMIT 10 OFFSET 20;
```

### **Фильтрация (WHERE)**
```sql
-- Простое условие
SELECT * FROM users WHERE age > 25;

-- LIKE (поиск по шаблону)
SELECT * FROM users WHERE username LIKE 'a%';

-- IN (проверка на вхождение)
SELECT * FROM users WHERE id IN (1, 2, 3);

-- BETWEEN (диапазон)
SELECT * FROM users WHERE age BETWEEN 20 AND 30;
```

### **Агрегация (GROUP BY)**
```sql
-- Количество пользователей
SELECT COUNT(*) FROM users;

-- Группировка по возрасту
SELECT age, COUNT(*) as user_count 
FROM users 
GROUP BY age 
HAVING COUNT(*) > 1;
```

---

## **5. Обновление данных (UPDATE)**
```sql
-- Обновление одной записи
UPDATE users 
SET email = 'new_email@example.com' 
WHERE id = 1;

-- Обновление нескольких записей
UPDATE users 
SET is_active = FALSE 
WHERE created_at < '2023-01-01';
```

---

## **6. Удаление данных (DELETE)**
```sql
-- Удаление одной записи
DELETE FROM users WHERE id = 1;

-- Удаление по условию
DELETE FROM users WHERE is_active = FALSE;
```

---

## **7. JOIN (соединение таблиц)**
### **Создание связанных таблиц**
```sql
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(100) NOT NULL,
    content TEXT
);
```

### **INNER JOIN (только совпадающие записи)**
```sql
SELECT u.username, p.title 
FROM users u
JOIN posts p ON u.id = p.user_id;
```

### **LEFT JOIN (все записи из левой таблицы)**
```sql
SELECT u.username, p.title 
FROM users u
LEFT JOIN posts p ON u.id = p.user_id;
```

### **RIGHT JOIN и FULL JOIN**
```sql
-- RIGHT JOIN (все записи из правой таблицы)
SELECT u.username, p.title 
FROM users u
RIGHT JOIN posts p ON u.id = p.user_id;

-- FULL JOIN (все записи из обеих таблиц)
SELECT u.username, p.title 
FROM users u
FULL JOIN posts p ON u.id = p.user_id;
```

---

## **8. Транзакции**
```sql
BEGIN;

-- Удаление пользователя и его постов
DELETE FROM posts WHERE user_id = 1;
DELETE FROM users WHERE id = 1;

-- Если ошибок нет, подтверждаем
COMMIT;

-- Если что-то пошло не так, откатываем
-- ROLLBACK;
```

---

## **9. Работа с JSON**
```sql
-- Создание таблицы с JSON-полем
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    attributes JSONB
);

-- Вставка JSON
INSERT INTO products (name, attributes) 
VALUES ('Laptop', '{"color": "black", "ram": 16}');

-- Выборка по JSON-полю
SELECT name FROM products 
WHERE attributes->>'color' = 'black';

-- Обновление JSON
UPDATE products 
SET attributes = jsonb_set(attributes, '{ram}', '32') 
WHERE id = 1;
```

---

## **10. Часто используемые функции**
```sql
-- Текущая дата и время
SELECT NOW();

-- Форматирование даты
SELECT TO_CHAR(NOW(), 'YYYY-MM-DD HH24:MI:SS');

-- Округление чисел
SELECT ROUND(3.14159, 2);  -- 3.14

-- Конкатенация строк
SELECT CONCAT(username, ' (', email, ')') FROM users;
```

---

### **Заключение**
Эти запросы покрывают 90% повседневной работы с PostgreSQL.  
Рекомендую сохранить их в `.sql`-файл и периодически повторять! 🚀  

Если нужно что-то конкретное (например, оконные функции или партиционирование) — напишите, дополню!
