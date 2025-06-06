
Ваш вопрос о реализации собственного rate limiter (ограничитель скорости) в Go касается важного механизма управления нагрузкой, который может быть полезен для вашего проекта "Learning Repeater" (Версия 2), например, чтобы ограничить запросы к API или БД. Я разберу это академически, подробно, с примерами, объясню алгоритмы и свяжу с вашим проектом.

---

### Что такое Rate Limiting?
Rate limiting — это техника, ограничивающая частоту выполнения операций (например, запросов в секунду) для предотвращения перегрузки системы. В контексте вашего опыта с Яндекс.Директ API это могло быть важно для соблюдения лимитов (например, 20 запросов/сек).

#### Популярные алгоритмы
1. **Token Bucket (Ведро с токенами)**:
   - Имеется "ведро" с токенами, пополняемое с фиксированной скоростью.
   - Каждый запрос берёт токен; если токенов нет, запрос ждёт или отклоняется.
2. **Leaky Bucket (Протекающее ведро)**:
   - Запросы поступают в "ведро", которое "протекает" с постоянной скоростью.
   - Если ведро переполняется, запросы отклоняются.
3. **Fixed Window (Фиксированное окно)**:
   - Считает запросы в фиксированном временном окне (например, 100 запросов в минуту).
   - Прост, но уязвим к всплескам на границах окна.
4. **Sliding Window (Скользящее окно)**:
   - Учитывает запросы в непрерывном окне времени, более точен.

Я реализую **Token Bucket**, так как он гибкий, широко используется (например, в `golang.org/x/time/rate`) и подходит для большинства случаев.

---

### Реализация Token Bucket Rate Limiter
#### Идея
- Ведро вмещает `capacity` токенов.
- Токены добавляются со скоростью `rate` (например, 10 токенов/сек).
- Каждый запрос требует 1 токен. Если токенов нет, запрос блокируется или отклоняется.

#### Код на Go
```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// RateLimiter реализует Token Bucket
type RateLimiter struct {
	capacity  int           // Максимум токенов
	tokens    int           // Текущие токены
	rate      time.Duration // Скорость пополнения (токенов в единицу времени)
	lastCheck time.Time     // Последнее обновление токенов
	mu        sync.Mutex    // Для потокобезопасности
}

// NewRateLimiter создаёт новый лимитер
func NewRateLimiter(capacity int, rate time.Duration) *RateLimiter {
	return &RateLimiter{
		capacity:  capacity,
		tokens:    capacity, // Изначально полное ведро
		rate:      rate,
		lastCheck: time.Now(),
	}
}

// updateTokens обновляет количество токенов
func (rl *RateLimiter) updateTokens() {
	rl.mu.Lock()
	defer rl.mu.Unlock()

	now := time.Now()
	elapsed := now.Sub(rl.lastCheck)
	tokensToAdd := int(elapsed / rl.rate) // Сколько токенов добавилось
	if tokensToAdd > 0 {
		rl.tokens = min(rl.capacity, rl.tokens+tokensToAdd)
		rl.lastCheck = now
	}
}

// min — вспомогательная функция
func min(a, b int) int {
	if a < b {
		return a
	}
	return b
}

// Allow проверяет, можно ли выполнить запрос
func (rl *RateLimiter) Allow() bool {
	rl.updateTokens()
	rl.mu.Lock()
	defer rl.mu.Unlock()

	if rl.tokens > 0 {
		rl.tokens--
		return true
	}
	return false
}

// Wait блокирует, пока токен не появится
func (rl *RateLimiter) Wait() {
	for !rl.Allow() {
		time.Sleep(rl.rate / time.Duration(rl.capacity)) // Ждём токен
	}
}

func main() {
	// Лимитер: 10 токенов, 1 токен каждые 100 мс (10 запросов/сек)
	rl := NewRateLimiter(10, 100*time.Millisecond)

	// Тест: 15 запросов
	for i := 0; i < 15; i++ {
		if rl.Allow() {
			fmt.Printf("Request %d allowed at %v\n", i, time.Now())
		} else {
			fmt.Printf("Request %d denied at %v\n", i, time.Now())
		}
		time.Sleep(50 * time.Millisecond) // Симуляция интервала запросов
	}
}
```

