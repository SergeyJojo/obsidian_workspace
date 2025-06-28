# Большой проект на Go: Многомодульный сервис с Kafka, gRPC, HTTP, PGX, Vault

Создам для вас структурированный проект, который демонстрирует лучшие практики разработки в больших проектах на Go. Этот проект будет включать:

1. Многослойную архитектуру
2. Поддержку нескольких протоколов (HTTP, gRPC)
3. Работу с Kafka (производитель и потребитель)
4. Интеграцию с PostgreSQL через PGX
5. Конфигурацию через Vault
6. Логирование
7. Тестирование (юнит, интеграционные)
8. Обработку JSON
9. Docker-образы
10. Makefile для управления

## Структура проекта

```
bigtech-project/
├── cmd/
│   ├── api/
│   │   └── main.go
│   └── worker/
│       └── main.go
├── internal/
│   ├── config/
│   ├── controller/
│   ├── domain/
│   ├── repository/
│   ├── service/
│   ├── transport/
│   │   ├── grpc/
│   │   └── http/
│   └── worker/
├── pkg/
│   ├── kafka/
│   ├── logger/
│   ├── postgres/
│   └── vault/
├── api/
│   └── proto/
├── deployments/
│   ├── docker-compose.yml
│   ├── api.Dockerfile
│   └── worker.Dockerfile
├── Makefile
├── go.mod
└── go.sum
```

## Реализация

### 1. Конфигурация (internal/config)

`internal/config/config.go`:
```go
package config

import (
	"context"
	"fmt"
	"time"

	"github.com/hashicorp/vault/api"
)

type Config struct {
	App        AppConfig
	HTTP       HTTPConfig
	GRPC       GRPCConfig
	Postgres   PostgresConfig
	Kafka      KafkaConfig
	Vault      VaultConfig
}

type AppConfig struct {
	Name    string
	Version string
	Env     string
}

type HTTPConfig struct {
	Port            string
	Timeout         time.Duration
	ShutdownTimeout time.Duration
}

type GRPCConfig struct {
	Port    string
	Timeout time.Duration
}

type PostgresConfig struct {
	URL string
}

type KafkaConfig struct {
	Brokers  []string
	Producer struct {
		Topic string
	}
	Consumer struct {
		Topic    string
		GroupID  string
		AutoAck  bool
		Workers  int
	}
}

type VaultConfig struct {
	Address string
	Token   string
	Path    string
}

func Load(ctx context.Context) (*Config, error) {
	vaultClient, err := api.NewClient(api.DefaultConfig())
	if err != nil {
		return nil, fmt.Errorf("failed to create vault client: %w", err)
	}

	// Получаем секреты из Vault
	secret, err := vaultClient.Logical().Read("secret/data/myapp")
	if err != nil {
		return nil, fmt.Errorf("failed to read vault secret: %w", err)
	}

	data := secret.Data["data"].(map[string]interface{})

	cfg := &Config{
		App: AppConfig{
			Name:    "bigtech-project",
			Version: "1.0.0",
			Env:     data["env"].(string),
		},
		HTTP: HTTPConfig{
			Port:            data["http_port"].(string),
			Timeout:         30 * time.Second,
			ShutdownTimeout: 5 * time.Second,
		},
		GRPC: GRPCConfig{
			Port:    data["grpc_port"].(string),
			Timeout: 30 * time.Second,
		},
		Postgres: PostgresConfig{
			URL: data["postgres_url"].(string),
		},
		Kafka: KafkaConfig{
			Brokers: []string{data["kafka_broker"].(string)},
			Producer: struct {
				Topic string
			}{
				Topic: data["kafka_producer_topic"].(string),
			},
			Consumer: struct {
				Topic    string
				GroupID  string
				AutoAck  bool
				Workers  int
			}{
				Topic:    data["kafka_consumer_topic"].(string),
				GroupID:  data["kafka_consumer_group_id"].(string),
				AutoAck:  data["kafka_consumer_auto_ack"].(bool),
				Workers:  int(data["kafka_consumer_workers"].(float64)),
			},
		},
		Vault: VaultConfig{
			Address: data["vault_address"].(string),
			Token:   data["vault_token"].(string),
			Path:    "secret/data/myapp",
		},
	}

	return cfg, nil
}
```

### 2. Доменные модели (internal/domain)

`internal/domain/user.go`:
```go
package domain

import "time"

type User struct {
	ID        string    `json:"id"`
	Name      string    `json:"name"`
	Email     string    `json:"email"`
	CreatedAt time.Time `json:"created_at"`
	UpdatedAt time.Time `json:"updated_at"`
}

type UserEvent struct {
	Type string `json:"type"`
	User User   `json:"user"`
}
```

