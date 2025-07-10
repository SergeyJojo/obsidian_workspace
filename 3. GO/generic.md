Вот готовая **шпаргалка по дженерикам (generics) в Go** в формате для Obsidian (Markdown):

```markdown
---
tags: [go, generics, cheat-sheet]
---

# Дженерики в Go

## 📌 Основной синтаксис

### Объявление generic-функции
```go
func FuncName[T any](arg T) T {
    return arg
}
```

### Объявление generic-структуры
```go
type Container[T any] struct {
    value T
}
```

## 🔗 Ограничения типов (constraints)

### Встроенные ограничения
- `any` - любой тип (аналог `interface{}`)
- `comparable` - типы, поддерживающие `==` и `!=`

### Примеры пользовательских ограничений
```go
type Number interface {
    int | int8 | int16 | int32 | int64 | 
    uint | uint8 | uint16 | uint32 | uint64 |
    float32 | float64
}

func Sum[T Number](a, b T) T {
    return a + b
}
```

## 🧩 Полезные шаблоны

### Generic Map
```go
func Map[T, U any](s []T, f func(T) U) []U {
    result := make([]U, len(s))
    for i, v := range s {
        result[i] = f(v)
    }
    return result
}
```

### Generic Filter
```go
func Filter[T any](s []T, f func(T) bool) []T {
    var result []T
    for _, v := range s {
        if f(v) {
            result = append(result, v)
        }
    }
    return result
}
```

## ⚠️ Ограничения
1. Нет специализации методов
2. Нет метапрограммирования
3. Нет операторов типа `+` в общем виде

## 🔥 Примеры использования

### Кэш с разными типами
```go
type Cache[T any] struct {
    data map[string]T
}

func (c *Cache[T]) Set(key string, value T) {
    c.data[key] = value
}

func (c *Cache[T]) Get(key string) (T, bool) {
    val, ok := c.data[key]
    return val, ok
}
```

### Стек любого типа
```go
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() T {
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item
}
```

## 📚 Полезные ссылки
- [Официальный туторил](https://go.dev/doc/tutorial/generics)
- [Type Parameters Proposal](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md)
```

### Как использовать в Obsidian:
1. Создайте новую заметку
2. Вставьте этот Markdown-код
3. Добавьте в теги `#go` и `#generics`
4. Свяжите с другими заметками по Go

Формат оптимизирован для Obsidian:
- Четкие заголовки с emoji
- Подсветка кода
- Теги для удобного поиска
- Ссылки на дополнительные материалы