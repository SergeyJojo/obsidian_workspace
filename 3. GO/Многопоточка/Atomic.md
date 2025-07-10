**Атомики** в Go предоставляют способ выполнения низкоуровневых операций с переменными, которые являются атомарными (непрерываемыми). Они обеспечивают безопасное обновление переменных в конкурентной среде без использования мьютексов, что делает их полезными для повышения производительности в высоконагруженных системах.

В Go атомики реализованы в пакете **`sync/atomic`**.

---

## **Основные операции с атомиками**

### **1. Чтение и запись атомарных значений**

- **`atomic.Load`**: Атомарное чтение значения.
- **`atomic.Store`**: Атомарная запись значения.

Пример:

```go
package main

import (
	"fmt"
	"sync/atomic"
)

func main() {
	var counter int32 = 10

	// Чтение
	value := atomic.LoadInt32(&counter)
	fmt.Println("Initial value:", value)

	// Запись
	atomic.StoreInt32(&counter, 20)
	fmt.Println("Updated value:", atomic.LoadInt32(&counter))
}
```

---

### **2. Атомарное добавление и вычитание**

- **`atomic.AddInt32`**: Атомарно увеличивает или уменьшает значение.
- Поддерживаются типы: `Int32`, `Int64`, `Uint32`, `Uint64`.

Пример:

```go
package main

import (
	"fmt"
	"sync/atomic"
)

func main() {
	var counter int32 = 0

	// Увеличиваем значение
	atomic.AddInt32(&counter, 5)
	fmt.Println("After increment:", counter)

	// Уменьшаем значение
	atomic.AddInt32(&counter, -2)
	fmt.Println("After decrement:", counter)
}
```

---

### **3. Сравнение и замена (`CompareAndSwap`)**

- **`atomic.CompareAndSwap`**: Сравнивает значение с ожидаемым и, если они равны, заменяет значение на новое.
- Возвращает `true`, если операция успешна, и `false`, если нет.

Пример:

```go
package main

import (
	"fmt"
	"sync/atomic"
)

func main() {
	var counter int32 = 10

	// Успешная замена
	swapped := atomic.CompareAndSwapInt32(&counter, 10, 20)
	fmt.Println("Swap success:", swapped, "New value:", counter)

	// Неудачная замена
	swapped = atomic.CompareAndSwapInt32(&counter, 10, 30)
	fmt.Println("Swap success:", swapped, "Value remains:", counter)
}
```

---

### **4. Загрузка и замена (`Swap`)**

- **`atomic.Swap`**: Атомарно заменяет значение и возвращает старое.
- Поддерживает типы: `Int32`, `Int64`, `Uint32`, `Uint64`, `Pointer`.

Пример:

```go
package main

import (
	"fmt"
	"sync/atomic"
)

func main() {
	var counter int32 = 10

	// Замена значения
	oldValue := atomic.SwapInt32(&counter, 30)
	fmt.Println("Old value:", oldValue, "New value:", counter)
}
```

---

### **5. Работа с указателями**

Пакет `sync/atomic` также предоставляет функции для работы с указателями:

- **`atomic.LoadPointer`** и **`atomic.StorePointer`**: Чтение и запись указателей.
- **`atomic.SwapPointer`** и **`atomic.CompareAndSwapPointer`**: Замена и сравнение указателей.

Пример:

```go
package main

import (
	"fmt"
	"sync/atomic"
	"unsafe"
)

func main() {
	var ptr unsafe.Pointer

	// Записываем указатель
	data := "Hello, Atomic!"
	atomic.StorePointer(&ptr, unsafe.Pointer(&data))

	// Читаем указатель
	loaded := atomic.LoadPointer(&ptr)
	fmt.Println("Loaded value:", *(*string)(loaded))
}
```

---

## **Пример использования атомиков в многопоточном приложении**

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var counter int32 = 0
	var wg sync.WaitGroup

	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			atomic.AddInt32(&counter, 1)
		}()
	}

	wg.Wait()
	fmt.Println("Final counter value:", counter) // Ожидается 100
}
```

---

## **Когда использовать атомики**

1. **Лёгкая альтернатива мьютексам:**
    
    - Если требуется простое обновление чисел или указателей, атомики обеспечивают более высокую производительность, чем `sync.Mutex`.
2. **Подходят для счётчиков:**
    
    - Увеличение/уменьшение значений, например, числа активных соединений или выполненных задач.
3. **Гарантия целостности данных:**
    
    - Обеспечивают атомарность операций, что предотвращает состояние гонки.

---

## **Ограничения атомиков**

1. **Не заменяют мьютексы в сложных сценариях:**
    
    - Атомики работают только с единичными значениями. Для более сложных структур лучше использовать мьютексы.
2. **Сложности в чтении/понимании кода:**
    
    - Код с атомиками менее очевиден, чем с использованием мьютексов.
3. **Ограниченный набор типов:**
    
    - Поддерживаются только базовые типы (`int32`, `int64`, `uint32`, `uint64`, `uintptr`, `Pointer`).

---

## **Итог**

Атомики в Go — это мощный инструмент для работы с конкурентными программами, обеспечивающий высокую производительность в простых сценариях. Однако их следует использовать только там, где требуется атомарное обновление одиночных переменных, и избегать их для управления сложными структурами данных, где лучше подойдут мьютексы или другие синхронизационные примитивы.