### 3. Репозиторий (internal/repository)

`internal/repository/user.go`:
```go
package repository

import (
	"context"
	"time"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"

	"bigtech-project/internal/domain"
)

type UserRepository interface {
	Create(ctx context.Context, user *domain.User) error
	GetByID(ctx context.Context, id string) (*domain.User, error)
	Update(ctx context.Context, user *domain.User) error
	Delete(ctx context.Context, id string) error
}

type userRepo struct {
	db *pgxpool.Pool
}

func NewUserRepository(db *pgxpool.Pool) UserRepository {
	return &userRepo{db: db}
}

func (r *userRepo) Create(ctx context.Context, user *domain.User) error {
	query := `INSERT INTO users (id, name, email, created_at, updated_at) 
	          VALUES ($1, $2, $3, $4, $5)`

	_, err := r.db.Exec(ctx, query, 
		user.ID, user.Name, user.Email, time.Now(), time.Now())

	return err
}

func (r *userRepo) GetByID(ctx context.Context, id string) (*domain.User, error) {
	query := `SELECT id, name, email, created_at, updated_at 
	          FROM users WHERE id = $1`

	var user domain.User
	err := r.db.QueryRow(ctx, query, id).Scan(
		&user.ID, &user.Name, &user.Email, &user.CreatedAt, &user.UpdatedAt)

	if err == pgx.ErrNoRows {
		return nil, nil
	}

	if err != nil {
		return nil, err
	}

	return &user, nil
}

func (r *userRepo) Update(ctx context.Context, user *domain.User) error {
	query := `UPDATE users SET name = $1, email = $2, updated_at = $3 
	          WHERE id = $4`

	_, err := r.db.Exec(ctx, query, 
		user.Name, user.Email, time.Now(), user.ID)

	return err
}

func (r *userRepo) Delete(ctx context.Context, id string) error {
	query := `DELETE FROM users WHERE id = $1`

	_, err := r.db.Exec(ctx, query, id)
	return err
}
```

### 4. Сервисный слой (internal/service)

`internal/service/user.go`:
```go
package service

import (
	"context"
	"errors"

	"bigtech-project/internal/domain"
	"bigtech-project/internal/repository"
	"bigtech-project/pkg/kafka"
	"bigtech-project/pkg/logger"
)

type UserService interface {
	CreateUser(ctx context.Context, user *domain.User) error
	GetUser(ctx context.Context, id string) (*domain.User, error)
	UpdateUser(ctx context.Context, user *domain.User) error
	DeleteUser(ctx context.Context, id string) error
}

type userService struct {
	repo        repository.UserRepository
	kafkaProd   kafka.Producer
	log         logger.Logger
}

func NewUserService(
	repo repository.UserRepository, 
	prod kafka.Producer, 
	log logger.Logger,
) UserService {
	return &userService{
		repo:      repo,
		kafkaProd: prod,
		log:       log,
	}
}

func (s *userService) CreateUser(ctx context.Context, user *domain.User) error {
	if user.Name == "" {
		return errors.New("name is required")
	}

	if err := s.repo.Create(ctx, user); err != nil {
		return err
	}

	event := domain.UserEvent{
		Type: "user.created",
		User: *user,
	}

	if err := s.kafkaProd.Produce(ctx, event); err != nil {
		s.log.Error("failed to produce user created event", "error", err)
	}

	return nil
}

func (s *userService) GetUser(ctx context.Context, id string) (*domain.User, error) {
	return s.repo.GetByID(ctx, id)
}

func (s *userService) UpdateUser(ctx context.Context, user *domain.User) error {
	if user.Name == "" {
		return errors.New("name is required")
	}

	if err := s.repo.Update(ctx, user); err != nil {
		return err
	}

	event := domain.UserEvent{
		Type: "user.updated",
		User: *user,
	}

	if err := s.kafkaProd.Produce(ctx, event); err != nil {
		s.log.Error("failed to produce user updated event", "error", err)
	}

	return nil
}

func (s *userService) DeleteUser(ctx context.Context, id string) error {
	user, err := s.repo.GetByID(ctx, id)
	if err != nil {
		return err
	}

	if user == nil {
		return nil
	}

	if err := s.repo.Delete(ctx, id); err != nil {
		return err
	}

	event := domain.UserEvent{
		Type: "user.deleted",
		User: *user,
	}

	if err := s.kafkaProd.Produce(ctx, event); err != nil {
		s.log.Error("failed to produce user deleted event", "error", err)
	}

	return nil
}
```

