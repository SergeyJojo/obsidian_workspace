
Вот чек-лист, который поможет тебе не забывать важные моменты при работе с SQL:

https://sql-academy.org/ru/interview-questions

### Основы запросов:

1. **SELECT**: Чтобы извлечь данные:
    
    ```sql
    SELECT column1, column2 FROM table WHERE condition;
    ```
    
2. **WHERE**: Для фильтрации данных.
    
    - Используй операторы: `=`, `>`, `<`, `BETWEEN`, `IN`, `LIKE`, `IS NULL`.
3. **ORDER BY**: Для сортировки результатов.
    
    - Сортировка по возрастанию или убыванию:
        
        ```sql
        ORDER BY column ASC|DESC;
        ```
        
4. **LIMIT**: Для ограничения количества выводимых строк.
    
    ```sql
    SELECT * FROM table LIMIT 10;
    ```
    

### Работа с объединениями:

1. **INNER JOIN**: Для выборки строк, которые совпадают в обеих таблицах.
    
    ```sql
    SELECT * FROM table1 INNER JOIN table2 ON table1.id = table2.id;
    ```
    
2. **LEFT JOIN (или LEFT OUTER JOIN)**: Для выборки всех строк из первой таблицы и совпадающих строк из второй.
    
    ```sql
    SELECT * FROM table1 LEFT JOIN table2 ON table1.id = table2.id;
    ```
    
3. **RIGHT JOIN (или RIGHT OUTER JOIN)**: Для выборки всех строк из второй таблицы и совпадающих строк из первой.
    
    ```sql
    SELECT * FROM table1 RIGHT JOIN table2 ON table1.id = table2.id;
    ```
    
4. **FULL OUTER JOIN**: Для получения всех строк из обеих таблиц.
    
    ```sql
    SELECT * FROM table1 FULL OUTER JOIN table2 ON table1.id = table2.id;
    ```
    

### Агрегация:

1. **GROUP BY**: Для группировки данных.
    
    ```sql
    SELECT column, COUNT(*) FROM table GROUP BY column;
    ```
    
2. **Агрегатные функции**:
    
    - **COUNT()** — подсчет строк.
    - **SUM()** — сумма значений.
    - **AVG()** — среднее значение.
    - **MIN()** — минимальное значение.
    - **MAX()** — максимальное значение.
3. **HAVING**: Для фильтрации результатов после группировки.
    
    ```sql
    SELECT column, COUNT(*) FROM table GROUP BY column HAVING COUNT(*) > 5;
    ```
    

### Подзапросы:

4. **В SELECT**: Подзапрос внутри основного запроса.
    
    ```sql
    SELECT column FROM table WHERE id = (SELECT id FROM other_table WHERE condition);
    ```
    
5. **В WHERE**: Использование подзапросов в условиях.
    
    ```sql
    SELECT * FROM table WHERE column IN (SELECT column FROM other_table);
    ```
    

### Операции с данными:

6. **INSERT**: Вставка данных.
    
    ```sql
    INSERT INTO table (column1, column2) VALUES (value1, value2);
    ```
    
7. **UPDATE**: Обновление данных.
    
    ```sql
    UPDATE table SET column1 = value1 WHERE condition;
    ```
    
8. **DELETE**: Удаление данных.
    
    ```sql
    DELETE FROM table WHERE condition;
    ```
    
9. **TRUNCATE**: Быстрое удаление всех данных из таблицы (без возможности восстановления).
    
    ```sql
    TRUNCATE TABLE table;
    ```
    

### Индексы:

10. **Создание индекса**:
    
    ```sql
    CREATE INDEX index_name ON table (column);
    ```
    
11. **Удаление индекса**:
    
    ```sql
    DROP INDEX index_name;
    ```
    

### Модификация структуры базы данных:

12. **ALTER TABLE**:
    
    - Добавить колонку:
        
        ```sql
        ALTER TABLE table ADD column_name datatype;
        ```
        
    - Изменить колонку:
        
        ```sql
        ALTER TABLE table MODIFY column_name new_datatype;
        ```
        
    - Удалить колонку:
        
        ```sql
        ALTER TABLE table DROP COLUMN column_name;
        ```
        
13. **CREATE TABLE**: Для создания таблицы.
    
    ```sql
    CREATE TABLE table_name (
        column1 datatype,
        column2 datatype
    );
    ```
    
14. **DROP TABLE**: Удаление таблицы.
    
    ```sql
    DROP TABLE table;
    ```
    

### Транзакции:

15. **BEGIN TRANSACTION**: Начало транзакции.
    
    ```sql
    BEGIN TRANSACTION;
    ```
    
16. **COMMIT**: Фиксация изменений.
    
    ```sql
    COMMIT;
    ```
    
17. **ROLLBACK**: Отмена изменений.
    
    ```sql
    ROLLBACK;
    ```
    

### Оптимизация:

18. **EXPLAIN**: Для анализа плана выполнения запроса.
    
    ```sql
    EXPLAIN SELECT * FROM table;
    ```
    
19. **Использование индексов**: Важность индексов на полях, по которым часто происходит фильтрация или сортировка.
    

---

Вот чек-лист для более продвинутых знаний SQL:

### 1. **Подзапросы (Subqueries)**:

- **Подзапросы в SELECT**: Используются для возвращения значения, которое можно использовать в основном запросе.
    
    ```sql
    SELECT name, (SELECT MAX(salary) FROM employees WHERE department = 'HR') AS max_salary
    FROM employees;
    ```
    
