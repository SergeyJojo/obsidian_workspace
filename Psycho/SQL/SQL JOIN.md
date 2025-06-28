**JOIN** в SQL — это оператор, который позволяет объединить данные из нескольких таблиц в одну результирующую выборку на основе заданного условия. Это один из самых мощных инструментов работы с реляционными базами данных.

---

## **Типы JOIN в SQL**

### **1. INNER JOIN**

- Возвращает только те строки, у которых есть совпадения в обеих таблицах.
- **Синтаксис:**
    
    ```sql
    SELECT columns
    FROM table1
    INNER JOIN table2
    ON table1.column = table2.column;
    ```
    
- **Пример:**
    
    ```sql
    SELECT employees.name, departments.name AS department
    FROM employees
    INNER JOIN departments
    ON employees.department_id = departments.id;
    ```
    

---

### **2. LEFT JOIN (LEFT OUTER JOIN)**

- Возвращает все строки из левой таблицы и совпадающие строки из правой. Если совпадений нет, для правой таблицы будут `NULL`.
    
- **Синтаксис:**
    
    ```sql
    SELECT columns
    FROM table1
    LEFT JOIN table2
    ON table1.column = table2.column;
    ```
    
- **Пример:**
    
    ```sql
    SELECT employees.name, departments.name AS department
    FROM employees
    LEFT JOIN departments
    ON employees.department_id = departments.id;
    ```
    
- **Результат:** Все сотрудники, включая тех, кто не привязан к отделу (NULL в столбце `department`).
    

---

### **3. RIGHT JOIN (RIGHT OUTER JOIN)**

- Возвращает все строки из правой таблицы и совпадающие строки из левой. Если совпадений нет, для левой таблицы будут `NULL`.
    
- **Синтаксис:**
    
    ```sql
    SELECT columns
    FROM table1
    RIGHT JOIN table2
    ON table1.column = table2.column;
    ```
    
- **Пример:**
    
    ```sql
    SELECT employees.name, departments.name AS department
    FROM employees
    RIGHT JOIN departments
    ON employees.department_id = departments.id;
    ```
    
- **Результат:** Все отделы, включая те, где нет сотрудников (NULL в столбце `name`).
    

---

### **4. FULL JOIN (FULL OUTER JOIN)**

- Возвращает все строки из обеих таблиц. Если совпадений нет, для отсутствующих данных будут `NULL`.
    
- **Синтаксис:**
    
    ```sql
    SELECT columns
    FROM table1
    FULL JOIN table2
    ON table1.column = table2.column;
    ```
    
- **Пример:**
    
    ```sql
    SELECT employees.name, departments.name AS department
    FROM employees
    FULL JOIN departments
    ON employees.department_id = departments.id;
    ```
    
- **Результат:** Все сотрудники и все отделы, включая те, где нет совпадений.
    

---

### **5. CROSS JOIN**

- Возвращает декартово произведение двух таблиц (каждая строка из первой таблицы соединяется с каждой строкой из второй таблицы).
    
- **Синтаксис:**
    
    ```sql
    SELECT columns
    FROM table1
    CROSS JOIN table2;
    ```
    
- **Пример:**
    
    ```sql
    SELECT employees.name, departments.name AS department
    FROM employees
    CROSS JOIN departments;
    ```
    
- **Результат:** Все возможные комбинации сотрудников и отделов.
    

---

### **6. SELF JOIN**

- Это особый случай, когда таблица соединяется сама с собой.
    
- **Синтаксис:**
    
    ```sql
    SELECT A.column1, B.column2
    FROM table A
    INNER JOIN table B
    ON A.column = B.column;
    ```
    
- **Пример:**
    
    ```sql
    SELECT E1.name AS employee, E2.name AS manager
    FROM employees E1
    INNER JOIN employees E2
    ON E1.manager_id = E2.id;
    ```
    
- **Результат:** Список сотрудников и их менеджеров.
    

---

### **Примеры задач с JOIN**

#### **Пример 1: Объединение данных о продажах**

Таблицы:

- `orders`:
    
    |order_id|customer_id|amount|
    |---|---|---|
    |1|101|500|
    |2|102|300|
    |3|103|400|
    
- `customers`:
    
    |customer_id|name|
    |---|---|
    |101|Alice|
    |102|Bob|
    |104|Charlie|
    

**Запрос:**

```sql
SELECT orders.order_id, customers.name, orders.amount
FROM orders
LEFT JOIN customers
ON orders.customer_id = customers.customer_id;
```

**Результат:**

|order_id|name|amount|
|---|---|---|
|1|Alice|500|
|2|Bob|300|
|3|NULL|400|

---

#### **Пример 2: Отделы без сотрудников**

```sql
SELECT departments.name
FROM departments
LEFT JOIN employees
ON departments.id = employees.department_id
WHERE employees.id IS NULL;
```

- Показывает отделы, где нет сотрудников.

---

## **Советы по работе с JOIN**