### 5. HTTP контроллер (internal/transport/http)

`internal/transport/http/user.go`:
```go
package http

import (
	"encoding/json"
	"net/http"
	"time"

	"github.com/go-chi/chi/v5"
	"github.com/go-chi/render"

	"bigtech-project/internal/domain"
	"bigtech-project/internal/service"
	"bigtech-project/pkg/logger"
)

type UserHandler struct {
	service service.UserService
	log     logger.Logger
}

func NewUserHandler(service service.UserService, log logger.Logger) *UserHandler {
	return &UserHandler{
		service: service,
		log:     log,
	}
}

func (h *UserHandler) Routes() chi.Router {
	r := chi.NewRouter()

	r.Post("/", h.CreateUser)
	r.Get("/{id}", h.GetUser)
	r.Put("/{id}", h.UpdateUser)
	r.Delete("/{id}", h.DeleteUser)

	return r
}

type CreateUserRequest struct {
	Name  string `json:"name"`
	Email string `json:"email"`
}

func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
	var req CreateUserRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		render.Status(r, http.StatusBadRequest)
		render.JSON(w, r, map[string]string{"error": "invalid request"})
		return
	}

	user := &domain.User{
		ID:    generateID(),
		Name:  req.Name,
		Email: req.Email,
	}

	if err := h.service.CreateUser(r.Context(), user); err != nil {
		h.log.Error("failed to create user", "error", err)
		render.Status(r, http.StatusInternalServerError)
		render.JSON(w, r, map[string]string{"error": "failed to create user"})
		return
	}

	render.Status(r, http.StatusCreated)
	render.JSON(w, r, user)
}

func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
	id := chi.URLParam(r, "id")

	user, err := h.service.GetUser(r.Context(), id)
	if err != nil {
		h.log.Error("failed to get user", "error", err, "id", id)
		render.Status(r, http.StatusInternalServerError)
		render.JSON(w, r, map[string]string{"error": "failed to get user"})
		return
	}

	if user == nil {
		render.Status(r, http.StatusNotFound)
		render.JSON(w, r, map[string]string{"error": "user not found"})
		return
	}

	render.JSON(w, r, user)
}

func (h *UserHandler) UpdateUser(w http.ResponseWriter, r *http.Request) {
	id := chi.URLParam(r, "id")

	var req CreateUserRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		render.Status(r, http.StatusBadRequest)
		render.JSON(w, r, map[string]string{"error": "invalid request"})
		return
	}

	user := &domain.User{
		ID:    id,
		Name:  req.Name,
		Email: req.Email,
	}

	if err := h.service.UpdateUser(r.Context(), user); err != nil {
		h.log.Error("failed to update user", "error", err, "id", id)
		render.Status(r, http.StatusInternalServerError)
		render.JSON(w, r, map[string]string{"error": "failed to update user"})
		return
	}

	render.JSON(w, r, user)
}

func (h *UserHandler) DeleteUser(w http.ResponseWriter, r *http.Request) {
	id := chi.URLParam(r, "id")

	if err := h.service.DeleteUser(r.Context(), id); err != nil {
		h.log.Error("failed to delete user", "error", err, "id", id)
		render.Status(r, http.StatusInternalServerError)
		render.JSON(w, r, map[string]string{"error": "failed to delete user"})
		return
	}

	render.Status(r, http.StatusNoContent)
}

func generateID() string {
	return time.Now().Format("20060102150405")
}
```

### 6. gRPC сервер (internal/transport/grpc)

`api/proto/user.proto`:
```proto
syntax = "proto3";

package user;

option go_package = "bigtech-project/api/proto;user";

service UserService {
    rpc CreateUser (CreateUserRequest) returns (UserResponse);
    rpc GetUser (GetUserRequest) returns (UserResponse);
    rpc UpdateUser (UpdateUserRequest) returns (UserResponse);
    rpc DeleteUser (DeleteUserRequest) returns (EmptyResponse);
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
}

message GetUserRequest {
    string id = 1;
}

message UpdateUserRequest {
    string id = 1;
    string name = 2;
    string email = 3;
}

message DeleteUserRequest {
    string id = 1;
}

message UserResponse {
    string id = 1;
    string name = 2;
    string email = 3;
    string created_at = 4;
    string updated_at = 5;
}

message EmptyResponse {}
```

