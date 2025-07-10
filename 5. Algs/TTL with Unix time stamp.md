
Вот минималистичный пример **TTL-кэша на Go**, с автоматическим удалением просроченных записей:

---

### ✅ `TTLCache` на Go

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type cacheItem struct {
	value      interface{}
	expiration int64 // UnixNano
}

type TTLCache struct {
	data  map[string]cacheItem
	mutex sync.RWMutex
	ttl   time.Duration
}

func NewTTLCache(ttl time.Duration) *TTLCache {
	cache := &TTLCache{
		data: make(map[string]cacheItem),
		ttl:  ttl,
	}
	go cache.cleanup()
	return cache
}

func (c *TTLCache) Set(key string, value interface{}) {
	c.mutex.Lock()
	defer c.mutex.Unlock()
	c.data[key] = cacheItem{
		value:      value,
		expiration: time.Now().Add(c.ttl).UnixNano(),
	}
}

func (c *TTLCache) Get(key string) (interface{}, bool) {
	c.mutex.RLock()
	item, found := c.data[key]
	c.mutex.RUnlock()
	if !found || time.Now().UnixNano() > item.expiration {
		return nil, false
	}
	return item.value, true
}

func (c *TTLCache) cleanup() {
	ticker := time.NewTicker(c.ttl / 2)
	for range ticker.C {
		now := time.Now().UnixNano()
		c.mutex.Lock()
		for k, v := range c.data {
			if now > v.expiration {
				delete(c.data, k)
			}
		}
		c.mutex.Unlock()
	}
}
```

---

### ✅ Пример использования:

```go
func main() {
	cache := NewTTLCache(5 * time.Second)

	cache.Set("foo", "bar")

	val, ok := cache.Get("foo")
	fmt.Println(val, ok) // bar true

	time.Sleep(6 * time.Second)

	val, ok = cache.Get("foo")
	fmt.Println(val, ok) // nil false (просрочено)
}
```

---

Хочешь — добавим:

- Размерные ограничения (`max size`)
    
- Статистику по hit/miss
    
- Ручное удаление ключей
    
- Индивидуальный TTL для каждой записи
    

Напиши, если нужно расширить.