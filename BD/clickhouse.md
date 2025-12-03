# ClickHouse: Высокопроизводительная колоночная СУБД

## Что такое ClickHouse?

**ClickHouse** — это колоночная система управления базами данных (СУБД) с открытым исходным кодом, разработанная Яндексом для обработки огромных объемов данных в реальном времени. Оптимизирована для аналитических запросов (OLAP).

## Ключевые особенности

### 1. **Колоночное хранение**
```sql
-- Вместо построчного хранения:
-- | id | user_id | timestamp | event |
-- | 1  | 100     | 2023-... | view  |
-- | 2  | 101     | 2023-... | click |

-- Данные хранятся колонками:
-- id:     [1, 2, 3, ...]
-- user_id: [100, 101, 102, ...]
-- event:  ["view", "click", "view", ...]
```

**Преимущества:**
- Сжатие данных (до 10-100x)
- Быстрое чтение только нужных колонок
- Эффективные агрегации

### 2. **Высокая производительность**
- Обработка миллиардов строк за секунды
- Параллельная и распределенная обработка
- Векторизованные вычисления

### 3. **Масштабируемость**
- Горизонтальное масштабирование
- Шардирование и репликация "из коробки"

## Архитектура и основные концепции

### Движки таблиц

```sql
-- MergeTree (основной движок)
CREATE TABLE events (
    timestamp DateTime,
    user_id UInt32,
    event_type String,
    page_url String
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (timestamp, user_id);

-- ReplicatedMergeTree (для репликации)
CREATE TABLE events (
    -- ...
) ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/events',
    '{replica}'
) PARTITION BY toYYYYMM(timestamp)
ORDER BY (timestamp, user_id);
```

### Распределенные таблицы

```sql
-- Локальная таблица на каждом шарде
CREATE TABLE events_local (...) ENGINE = MergeTree()...;

-- Распределенная таблица-представление
CREATE TABLE events AS events_local
ENGINE = Distributed(
    cluster_name,    -- имя кластера
    database_name,   -- БД
    events_local,    -- локальная таблица
    rand()           -- функция шардирования
);
```

## Практическое использование

### Установка и запуск
```bash
# Ubuntu/Debian
sudo apt-get install clickhouse-server clickhouse-client

# Запуск
sudo service clickhouse-server start

# Клиент
clickhouse-client
```

### Создание БД и таблиц
```sql
CREATE DATABASE analytics;

CREATE TABLE analytics.page_views (
    event_date Date,
    event_time DateTime,
    user_id UInt32,
    page_url String,
    session_id String,
    country_code FixedString(2),
    browser String,
    is_mobile Bool
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, user_id, session_id)
SETTINGS index_granularity = 8192;
```

### Вставка данных
```sql
INSERT INTO analytics.page_views VALUES
('2023-10-01', '2023-10-01 10:00:00', 1001, '/home', 'sess_001', 'US', 'Chrome', true),
('2023-10-01', '2023-10-01 10:01:00', 1002, '/products', 'sess_002', 'DE', 'Firefox', false);
```

Или массовая вставка из файла:
```bash
clickhouse-client --query="INSERT INTO analytics.page_views FORMAT CSV" < data.csv
```

## Примеры аналитических запросов

### 1. Агрегации
```sql
-- Статистика по дням
SELECT 
    event_date,
    count() AS page_views,
    uniq(user_id) AS unique_users,
    uniq(session_id) AS unique_sessions,
    avgIf(1, is_mobile = 1) AS mobile_ratio
FROM analytics.page_views
WHERE event_date >= '2023-10-01'
GROUP BY event_date
ORDER BY event_date;
```

### 2. Воронка событий
```sql
SELECT
    sumIf(1, page_url = '/home') AS home_views,
    sumIf(1, page_url = '/products') AS product_views,
    sumIf(1, page_url = '/cart') AS cart_views,
    sumIf(1, page_url = '/checkout') AS checkout_views
FROM analytics.page_views
WHERE event_date = '2023-10-01'
  AND user_id IN (
      SELECT user_id 
      FROM analytics.page_views 
      WHERE page_url = '/home'
  );
```