`internal/transport/grpc/user.go`:
```go
package grpc

import (
	"context"
	"time"

	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"

	"bigtech-project/api/proto"
	"bigtech-project/internal/domain"
	"bigtech-project/internal/service"
	"bigtech-project/pkg/logger"
)

type UserServer struct {
	proto.UnimplementedUserServiceServer
	service service.UserService
	log     logger.Logger
}

func NewUserServer(service service.UserService, log logger.Logger) *UserServer {
	return &UserServer{
		service: service,
		log:     log,
	}
}

func (s *UserServer) CreateUser(ctx context.Context, req *proto.CreateUserRequest) (*proto.UserResponse, error) {
	user := &domain.User{
		ID:    generateID(),
		Name:  req.Name,
		Email: req.Email,
	}

	if err := s.service.CreateUser(ctx, user); err != nil {
		s.log.Error("failed to create user", "error", err)
		return nil, status.Error(codes.Internal, "failed to create user")
	}

	return s.domainToProto(user), nil
}

func (s *UserServer) GetUser(ctx context.Context, req *proto.GetUserRequest) (*proto.UserResponse, error) {
	user, err := s.service.GetUser(ctx, req.Id)
	if err != nil {
		s.log.Error("failed to get user", "error", err, "id", req.Id)
		return nil, status.Error(codes.Internal, "failed to get user")
	}

	if user == nil {
		return nil, status.Error(codes.NotFound, "user not found")
	}

	return s.domainToProto(user), nil
}

func (s *UserServer) UpdateUser(ctx context.Context, req *proto.UpdateUserRequest) (*proto.UserResponse, error) {
	user := &domain.User{
		ID:    req.Id,
		Name:  req.Name,
		Email: req.Email,
	}

	if err := s.service.UpdateUser(ctx, user); err != nil {
		s.log.Error("failed to update user", "error", err, "id", req.Id)
		return nil, status.Error(codes.Internal, "failed to update user")
	}

	return s.domainToProto(user), nil
}

func (s *UserServer) DeleteUser(ctx context.Context, req *proto.DeleteUserRequest) (*proto.EmptyResponse, error) {
	if err := s.service.DeleteUser(ctx, req.Id); err != nil {
		s.log.Error("failed to delete user", "error", err, "id", req.Id)
		return nil, status.Error(codes.Internal, "failed to delete user")
	}

	return &proto.EmptyResponse{}, nil
}

func (s *UserServer) domainToProto(user *domain.User) *proto.UserResponse {
	return &proto.UserResponse{
		Id:        user.ID,
		Name:      user.Name,
		Email:     user.Email,
		CreatedAt: user.CreatedAt.Format(time.RFC3339),
		UpdatedAt: user.UpdatedAt.Format(time.RFC3339),
	}
}
```

### 7. Kafka интеграция (pkg/kafka)

`pkg/kafka/producer.go`:
```go
package kafka

import (
	"context"
	"encoding/json"

	"github.com/segmentio/kafka-go"
	"github.com/segmentio/kafka-go/protocol"
)

type Producer interface {
	Produce(ctx context.Context, message interface{}) error
	Close() error
}

type producer struct {
	writer *kafka.Writer
}

func NewProducer(brokers []string, topic string) Producer {
	return &producer{
		writer: &kafka.Writer{
			Addr:     kafka.TCP(brokers...),
			Topic:    topic,
			Balancer: &kafka.LeastBytes{},
			Async:    true,
		},
	}
}

func (p *producer) Produce(ctx context.Context, message interface{}) error {
	msg, err := json.Marshal(message)
	if err != nil {
		return err
	}

	return p.writer.WriteMessages(ctx, kafka.Message{
		Value: msg,
		Headers: []protocol.Header{
			{
				Key:   "event-type",
				Value: []byte(getEventType(message)),
			},
		},
	})
}

func (p *producer) Close() error {
	return p.writer.Close()
}

func getEventType(message interface{}) string {
	switch m := message.(type) {
	case interface{ GetType() string }:
		return m.GetType()
	default:
		return "unknown"
	}
}
```

