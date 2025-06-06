[[Mutex]]

Пакет **`sync`** в Go предоставляет примитивы синхронизации для работы с конкурентными программами. Он широко используется для управления доступом к общим ресурсам, синхронизации горутин и решения проблем конкурентности, таких как гонки данных.

---

## **Основные структуры и функции пакета `sync`**

### **1. `sync.Mutex` (Мьютекс)**

Мьютекс (mutual exclusion) используется для защиты критических секций кода, чтобы только одна горутина могла получить доступ к общему ресурсу в данный момент.

#### **Основные методы:**

- **`Lock()`**: Захватывает мьютекс. Если мьютекс уже захвачен, горутина блокируется, пока мьютекс не освободится.
- **`Unlock()`**: Освобождает мьютекс.

#### **Пример:**

```go
package main

import (
	"fmt"
	"sync"
)

var counter int
var mu sync.Mutex

func increment(wg *sync.WaitGroup) {
	defer wg.Done()
	mu.Lock()
	defer mu.Unlock()
	counter++
}

func main() {
	var wg sync.WaitGroup

	for i := 0; i < 10; i++ {
		wg.Add(1)
		go increment(&wg)
	}

	wg.Wait()
	fmt.Println("Final counter:", counter)
}
```

---

### **2. `sync.RWMutex` (Читающий/пишущий мьютекс)**

Позволяет нескольким горутинам одновременно читать данные, но запрещает доступ к данным, если идёт запись.

#### **Основные методы:**

- **`RLock()` и `RUnlock()`**: Блокирует мьютекс для чтения.
- **`Lock()` и `Unlock()`**: Блокирует мьютекс для записи.

#### **Пример:**

```go
package main

import (
	"fmt"
	"sync"
)

var data = make(map[string]string)
var mu sync.RWMutex

func readData(key string, wg *sync.WaitGroup) {
	defer wg.Done()
	mu.RLock()
	defer mu.RUnlock()
	fmt.Println("Reading:", key, "=", data[key])
}

func writeData(key, value string, wg *sync.WaitGroup) {
	defer wg.Done()
	mu.Lock()
	defer mu.Unlock()
	data[key] = value
	fmt.Println("Written:", key, "=", value)
}

func main() {
	var wg sync.WaitGroup

	wg.Add(3)
	go writeData("name", "Alice", &wg)
	go readData("name", &wg)
	go readData("name", &wg)

	wg.Wait()
}
```

---

### **3. `sync.WaitGroup`**

Используется для ожидания завершения группы горутин. Очень полезен для синхронизации горутин.

#### **Основные методы:**

- **`Add(delta int)`**: Увеличивает счётчик ожидания на `delta`.
- **`Done()`**: Уменьшает счётчик на 1.
- **`Wait()`**: Блокирует выполнение до тех пор, пока счётчик не станет равным 0.

#### **Пример:**

```go
package main

import (
	"fmt"
	"sync"
)

func worker(id int, wg *sync.WaitGroup) {
	defer wg.Done()
	fmt.Printf("Worker %d started\n", id)
	// Имитация работы
	fmt.Printf("Worker %d finished\n", id)
}

func main() {
	var wg sync.WaitGroup

	for i := 1; i <= 5; i++ {
		wg.Add(1)
		go worker(i, &wg)
	}

	wg.Wait()
	fmt.Println("All workers are done")
}
```

---

### **4. `sync.Once`**

Гарантирует, что определённый код выполнится только **один раз**, даже если он вызывается из нескольких горутин.

#### **Основные методы:**

- **`Do(f func())`**: Выполняет функцию `f` только один раз.

#### **Пример:**

```go
package main

import (
	"fmt"
	"sync"
)

var once sync.Once

func initialize() {
	fmt.Println("Initialization")
}

func main() {
	for i := 0; i < 3; i++ {
		go once.Do(initialize)
	}
}
```

**Вывод:** Функция `initialize` выполнится только один раз, независимо от количества вызовов.

---

### **5. `sync.Cond`**

Используется для уведомления одной или нескольких горутин о том, что произошло какое-то событие. Полезно для реализации сложных сценариев синхронизации.

#### **Основные методы:**

- **`Wait()`**: Ожидает, пока будет вызван метод `Signal()` или `Broadcast()`.
- **`Signal()`**: Пробуждает одну горутину, ожидающую условие.
- **`Broadcast()`**: Пробуждает всех, кто ждёт условие.

#### **Пример:**

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var mu sync.Mutex
	cond := sync.NewCond(&mu)
	queue := []int{}

	// Consumer
	go func() {
		mu.Lock()
		for len(queue) == 0 {
			cond.Wait()
		}
		fmt.Println("Consumed:", queue[0])
		queue = queue[1:]
		mu.Unlock()
	}()

	// Producer
	go func() {
		mu.Lock()
		queue = append(queue, 42)
		fmt.Println("Produced: 42")
		cond.Signal()
		mu.Unlock()
	}()

	// Даем время горутинам завершиться
	select {}
}
```

---

### **6. `sync.Pool`**

Используется для повторного использования объектов и минимизации аллокаций.

#### **Основные методы:**

- **`Get()`**: Получает объект из пула. Если пула нет, вызывает функцию `New`.
- **`Put(x interface{})`**: Возвращает объект в пул.

#### **Пример:**

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	pool := sync.Pool{
		New: func() interface{} {
			return "default value"
		},
	}

	// Извлечение объекта
	fmt.Println(pool.Get()) // "default value"

	// Возврат объекта
	pool.Put("custom value")
	fmt.Println(pool.Get()) // "custom value"
}
```

---

## **Когда использовать `sync`**

- **`sync.Mutex` / `sync.RWMutex`**: Для защиты критических секций, где доступ к общим данным должен быть синхронизирован.
- **`sync.WaitGroup`**: Для синхронизации завершения горутин.
- **`sync.Once`**: Для выполнения кода (например, инициализации) только один раз.
- **`sync.Cond`**: Для сигнализации между горутинами.
- **`sync.Pool`**: Для повышения производительности за счёт уменьшения частоты аллокаций.

---

## **Плюсы использования `sync`**

1. Высокая производительность: Эти примитивы синхронизации работают быстро за счёт низкоуровневых оптимизаций.
2. Простота использования: Предоставляют стандартные интерфейсы для синхронизации без необходимости изобретать свои решения.
3. Минимизация ошибок: Использование стандартных примитивов снижает вероятность ошибок в конкурентных программах.

---

## **Минусы**

1. Сложность отладки: Неправильное использование мьютексов или других примитивов может привести к взаимным блокировкам (deadlocks).
2. Ограниченность: Для более сложных сценариев лучше использовать каналы.
3. Риск забыть вызов `Unlock()` или `Done()`.

---

## **Вывод**

Пакет `sync` в Go предоставляет надёжные и производительные примитивы для синхронизации. На собеседовании важно уметь объяснить и продемонстрировать использование ключевых компонентов пакета (`Mutex`, `WaitGroup`, `Once`) и рассказать, когда их применять, а также в чём их преимущества и ограничения.