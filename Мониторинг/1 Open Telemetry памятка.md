### **Памятка: OpenTelemetry в Go**  
Минималистичный гайд по основным операциям.

---

#### **1. Инициализация**  
```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/jaeger"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
)

// Создаем экспортер (Jaeger)
exp, _ := jaeger.New(jaeger.WithCollectorEndpoint(jaeger.WithEndpoint("http://localhost:14268/api/traces")))

// Настраиваем провайдер трейсов
tp := sdktrace.NewTracerProvider(
    sdktrace.WithBatcher(exp),
    sdktrace.WithResource(resource.NewWithAttributes(
        semconv.SchemaURL,
        semconv.ServiceNameKey.String("my-service"),
    )),
)
otel.SetTracerProvider(tp)
```

---

#### **2. Трейсинг**  
```go
// Создаем span
ctx, span := otel.Tracer("app").Start(ctx, "operation-name")
defer span.End()

// Добавляем атрибуты
span.SetAttributes(
    attribute.String("key", "value"),
    attribute.Int("count", 42),
)

// Запись ошибки
span.RecordError(err)
```

---

#### **3. Метрики**  
```go
import "go.opentelemetry.io/otel/metric"

meter := otel.Meter("app")
counter, _ := meter.Int64Counter("requests.count")

// Инкремент метрики
counter.Add(ctx, 1)
```

---

#### **4. Логирование (в трейсы)**  
```go
span.AddEvent("log-event", 
    trace.WithAttributes(
        attribute.String("message", "something happened"),
    ),
)
```

---

#### **5. Контекст**  
```go
// Передача контекста между функциями
func Handler(ctx context.Context) {
    _, span := otel.Tracer("app").Start(ctx, "handler")
    defer span.End()
    AnotherFunction(ctx)
}
```

---

#### **6. Инструментация HTTP**  
```go
import "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"

// Клиент
client := &http.Client{
    Transport: otelhttp.NewTransport(http.DefaultTransport),
}

// Сервер
http.Handle("/", otelhttp.NewHandler(handler, "endpoint-name"))
```

---

**Ключевые компоненты:**  
- `Tracer` – трейсинг операций  
- `Meter` – сбор метрик  
- `Resource` – описание сервиса  
- `Context` – передача трейсов  

🚀 **Готово для интеграции в микросервисы**