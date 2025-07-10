#postgres



1. **Q: Как посмотреть список всех БД?** ::A: `\l` в psql или `SELECT datname FROM pg_database;`
<!--SR:!2025-07-01,3,250-->
2. **Q: Чем `VACUUM` отличается от `VACUUM FULL`?** ::A: `VACUUM` не блокирует таблицу, `VACUUM FULL` перезаписывает её целиком с блокировкой
<!--SR:!2025-07-01,3,250-->
3. **Q: Что делает `ANALYZE`?** ::A: Собирает статистику для оптимизатора запросов
<!--SR:!2025-07-01,3,250-->
4. **Q: Когда использовать BRIN-индекс?** ::A: Для больших таблиц с линейно упорядоченными данными (например, timestamp)
<!--SR:!2025-07-02,3,250-->
5. **Q: Почему индекс может не использоваться?** ::A: Недостаточно данных, неверный тип условия или устаревшая статистика
<!--SR:!2025-07-02,3,250-->
6. **Q: Что такое covering index?** ::A: Индекс, содержащий все поля запроса (использует `INCLUDE`)
<!--SR:!2025-07-01,3,250-->
7. **Q: Как найти медленные запросы?** ::A: Через `pg_stat_statements` или лог с `log_min_duration_statement`
<!--SR:!2025-07-01,3,250-->
8. **Q: Зачем нужен work_mem?** ::A: Определяет память для сортировки/хеширования. Мало памяти = запись на диск
<!--SR:!2025-07-01,3,250-->
9. **Q: Как CTE влияет на производительность?** ::A: До PG12 материализовалась, в 12+ можно контролировать через `MATERIALIZED`/`NOT MATERIALIZED`
<!--SR:!2025-07-02,3,250-->
10. **Q: Чем физическая репликация отличается от логической?** ::A: Физическая копирует байты WAL, логическая - изменения на уровне строк
<!--SR:!2025-07-01,3,250-->
11. **Q: Что делает pg_rewind?** ::A: Синхронизирует мастер и реплику без полного копирования
<!--SR:!2025-07-01,3,250-->
12. **Q: Как проверить лаг реплики?** ::A: `SELECT pg_current_wal_lsn() - replay_lsn FROM pg_stat_replication;`
<!--SR:!2025-07-01,3,250-->
13. **Q: Как создать TTL для строк?** ::A: Через триггер или расширение `pg_partman`
<!--SR:!2025-07-01,3,250-->
14. **Q: Что такое LATERAL JOIN?** ::A: Позволяет подзапросу ссылаться на поля из предыдущих таблиц в FROM
<!--SR:!2025-07-01,3,250-->
15. **Q: Как посмотреть активные подключения?** ::A: `SELECT * FROM pg_stat_activity;`
<!--SR:!2025-07-01,3,250-->
16. **Q: Что делает HStore?** ::A: Добавляет тип данных "ключ-значение" в колонку
<!--SR:!2025-07-01,3,250-->
17. **Q: Как ускорить вставку множества строк?** ::A: Использовать `COPY` вместо `INSERT`, отключить индексы/триггеры на время вставки
<!--SR:!2025-07-01,3,250-->
18. **Q: Что такое WAL?** ::A: Write-Ahead Log - журнал изменений для восстановления и репликации
<!--SR:!2025-07-01,3,250-->
19. **Q: Как сделать рестарт сервера без остановки?** ::A: `pg_ctl reload` или `SELECT pg_reload_conf();`
<!--SR:!2025-07-01,3,250-->
20. **Q: Что проверяет CHECK-ограничение?** ::A: Условие для значений в столбце (например, `price > 0`)
<!--SR:!2025-07-01,3,250-->



**Q: Как найти дубликаты в таблице?** ::A: `SELECT email, COUNT(*) FROM users GROUP BY email HAVING COUNT(*) > 1;`
<!--SR:!2025-07-01,3,250-->

**Q: Как выбрать 10 случайных записей?** ::A: `SELECT * FROM products ORDER BY random() LIMIT 10;`
<!--SR:!2025-07-02,3,250-->

**Q: Как обновить данные с JOIN?** ::A: `UPDATE orders SET status = 'shipped' FROM users WHERE orders.user_id = users.id AND users.country = 'US';`
<!--SR:!2025-07-01,3,250-->

