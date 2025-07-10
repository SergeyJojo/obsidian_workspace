
Отлично, разберём три важных SQL-конструкции с примерами: `FILTER`, `PARTITION BY`, `CTE`. Всё на примерах, с пояснениями.

---

## 📌 1. `FILTER` в агрегатных функциях (альтернатива `CASE WHEN`)

```sql
SELECT 
  COUNT(*) AS total_orders,
  COUNT(*) FILTER (WHERE status = 'paid') AS paid_orders,
  SUM(amount) FILTER (WHERE status = 'refunded') AS refunded_total
FROM orders;
```

`FILTER` используется внутри агрегатной функции, чтобы ограничить строки по условию, **не влияя** на весь `WHERE` запроса. Удобно и читаемо.

---

## 📌 2. `PARTITION BY` в оконных функциях

```sql
SELECT 
  user_id,
  order_id,
  amount,
  SUM(amount) OVER (PARTITION BY user_id ORDER BY order_date) AS running_total
FROM orders;
```

🧠 Здесь `PARTITION BY` — это как `GROUP BY`, но **для оконной функции**, не агрегирует весь результат. Ты видишь промежуточную сумму заказов по каждому пользователю.

---

Ещё один пример — ранжирование:

```sql
SELECT 
  user_id,
  order_id,
  amount,
  RANK() OVER (PARTITION BY user_id ORDER BY amount DESC) AS rank_within_user
FROM orders;
```

---

## 📌 3. CTE — общие табличные выражения

```sql
WITH high_value_orders AS (
  SELECT * FROM orders WHERE amount > 1000
)
SELECT user_id, COUNT(*) AS big_orders
FROM high_value_orders
GROUP BY user_id;
```

`WITH` создаёт временную именованную таблицу. Можно вкладывать, соединять, объединять:

```sql
WITH recent_orders AS (
  SELECT * FROM orders WHERE order_date > now() - interval '30 days'
),
user_sums AS (
  SELECT user_id, SUM(amount) AS total FROM recent_orders GROUP BY user_id
)
SELECT * FROM user_sums WHERE total > 500;
```

---

Если хочешь — добавим примеры с:

- рекурсивными `CTE`
    
- `LAG`, `LEAD`, `FIRST_VALUE` / `LAST_VALUE`
    
- комбинированием оконных и агрегатных функций
    

Что показать дальше?