`pkg/kafka/consumer.go`:
```go
package kafka

import (
	"context"
	"encoding/json"
	"sync"

	"github.com/segmentio/kafka-go"
	"go.uber.org/zap"
)

type ConsumerHandler func(ctx context.Context, message []byte) error

type Consumer interface {
	Start(ctx context.Context) error
	Close() error
}

type consumer struct {
	reader  *kafka.Reader
	handler ConsumerHandler
	log     *zap.Logger
	wg      sync.WaitGroup
}

func NewConsumer(brokers []string, topic, groupID string, handler ConsumerHandler, log *zap.Logger) Consumer {
	return &consumer{
		reader: kafka.NewReader(kafka.ReaderConfig{
			Brokers: brokers,
			Topic:   topic,
			GroupID: groupID,
		}),
		handler: handler,
		log:     log,
	}
}

func (c *consumer) Start(ctx context.Context) error {
	for {
		select {
		case <-ctx.Done():
			return nil
		default:
			msg, err := c.reader.FetchMessage(ctx)
			if err != nil {
				c.log.Error("failed to fetch message", zap.Error(err))
				continue
			}

			c.wg.Add(1)
			go func(m kafka.Message) {
				defer c.wg.Done()

				if err := c.handler(ctx, m.Value); err != nil {
					c.log.Error("failed to handle message", 
						zap.Error(err), 
						zap.ByteString("value", m.Value))
					return
				}

				if err := c.reader.CommitMessages(ctx, m); err != nil {
					c.log.Error("failed to commit message", zap.Error(err))
				}
			}(msg)
		}
	}
}

func (c *consumer) Close() error {
	c.wg.Wait()
	return c.reader.Close()
}
```

### 8. Логирование (pkg/logger)

`pkg/logger/logger.go`:
```go
package logger

import (
	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
)

type Logger interface {
	Debug(msg string, fields ...Field)
	Info(msg string, fields ...Field)
	Warn(msg string, fields ...Field)
	Error(msg string, fields ...Field)
	Fatal(msg string, fields ...Field)
	With(fields ...Field) Logger
}

type Field struct {
	Key   string
	Value interface{}
}

type zapLogger struct {
	logger *zap.Logger
}

func New(level string) (Logger, error) {
	cfg := zap.NewProductionConfig()
	cfg.EncoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder

	lvl, err := zapcore.ParseLevel(level)
	if err != nil {
		return nil, err
	}
	cfg.Level = zap.NewAtomicLevelAt(lvl)

	zapLogger, err := cfg.Build()
	if err != nil {
		return nil, err
	}

	return &zapLogger{logger: zapLogger}, nil
}

func (l *zapLogger) Debug(msg string, fields ...Field) {
	l.logger.Debug(msg, l.toZapFields(fields)...)
}

func (l *zapLogger) Info(msg string, fields ...Field) {
	l.logger.Info(msg, l.toZapFields(fields)...)
}

func (l *zapLogger) Warn(msg string, fields ...Field) {
	l.logger.Warn(msg, l.toZapFields(fields)...)
}

func (l *zapLogger) Error(msg string, fields ...Field) {
	l.logger.Error(msg, l.toZapFields(fields)...)
}

func (l *zapLogger) Fatal(msg string, fields ...Field) {
	l.logger.Fatal(msg, l.toZapFields(fields)...)
}

func (l *zapLogger) With(fields ...Field) Logger {
	return &zapLogger{logger: l.logger.With(l.toZapFields(fields)...)}
}

func (l *zapLogger) toZapFields(fields []Field) []zap.Field {
	zapFields := make([]zap.Field, len(fields))
	for i, f := range fields {
		zapFields[i] = zap.Any(f.Key, f.Value)
	}
	return zapFields
}
```

### 9. PostgreSQL (pkg/postgres)

`pkg/postgres/postgres.go`:
```go
package postgres

import (
	"context"
	"fmt"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
)

type Postgres struct {
	Pool *pgxpool.Pool
}

func New(ctx context.Context, connString string) (*Postgres, error) {
	config, err := pgxpool.ParseConfig(connString)
	if err != nil {
		return nil, fmt.Errorf("failed to parse postgres config: %w", err)
	}

	config.MaxConns = 10
	config.MinConns = 2
	config.MaxConnLifetime = time.Hour
	config.MaxConnIdleTime = 30 * time.Minute
	config.HealthCheckPeriod = time.Minute

	pool, err := pgxpool.NewWithConfig(ctx, config)
	if err != nil {
		return nil, fmt.Errorf("failed to create postgres pool: %w", err)
	}

	if err := pool.Ping(ctx); err != nil {
		return nil, fmt.Errorf("failed to ping postgres: %w", err)
	}

	return &Postgres{Pool: pool}, nil
}

func (p *Postgres) Close() {
	p.Pool.Close()
}
```

### 10. Vault (pkg/vault)

