
Конечно! Ниже — **пошаговый гайд по работе с Apache Kafka в Go**, включая конкретные операции: отправка, чтение, группировка потребителей, сериализация и т.д.

---

# 📚 Гайд по работе с Kafka в Go

## 🛠 Технологии:

- Язык: **Go (Golang)**
- Библиотека: **[sarama](https://github.com/Shopify/sarama)** — самая популярная библиотека для работы с Kafka на Go
- Формат сообщений: JSON (но можно использовать Protobuf, Avro и др.)

---

## ✅ Установка

```bash
go get github.com/Shopify/sarama
```

---

## 🧩 1. Отправка сообщения в Kafka (Producer)

```go
package main

import (
	"fmt"
	"log"

	"github.com/Shopify/sarama"
)

func main() {
	// Настройки продюсера
	config := sarama.NewConfig()
	config.Producer.Return.Successes = true // Возвращать успехи

	// Создаем продюсера
	producer, err := sarama.NewSyncProducer([]string{"localhost:9092"}, config)
	if err != nil {
		log.Fatalln("Error creating producer:", err)
	}
	defer producer.Close()

	// Сообщение
	msg := &sarama.ProducerMessage{
		Topic: "my-topic",
		Value: sarama.StringEncoder("Hello from Go producer!"),
	}

	// Отправляем
	partition, offset, err := producer.SendMessage(msg)
	if err != nil {
		log.Fatalln("Failed to send message:", err)
	}

	fmt.Printf("Message sent to partition %d at offset %d\n", partition, offset)
}
```

---

## 🧩 2. Чтение сообщений из Kafka (Consumer)

```go
package main

import (
	"fmt"
	"log"

	"github.com/Shopify/sarama"
)

func main() {
	// Создаем конфигурацию
	config := sarama.NewConfig()
	config.Consumer.Group.Session.Timeout = 10 * time.Second
	config.Consumer.Group.Heartbeat.Interval = 3 * time.Second
	config.Consumer.Fetch.Default = 10e6 // 10MB

	// Создаем новый потребитель
	consumer, err := sarama.NewConsumerGroup([]string{"localhost:9092"}, "my-group", config)
	if err != nil {
		log.Fatalln("Error creating consumer group:", err)
	}
	defer consumer.Close()

	fmt.Println("Consumer is running...")

	// Запуск потребителя
	for {
		topics := []string{"my-topic"}
		handler := exampleConsumerGroupHandler{}
		err := consumer.Consume(context.Background(), topics, handler)
		if err != nil {
			log.Fatalln("Error consuming:", err)
		}
	}
}

// Реализация обработчика
type exampleConsumerGroupHandler struct{}

func (h exampleConsumerGroupHandler) Setup(session sarama.ConsumerGroupSession) error   { return nil }
func (h exampleConsumerGroupHandler) Cleanup(session sarama.ConsumerGroupSession) error { return nil }

func (h exampleConsumerGroupHandler) ConsumeClaim(session sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
	for message := range claim.Messages() {
		fmt.Printf("Received message: key=%s, value=%s, topic=%s, partition=%d, offset=%d\n",
			message.Key, message.Value, message.Topic, message.Partition, message.Offset)
		session.MarkMessage(message, "")
	}
	return nil
}
```

---

## 🧩 3. Работа с Consumer Group

Kafka позволяет нескольким потребителям объединяться в группы (`consumer group`), чтобы распределять нагрузку между ними.

- Все потребители одной группы **не получают одно и то же сообщение дважды**.
- Разные группы могут читать одни и те же сообщения.

### Пример:

```go
config := sarama.NewConfig()
config.Consumer.Group.Rebalance.Strategy = sarama.BalanceStrategyRange
```

---

## 🧩 4. Сериализация данных (JSON)

Можно отправлять и читать структуры в формате JSON:

### Отправка:

```go
type MyMessage struct {
	ID   int    `json:"id"`
	Text string `json:"text"`
}

msg := MyMessage{ID: 1, Text: "Hello JSON!"}
jsonBytes, _ := json.Marshal(msg)

producerMsg := &sarama.ProducerMessage{
	Topic: "json-topic",
	Value: sarama.ByteEncoder(jsonBytes),
}
```

### Получение:

```go
var received MyMessage
err := json.Unmarshal(message.Value, &received)
if err != nil {
	fmt.Println("Error decoding JSON:", err)
} else {
	fmt.Printf("Decoded: %+v\n", received)
}
```

---

## 🧩 5. Использование Partitioner

Если нужно отправлять данные в определённый `partition`, используйте `Partitioner`.

```go
config.Producer.Partitioner = sarama.NewManualPartitioner // или Hash, RoundRobin и т.д.

msg := &sarama.ProducerMessage{
	Topic:     "my-topic",
	Value:     sarama.StringEncoder("Custom partition"),
	Partition: 1,
}
```

---

## 🧩 6. Управление Offset

Kafka сохраняет позиции (offsets) потребителей:

- Можно указать начальный offset:
  
```go
config.Consumer.Offsets.Initial = sarama.OffsetOldest // или OffsetNewest
```

- Можно вручную коммитить offset:

```go
session.MarkMessage(message, "")
```

---

## 🧩 7. Установка заголовков (Headers)

Начиная с Kafka 0.11 поддерживается метаданные в виде заголовков.

### Отправка:

```go
msg := &sarama.ProducerMessage{
	Topic: "headers-topic",
	Value: sarama.StringEncoder("Hello with headers"),
	Headers: []sarama.RecordHeader{
		{Key: []byte("user-id"), Value: []byte("123")},
		{Key: []byte("source"), Value: []byte("web")},
	},
}
```

### Приём:

```go
for _, header := range message.Headers {
	fmt.Printf("Header: %s = %s\n", header.Key, header.Value)
}
```

---

## 🧩 8. Health Check и мониторинг

- Проверяйте соединение через `Metadata` API:

```go
client, err := sarama.NewClient([]string{"localhost:9092"}, nil)
if err != nil {
	log.Fatal(err)
}
defer client.Close()

topics, _ := client.Topics()
fmt.Println("Available topics:", topics)
```

---

## 🧰 Полезные утилиты и инструменты

| Назначение | Инструмент |
|-----------|------------|
| CLI клиент | `kafka-topics.sh`, `kafka-console-producer.sh`, `kafka-console-consumer.sh` |
| Мониторинг | [Kafka Exporter + Prometheus](https://github.com/danielqsj/kafka_exporter) |
| UI       | [Kafka Manager](https://github.com/yahoo/kafka-manager), [AKHQ](https://github.com/tchiotludo/akhq) |

---

## 📌 Лучшие практики

- Используйте **Consumer Groups** для масштабирования
- Используйте **JSON / Protobuf** для сериализации
- Не забывайте про **коммит offset'ов**
- Для высокой нагрузки настраивайте **BatchSize, FetchSize, Workers**
- Логируйте ошибки и делайте retry при ошибках

---

## 📚 Полезные ссылки

- [Sarama GitHub](https://github.com/Shopify/sarama)
- [Apache Kafka Docs](https://kafka.apache.org/documentation/)
- [Kafka Tutorial for Beginners](https://www.confluent.io/resources/kafka-the-definitive-guide/)
- [Kafka in Go: Best Practices](https://www.oreilly.com/library/view/kafka-in-go/2f4c0253-dba0-4bfa-a14d-7af995fb6763.xhtml)

---

Хочешь пример с использованием **Protobuf**, **Sarama wrappers**, **автоматического создания топиков**, или **интеграции с тестами**? Напиши — покажу!