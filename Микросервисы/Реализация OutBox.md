
# **Реализация паттерна Outbox в Go с отправкой в Kafka**

Паттерн Outbox используется для надежной доставки сообщений в брокер (например, Kafka) в рамках транзакции с основным действием. Вот промышленная реализация на Go.

---

## **Архитектура решения**
1. **Транзакция в БД** сохраняет данные + сообщение в таблицу `outbox`
2. **Отдельный воркер** читает `outbox` и отправляет сообщения в Kafka
3. **Гарантия доставки**: подтверждение отправки удаляет запись из `outbox`

---

## **1. Модели и конфигурация**

### **Конфиг (config.go)**
```go
type Config struct {
    DB struct {
        DSN string `yaml:"dsn"`
    }
    Kafka struct {
        Brokers []string `yaml:"brokers"`
        Topic   string   `yaml:"topic"`
    }
    Worker struct {
        BatchSize int           `yaml:"batch_size"`
        Interval  time.Duration `yaml:"interval"`
    }
}
```

### **Модель Outbox (models.go)**
```go
type OutboxMessage struct {
    ID        string    `db:"id"`
    Topic     string    `db:"topic"`
    Key       []byte    `db:"key"`
    Value     []byte    `db:"value"`
    CreatedAt time.Time `db:"created_at"`
    Sent      bool      `db:"sent"`
}
```

---

## **2. Репозиторий для работы с Outbox (repository.go)**

```go
type OutboxRepository struct {
    db *sqlx.DB
}

func NewOutboxRepository(db *sqlx.DB) *OutboxRepository {
    return &OutboxRepository{db: db}
}

// SaveInTransaction сохраняет сообщение в рамках транзакции
func (r *OutboxRepository) SaveInTransaction(tx *sqlx.Tx, msg *OutboxMessage) error {
    query := `
        INSERT INTO outbox (id, topic, key, value, created_at, sent)
        VALUES (:id, :topic, :key, :value, :created_at, :sent)`
    _, err := tx.NamedExec(query, msg)
    return err
}

// GetUnsent получает непрочитанные сообщения
func (r *OutboxRepository) GetUnsent(limit int) ([]OutboxMessage, error) {
    var messages []OutboxMessage
    err := r.db.Select(&messages, 
        "SELECT * FROM outbox WHERE sent = false ORDER BY created_at LIMIT $1", limit)
    return messages, err
}

// MarkAsSent помечает сообщение как отправленное
func (r *OutboxRepository) MarkAsSent(tx *sqlx.Tx, id string) error {
    _, err := tx.Exec("UPDATE outbox SET sent = true WHERE id = $1", id)
    return err
}
```

---

## **3. Сервис с Outbox (service.go)**

```go
type OrderService struct {
    db      *sqlx.DB
    outbox  *OutboxRepository
    kafka   *kafka.Writer
}

func NewOrderService(cfg Config) (*OrderService, error) {
    db, err := sqlx.Connect("postgres", cfg.DB.DSN)
    if err != nil {
        return nil, fmt.Errorf("db connect: %w", err)
    }

    kafkaWriter := &kafka.Writer{
        Addr:     kafka.TCP(cfg.Kafka.Brokers...),
        Topic:    cfg.Kafka.Topic,
        Balancer: &kafka.Hash{},
    }

    return &OrderService{
        db:     db,
        outbox: NewOutboxRepository(db),
        kafka:  kafkaWriter,
    }, nil
}

// CreateOrder создает заказ и сохраняет событие в Outbox
func (s *OrderService) CreateOrder(order *Order) error {
    tx, err := s.db.Beginx()
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }
    defer tx.Rollback()

    // 1. Сохраняем заказ в БД
    if err := s.saveOrder(tx, order); err != nil {
        return err
    }

    // 2. Сохраняем событие в Outbox
    event := OutboxMessage{
        ID:        uuid.New().String(),
        Topic:     "orders",
        Key:       []byte(order.ID),
        Value:     s.orderToJSON(order),
        CreatedAt: time.Now(),
        Sent:      false,
    }
    if err := s.outbox.SaveInTransaction(tx, &event); err != nil {
        return fmt.Errorf("save outbox: %w", err)
    }

    return tx.Commit()
}
```

---

## **4. Воркер для отправки в Kafka (worker.go)**

```go
func (s *OrderService) RunWorker(ctx context.Context) {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            if err := s.processOutbox(ctx); err != nil {
                log.Printf("process outbox error: %v", err)
            }
        }
    }
}

func (s *OrderService) processOutbox(ctx context.Context) error {
    // 1. Получаем пачку непрочитанных сообщений
    messages, err := s.outbox.GetUnsent(100)
    if err != nil {
        return fmt.Errorf("get unsent: %w", err)
    }

    // 2. Отправляем каждое в Kafka
    for _, msg := range messages {
        if err := s.sendToKafka(ctx, msg); err != nil {
            return fmt.Errorf("send to kafka: %w", err)
        }

        // 3. Помечаем как отправленное в транзакции
        tx, err := s.db.Beginx()
        if err != nil {
            return fmt.Errorf("begin tx: %w", err)
        }

        if err := s.outbox.MarkAsSent(tx, msg.ID); err != nil {
            tx.Rollback()
            return fmt.Errorf("mark as sent: %w", err)
        }

        if err := tx.Commit(); err != nil {
            return fmt.Errorf("commit tx: %w", err)
        }
    }

    return nil
}

func (s *OrderService) sendToKafka(ctx context.Context, msg OutboxMessage) error {
    return s.kafka.WriteMessages(ctx, kafka.Message{
        Key:   msg.Key,
        Value: msg.Value,
    })
}
```

---

## **5. Запуск сервиса (main.go)**

```go
func main() {
    var cfg Config
    // Загрузка конфига из файла...

    svc, err := NewOrderService(cfg)
    if err != nil {
        log.Fatalf("create service: %v", err)
    }

    // Запускаем воркер в отдельной горутине
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    go svc.RunWorker(ctx)

    // HTTP-сервер для создания заказов
    http.HandleFunc("/orders", func(w http.ResponseWriter, r *http.Request) {
        var order Order
        if err := json.NewDecoder(r.Body).Decode(&order); err != nil {
            http.Error(w, err.Error(), http.StatusBadRequest)
            return
        }

        if err := svc.CreateOrder(&order); err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }

        w.WriteHeader(http.StatusCreated)
    })

    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

## **6. Дополнительные улучшения для продакшена**
1. **Ретри отправки** при ошибках Kafka (например, через `github.com/avast/retry-go`)
2. **Метрики** (Prometheus) для мониторинга:
   - Количество сообщений в Outbox
   - Время обработки батча
3. **Dead Letter Queue** для сообщений, которые не удалось отправить после N попыток
4. **Схема БД** для таблицы `outbox`:
   ```sql
   CREATE TABLE outbox (
       id         TEXT PRIMARY KEY,
       topic      TEXT NOT NULL,
       key        BYTEA,
       value      BYTEA NOT NULL,
       created_at TIMESTAMP NOT NULL,
       sent       BOOLEAN DEFAULT false
   );
   CREATE INDEX outbox_unsent_idx ON outbox (created_at) WHERE sent = false;
   ```

---

## **Итог**
Эта реализация обеспечивает:
- **Атомарность**: данные и сообщение сохраняются в одной транзакции
- **Надежность**: воркер гарантирует доставку
- **Масштабируемость**: батчевая обработка и параллелизм

Для промышленного использования добавьте обработку ошибок и мониторинг.