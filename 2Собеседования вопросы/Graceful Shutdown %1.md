
Для корректного завершения работы приложения с учетом правильной последовательности закрытия сервисов, нам нужно выполнить несколько шагов:

1. **Закрытие входящих запросов в HTTP сервере** — при завершении работы HTTP сервера важно сначала остановить приём новых запросов.
2. **Завершение работы сервисов** — сервис может иметь зависимость от базы данных, и завершать его нужно до завершения работы базы данных.
3. **Закрытие соединения с базой данных** — база данных должна быть закрыта последней, чтобы все запросы к базе завершились корректно.

Основной задачей является правильная синхронизация этих шагов. Для этого будем использовать сигналы ОС для получения сигнала остановки и корректно закрывать сервисы.

### Решение:

```go
package main

import (
	"log"
	"os"
	"os/signal"
	"syscall"
	"time"
)

// Closable интерфейс для сервисов, которые нужно корректно закрывать
type Closable interface {
	Close() error
}

// NewDatabaseClient - имитация создания клиента базы данных
func NewDatabaseClient() Closable {
	// Реализуем подключение к базе данных
	return &DatabaseClient{}
}

// MyCoolService - пример бизнес-логики, которая зависит от базы данных
func MyCoolService(db Closable) Closable {
	// Реализуем логику сервиса
	return &Service{db: db}
}

// DatabaseClient - структура, представляющая соединение с базой данных
type DatabaseClient struct {
	// Здесь можно хранить соединение с базой
}

// Close - метод для закрытия подключения к базе данных
func (d *DatabaseClient) Close() error {
	// Реализуем корректное закрытие базы данных
	log.Println("Closing database connection")
	time.Sleep(1 * time.Second) // Симуляция времени закрытия соединения
	return nil
}

// Service - структура для бизнес-логики
type Service struct {
	db Closable
}

// Close - метод для завершения работы сервиса
func (s *Service) Close() error {
	// Закрытие ресурсов, связанных с сервисом
	log.Println("Closing service")
	time.Sleep(1 * time.Second) // Симуляция времени закрытия
	return nil
}

// HttpServer - структура для HTTP сервера
type HttpServer struct{}

// NewServer - создание нового HTTP сервера
func (h *HttpServer) Register(pattern string, handler func()) {
	// Регистрация обработчиков для серверов
}

// ListenAndServeNonBlocking - запускаем сервер в фоновом режиме
func (h *HttpServer) ListenAndServeNonBlocking() {
	// Симуляция работы сервера
	log.Println("Server is listening...")
}

// Close - завершение работы сервера
func (h *HttpServer) Close() error {
	// Остановка сервера
	log.Println("Stopping HTTP server")
	time.Sleep(1 * time.Second) // Симуляция времени остановки
	return nil
}

func main() {
	// Инициализация сервисов
	database := NewDatabaseClient()
	service := MyCoolService(database)
	httpServer := &HttpServer{}

	// Регистрация маршрутов для HTTP сервера
	httpServer.Register("/", func() {
		// Обработка запроса
	})

	// Запуск сервера в неблокирующем режиме
	httpServer.ListenAndServeNonBlocking()

	// Ожидание сигнала для завершения
	stopChan := make(chan os.Signal, 1)
	signal.Notify(stopChan, syscall.SIGINT, syscall.SIGTERM)
	<-stopChan

	// Закрытие сервисов в правильной последовательности

	// 1. Закрыть сервер, чтобы не принимать новые запросы
	if err := httpServer.Close(); err != nil {
		log.Printf("Error stopping HTTP server: %v", err)
	}

	// 2. Завершаем работу сервиса, если он выполняет фоновую работу с базой данных
	if err := service.Close(); err != nil {
		log.Printf("Error closing service: %v", err)
	}

	// 3. Закрываем соединение с базой данных
	if err := database.Close(); err != nil {
		log.Printf("Error closing database connection: %v", err)
	}

	log.Println("Graceful shutdown completed")
}
```

### Разбор решения:

1. **Интерфейс `Closable`**:
    
    - Создан интерфейс `Closable`, который реализуют все сервисы (HTTP сервер, бизнес-логика и база данных). Это позволяет унифицировать процесс закрытия сервисов.
2. **Инициализация сервисов**:
    
    - Создаём объекты базы данных, сервиса и HTTP сервера. Каждый из них реализует интерфейс `Closable`.
3. **Завершение работы в правильной последовательности**:
    
    - **HTTP сервер** закрывается первым, чтобы остановить приём новых запросов.
    - **Сервис** завершает работу с базой данных, его нужно закрыть перед базой данных.
    - **База данных** закрывается последней, чтобы все запросы к базе были завершены корректно.
4. **Обработка сигналов**:
    
    - Используем канал `stopChan` для ожидания сигналов ОС (SIGINT, SIGTERM), которые означают необходимость завершения работы приложения.
5. **Graceful shutdown**:
    
    - После получения сигнала мы последовательно закрываем сервисы. Используем `time.Sleep` в каждом из методов `Close`, чтобы имитировать время на завершение работы (например, очистка ресурсов, завершение фонов работы).
6. **Обработка ошибок**:
    
    - При закрытии сервисов проверяются возможные ошибки, которые могут возникнуть, и они логируются.

### Важные моменты:

- **Параллельное закрытие**: Можно улучшить код, сделав закрытие сервисов параллельным для более быстрой завершения работы приложения. Например, использовать горутины для закрытия независимых сервисов.
- **Timeouts**: Можно добавить тайм-ауты для ожидания завершения работы сервисов, чтобы не зависнуть в случае, если какой-то сервис не может корректно завершиться. Например, можно использовать `context.WithTimeout`.

### Пример с таймаутами для graceful shutdown:

```go
func main() {
	// ...

	// Устанавливаем контекст с таймаутом для graceful shutdown
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	stopChan := make(chan os.Signal, 1)
	signal.Notify(stopChan, syscall.SIGINT, syscall.SIGTERM)
	<-stopChan

	// Завершаем сервисы с таймаутом
	doneChan := make(chan struct{})
	go func() {
		// Закрытие сервисов
		if err := httpServer.Close(); err != nil {
			log.Printf("Error stopping HTTP server: %v", err)
		}
		if err := service.Close(); err != nil {
			log.Printf("Error closing service: %v", err)
		}
		if err := database.Close(); err != nil {
			log.Printf("Error closing database connection: %v", err)
		}
		close(doneChan)
	}()

	select {
	case <-doneChan:
		log.Println("Graceful shutdown completed")
	case <-ctx.Done():
		log.Println("Timeout reached, shutting down")
	}
}
```

В этом случае мы добавляем таймаут для завершения работы, чтобы не зависеть от длительного закрытия сервисов.

### Заключение:

Мы создали систему **graceful shutdown**, которая корректно завершает работу всех сервисов в правильной последовательности, используя сигнал от операционной системы и методы закрытия для каждого компонента.