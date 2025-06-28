
### TTL Кэш на Go

**TTL (Time To Live)** — это время жизни элемента в кэше. После истечения этого времени элемент становится недействительным и удаляется из кэша.

Для реализации TTL кэша можно использовать структуру данных, аналогичную LRU, но с добавлением временных меток, чтобы отслеживать время жизни каждого элемента.

#### Структура TTL кэша

10. Для каждого элемента хранится не только его значение, но и время, когда оно должно истечь.
11. Когда мы получаем элемент, нужно проверять, не истек ли TTL.
12. Если TTL истек, элемент удаляется из кэша.

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// Структура для представления элемента кэша
type TTLCacheItem struct {
	value     string
	expiry    time.Time
}

// Структура для TTL кэша
type TTLCache struct {
	cache     map[string]TTLCacheItem
	mu        sync.RWMutex
	ttl       time.Duration
}

// Конструктор для создания TTL кэша
func NewTTLCache(ttl time.Duration) *TTLCache {
	return &TTLCache{
		cache: make(map[string]TTLCacheItem),
		ttl:   ttl,
	}
}

// Метод для извлечения элемента из кэша
func (c *TTLCache) Get(key string) (string, bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()

	item, found := c.cache[key]
	if !found || time.Now().After(item.expiry) {
		// Если элемент не найден или TTL истек, удаляем элемент
		delete(c.cache, key)
		return "", false
	}

	return item.value, true
}

// Метод для добавления элемента в кэш с TTL
func (c *TTLCache) Put(key, value string) {
	c.mu.Lock()
	defer c.mu.Unlock()

	c.cache[key] = TTLCacheItem{
		value:  value,
		expiry: time.Now().Add(c.ttl), // Устанавливаем время истечения
	}
}

// Метод для удаления элемента
func (c *TTLCache) Remove(key string) {
	c.mu.Lock()
	defer c.mu.Unlock()

	delete(c.cache, key)
}

// Пример использования
func main() {
	cache := NewTTLCache(5 * time.Second) // TTL = 5 секунд

	cache.Put("key1", "value1")
	cache.Put("key2", "value2")

	time.Sleep(3 * time.Second) // Подождем 3 секунды

	if value, found := cache.Get("key1"); found {
		fmt.Println("key1:", value) // "key1: value1"
	}

	time.Sleep(3 * time.Second) // Подождем еще 3 секунды, TTL истекает

	if value, found := cache.Get("key1"); !found {
		fmt.Println("key1 not found") // "key1 not found"
	}
}
```

### Пояснение:

13. **TTLCache**:
    
    - `cache` — хеш-таблица для хранения элементов кэша.
    - `ttl` — продолжительность жизни элементов в кэше.
    - `mu` — мьютекс для синхронизации доступа.
14. **Метод `Put`**:
    
    - Добавляет элемент в кэш с установкой времени его истечения (`expiry`), которое будет через `ttl` от текущего времени.
15. **Метод `Get`**:
    
    - Проверяет, не истек ли TTL для элемента. Если элемент не найден или его TTL истек, удаляет его из кэша.
16. **Метод `Remove`**:
    
    - Удаляет элемент по ключу.

---

### Заключение

Оба кэша — LRU и TTL — позволяют эффективно управлять памятью и оптимизировать работу с данными. В зависимости от ваших задач (управление порядком доступа или управление временем жизни данных) можно выбрать подходящий алгоритм. LRU кэш полезен для использования с ограниченной памятью, где важно удалять наименее часто используемые элементы, а TTL кэш — для работы с данными, которые должны жить в кэше только ограниченное время.