`pkg/vault/vault.go`:
```go
package vault

import (
	"context"
	"fmt"

	"github.com/hashicorp/vault/api"
)

type Vault struct {
	client *api.Client
}

func New(address, token string) (*Vault, error) {
	config := api.DefaultConfig()
	config.Address = address

	client, err := api.NewClient(config)
	if err != nil {
		return nil, fmt.Errorf("failed to create vault client: %w", err)
	}

	client.SetToken(token)

	return &Vault{client: client}, nil
}

func (v *Vault) ReadSecret(path string) (map[string]interface{}, error) {
	secret, err := v.client.Logical().Read(path)
	if err != nil {
		return nil, fmt.Errorf("failed to read secret: %w", err)
	}

	if secret == nil || secret.Data == nil {
		return nil, fmt.Errorf("secret not found")
	}

	return secret.Data, nil
}
```

### 11. Основной сервер (cmd/api/main.go)

```go
package main

import (
	"context"
	"net"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"bigtech-project/api/proto"
	"bigtech-project/internal/config"
	"bigtech-project/internal/repository"
	"bigtech-project/internal/service"
	"bigtech-project/internal/transport/grpc"
	"bigtech-project/internal/transport/http"
	"bigtech-project/pkg/kafka"
	"bigtech-project/pkg/logger"
	"bigtech-project/pkg/postgres"

	"github.com/go-chi/chi/v5"
	"google.golang.org/grpc"
)

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// Загрузка конфигурации
	cfg, err := config.Load(ctx)
	if err != nil {
		panic(err)
	}

	// Инициализация логгера
	log, err := logger.New("info")
	if err != nil {
		panic(err)
	}

	// Подключение к PostgreSQL
	pg, err := postgres.New(ctx, cfg.Postgres.URL)
	if err != nil {
		log.Fatal("failed to connect to postgres", logger.Field{"error", err})
	}
	defer pg.Close()

	// Инициализация Kafka Producer
	kafkaProducer := kafka.NewProducer(cfg.Kafka.Brokers, cfg.Kafka.Producer.Topic)
	defer kafkaProducer.Close()

	// Инициализация репозиториев
	userRepo := repository.NewUserRepository(pg.Pool)

	// Инициализация сервисов
	userService := service.NewUserService(userRepo, kafkaProducer, log)

	// Инициализация HTTP сервера
	httpServer := initHTTPServer(cfg, userService, log)

	// Инициализация gRPC сервера
	grpcServer := initGRPCServer(cfg, userService, log)

	// Запуск серверов
	go func() {
		log.Info("starting HTTP server", logger.Field{"port", cfg.HTTP.Port})
		if err := httpServer.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatal("failed to start HTTP server", logger.Field{"error", err})
		}
	}()

	go func() {
		lis, err := net.Listen("tcp", ":"+cfg.GRPC.Port)
		if err != nil {
			log.Fatal("failed to listen", logger.Field{"error", err})
		}

		log.Info("starting gRPC server", logger.Field{"port", cfg.GRPC.Port})
		if err := grpcServer.Serve(lis); err != nil {
			log.Fatal("failed to start gRPC server", logger.Field{"error", err})
		}
	}()

	// Ожидание сигналов завершения
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	log.Info("shutting down servers...")

	// Graceful shutdown
	ctx, shutdownCancel := context.WithTimeout(context.Background(), cfg.HTTP.ShutdownTimeout)
	defer shutdownCancel()

	if err := httpServer.Shutdown(ctx); err != nil {
		log.Error("failed to shutdown HTTP server", logger.Field{"error", err})
	}

	grpcServer.GracefulStop()
}

func initHTTPServer(cfg *config.Config, userService service.UserService, log logger.Logger) *http.Server {
	r := chi.NewRouter()

	// Инициализация HTTP обработчиков
	userHandler := http.NewUserHandler(userService, log)

	// Маршрутизация
	r.Mount("/users", userHandler.Routes())

	return &http.Server{
		Addr:    ":" + cfg.HTTP.Port,
		Handler: r,
	}
}

func initGRPCServer(cfg *config.Config, userService service.UserService, log logger.Logger) *grpc.Server {
	srv := grpc.NewServer()

	// Регистрация gRPC сервисов
	userServer := grpc.NewUserServer(userService, log)
	proto.RegisterUserServiceServer(srv, userServer)

	return srv
}
```

### 12. Воркер для Kafka (cmd/worker/main.go)

