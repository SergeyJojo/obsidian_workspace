
В языке Go работа со временем осуществляется через пакет **`time`**, который входит в стандартную библиотеку. Этот пакет предоставляет множество функций и структур для работы с датами и временем.

---

### Основные структуры и функции

1. **`time.Time`** — структура для представления времени.
2. **`time.Duration`** — структура для представления промежутка времени (например, 1 час, 30 минут).
3. **Основные функции**:
   - `time.Now()` — возвращает текущее время.
   - `time.Parse()` — парсит строку в объект времени.
   - `time.Format()` — форматирует объект времени в строку.
   - `time.Add()` — добавляет промежуток времени.
   - `time.Sub()` — вычисляет разницу между двумя временными отметками.

---

### Примеры работы

#### 1. Получение текущего времени
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	now := time.Now() // Текущее время
	fmt.Println("Current time:", now)
}
```

---

#### 2. Форматирование времени
Go использует специфичный метод для форматирования времени. В качестве шаблона используется фиксированная дата: `Mon Jan 2 15:04:05 MST 2006`. Эти значения заменяются на реальные при форматировании.

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	now := time.Now()

	// Форматирование в строку
	fmt.Println("Default format:", now.String())
	fmt.Println("Custom format:", now.Format("02-01-2006 15:04:05"))

	// Форматирование только даты или времени
	fmt.Println("Date only:", now.Format("02-01-2006"))
	fmt.Println("Time only:", now.Format("15:04:05"))
}
```

---

#### 3. Парсинг строки в дату
Если у вас есть строка, представляющая дату и время, вы можете преобразовать её в объект `time.Time`.

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	layout := "02-01-2006 15:04:05" // Шаблон для парсинга
	dateStr := "13-04-2025 15:44:30"

	parsedTime, err := time.Parse(layout, dateStr)
	if err != nil {
		fmt.Println("Error parsing date:", err)
		return
	}

	fmt.Println("Parsed time:", parsedTime)
}
```

---

#### 4. Вычисление разницы между датами
Для нахождения разницы между двумя временными объектами используется метод `Sub`.

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	start := time.Date(2025, 4, 13, 12, 0, 0, 0, time.UTC)
	end := time.Now()

	duration := end.Sub(start) // Разница между временем
	fmt.Println("Duration:", duration)

	// Преобразование в часы, минуты, секунды
	fmt.Println("Hours:", duration.Hours())
	fmt.Println("Minutes:", duration.Minutes())
	fmt.Println("Seconds:", duration.Seconds())
}
```

---

#### 5. Добавление и вычитание времени
Вы можете добавлять или вычитать промежутки времени из объекта `time.Time` с помощью метода `Add`.

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	now := time.Now()

	// Добавляем 1 день
	tomorrow := now.Add(24 * time.Hour)
	fmt.Println("Tomorrow:", tomorrow)

	// Вычитаем 2 часа
	twoHoursAgo := now.Add(-2 * time.Hour)
	fmt.Println("Two hours ago:", twoHoursAgo)
}
```

---

#### 6. Таймеры и задержки
Для установки задержек используются функции `time.Sleep` и структуры `time.Timer` и `time.Ticker`.

**Пример с `time.Sleep`:**
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	fmt.Println("Start")
	time.Sleep(2 * time.Second) // Задержка 2 секунды
	fmt.Println("End")
}
```

**Пример с `time.Ticker`:**
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ticker := time.NewTicker(1 * time.Second)
	defer ticker.Stop()

	for i := 0; i < 5; i++ {
		<-ticker.C
		fmt.Println("Tick", i+1)
	}
}
```

---

#### 7. Тайм-ауты и работа с каналами
Для создания тайм-аутов удобно использовать `time.After`.

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	fmt.Println("Waiting for 3 seconds...")

	select {
	case <-time.After(3 * time.Second):
		fmt.Println("Timeout reached!")
	}
}
```

---

#### 8. Работа с часовыми поясами
Go позволяет учитывать часовые пояса с помощью объекта `time.Location`.

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	location, err := time.LoadLocation("America/New_York")
	if err != nil {
		fmt.Println("Error loading location:", err)
		return
	}

	now := time.Now()
	fmt.Println("Local time:", now)

	newYorkTime := now.In(location)
	fmt.Println("New York time:", newYorkTime)
}
```

---

#### 9. Сравнение времени
Для сравнения временных отметок используются методы `Before`, `After` и `Equal`.

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	t1 := time.Date(2025, 4, 13, 10, 0, 0, 0, time.UTC)
	t2 := time.Now()

	fmt.Println("t1 is before t2:", t1.Before(t2))
	fmt.Println("t1 is after t2:", t1.After(t2))
	fmt.Println("t1 is equal to t2:", t1.Equal(t2))
}
```

---

### Полезные функции и советы
1. **Текущая метка времени (Unix Timestamp):**
   ```go
   now := time.Now()
   fmt.Println("Unix:", now.Unix())       // Секунды с 1970
   fmt.Println("UnixNano:", now.UnixNano()) // Наносекунды с 1970
   ```

2. **Создание `time.Duration`:**
   ```go
   duration := 2 * time.Hour + 30 * time.Minute
   fmt.Println("Duration:", duration)
   ```

3. **Конвертация `time.Duration` в другие единицы:**
   ```go
   d := 90 * time.Minute
   fmt.Println("Hours:", d.Hours())
   fmt.Println("Minutes:", d.Minutes())
   fmt.Println("Seconds:", d.Seconds())
   ```

4. **Фиксированный интервал времени (`time.Tick`):**
   ```go
   for range time.Tick(1 * time.Second) {
       fmt.Println("Tick")
   }
   ```

---

### Итог
Пакет `time` предоставляет мощные инструменты для работы с временем, такими как:
- Работа с текущим временем (`Now`, `Unix`).
- Форматирование (`Format`) и парсинг (`Parse`).
- Вычисление разницы времени (`Sub`).
- Таймеры и интервалы (`Sleep`, `Ticker`).
- Работа с часовыми поясами (`LoadLocation`, `In`).

Эти функции делают работу со временем в Go простой и эффективной.