#### Как работает
1. **Инициализация**:
   - `capacity=10`: Ведро вмещает 10 токенов.
   - `rate=100ms`: 1 токен добавляется каждые 100 мс (10 токенов/сек).
2. **Обновление токенов**:
   - `updateTokens()` вычисляет, сколько времени прошло с последнего обновления, и добавляет токены, не превышая `capacity`.
3. **Проверка**:
   - `Allow()` возвращает `true`, если есть токен, и уменьшает их число.
   - `Wait()` блокирует, пока токен не появится.

#### Вывод примера
```
Request 0 allowed at ...
Request 1 allowed at ...
...
Request 9 allowed at ...
Request 10 denied at ...
Request 11 denied at ...
```
- Первые 10 запросов проходят, затем лимит исчерпан до появления новых токенов.

---

### Альтернативная реализация: Fixed Window
#### Идея
- Считаем запросы в окне (например, 1 секунда).
- Сбрасываем счётчик в начале нового окна.

#### Код
```go
type FixedWindowLimiter struct {
	limit     int
	count     int
	window    time.Duration
	startTime time.Time
	mu        sync.Mutex
}

func NewFixedWindowLimiter(limit int, window time.Duration) *FixedWindowLimiter {
	return &FixedWindowLimiter{
		limit:     limit,
		window:    window,
		count:     0,
		startTime: time.Now(),
	}
}

func (fw *FixedWindowLimiter) Allow() bool {
	fw.mu.Lock()
	defer fw.mu.Unlock()

	now := time.Now()
	if now.Sub(fw.startTime) >= fw.window {
		fw.count = 0
		fw.startTime = now
	}
	if fw.count < fw.limit {
		fw.count++
		return true
	}
	return false
}

func main() {
	fw := NewFixedWindowLimiter(5, time.Second)
	for i := 0; i < 10; i++ {
		if fw.Allow() {
			fmt.Printf("Request %d allowed\n", i)
		} else {
			fmt.Printf("Request %d denied\n", i)
		}
		time.Sleep(100 * time.Millisecond)
	}
}
```

#### Проблема
- Всплески на границах окна: 5 запросов в конце секунды + 5 в начале следующей = 10 за 0.1 сек, что нарушает равномерность.

---

### Связь с "Learning Repeater"
#### Применение
1. **Content Service**:
   - Ограничение запросов к PostgreSQL:
     ```go
     rl := NewRateLimiter(100, 10*time.Millisecond) // 100 запросов/сек
     if rl.Allow() {
         db.Exec("SELECT * FROM questions")
     } else {
         return errors.New("rate limit exceeded")
     }
     ```
2. **Progress Service**:
   - Защита Redis от перегрузки:
     ```go
     rl := NewRateLimiter(50, 20*time.Millisecond) // 50 запросов/сек
     rl.Wait() // Блокируем, если лимит исчерпан
     redis.Set("user:123:learned", "5")
     ```
3. **Reminder Service**:
   - Ограничение отправки напоминаний через Kafka:
     ```go
     rl := NewRateLimiter(10, 100*time.Millisecond)
     if rl.Allow() {
         producer.SendMessage(msg)
     }
     ```

#### Почему свой?
- Готовый `golang.org/x/time/rate` хорош, но свой лимитер даёт:
  - Полный контроль (например, кастомная логика отказа).
  - Изучение принципов (полезно для интервью).

---

### Итог
- **Token Bucket**: Гибкий, равномерный, с токенами.
- **Fixed Window**: Простой, но уязвим к всплескам.
- **Реализация**: `Allow()` для проверки, `Wait()` для блокировки, `sync.Mutex` для потокобезопасности.
- **Проект**: Применяйте для защиты БД, API или Kafka от перегрузки.

Если хотите углубиться (например, sliding window или профилирование), дайте знать!