```go
package main

import (
	"context"
	"encoding/json"
	"os"
	"os/signal"
	"syscall"

	"bigtech-project/internal/config"
	"bigtech-project/internal/domain"
	"bigtech-project/internal/repository"
	"bigtech-project/internal/service"
	"bigtech-project/pkg/kafka"
	"bigtech-project/pkg/logger"
	"bigtech-project/pkg/postgres"
)

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// Загрузка конфигурации
	cfg, err := config.Load(ctx)
	if err != nil {
		panic(err)
	}

	// Инициализация логгера
	log, err := logger.New("info")
	if err != nil {
		panic(err)
	}

	// Подключение к PostgreSQL
	pg, err := postgres.New(ctx, cfg.Postgres.URL)
	if err != nil {
		log.Fatal("failed to connect to postgres", logger.Field{"error", err})
	}
	defer pg.Close()

	// Инициализация репозиториев
	userRepo := repository.NewUserRepository(pg.Pool)

	// Инициализация сервисов
	userService := service.NewUserService(userRepo, nil, log)

	// Инициализация Kafka Consumer
	consumer := kafka.NewConsumer(
		cfg.Kafka.Brokers,
		cfg.Kafka.Consumer.Topic,
		cfg.Kafka.Consumer.GroupID,
		func(ctx context.Context, msg []byte) error {
			var event domain.UserEvent
			if err := json.Unmarshal(msg, &event); err != nil {
				log.Error("failed to unmarshal message", logger.Field{"error", err})
				return nil
			}

			switch event.Type {
			case "user.created":
				log.Info("user created event received", logger.Field{"user_id", event.User.ID})
			case "user.updated":
				log.Info("user updated event received", logger.Field{"user_id", event.User.ID})
			case "user.deleted":
				log.Info("user deleted event received", logger.Field{"user_id", event.User.ID})
			default:
				log.Warn("unknown event type received", logger.Field{"type", event.Type})
			}

			return nil
		},
		log.(*logger.zapLogger).logger,
	)

	// Запуск consumer
	go func() {
		log.Info("starting kafka consumer")
		if err := consumer.Start(ctx); err != nil {
			log.Fatal("failed to start consumer", logger.Field{"error", err})
		}
	}()

	// Ожидание сигналов завершения
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	log.Info("shutting down worker...")

	// Graceful shutdown
	if err := consumer.Close(); err != nil {
		log.Error("failed to close consumer", logger.Field{"error", err})
	}
}
```

### 13. Docker-compose (deployments/docker-compose.yml)

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:14
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: bigtech
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  zookeeper:
    image: confluentinc/cp-zookeeper:7.0.1
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:7.0.1
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
    ports:
      - "29092:29092"
      - "9092:9092"

  vault:
    image: vault:1.9.0
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: "root"
      VAULT_DEV_LISTEN_ADDRESS: "0.0.0.0:8200"
    ports:
      - "8200:8200"
    volumes:
      - ./vault:/vault/data
    cap_add:
      - IPC_LOCK

  api:
    build:
      context: ../..
      dockerfile: deployments/api.Dockerfile
    depends_on:
      - postgres
      - kafka
      - vault
    environment:
      VAULT_ADDR: "http://vault:8200"
      VAULT_TOKEN: "root"
    ports:
      - "8080:8080"
      - "50051:50051"

  worker:
    build:
      context: ../..
      dockerfile: deployments/worker.Dockerfile
    depends_on:
      - postgres
      - kafka
      - vault
    environment:
      VAULT_ADDR: "http://vault:8200"
      VAULT_TOKEN: "root"

volumes:
  postgres_data:
```

### 14. Dockerfile для API (deployments/api.Dockerfile)

```dockerfile
FROM golang:1.19-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o api ./cmd/api

FROM alpine:latest

WORKDIR /app

COPY --from=builder /app/api .
COPY --from=builder /app/api/proto ./api/proto

EXPOSE 8080 50051

CMD ["./api"]
```

### 15. Dockerfile для Worker (deployments/worker.Dockerfile)

```dockerfile
FROM golang:1.19-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o worker ./cmd/worker

FROM alpine:latest

WORKDIR /app

COPY --from=builder /app/worker .

CMD ["./worker"]
```

### 16. Makefile

```makefile
.PHONY: build run test migrate lint proto

build:
	go build -o bin/api ./cmd/api
	go build -o bin/worker ./cmd/worker

run:
	go run ./cmd/api

test:
	go test -v -cover ./...

migrate:
	migrate -path migrations -database "postgres://postgres:postgres@localhost:5432/bigtech?sslmode=disable" up

lint:
	golangci-lint run

proto:
	protoc --go_out=. --go_opt=paths=source_relative \
		--go-grpc_out=. --go-grpc_opt=paths=source_relative \
		api/proto/user.proto

