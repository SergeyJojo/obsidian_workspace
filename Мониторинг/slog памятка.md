### **Памятка: slog в Go**  
Минималистичная шпаргалка по работе с пакетом `log/slog`.

---

#### **1. Инициализация**
```go
import "log/slog"

// Текстовый логгер (stdout)
logger := slog.New(slog.NewTextHandler(os.Stdout, nil))

// JSON-логгер (файл)
file, _ := os.Create("app.log")
jsonLogger := slog.New(slog.NewJSONHandler(file, nil))
```

---

#### **2. Уровни логирования**
```go
logger.Debug("Отладочное сообщение")
logger.Info("Информационное сообщение")
logger.Warn("Предупреждение")
logger.Error("Ошибка", "err", err)
```

---

#### **3. Контекст и атрибуты**
```go
// С атрибутами
logger.Info("Запрос", "method", "GET", "path", "/api")

// С группой атрибутов
logger.Info("DB Query",
    slog.String("query", "SELECT * FROM users"),
    slog.Group("timing",
        slog.Int("duration_ms", 120),
    ),
)
```

---

#### **4. Настройка логгера**
```go
opts := &slog.HandlerOptions{
    Level: slog.LevelDebug, // Уровень логирования
    AddSource: true,       // Добавлять источник вызова
}

logger := slog.New(slog.NewJSONHandler(os.Stderr, opts))
```

---

#### **5. Глобальный логгер**
```go
slog.SetDefault(logger)
slog.Info("Сообщение через глобальный логгер")
```

---

#### **6. Логирование ошибок**
```go
if err != nil {
    logger.Error("Ошибка выполнения",
        slog.String("error", err.Error()),
        slog.Any("metadata", data),
    )
}
```

---

**Формат:**  
- Только ключевые методы  
- Без лишних параметров  
- Готово к использованию  

🚀 **Оптимизировано для быстрого копирования в код**