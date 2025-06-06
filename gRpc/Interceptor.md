
Интерцепторы (interceptors) в gRPC — это мощный механизм, который позволяет перехватывать и обрабатывать запросы и ответы на уровне сервера или клиента. Они используются для добавления дополнительной логики, такой как логирование, аутентификация, метрики, обработка ошибок или модификация данных, без необходимости изменять основной код обработчиков. В этом ответе я подробно объясню, как работают интерцепторы в gRPC на Go, какие типы интерцепторов бывают, и приведу примеры их использования.

---

Что такое интерцепторы?

Интерцепторы — это middleware (посредники) в gRPC, которые вставляются в цепочку обработки вызовов. Они позволяют:

- Перехватывать запросы перед тем, как они попадут в обработчик (на сервере) или отправятся на сервер (на клиенте).
    
- Обрабатывать ответы перед их возвращением клиенту (на сервере) или обработкой клиентом.
    
- Выполнять сквозные задачи (например, замер времени выполнения, валидацию токенов).
    

В Go gRPC интерцепторы делятся на два основных типа:

1. Унарные интерцепторы (Unary Interceptors) — для унарных RPC (один запрос — один ответ).
    
2. Потоковые интерцепторы (Stream Interceptors) — для потоковых RPC (множественные запросы/ответы).
    

---

Типы интерцепторов

1. Унарные интерцепторы

- Применяются к унарным вызовам (стандартный запрос-ответ).
    
- На сервере: grpc.UnaryServerInterceptor.
    
- На клиенте: grpc.UnaryClientInterceptor.
    

Сигнатура унарного серверного интерцептора:

go

```go
type UnaryServerInterceptor func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (resp interface{}, err error)
```

- ctx: Контекст запроса.
    
- req: Входящий запрос.
    
- info: Метаданные о вызове (например, имя метода).
    
- handler: Следующий обработчик в цепочке.
    

Сигнатура унарного клиентского интерцептора:

go

```go
type UnaryClientInterceptor func(ctx context.Context, method string, req, reply interface{}, cc *ClientConn, invoker UnaryInvoker, opts ...CallOption) error
```

2. Потоковые интерцепторы

- Применяются к потоковым вызовам (Server Streaming, Client Streaming, Bidirectional Streaming).
    
- На сервере: grpc.StreamServerInterceptor.
    
- На клиенте: grpc.StreamClientInterceptor.
    

Сигнатура потокового серверного интерцептора:

go

```go
type StreamServerInterceptor func(srv interface{}, ss ServerStream, info *StreamServerInfo, handler StreamHandler) error
```

---

Как работают интерцепторы?

1. На сервере:
    
    - Интерцептор вызывается перед тем, как запрос попадет в основной обработчик (например, метод сервиса).
        
    - Он может модифицировать контекст, проверить метаданные, записать логи и т.д.
        
    - После выполнения логики интерцептор вызывает handler для продолжения обработки или возвращает ошибку, прерывая цепочку.
        
2. На клиенте:
    
    - Интерцептор вызывается перед отправкой запроса на сервер.
        
    - Может добавлять метаданные (например, токены авторизации), логировать запросы или обрабатывать ошибки ответа.
        
3. Цепочка интерцепторов:
    
    - Можно задать несколько интерцепторов, и они выполняются в порядке их регистрации (как слои в луковице).
        

---

Пример: Унарный серверный интерцептор

Допустим, мы хотим логировать время выполнения каждого gRPC-вызова на сервере.

go

```go
package main

import (
    "context"
    "log"
    "time"

    "google.golang.org/grpc"
)

// Логирующий интерцептор
func loggingInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    start := time.Now()

    // Вызываем следующий обработчик в цепочке
    resp, err := handler(ctx, req)

    // Логируем время выполнения
    log.Printf("Method: %s, Duration: %s, Error: %v", info.FullMethod, time.Since(start), err)
    return resp, err
}

// Пример сервиса
type server struct {
    pb.UnimplementedGreeterServer
}

func (s *server) SayHello(ctx context.Context, req *pb.HelloRequest) (*pb.HelloReply, error) {
    return &pb.HelloReply{Message: "Hello, " + req.Name}, nil
}

func main() {
    lis, _ := net.Listen("tcp", ":50051")
    s := grpc.NewServer(
        grpc.UnaryInterceptor(loggingInterceptor), // Регистрируем интерцептор
    )
    pb.RegisterGreeterServer(s, &server{})
    s.Serve(lis)
}
```

Что происходит:

- Каждый вызов SayHello проходит через loggingInterceptor.
    
- Интерцептор замеряет время и логирует метод, длительность и ошибку (если есть).
    

---

Пример: Унарный клиентский интерцептор

Теперь добавим интерцептор на клиент, который добавляет токен в метаданные.

go

```go
package main

import (
    "context"
    "log"

    "google.golang.org/grpc"
    "google.golang.org/grpc/metadata"
)

func authInterceptor(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
    // Добавляем токен в метаданные
    ctx = metadata.AppendToOutgoingContext(ctx, "authorization", "Bearer my-token")
    
    // Логируем запрос
    log.Printf("Calling method: %s", method)
    
    // Вызываем сервер
    err := invoker(ctx, method, req, reply, cc, opts...)
    return err
}

func main() {
    conn, _ := grpc.Dial(
        "localhost:50051",
        grpc.WithInsecure(),
        grpc.WithUnaryInterceptor(authInterceptor), // Регистрируем интерцептор
    )
    defer conn.Close()

    client := pb.NewGreeterClient(conn)
    resp, _ := client.SayHello(context.Background(), &pb.HelloRequest{Name: "World"})
    log.Println(resp.Message)
}
```

Что происходит:

- Перед вызовом сервера интерцептор добавляет токен в метаданные и логирует метод.
    

---

Пример: Потоковый интерцептор

Для потокового RPC (например, bidirectional streaming) интерцептор работает с объектом ServerStream.

go

```go
func streamLoggingInterceptor(srv interface{}, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
    log.Printf("Stream started: %s", info.FullMethod)
    err := handler(srv, ss)
    log.Printf("Stream finished: %s, Error: %v", info.FullMethod, err)
    return err
}
```

Регистрация:

go

```go
s := grpc.NewServer(
    grpc.StreamInterceptor(streamLoggingInterceptor),
)
```

---

Полезные библиотеки

- grpc-ecosystem/go-grpc-middleware: Содержит готовые интерцепторы для логирования, метрик, ретраев, валидации и т.д.
    
    - Пример: grpc_zap для интеграции с Zap-логгером.
        
    - Установка: go get github.com/grpc-ecosystem/go-grpc-middleware.
        

Пример использования:

go

```go
import (
    "github.com/grpc-ecosystem/go-grpc-middleware/logging/zap"
    "go.uber.org/zap"
)

logger, _ := zap.NewProduction()
s := grpc.NewServer(
    grpc.UnaryInterceptor(grpc_zap.UnaryServerInterceptor(logger)),
)
```

---

Итог

- Унарные интерцепторы: Для одиночных запросов, регистрируются через grpc.UnaryInterceptor.
    
- Потоковые интерцепторы: Для потоков, регистрируются через grpc.StreamInterceptor.
    
- Используются для логирования, аутентификации, метрик и других сквозных задач.
    
- Регистрируются при создании сервера (grpc.NewServer) или клиента (grpc.Dial).
    

Интерцепторы — это гибкий способ расширить функциональность gRPC без изменения бизнес-логики. Если у тебя есть конкретная задача (например, авторизация или метрики), напиши — я помогу с примером!