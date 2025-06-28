
Вот детальная логика и последовательность реализации LRU-кэша с использованием двусвязного списка и заглушек (dummy nodes) head/tail:

### 1. Структуры данных

```go
type Node struct {
    key   int
    value int
    prev  *Node
    next  *Node
}

type LRUCache struct {
    capacity int
    cache    map[int]*Node
    head     *Node  // Dummy head (всегда первый элемент)
    tail     *Node  // Dummy tail (всегда последний)
}
```

### 2. Инициализация

```go
func NewLRUCache(capacity int) *LRUCache {
    // Создаем заглушки
    head := &Node{}
    tail := &Node{}
    
    // Связываем их между собой
    head.next = tail
    tail.prev = head
    
    return &LRUCache{
        capacity: capacity,
        cache:    make(map[int]*Node),
        head:     head,
        tail:     tail,
    }
}
```

### 3. Вспомогательные методы работы со списком

```go
// Добавление узла сразу после head (новые элементы в начало)
func (l *LRUCache) addNode(node *Node) {
    node.prev = l.head
    node.next = l.head.next
    
    l.head.next.prev = node
    l.head.next = node
}

// Удаление узла из списка
func (l *LRUCache) removeNode(node *Node) {
    prev := node.prev
    next := node.next
    
    prev.next = next
    next.prev = prev
}

// Перемещение узла в начало
func (l *LRUCache) moveToHead(node *Node) {
    l.removeNode(node)
    l.addNode(node)
}

// Удаление последнего узла (перед tail)
func (l *LRUCache) popTail() *Node {
    node := l.tail.prev
    l.removeNode(node)
    return node
}
```

### 4. Основные операции

```go
func (l *LRUCache) Get(key int) int {
    if node, ok := l.cache[key]; ok {
        l.moveToHead(node)  // Обновляем позицию
        return node.value
    }
    return -1
}

func (l *LRUCache) Put(key int, value int) {
    // Если ключ уже существует
    if node, ok := l.cache[key]; ok {
        node.value = value    // Обновляем значение
        l.moveToHead(node)    // Перемещаем в начало
        return
    }
    
    // Создаем новый узел
    newNode := &Node{
        key:   key,
        value: value,
    }
    
    // Добавляем в кэш и список
    l.cache[key] = newNode
    l.addNode(newNode)
    
    // Проверка на переполнение
    if len(l.cache) > l.capacity {
        tail := l.popTail()    // Удаляем последний
        delete(l.cache, tail.key)
    }
}
```

### 5. Визуализация работы

```
До добавления:
Head <-> Tail

После Put(1,10):
Head <-> [1:10] <-> Tail

После Put(2,20):
Head <-> [2:20] <-> [1:10] <-> Tail

После Get(1):
Head <-> [1:10] <-> [2:20] <-> Tail

После Put(3,30) (при capacity=2):
Head <-> [3:30] <-> [1:10] <-> Tail
(элемент 2:20 удален)
```

### 6. Ключевые особенности реализации

1. **Заглушки (Dummy Nodes)**
   - `head` и `tail` никогда не содержат данных
   - Упрощают логику добавления/удаления (нет проверок на nil)

2. **Сложность операций**
   - Get: O(1) (хэш-мапа + константное время перемещения)
   - Put: O(1) 

3. **Потокобезопасность**
   - Требуется `sync.Mutex` для конкурентного доступа

4. **Оптимизации памяти**
   - Предвыделение узлов (object pool) для частых обновлений

### 7. Полный пример с синхронизацией

```go
import "sync"

type ThreadSafeLRUCache struct {
    lru    *LRUCache
    mutex  sync.Mutex
}

func (t *ThreadSafeLRUCache) Get(key int) int {
    t.mutex.Lock()
    defer t.mutex.Unlock()
    return t.lru.Get(key)
}

func (t *ThreadSafeLRUCache) Put(key int, value int) {
    t.mutex.Lock()
    defer t.mutex.Unlock()
    t.lru.Put(key, value)
}
```

### 8. Тестирование

```go
func TestLRU(t *testing.T) {
    cache := NewLRUCache(2)
    
    cache.Put(1, 1)
    cache.Put(2, 2)
    
    if val := cache.Get(1); val != 1 {
        t.Errorf("Expected 1, got %d", val)
    }
    
    cache.Put(3, 3)  // Должен вытеснить ключ 2
    
    if val := cache.Get(2); val != -1 {
        t.Errorf("Expected -1, got %d", val)
    }
}
``` 

Такой подход обеспечивает:
- Четкое разделение логики
- Минимальное количество проверок граничных условий
- Легкую расширяемость для новых функций (например, TTL)