1. **Оптимизация производительности:**
    
    - Убедитесь, что для столбцов, используемых в `ON` или `WHERE`, существуют индексы.
2. **Избегайте лишних JOIN:**
    
    - Используйте только те таблицы, которые необходимы для выполнения запроса.
3. **Алиасы для таблиц:**
    
    - Используйте короткие алиасы для упрощения чтения:
        
        ```sql
        SELECT e.name, d.name AS department
        FROM employees e
        INNER JOIN departments d
        ON e.department_id = d.id;
        ```
        
4. **Работа с дубликатами:**
    
    - Используйте `DISTINCT`, если результат содержит повторяющиеся строки.

---

**Итог:** JOIN — это мощный инструмент для объединения данных из нескольких таблиц. Использование правильного типа JOIN зависит от задачи: INNER JOIN для совпадений, LEFT JOIN для данных с дополнительной информацией, FULL JOIN для объединения всех данных, а CROSS JOIN для декартового произведения.


## **7️⃣ `NATURAL JOIN`**

### **Описание:**

`NATURAL JOIN` автоматически выполняет соединение на всех столбцах с одинаковыми именами в обеих таблицах. PostgreSQL не требует явного указания условий соединения.

### **Пример:**

```sql
SELECT *
FROM table1
NATURAL JOIN table2;
```

Будет выполнено соединение по всем столбцам с одинаковыми именами.

---

## **8️⃣ `JOIN USING`**

### **Описание:**

`JOIN USING` используется, когда вы хотите соединить таблицы по одному или нескольким столбцам с одинаковыми именами. В отличие от `NATURAL JOIN`, здесь нужно явно указать столбцы.

### **Пример:**

```sql
SELECT *
FROM table1
JOIN table2 USING (id);
```

Этот запрос объединяет таблицы по столбцу `id`.

---

## **9️⃣ `LEFT SEMI JOIN` (псевдоним для подзапросов, поддерживается через EXISTS)**

### **Описание:**

`LEFT SEMI JOIN` — это тип соединения, который фактически возвращает только строки из левой таблицы, для которых существует хотя бы одна строка в правой таблице. В PostgreSQL его можно имитировать с помощью `EXISTS`.

### **Пример (с использованием `EXISTS`):**

```sql
SELECT *
FROM table1 t1
WHERE EXISTS (SELECT 1 FROM table2 t2 WHERE t2.id = t1.id);
```

---

## **🔟 `RIGHT SEMI JOIN` (псевдоним для подзапросов, поддерживается через EXISTS)**

### **Описание:**

`RIGHT SEMI JOIN` — это соединение, которое возвращает только строки из правой таблицы, для которых существует хотя бы одна строка в левой таблице. В PostgreSQL это можно реализовать с помощью `EXISTS`.

### **Пример (с использованием `EXISTS`):**

```sql
SELECT *
FROM table2 t2
WHERE EXISTS (SELECT 1 FROM table1 t1 WHERE t1.id = t2.id);
```

---

## **11️⃣ `ANTI JOIN`**

### **Описание:**

`ANTI JOIN` возвращает строки из левой таблицы, для которых нет соответствующих строк в правой таблице. Это аналог `LEFT JOIN` с фильтрацией по `NULL` в правой таблице.

### **Пример (с использованием `NOT EXISTS`):**

```sql
SELECT *
FROM table1 t1
WHERE NOT EXISTS (SELECT 1 FROM table2 t2 WHERE t2.id = t1.id);
```

---

### **📌 Итоговые типы JOIN в PostgreSQL:**

1. **`INNER JOIN`** — возвращает только совпавшие строки.
2. **`LEFT JOIN` / `LEFT OUTER JOIN`** — возвращает все строки из левой таблицы и совпавшие из правой.
3. **`RIGHT JOIN` / `RIGHT OUTER JOIN`** — возвращает все строки из правой таблицы и совпавшие из левой.
4. **`FULL JOIN` / `FULL OUTER JOIN`** — возвращает строки, которые есть в обеих таблицах или только в одной из них.
5. **`CROSS JOIN`** — возвращает декартово произведение строк двух таблиц.
6. **`SELF JOIN`** — соединяет таблицу с самой собой.
7. **`NATURAL JOIN`** — выполняет соединение по столбцам с одинаковыми именами.
8. **`JOIN USING`** — выполняет соединение по заданным столбцам с одинаковыми именами.
9. **`LEFT SEMI JOIN`** — возвращает строки из левой таблицы, для которых существует строка в правой таблице.
10. **`RIGHT SEMI JOIN`** — возвращает строки из правой таблицы, для которых существует строка в левой таблице.
11. **`ANTI JOIN`** — возвращает строки из левой таблицы, для которых нет соответствующих строк в правой.

---

Все эти типы `JOIN` предоставляют гибкость для эффективного выполнения операций с данными в PostgreSQL и позволяют выбирать наиболее подходящий метод в зависимости от ваших нужд в запросах.