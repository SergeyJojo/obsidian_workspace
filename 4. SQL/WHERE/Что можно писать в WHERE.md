В `WHERE` в PostgreSQL можно использовать очень широкий набор **операций и выражений**. Вот основные группы:

---

### ✅ **1. Сравнения**

```sql
=   -- равно
<>  -- не равно
!=  -- не равно (синоним <>)
<   -- меньше
>   -- больше
<=  -- меньше или равно
>=  -- больше или равно
```

```sql
WHERE age >= 18
WHERE name != 'Alice'
```

---

### ✅ **2. Логические операторы**

```sql
AND      -- логическое И
OR       -- логическое ИЛИ
NOT      -- отрицание
```

```sql
WHERE age > 18 AND city = 'Berlin'
WHERE NOT is_active
```

---

### ✅ **3. BETWEEN, IN, LIKE**

```sql
BETWEEN a AND b    -- значение в диапазоне
IN (v1, v2, ...)   -- значение в списке
LIKE / ILIKE       -- шаблон строки (ILIKE — нечувствительно к регистру)
```

```sql
WHERE salary BETWEEN 1000 AND 3000
WHERE country IN ('DE', 'FR', 'ES')
WHERE name LIKE 'A%'   -- начинается на A
```

---

### ✅ **4. IS NULL / IS NOT NULL**

```sql
WHERE deleted_at IS NULL
WHERE middle_name IS NOT NULL
```

---

### ✅ **5. Операции со строками**

```sql
WHERE LENGTH(name) > 10
WHERE LOWER(email) = 'test@example.com'
WHERE POSITION('@' IN email) > 0
```

---

### ✅ **6. Работа с датами**

```sql
WHERE created_at > NOW() - INTERVAL '7 days'
WHERE EXTRACT(year FROM created_at) = 2024
```

---

### ✅ **7. Подзапросы**

```sql
WHERE user_id IN (SELECT id FROM users WHERE active = true)
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id)
```

---

### ✅ **8. Полнотекстовый поиск**

```sql
WHERE to_tsvector(title) @@ to_tsquery('postgres')
```

---

### ✅ **9. Операторы массивов**

```sql
@>   -- содержит
<@   -- содержится в
&&   -- пересекается

WHERE tags @> ARRAY['news']
WHERE ARRAY[1,2] && ARRAY[2,3]
```

---

### ✅ **10. Кастомные функции**

Ты можешь использовать:

```sql
WHERE my_custom_function(value) = true
```

---

Если хочешь, могу дать таблицу с описанием всех операторов PostgreSQL или показать конкретные примеры из реальной схемы.