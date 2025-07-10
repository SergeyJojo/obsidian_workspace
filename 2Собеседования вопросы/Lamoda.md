Вот отформатированный и исправленный код с правильным индексом и обработкой ошибок:

```go
package main

import (
	"fmt"
	"sync"
)

type listSKU []string

// Метод для получения последнего SKU
func (l listSKU) getLastSKU() string {
	// Проверка на пустой слайс
	if len(l) == 0 {
		panic("listSKU is empty") // Паника, если слайс пуст
	}
	return l[len(l)-1] // Индекс последнего элемента слайса
}

func main() {
	// Инициализация слайса с SKU
	items := listSKU{
		"MP990099991",
		"MP900000002",
		"MP000000003",
		"MP000000004",
		"MP000000005",
	}

	// Создание и ожидание горутины
	wg := new(sync.WaitGroup)
	wg.Add(1)

	// Запуск горутины для получения последнего SKU
	go func() {
		defer func() {
			// Обработка возможной паники
			if r := recover(); r != nil {
				fmt.Println("Recovered from panic:", r)
			}
		}()

		// Получение последнего SKU
		lastItem := items.getLastSKU()
		fmt.Printf("Last SKU is: %s\n", lastItem)

		wg.Done()
	}()

	// Ожидание завершения горутины
	wg.Wait()
	fmt.Println("Program completed.")
}
```

### Пояснение изменений:

1. **Проверка на пустой слайс**: В методе `getLastSKU()` добавлена проверка на пустой слайс, чтобы избежать паники.
    
2. **Использование правильного индекса**: В методе `getLastSKU()` исправлен индекс, чтобы получить последний элемент слайса с помощью `len(l)-1`.
    
3. **Обработка паники с `defer` и `recover`**: В горутине добавлена конструкция `defer` с `recover()`, чтобы обработать паники и избежать аварийного завершения программы.
    
4. **Форматирование**: Код отформатирован для лучшей читаемости, с правильным отступом и структурой.
    

### Пример вывода:

```
Last SKU is: MP000000005
Program completed.
```

Теперь код обрабатывает паники и работает корректно, не вызывая аварийных завершений.

В вашем коде возникает паника, потому что в методе `getLastSKU()` вы пытаетесь обратиться к индексу `len(l)`, который выходит за пределы слайса. В Go индексы слайсов начинаются с 0, и последний элемент слайса имеет индекс `len(l)-1`. Использование индекса `len(l)` вызывает панику, потому что этого индекса не существует.

Чтобы избежать паники, необходимо поправить индекс и использовать `len(l)-1` для получения последнего элемента слайса.

Кроме того, чтобы обработать панику в горутине и не остановить выполнение программы, мы можем использовать механизм **`defer`** и **`recover`** для перехвата ошибки.

### Исправленный код с обработкой паники:

```go
package main

import (
	"fmt"
	"sync"
)

type listSKU []string

func (l listSKU) getLastSKU() string {
	// Используем правильный индекс
	if len(l) == 0 {
		panic("listSKU is empty") // Если список пустой, вызываем панику
	}
	return l[len(l)-1] // Индекс последнего элемента
}

func main() {
	items := listSKU{
		"MP990099991",
		"MP900000002",
		"MP000000003",
		"MP000000004",
		"MP000000005",
	}

	wg := new(sync.WaitGroup)
	wg.Add(1)

	// Используем defer и recover для перехвата паники в горутине
	go func() {
		defer func() {
			if r := recover(); r != nil {
				fmt.Println("Recovered from panic:", r)
			}
		}()

		// Получаем последний SKU
		lastItem := items.getLastSKU()
		fmt.Printf("Last SKU is: %s\n", lastItem)

		wg.Done()
	}()

	wg.Wait()
	fmt.Println("Program completed.")
}
```

### Изменения:

1. **Исправление индекса**:
    
    - Метод `getLastSKU()` теперь использует индекс `len(l)-1`, что соответствует последнему элементу слайса.
2. **Обработка паники с `defer` и `recover`**:
    
    - В горутине мы добавили конструкцию `defer` с функцией `recover()`. Эта конструкция позволяет "поймать" панику и продолжить выполнение программы. Если в горутине возникает паника, программа не завершится аварийно, а вместо этого будет напечатано сообщение о восстановлении от паники.
