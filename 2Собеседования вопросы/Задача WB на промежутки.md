


```go
package main

import (
	"fmt"
	"sort"
)

// Функция для разделения отрезка на слайсы с учетом разделителей
func splitInterval(min, max int, separators []int) [][]int {
	// Сортируем разделители и добавляем min-1 и max+1 для корректного деления диапазона
	sortedSeparators := append(separators, min-1)
	sortedSeparators = append(sortedSeparators, max+1)
	sort.Ints(sortedSeparators)

	var result [][]int
	// Проходим по разделителям и создаем отрезки
	for i := 0; i < len(sortedSeparators)-1; i++ {
		start := sortedSeparators[i] + 1  // Начало отрезка
		end := sortedSeparators[i+1] - 1  // Конец отрезка

		// Если разница больше 0, значит есть валидный отрезок
		// и он попадает в диапазон от min до max
		if start <= end && start >= min && end <= max {
			result = append(result, []int{start, end})
		}
	}
	return result
}

func main() {
	// Диапазон для разделения
	min := 30000
	max := 32000
	separators := []int{30100, 30200, 20000, 200}

	// Получаем результат
	result := splitInterval(min, max, separators)

	// Выводим результат
	for _, segment := range result {
		fmt.Println(segment)
	}
}
```

