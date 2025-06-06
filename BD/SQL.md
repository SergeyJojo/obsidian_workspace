[[BD/Postgres/ОСНОВЫ]]
[[SQL JOIN]]
[[Индексы]]
[[УРОВНИ ИЗОЛЯЦИЙ]]
[[АГРЕГИРУЮЩИЕ ФУНКЦИИ]]

Окей, сейчас я тебе расскажу про **SQL**, начиная с продвинутых трюков, малоизвестных фич и редких случаев их применения, которые точно произведут впечатление на интервьюере. Это будут темы, которые обычно не обсуждаются в стандартных вопросах, но демонстрируют глубокое понимание работы **SQL** и **оптимизации запросов**.

---

## **1. Рекурсивные CTE (Common Table Expressions)**

Рекурсивные CTE — мощный инструмент для работы с деревьями, графами и иерархическими структурами.

---

### **1.1. Построение иерархии**

Предположим, у нас есть таблица сотрудников, где каждый сотрудник подчиняется другому:

```sql
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    manager_id INT REFERENCES employees(id)
);
```

#### **Пример: Найти все подчинённые одного менеджера**

```sql
WITH RECURSIVE employee_hierarchy AS (
    SELECT id, name, manager_id, 1 AS level
    FROM employees
    WHERE manager_id IS NULL -- Начинаем с верхнего уровня (CEO)

    UNION ALL

    SELECT e.id, e.name, e.manager_id, eh.level + 1
    FROM employees e
    INNER JOIN employee_hierarchy eh ON e.manager_id = eh.id
)
SELECT * FROM employee_hierarchy ORDER BY level, name;
```

**Результат**:

```
id | name       | manager_id | level
------------------------------------
1  | CEO        | NULL       | 1
2  | Manager A  | 1          | 2
3  | Employee 1 | 2          | 3
4  | Employee 2 | 2          | 3
```

---

### **1.2. Вычисление пути до корня**

Добавим путь в формате строки:

```sql
WITH RECURSIVE employee_hierarchy AS (
    SELECT id, name, manager_id, CAST(name AS TEXT) AS path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    SELECT e.id, e.name, e.manager_id, CONCAT(eh.path, ' -> ', e.name)
    FROM employees e
    INNER JOIN employee_hierarchy eh ON e.manager_id = eh.id
)
SELECT * FROM employee_hierarchy;
```

**Результат**:

```
id | name       | manager_id | path
------------------------------------------
1  | CEO        | NULL       | CEO
2  | Manager A  | 1          | CEO -> Manager A
3  | Employee 1 | 2          | CEO -> Manager A -> Employee 1
```

---

## **2. Индексы и их малоизвестные возможности**

---

### **2.1. Использование частичных индексов**

Частичные индексы позволяют индексировать только подмножество данных.

#### **Пример: Только активные пользователи**

```sql
CREATE INDEX idx_active_users ON users (email) WHERE is_active = true;
```

Преимущество:

- Индекс содержит только активных пользователей, уменьшая его размер.
- Значительно ускоряет запросы вида:
    
    ```sql
    SELECT email FROM users WHERE is_active = true;
    ```
    

---

### **2.2. Индексы на выражениях**

Вы можете индексировать не только столбцы, но и вычисляемые выражения.

#### **Пример: Ускорение поиска по нижнему регистру**

```sql
CREATE INDEX idx_lower_email ON users (LOWER(email));
```

Теперь запрос:

```sql
SELECT * FROM users WHERE LOWER(email) = 'example@gmail.com';
```

использует индекс.

---

### **2.3. GiST и GIN индексы**

- **GIN (Generalized Inverted Index)**: Идеален для полнотекстового поиска и работы с JSONB.
- **GiST (Generalized Search Tree)**: Подходит для поиска по диапазонам или географическим данным.

#### **Пример: GIN для JSONB**

```sql
CREATE INDEX idx_jsonb_tags ON posts USING gin (tags jsonb_path_ops);

SELECT * FROM posts WHERE tags @> '{"category": "tech"}';
```

---

## **3. Window Functions (Оконные функции)**

Оконные функции позволяют выполнять операции над строками без группировки.

---

### **3.1. Нумерация строк**

#### **Пример: Пронумеровать строки в группе**

```sql
SELECT
    department,
    name,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rank
FROM employees;
```

**Результат**:

```
department | name       | rank
--------------------------------
HR         | Alice      | 1
HR         | Bob        | 2
IT         | Charlie    | 1
IT         | Dave       | 2
```

---

### **3.2. Сумма и среднее по окну**

#### **Пример: Скользящее среднее**

```sql
SELECT
    date,
    sales,
    AVG(sales) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS moving_avg
FROM sales;
```

---

## **4. Оптимизация запросов**

---

### **4.1. EXPLAIN и EXPLAIN ANALYZE**

Используйте `EXPLAIN` для анализа выполнения запросов.

#### **Пример анализа запроса**

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'example@gmail.com';
```

**Результат** (пример):

```
Index Scan using idx_email on users  (cost=0.25..8.27 rows=1 width=64)
```

---

### **4.2. Common Pitfalls (ошибки)**

1. **Фильтры после `JOIN`**:
    
    - Плохо:
        
        ```sql
        SELECT * FROM orders o
        JOIN users u ON o.user_id = u.id
        WHERE u.is_active = true;
        ```
        
    - Хорошо:
        
        ```sql
        SELECT * FROM orders o
        JOIN users u ON o.user_id = u.id AND u.is_active = true;
        ```
        
2. **Избегайте `SELECT *`**:
    
    - Указывайте только нужные столбцы:
        
        ```sql
        SELECT id, name FROM users;
        ```
        

---

### **4.3. Materialized Views**

Материализованные представления сохраняют результаты запросов, которые можно обновлять по расписанию.

#### **Пример: Создание отчёта**

```sql
CREATE MATERIALIZED VIEW sales_report AS
SELECT region, SUM(amount) AS total_sales
FROM sales
GROUP BY region;

-- Обновление представления
REFRESH MATERIALIZED VIEW sales_report;
```

---

## **5. Работа с большими данными**

---

### **5.1. Партицирование таблиц**

Партицирование разбивает таблицу на логические части.

#### **Пример: Партицирование по дате**

```sql
CREATE TABLE logs (
    id SERIAL PRIMARY KEY,
    log_date DATE NOT NULL,
    message TEXT
) PARTITION BY RANGE (log_date);

CREATE TABLE logs_2023 PARTITION OF logs FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');
```

---

### **5.2. Parallel Query Execution**

PostgreSQL поддерживает параллельное выполнение запросов для ускорения.

#### **Пример: Параллельная агрегация**

```sql
SET max_parallel_workers_per_gather = 4;

SELECT SUM(sales) FROM large_table;
```

---

## **6. Интересные SQL-трюки**

---

### **6.1. Генерация последовательностей**

#### **Пример: Генерация дат**

```sql
SELECT generate_series('2023-01-01'::DATE, '2023-12-31'::DATE, '1 month') AS month;
```

---

### **6.2. Pivot-таблицы**

#### **Пример: Поворот таблицы**

```sql
SELECT *
FROM crosstab(
    'SELECT department, month, sales FROM sales_data ORDER BY 1, 2',
    'VALUES (''Jan''), (''Feb''), (''Mar'')'
) AS ct(department TEXT, Jan INT, Feb INT, Mar INT);
```

---

Если хочешь ещё **глубже** или нужно покрыть темы вроде **PostGIS**, **многослойных транзакций**, или **рекурсивных графов**, просто скажи **"ещё"!** 😊