docker-build:
	docker-compose -f deployments/docker-compose.yml build

docker-up:
	docker-compose -f deployments/docker-compose.yml up -d

docker-down:
	docker-compose -f deployments/docker-compose.yml down

docker-logs:
	docker-compose -f deployments/docker-compose.yml logs -f
```

## Тестирование

### Юнит-тест для сервиса (internal/service/user_test.go)

```go
package service_test

import (
	"context"
	"errors"
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/mock"

	"bigtech-project/internal/domain"
	"bigtech-project/internal/repository"
	"bigtech-project/internal/service"
	"bigtech-project/pkg/kafka"
	"bigtech-project/pkg/logger"
)

type MockUserRepository struct {
	mock.Mock
}

func (m *MockUserRepository) Create(ctx context.Context, user *domain.User) error {
	args := m.Called(ctx, user)
	return args.Error(0)
}

func (m *MockUserRepository) GetByID(ctx context.Context, id string) (*domain.User, error) {
	args := m.Called(ctx, id)
	return args.Get(0).(*domain.User), args.Error(1)
}

func (m *MockUserRepository) Update(ctx context.Context, user *domain.User) error {
	args := m.Called(ctx, user)
	return args.Error(0)
}

func (m *MockUserRepository) Delete(ctx context.Context, id string) error {
	args := m.Called(ctx, id)
	return args.Error(0)
}

type MockKafkaProducer struct {
	mock.Mock
}

func (m *MockKafkaProducer) Produce(ctx context.Context, message interface{}) error {
	args := m.Called(ctx, message)
	return args.Error(0)
}

func (m *MockKafkaProducer) Close() error {
	args := m.Called()
	return args.Error(0)
}

func TestUserService_CreateUser(t *testing.T) {
	tests := []struct {
		name        string
		user        *domain.User
		repoError   error
		prodError   error
		expectError bool
	}{
		{
			name: "success",
			user: &domain.User{
				Name:  "Test User",
				Email: "test@example.com",
			},
			repoError:   nil,
			prodError:   nil,
			expectError: false,
		},
		{
			name: "empty name",
			user: &domain.User{
				Name:  "",
				Email: "test@example.com",
			},
			repoError:   nil,
			prodError:   nil,
			expectError: true,
		},
		{
			name: "repository error",
			user: &domain.User{
				Name:  "Test User",
				Email: "test@example.com",
			},
			repoError:   errors.New("repository error"),
			prodError:   nil,
			expectError: true,
		},
		{
			name: "producer error",
			user: &domain.User{
				Name:  "Test User",
				Email: "test@example.com",
			},
			repoError:   nil,
			prodError:   errors.New("producer error"),
			expectError: false, // Producer error is logged but doesn't fail the operation
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			ctx := context.Background()
			mockRepo := new(MockUserRepository)
			mockProd := new(MockKafkaProducer)
			log := logger.NewTestLogger()

			mockRepo.On("Create", ctx, mock.AnythingOfType("*domain.User")).Return(tt.repoError)
			if tt.repoError == nil {
				mockProd.On("Produce", ctx, mock.AnythingOfType("domain.UserEvent")).Return(tt.prodError)
			}

			service := service.NewUserService(mockRepo, mockProd, log)

			err := service.CreateUser(ctx, tt.user)

			if tt.expectError {
				assert.Error(t, err)
			} else {
				assert.NoError(t, err)
				assert.NotEmpty(t, tt.user.ID)
			}

			mockRepo.AssertExpectations(t)
			if tt.repoError == nil {
				mockProd.AssertExpectations(t)
			}
		})
	}
}
```

## Как использовать проект

1. Клонируйте репозиторий
2. Установите зависимости: `go mod download`
3. Запустите инфраструктуру: `make docker-up`
4. Инициализируйте Vault (добавьте конфигурацию)
5. Запустите API: `go run cmd/api/main.go`
6. Запустите Worker: `go run cmd/worker/main.go`

## Заключение

Этот проект демонстрирует лучшие практики разработки больших приложений на Go:

1. Четкое разделение на слои (transport, service, repository)
2. Интеграция с внешними сервисами (Kafka, PostgreSQL, Vault)
3. Поддержка нескольких протоколов (HTTP, gRPC)
4. Централизованная конфигурация
5. Логирование
6. Тестирование
7. Docker-развертывание

Вы можете расширять этот проект, добавляя новые функции, улучшая обработку ошибок и добавляя больше интеграционных тестов.