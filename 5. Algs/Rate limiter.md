
Вот реализация **простого, но эффективного rate limiter'а на Go** без внешних библиотек. Поддерживает ограничение запросов в секунду (RPS) с возможностью настройки лимита.

---

### 🔹 **Token Bucket Rate Limiter** (на каналах)
```go
package main

import (
	"time"
)

type RateLimiter struct {
	tokens chan struct{}
	stop   chan struct{}
}

// NewRateLimiter создает лимитер с maxRequests в секунду.
func NewRateLimiter(maxRequests int) *RateLimiter {
	rl := &RateLimiter{
		tokens: make(chan struct{}, maxRequests),
		stop:   make(chan struct{}),
	}

	// Заполняем "ведро" токенами
	for i := 0; i < maxRequests; i++ {
		rl.tokens <- struct{}{}
	}

	// Запускаем пополнение токенов
	go rl.refillTokens(maxRequests)
	return rl
}

// refillTokens добавляет токены каждую секунду.
func (rl *RateLimiter) refillTokens(maxRequests int) {
	ticker := time.NewTicker(time.Second)
	defer ticker.Stop()

	for {
		select {
		case <-ticker.C:
			// Пополняем до максимума, не блокируясь
			for i := len(rl.tokens); i < maxRequests; i++ {
				select {
				case rl.tokens <- struct{}{}:
				default:
					break
				}
			}
		case <-rl.stop:
			return
		}
	}
}

// Allow проверяет, доступен ли токен.
func (rl *RateLimiter) Allow() bool {
	select {
	case <-rl.tokens:
		return true
	default:
		return false
	}
}

// Stop останавливает лимитер.
func (rl *RateLimiter) Stop() {
	close(rl.stop)
}
```

---

### 🔹 **Как это работает?**
1. **Token Bucket Algorithm**:
   - `tokens` — канал с "токенами" (максимум `maxRequests` штук).
   - Каждый запрос забирает 1 токен (`<-tokens`).
   - Если токенов нет — запрос отклоняется (`false`).

2. **Пополнение**:
   - Каждую секунду канал заполняется токенами до `maxRequests`.

3. **Гарантии**:
   - Не больше `maxRequests` вызовов в секунду.
   - Потокобезопасность (горутины могут вызывать `Allow()` конкурентно).

---

### 🔹 **Пример использования**
```go
func main() {
	limiter := NewRateLimiter(5) // 5 RPS
	defer limiter.Stop()

	for i := 0; i < 10; i++ {
		if limiter.Allow() {
			println("Request", i, "✅")
		} else {
			println("Request", i, "❌ (Rate limit exceeded)")
		}
		time.Sleep(200 * time.Millisecond) // Имитация запросов
	}
}
```

**Вывод**:
```
Request 0 ✅
Request 1 ✅
Request 2 ✅
Request 3 ✅
Request 4 ✅
Request 5 ❌ (Rate limit exceeded)
Request 6 ✅  (Новая секунда)
...
```

---

### 🔹 **Плюсы реализации**
- **Простота**: Только стандартная библиотека.
- **Гибкость**: Можно настраивать `maxRequests`.
- **Потокобезопасность**: Не требует `sync.Mutex`.

---

### 🔹 **Ограничения**
- Нет "бурстного" режима (как в `rate.NewLimiter(5, 10)`).
- Нет распределенного лимита (только для одного процесса).

Для продвинутых сценариев лучше использовать `golang.org/x/time/rate`, но эта реализация отлично подходит для базовых задач! 🚀