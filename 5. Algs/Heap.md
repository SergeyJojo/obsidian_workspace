# Реализация кучи (heap) на Go

В Go нет встроенной реализации кучи, но её можно легко реализовать с использованием интерфейсов из пакета `container/heap`. Вот полная реализация минимальной и максимальной кучи:

```go
package main

import (
	"container/heap"
	"fmt"
)

// MinHeap реализация
type MinHeap []int

func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *MinHeap) Push(x interface{}) {
	*h = append(*h, x.(int))
}

func (h *MinHeap) Pop() interface{} {
	old := *h
	n := len(old)
	x := old[n-1]
	*h = old[0 : n-1]
	return x
}

// MaxHeap реализация
type MaxHeap []int

func (h MaxHeap) Len() int           { return len(h) }
func (h MaxHeap) Less(i, j int) bool { return h[i] > h[j] }
func (h MaxHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *MaxHeap) Push(x interface{}) {
	*h = append(*h, x.(int))
}

func (h *MaxHeap) Pop() interface{} {
	old := *h
	n := len(old)
	x := old[n-1]
	*h = old[0 : n-1]
	return x
}

func main() {
	// Пример использования минимальной кучи
	minH := &MinHeap{2, 1, 5}
	heap.Init(minH)
	heap.Push(minH, 3)
	fmt.Printf("Минимальный элемент: %d\n", (*minH)[0])
	for minH.Len() > 0 {
		fmt.Printf("%d ", heap.Pop(minH))
	}
	fmt.Println()

	// Пример использования максимальной кучи
	maxH := &MaxHeap{2, 1, 5}
	heap.Init(maxH)
	heap.Push(maxH, 3)
	fmt.Printf("Максимальный элемент: %d\n", (*maxH)[0])
	for maxH.Len() > 0 {
		fmt.Printf("%d ", heap.Pop(maxH))
	}
	fmt.Println()
}
```

## Как это работает:

1. Мы определяем два типа: `MinHeap` и `MaxHeap`, которые являются срезами целых чисел.
2. Для каждого типа реализуем 5 методов, требуемых интерфейсом `heap.Interface`:
   - `Len()` - возвращает количество элементов
   - `Less(i, j int)` - определяет порядок элементов (для min heap i < j, для max heap i > j)
   - `Swap(i, j int)` - меняет элементы местами
   - `Push(x interface{})` - добавляет элемент в конец среза
   - `Pop() interface{}` - удаляет и возвращает последний элемент

3. Пакет `container/heap` предоставляет функции `Init`, `Push` и `Pop`, которые поддерживают свойства кучи.

## Пример вывода программы:
```
Минимальный элемент: 1
1 2 3 5 
Максимальный элемент: 5
5 3 2 1 
```

Вы можете использовать эту реализацию как основу для более сложных структур данных или алгоритмов, требующих кучи.