- **Коррелированные подзапросы**: Эти подзапросы используют данные внешнего запроса. Они выполняются для каждой строки внешнего запроса.
    
    ```sql
    SELECT name, salary
    FROM employees e
    WHERE salary > (SELECT AVG(salary) FROM employees WHERE department = e.department);
    ```
    

### 2. **Окна и аналитические функции**:

- **Окно**: Это диапазон строк, с которыми работает запрос. В отличие от GROUP BY, оконные функции не сжимаются в одну строку, а возвращают результат для каждой строки.
    
- **Пример оконной функции**:
    
    ```sql
    SELECT name, salary, RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rank
    FROM employees;
    ```
    
- **Другие аналитические функции**:
    
    - **ROW_NUMBER()**: Нумерация строк.
    - **DENSE_RANK()**: Нумерация с сохранением одинаковых рангов.
    - **NTILE()**: Разбиение данных на несколько групп.

### 3. **Создание и использование представлений (Views)**:

- **Создание представления**: Представления позволяют сохранять часто используемые запросы.
    
    ```sql
    CREATE VIEW employee_salary AS
    SELECT name, salary, department FROM employees WHERE salary > 50000;
    ```
    
- **Использование представлений**:
    
    ```sql
    SELECT * FROM employee_salary;
    ```
    
- **Обновление представлений**: Для обновления представлений необходимо использовать `CREATE OR REPLACE VIEW`.
    
    ```sql
    CREATE OR REPLACE VIEW employee_salary AS
    SELECT name, salary, department, age FROM employees WHERE salary > 50000;
    ```
    

### 4. **Индексирование**:

- **Типы индексов**:
    
    - **UNIQUE Index**: Обеспечивает уникальность значений в столбце.
    - **Full-text index**: Используется для текстовых поисков.
    - **Composite index**: Индекс, включающий несколько столбцов.
- **Создание уникального индекса**:
    
    ```sql
    CREATE UNIQUE INDEX idx_employee_id ON employees (employee_id);
    ```
    
- **Удаление индекса**:
    
    ```sql
    DROP INDEX idx_employee_id;
    ```
    

### 5. **Триггеры (Triggers)**:

- **Создание триггера**: Триггеры позволяют автоматически выполнять действия при вставке, обновлении или удалении данных.
    
    ```sql
    CREATE TRIGGER before_insert_employee
    BEFORE INSERT ON employees
    FOR EACH ROW
    SET NEW.created_at = NOW();
    ```
    
- **Типы триггеров**:
    
    - **BEFORE**: Действие выполняется до изменения данных.
    - **AFTER**: Действие выполняется после изменения данных.

### 6. **Транзакции**:

- **Сложные транзакции**: Использование нескольких операций внутри одной транзакции для обеспечения атомарности.
    
    ```sql
    BEGIN TRANSACTION;
    UPDATE employees SET salary = salary * 1.1 WHERE department = 'Sales';
    INSERT INTO audit_log (action) VALUES ('Salary update');
    COMMIT;
    ```
    
- **Управление транзакциями**:
    
    - **SAVEPOINT**: Установка точки сохранения, к которой можно откатиться.
        
        ```sql
        SAVEPOINT sp1;
        ```
        
    - **ROLLBACK TO SAVEPOINT**: Откат к сохраненной точке.
        
        ```sql
        ROLLBACK TO SAVEPOINT sp1;
        ```
        

### 7. **Оптимизация запросов**:

- **EXPLAIN**: Используется для анализа плана выполнения запроса.
    
    ```sql
    EXPLAIN SELECT * FROM employees WHERE department = 'HR';
    ```
    
- **Индексирование**: Использование индексов на полях, которые часто используются в фильтрах и для сортировки.
    
- **Использование JOIN**: Правильное использование типов соединений (INNER JOIN, LEFT JOIN, и т.д.), чтобы избежать ненужных операций и увеличить производительность.
    
- **Нормализация**:
    
    - Приведение базы данных к 3NF (Третья нормальная форма) для предотвращения избыточности данных.
    - Разделение таблиц на логические блоки для упрощения запросов и улучшения производительности.

### 8. **Хранимые процедуры и функции**:

- **Создание хранимой процедуры**: Хранимые процедуры могут выполнять несколько операций и возвращать результаты.
    
    ```sql
    CREATE PROCEDURE get_employee_info (IN emp_id INT)
    BEGIN
        SELECT name, salary FROM employees WHERE employee_id = emp_id;
    END;
    ```
    
- **Вызов хранимой процедуры**:
    
    ```sql
    CALL get_employee_info(1);
    ```
    
- **Создание функции**: Функции могут быть использованы для вычислений и возвращать значения.
    
    ```sql
    CREATE FUNCTION get_bonus (salary DECIMAL)
    RETURNS DECIMAL
    BEGIN
        RETURN salary * 0.1;
    END;
    ```
    

### 9. **Партиционирование таблиц**:

- **Партиционирование** позволяет разделить таблицу на более мелкие части (партиции), улучшая производительность запросов на большие объемы данных.
    
    ```sql
    CREATE TABLE employees (
        employee_id INT,
        name VARCHAR(100),
        department VARCHAR(50)
    )
    PARTITION BY RANGE (employee_id) (
        PARTITION p0 VALUES LESS THAN (100),
        PARTITION p1 VALUES LESS THAN (200)
    );
    ```
    

### 10. **Масштабирование и репликация**:

- **Мастер-слейв репликация**: Репликация позволяет создавать резервные копии данных и масштабировать чтение.
    
- **Шардирование**: Деление базы данных на более мелкие, независимые части для улучшения производительности.
    

---

Этот чек-лист охватывает более сложные и продвинутые темы в SQL. Регулярная практика этих концепций позволит тебе значительно углубить знания и лучше справляться с более сложными задачами.