### 3. Сессии пользователей
```sql
SELECT
    user_id,
    windowFunnel(3600)(event_time, page_url = '/home', page_url = '/products', page_url = '/cart') AS funnel_steps,
    dateDiff('second', min(event_time), max(event_time)) AS session_duration
FROM analytics.page_views
WHERE event_date = '2023-10-01'
GROUP BY user_id, session_id
HAVING funnel_steps > 1;
```

## Продвинутые функции

### Массивы и вложенные структуры
```sql
CREATE TABLE user_behavior (
    user_id UInt32,
    dates Array(Date),
    actions Array(String),
    scores Array(Int32)
) ENGINE = MergeTree()
ORDER BY user_id;

-- Работа с массивами
SELECT
    user_id,
    arrayFilter((date, action) -> action = 'purchase', dates, actions) AS purchase_dates,
    arraySum(scores) AS total_score
FROM user_behavior;
```

### Оконные функции
```sql
SELECT
    user_id,
    event_time,
    page_url,
    count() OVER (PARTITION BY user_id ORDER BY event_time ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cumulative_views
FROM analytics.page_views
ORDER BY user_id, event_time;
```

## Интеграция с другими системами

### Выгрузка данных
```sql
-- В CSV
SELECT * FROM analytics.page_views 
INTO OUTFILE 'views.csv'
FORMAT CSVWithNames;

-- В PostgreSQL
CREATE TABLE pg_table AS postgresql(
    'host:port', 'database', 'table', 'user', 'password',
    'SELECT * FROM analytics.page_views'
);
```

### Загрузка данных из Kafka
```sql
CREATE TABLE kafka_queue (
    message String
) ENGINE = Kafka(
    'kafka-broker:9092',
    'topic_name',
    'consumer_group',
    'JSONEachRow'
);

CREATE TABLE target_table (
    -- поля
) ENGINE = MergeTree()...;

CREATE MATERIALIZED VIEW consumer TO target_table AS
SELECT * FROM kafka_queue;
```

## Мониторинг и оптимизация

### Системные таблицы
```sql
-- Мониторинг запросов
SELECT * FROM system.query_log 
WHERE event_time > now() - 3600
ORDER BY event_time DESC;

-- Использование памяти
SELECT * FROM system.metrics
WHERE metric LIKE '%Memory%';

-- Размеры таблиц
SELECT 
    table,
    formatReadableSize(sum(bytes)) AS size,
    sum(rows) AS rows
FROM system.parts
WHERE active
GROUP BY table;
```

### Оптимизация запросов
```sql
-- Используйте EXPLAIN для анализа планов
EXPLAIN PIPELINE
SELECT count() FROM analytics.page_views
WHERE event_date = '2023-10-01';

-- Индексы
ALTER TABLE analytics.page_views 
ADD INDEX url_index page_url TYPE bloom_filter GRANULARITY 1;
```

## Best Practices

### 1. **Правильное партиционирование**
```sql
-- По месяцам для данных за несколько лет
PARTITION BY toYYYYMM(event_date)

-- По дням для горячих данных
PARTITION BY event_date
```

### 2. **Оптимальный ORDER BY**
```sql
-- Самые частые фильтры первыми
ORDER BY (event_date, user_id, event_type)
```

### 3. **Настройки для больших таблиц**
```sql
SETTINGS 
    index_granularity = 8192,
    min_bytes_for_wide_part = 104857600; -- 100MB
```

## Типичные use cases

### ✅ Хорошо подходит:
- Web analytics
- IoT и телеметрия
- Financial analytics
- Ad tech и RTB
- Log analysis
- Business intelligence

### ❌ Не подходит для:
- Транзакционные системы (OLTP)
- Частые точечные UPDATE/DELETE
- Системы с strict ACID
- Много join между большими таблицами

## Заключение

ClickHouse — это мощный инструмент для аналитики больших данных, который обеспечивает:
- **Скорость**: Обработка миллиардов строк за секунды
- **Эффективность**: Высокое сжатие данных
- **Масштабируемость**: Простое горизонтальное масштабирование
- **SQL-совместимость**: Знакомый синтаксис для аналитиков

Идеальный выбор для систем, где важна скорость выполнения сложных аналитических запросов на больших объемах данных.