**Q: Как удалить дубликаты, оставив одну запись?** ::A: `DELETE FROM products WHERE ctid NOT IN (SELECT min(ctid) FROM products GROUP BY product_id);`
<!--SR:!2025-07-01,3,250-->

**Q: Как сделать пагинацию?** ::A: `SELECT * FROM posts ORDER BY created_at DESC OFFSET 20 LIMIT 10;`
<!--SR:!2025-07-01,3,250-->

**Q: Как найти записи по частичному совпадению текста?** ::A: `SELECT * FROM articles WHERE content ILIKE '%postgres%';`
<!--SR:!2025-07-01,3,250-->

**Q: Как посчитать кумулятивную сумму?** ::A: `SELECT date, revenue, SUM(revenue) OVER (ORDER BY date) FROM sales;`
<!--SR:!2025-07-01,3,250-->

**Q: Как сравнить текущую строку с предыдущей?** ::A: `SELECT id, value, LAG(value) OVER (ORDER BY id) AS prev_value FROM metrics;`
<!--SR:!2025-07-01,3,250-->

**Q: Как развернуть JSON в таблицу?** ::A: `SELECT * FROM jsonb_to_recordset('[{"id":1},{"id":2}]'::jsonb) AS t(id int);`
<!--SR:!2025-07-01,3,250-->

**Q: Как найти "пропущенные" ID?** ::A: `SELECT generate_series(1,100) AS missing_id EXCEPT SELECT id FROM items;`
<!--SR:!2025-07-01,3,250-->

**Q: Как проверить использование индекса?** ::A: `EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';`
<!--SR:!2025-07-01,3,250-->

**Q: Как создать составной индекс?** ::A: `CREATE INDEX idx_users_name_email ON users(last_name, email);`
<!--SR:!2025-07-01,3,250-->

**Q: Как найти самые медленные запросы?** ::A: `SELECT query, total_time FROM pg_stat_statements ORDER BY total_time DESC LIMIT 10;`
<!--SR:!2025-07-01,3,250-->

**Q: Как ускорить вставку многих строк?** ::A: Использовать `COPY products FROM '/path/to/file.csv' WITH CSV;` вместо INSERT
<!--SR:!2025-07-02,3,250-->

**Q: Как разделить данные на группы (NTILE)?** ::A: `SELECT name, salary, NTILE(4) OVER (ORDER BY salary) AS quartile FROM employees;`
<!--SR:!2025-07-01,3,250-->

**Q: Как извлечь значение из JSON?** ::A: `SELECT data->>'name' FROM users WHERE data @> '{"active":true}';`
<!--SR:!2025-07-01,3,250-->

**Q: Как обновить поле в JSONB?** ::A: `UPDATE products SET attributes = jsonb_set(attributes, '{color}', '"red"') WHERE id = 1;`
<!--SR:!2025-07-01,3,250-->

**Q: Как использовать временные результаты (WITH)?** ::A: `WITH top_users AS (SELECT id FROM users ORDER BY rating DESC LIMIT 10) SELECT * FROM top_users JOIN orders ON orders.user_id = top_users.id;`
<!--SR:!2025-07-01,3,250-->

**Q: Как выделить неделю из даты?** ::A: `SELECT date_trunc('week', created_at) AS week_start FROM orders;`
<!--SR:!2025-07-01,3,250-->

**Q: Как найти интервал между датами?** ::A: `SELECT order_date, shipment_date, (shipment_date - order_date) AS days_to_ship FROM orders;`
<!--SR:!2025-07-01,3,250-->

Вот карточки по PostgreSQL в нужном формате:

### WAL (Write-Ahead Log)
Q: Что такое WAL в PostgreSQL? ::A: Журнал предзаписи для обеспечения надежности и восстановления данных
Q: Как WAL влияет на производительность? ::A: Замедляет запись (двойная запись), но ускоряет восстановление
Q: Основные параметры WAL? ::A: wal_level, checkpoint_timeout, wal_buffers

### MVCC
Q: Как работает MVCC в PostgreSQL? ::A: Хранение нескольких версий строк с отметками xmin/xmax
Q: Проблемы MVCC? ::A: Блокировки версий и необходимость VACUUM
Q: Как посмотреть текущий xid? ::A: SELECT txid_current();

