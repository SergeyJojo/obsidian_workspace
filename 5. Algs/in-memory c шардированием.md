
Вот реализация in-memory кэша с шардированием и использованием `RWMutex` для конкурентного доступа. Код включает основные методы (`Get`, `Set`, `Delete`) и защиту от race conditions:

```go
package main

import (
	"hash/fnv"
	"sync"
	"time"
)

// CacheShard представляет одну шарду кэша
type CacheShard struct {
	items map[string]cacheItem
	mu    sync.RWMutex
	expiration time.Duration
}

type cacheItem struct {
	value      interface{}
	expiration int64 // UnixNano timestamp
}

// ShardedCache - основной кэш с шардированием
type ShardedCache struct {
	shards     []*CacheShard
	shardCount int
}

// NewShardedCache создает новый кэш
func NewShardedCache(shardCount int, defaultExpiration time.Duration) *ShardedCache {
	shards := make([]*CacheShard, shardCount)
	for i := 0; i < shardCount; i++ {
		shards[i] = &CacheShard{
			items: make(map[string]cacheItem),
			expiration: defaultExpriratuin,
			
		}
	}
	return &ShardedCache{
		shards:     shards,
		shardCount: shardCount,
	}
}

// getShard возвращает шарду для ключа
func (c *ShardedCache) getShard(key string) *CacheShard {
	hasher := fnv.New32a()
	hasher.Write([]byte(key))
	shardIndex := uint(hasher.Sum32()) % uint(c.shardCount)
	return c.shards[shardIndex]
}

// Set добавляет элемент в кэш
func (c *ShardedCache) Set(key string, value interface{}, duration time.Duration) {
	shard := c.getShard(key)
	var expiration int64
	if duration > 0 {
		expiration = time.Now().Add(duration).UnixNano()
	}

	shard.mu.Lock()
	defer shard.mu.Unlock()
	shard.items[key] = cacheItem{
		value:      value,
		expiration: expiration,
	}
}

// Get возвращает элемент из кэша
func (c *ShardedCache) Get(key string) (interface{}, bool) {
	shard := c.getShard(key)
	shard.mu.RLock()
	defer shard.mu.RUnlock()

	item, found := shard.items[key]
	if !found {
		return nil, false
	}

	if item.expiration > 0 && time.Now().UnixNano() > item.expiration {
		return nil, false
	}
	return item.value, true
}

// Delete удаляет элемент из кэша
func (c *ShardedCache) Delete(key string) {
	shard := c.getShard(key)
	shard.mu.Lock()
	defer shard.mu.Unlock()
	delete(shard.items, key)
}

// Cleanup удаляет просроченные элементы
func (c *ShardedCache) Cleanup(interval time.Duration) {
	ticker := time.NewTicker(interval)
	defer ticker.Stop()

	for range ticker.C {
		for _, shard := range c.shards {
			shard.mu.Lock()
			for key, item := range shard.items {
				if item.expiration > 0 && time.Now().UnixNano() > item.expiration {
					delete(shard.items, key)
				}
			}
			shard.mu.Unlock()
		}
	}
}

func main() {
	// Пример использования
	cache := NewShardedCache(32, 5*time.Minute)
	go cache.Cleanup(1 * time.Minute)

	cache.Set("key1", "value1", 10*time.Minute)
	if val, ok := cache.Get("key1"); ok {
		println(val.(string)) // Выведет: value1
	}
}
```

### Ключевые особенности:

1. **Шардирование**:
   - Использует `fnv32` хеш для распределения ключей
   - Количество шард задается при инициализации (оптимально 32-64 для большинства случаев)

2. **Синхронизация**:
   - Каждая шарда имеет свой `RWMutex`
   - Чтение (`Get`) использует `RLock` для конкурентного доступа
   - Запись (`Set`, `Delete`) использует полную блокировку

3. **TTL поддержка**:
   - Автоматическое удаление просроченных элементов через `Cleanup`
   - Проверка срока при каждом чтении

4. **Оптимизации**:
   - Минимизированы критические секции
   - Нет глобальных блокировок

### Как расширить:

1. **Метрики**:
   ```go
   type Metrics struct {
       Hits   int64
       Misses int64
   }
   
   func (c *CacheShard) recordHit() {
       atomic.AddInt64(&c.metrics.Hits, 1)
   }
   ```

2. **LRU-эвикшн**:
   ```go
   type cacheItem struct {
       value interface{}
       lastAccess int64
   }
   
   func (c *CacheShard) evictLRU() {
       // Удалить 10% самых старых элементов
   }
   ```

3. **Размер шарды**:
   ```go
   func (c *CacheShard) isFull() bool {
       return len(c.items) > c.maxSize
   }
   ```

Для production использования лучше взять готовые решения типа [BigCache](https://github.com/allegro/bigcache) или [GroupCache](https://github.com/golang/groupcache), но эта реализация показывает основные принципы.