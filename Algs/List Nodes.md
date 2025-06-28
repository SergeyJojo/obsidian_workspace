
Использование **фиктивных (фейковых) `head` и `tail` узлов** — отличный способ упростить операции с двусвязным списком.

🔹 **Фиктивные `head` и `tail`**:

- **`head`** — не содержит данных, но указывает на первый реальный элемент.
    
- **`tail`** — не содержит данных, но указывает на последний реальный элемент.
    
- **Не нужно проверять `nil` при добавлении и удалении узлов**.
    

---

## 🔥 **Реализация двусвязного списка с фиктивными узлами**

```go
package main

import "fmt"

// Узел списка
type Node struct {
    Value int
    Prev  *Node
    Next  *Node
}

// Двусвязный список с фиктивными head и tail
type LinkedList struct {
    Head *Node // Фиктивный head
    Tail *Node // Фиктивный tail
}

// Создание нового списка
func NewLinkedList() *LinkedList {
    list := &LinkedList{}
    list.Head = &Node{} // Фиктивный head
    list.Tail = &Node{} // Фиктивный tail

    list.Head.Next = list.Tail
    list.Tail.Prev = list.Head
    return list
}

// 🔹 Добавление узла в конец
func (list *LinkedList) AddToTail(value int) {
    newNode := &Node{Value: value}

    last := list.Tail.Prev // Предыдущий реальный узел
    last.Next = newNode
    newNode.Prev = last
    newNode.Next = list.Tail
    list.Tail.Prev = newNode
}

// 🔹 Добавление узла в начало
func (list *LinkedList) AddToHead(value int) {
    newNode := &Node{Value: value}

    first := list.Head.Next // Первый реальный узел
    list.Head.Next = newNode
    newNode.Prev = list.Head
    newNode.Next = first
    first.Prev = newNode
}

// 🔹 Удаление узла по значению
func (list *LinkedList) Remove(value int) {
    for current := list.Head.Next; current != list.Tail; current = current.Next {
        if current.Value == value {
            current.Prev.Next = current.Next
            current.Next.Prev = current.Prev
            return
        }
    }
}

// 🔹 Переворот списка
func (list *LinkedList) Reverse() {
    current := list.Head
    for current != nil {
        current.Next, current.Prev = current.Prev, current.Next
        current = current.Prev // Теперь движение идет в обратную сторону
    }
    list.Head, list.Tail = list.Tail, list.Head
}

// 🔹 Вывод списка
func (list *LinkedList) Print() {
    for current := list.Head.Next; current != list.Tail; current = current.Next {
        fmt.Print(current.Value, " ")
    }
    fmt.Println()
}

// 🔹 Тестирование
func main() {
    list := NewLinkedList()

    list.AddToTail(1)
    list.AddToTail(2)
    list.AddToTail(3)
    list.Print() // 1 2 3

    list.AddToHead(0)
    list.Print() // 0 1 2 3

    list.Remove(2)
    list.Print() // 0 1 3

    list.Reverse()
    list.Print() // 3 1 0
}
```

---

## 🔥 **Как работают фиктивные `head` и `tail`?**

1. **`head` и `tail` существуют всегда**
    
    - `head.Next` указывает на первый реальный элемент.
        
    - `tail.Prev` указывает на последний реальный элемент.
        
2. **Добавление элементов**
    
    - В конец: `tail.Prev = newNode`, `newNode.Next = tail`.
        
    - В начало: `head.Next = newNode`, `newNode.Prev = head`.
        
3. **Удаление узла**
    
    - Просто меняем `Next` и `Prev` у соседних элементов.
        
4. **Переворот списка**
    
    - Переставляем `Next` и `Prev` у каждого узла.
        
    - Меняем `head` и `tail` местами.
        

---

## 🔥 **Плюсы такого подхода**

✅ **Нет проверки на `nil`** → Упрощает код  
✅ **Гарантия наличия `head` и `tail`** → Не сломаем список случайно  
✅ **Эффективность** → Операции `O(1)`, кроме `Remove`, который `O(n)`

🔹 **Такой подход используют в стандартных реализациях двусвязных списков!** 🚀

Это без фейковых головы и хвоста
### 🔹 Принцип работы с двусвязным и односвязным списками