### ACID
Q: Что означает A в ACID? ::A: Atomicity (Атомарность) - транзакция выполняется целиком или не выполняется
Q: Что означает I в ACID? ::A: Isolation (Изолированность) - транзакции не мешают друг другу
Q: Как PostgreSQL обеспечивает Durability? ::A: Через синхронную запись WAL на диск

### CAP-теорема
Q: Что означает CAP? ::A: Consistency, Availability, Partition tolerance
Q: Какую часть CAP жертвует PostgreSQL? ::A: Availability (при сетевых разделах)
Q: Почему PostgreSQL CP-система? ::A: Гарантирует консистентность даже при потере доступности

### Уровни изоляции
Q: Какие уровни изоляции есть в PostgreSQL? ::A: Read uncommitted, Read committed, Repeatable read, Serializable
Q: Уровень по умолчанию? ::A: Read committed
Q: Как избежать фантомного чтения? ::A: Использовать Serializable или блокировки

### Блокировки
Q: Как посмотреть текущие блокировки? ::A: SELECT * FROM pg_locks;
Q: Разница между LOCK и SELECT FOR UPDATE? ::A: LOCK блокирует всю таблицу, SELECT FOR UPDATE - только строки
Q: Что такое deadlock? ::A: Взаимная блокировка двух транзакций

### EXPLAIN ANALYZE
Q: Как включить вывод плана запроса? ::A: EXPLAIN ANALYZE SELECT...
Q: Что означает "Seq Scan"? ::A: Полное сканирование таблицы (плохо для больших таблиц)
Q: Как понять стоимость в плане? ::A: Первое число - стоимость запуска, второе - общая

### Буферы
Q: Что такое shared_buffers? ::A: Кеш PostgreSQL в оперативной памяти
Q: Как проверить эффективность кеша? ::A: SELECT * FROM pg_stat_bgwriter;
Q: Размер shared_buffers по умолчанию? ::A: 128MB (рекомендуется 25% от RAM)

### Work Mem
Q: Что контролирует work_mem? ::A: Память для операций сортировки и хеш-таблиц
Q: Как подобрать work_mem? ::A: Начинать с 4MB, увеличивать при сортировке на диске
Q: Где видно переполнение work_mem? ::A: В EXPLAIN ANALYZE как "Disk: I/O"

### Индексы
Q: Типы индексов в PostgreSQL? ::A: B-tree, Hash, GiST, SP-GiST, GIN, BRIN
Q: Когда использовать GIN? ::A: Для составных типов (массивы, json, full-text)
Q: Как проверить использование индекса? ::A: EXPLAIN ANALYZE и pg_stat_all_indexes

### B-tree
Q: Структура B-tree? ::A: Сбалансированное дерево с ключами в узлах
Q: Когда B-tree неэффективен? ::A: При низкой селективности (мало уникальных значений)
Q: Как перестроить индекс? ::A: REINDEX INDEX имя_индекса

### Join-алгоритмы
Q: Когда используется Nested Loop? ::A: При маленьких таблицах или наличии индексов
Q: Когда Hash Join эффективен? ::A: При соединении больших таблиц без индексов
Q: Преимущество Merge Join? ::A: Не требует памяти, но требует сортировки данных

### COPY
Q: Как загрузить данные из CSV? ::A: COPY table FROM 'file.csv' WITH CSV HEADER;
Q: Как экспортировать данные? ::A: COPY table TO 'file.csv' WITH CSV DELIMITER ';'
Q: Разница между COPY и \copy? ::A: COPY работает на сервере, \copy - на клиенте

### CTE (WITH)
Q: Что такое CTE? ::A: Common Table Expression - временный результат запроса
Q: Когда использовать MATERIALIZED? ::A: Когда CTE используется несколько раз
Q: Как рекурсивные CTE? ::A: WITH RECURSIVE для иерархических данных

### Дополнительные
Q: Как посмотреть активные подключения? ::A: SELECT * FROM pg_stat_activity;
Q: Как проверить размер таблицы? ::A: SELECT pg_size_pretty(pg_total_relation_size('table'));
Q: Как найти медленные запросы? ::A: Включить log_min_duration_statement и анализировать лог