3. **Проверка на пустой слайс**:
    
    - Мы добавили проверку на пустоту слайса внутри метода `getLastSKU()`, чтобы предотвратить панику, если слайс пустой.

### Пример вывода:

```
Last SKU is: MP000000005
Program completed.
```

Теперь программа безопасно обрабатывает возможные ошибки и продолжает выполнение, даже если что-то пойдет не так внутри горутины.



SQL

```sql
CREATE TABLE customer (
    id INTEGER PRIMARY KEY,
    email VARCHAR(100) NOT NULL,
    country CHAR(2) NOT NULL
);

CREATE TABLE cart_item (
    id INTEGER PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    title VARCHAR(20) NOT NULL,
    amount INTEGER NOT NULL,
    price INTEGER NOT NULL
);
```

```sql
# Вывести построчно всех кастомеров (id, email) и все элементы корзины пользователя (title, amount)
SELECT c.id, c.email, ci.title, ci.amount
FROM customer c
JOIN cart_item ci ON c.id = ci.customer_id;
```

```sql
# Вывести топ-10 клиентов (id, email) по общей стоимости товаров в корзине
SELECT c.id, c.email, SUM(ci.amount * ci.price) AS total_cost
FROM customer c
JOIN cart_item ci ON c.id = ci.customer_id
GROUP BY c.id, c.email
ORDER BY total_cost DESC
LIMIT 10;
```

```sql
# Интересуют только покупатели из России, которые положили в корзину товаров не менее чем на 1000 рублей
SELECT c.id, c.email, SUM(ci.amount * ci.price) AS total_cost
FROM customer c
JOIN cart_item ci ON c.id = ci.customer_id
WHERE c.country = 'RU'
GROUP BY c.id, c.email
HAVING total_cost >= 1000;
```

```sql
# Будет ли применяться индекс?
CREATE INDEX myIdx ON carts (sku, country, customer_id);
SELECT * FROM carts WHERE sku = 192 AND country = 'ru';
```

```sql
# А так?
CREATE INDEX myIdx ON carts (sku, country, customer_id);
SELECT * FROM carts WHERE country = 'ru' AND customer_id = 10;
```

Извините за путаницу. Давайте рассмотрим ваши запросы и ответы на них.

### 1. **Запрос с условиями `sku = 192 AND country = 'ru'`**:

```sql
CREATE INDEX myIdx ON carts (sku, country, customer_id);
SELECT * FROM carts WHERE sku = 192 AND country = 'ru';
```

**Ответ**: В этом запросе **индекс будет использован**. Индекс был создан по столбцам `sku`, `country` и `customer_id`, и запрос использует как `sku`, так и `country`, что позволяет базе данных эффективно использовать индекс для быстрого поиска.

### 2. **Запрос с условиями `country = 'ru' AND customer_id = 10`**:

```sql
CREATE INDEX myIdx ON carts (sku, country, customer_id);
SELECT * FROM carts WHERE country = 'ru' AND customer_id = 10;
```

**Ответ**: В этом запросе **индекс не будет использован эффективно**. Порядок столбцов в индексе важен. Индекс был создан по столбцам `sku`, `country` и `customer_id`, но запрос фильтрует только по `country` и `customer_id`. Так как в индексе первым идет `sku`, база данных может не использовать индекс полностью, что приведет к снижению эффективности запроса.

### Рекомендация:

Для второго запроса рекомендуется создать индекс, включающий только столбцы `country` и `customer_id`, так как это более подходит для вашего запроса.

```sql
CREATE INDEX myIdxCountryCustomer ON carts (country, customer_id);
```

Этот индекс будет использоваться эффективно для запроса, который фильтрует только по `country` и `customer_id`.

### Резюме:

1. В первом запросе индекс будет использоваться эффективно, так как все условия фильтрации соответствуют порядку столбцов в индексе.
2. Во втором запросе индекс не будет использоваться эффективно из-за порядка столбцов. Для улучшения производительности нужно создать индекс с правильным порядком столбцов `country` и `customer_id`.