Все операции с **связными списками** сводятся к **перестановке указателей**.  
Главное правило: **сначала меняем указатели, потом удаляем/добавляем узел**.

---

# 📌 **1. Основные операции с двусвязным списком**

Каждый узел двусвязного списка содержит:

```go
type Node struct {
    Value int
    Prev  *Node
    Next  *Node
}
```

---

## ✅ **Добавление узла в начало**

1. Создаём новый узел.
    
2. Меняем указатели `Next` у нового узла.
    
3. Меняем `Prev` у старого `head`.
    
4. Обновляем `head`.
    

```go
func addToHead(head **Node, value int) {
    newNode := &Node{Value: value, Next: *head}
    if *head != nil {
        (*head).Prev = newNode
    }
    *head = newNode
}
```

---

## ✅ **Добавление узла в конец**

1. Двигаемся до последнего узла.
    
2. Меняем `Next` у последнего узла.
    
3. Меняем `Prev` у нового узла.
    

```go
func addToTail(head **Node, value int) { // Двойной указатель на head
    if *head == nil {
        *head = &Node{Value: value}
        return
    
    }
    tail := head
    for tail.Next != nil {
        tail = tail.Next
    }
    newNode := &Node{Value: value, Prev: tail}
    tail.Next = newNode
}
```

---

## ✅ **Удаление узла**

1. Перенастраиваем указатели `Prev` и `Next` у соседних узлов.
    
2. Освобождаем память (в Go GC сам это делает).
    

```go
func deleteNode(head **Node, target *Node) {
    if target == nil {
        return
    }
    if target.Prev != nil {
        target.Prev.Next = target.Next
    } else {
        *head = target.Next
    }
    if target.Next != nil {
        target.Next.Prev = target.Prev
    }
}
```

---

## ✅ **Разворот двусвязного списка**

1. Проходим по списку и меняем `Next` ↔ `Prev` для каждого узла.
    
2. В конце `head` становится `tail`.
    

```go
func reverseDoublyLinkedList(head **Node) {
    var temp *Node
    current := *head
    for current != nil {
        temp = current.Prev
        current.Prev = current.Next
        current.Next = temp
        current = current.Prev // идем дальше по новому Next
    }
    if temp != nil {
        *head = temp.Prev
    }
}
```

---

# 📌 **2. Основные операции с односвязным списком**

Каждый узел содержит только `Next`:

```go
type Node struct {
    Value int
    Next  *Node
}
```

---

## ✅ **Добавление узла в начало**

```go
func addToHead(head **Node, value int) {
    newNode := &Node{Value: value, Next: *head}
    *head = newNode
}
```

---

## ✅ **Добавление узла в конец**

```go
func addToTail(head *Node, value int) {
    if head == nil {
        head = &Node{Value: value}
        return
    }
    tail := head
    for tail.Next != nil {
        tail = tail.Next
    }
    tail.Next = &Node{Value: value}
}
```

---

## ✅ **Удаление узла**

1. Ищем узел.
    
2. Перенастраиваем `Next` предыдущего узла.
    

```go
func deleteNode(head **Node, value int) {
    if *head == nil {
        return
    }
    if (*head).Value == value {
        *head = (*head).Next
        return
    }
    prev := *head
    for prev.Next != nil && prev.Next.Value != value {
        prev = prev.Next
    }
    if prev.Next != nil {
        prev.Next = prev.Next.Next
    }
}
```

---

## ✅ **Разворот односвязного списка**

1. Проходим по списку.
    
2. Меняем `Next` на предыдущий узел.
    

```go
func reverseSinglyLinkedList(head **Node) {
    var prev, next *Node
    current := *head
    for current != nil {
        next = current.Next
        current.Next = prev
        prev = current
        current = next
    }
    *head = prev
}
```

---

### 🔹 **Вывод**

Все операции сводятся к **изменению указателей**.

- **Двусвязный список** даёт быстрый доступ назад (`Prev`), но требует **двойного обновления указателей**.
    
- **Односвязный список** проще, но **нельзя двигаться назад**.
    
- **Разворот** — всегда **меняем `Next` на предыдущее значение**.
    

Какие моменты остались неясны? 😊