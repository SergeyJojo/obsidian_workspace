
### Решение на Go (Golang) для задачи "Summary Ranges"

Вот эффективное решение на Go, которое преобразует отсортированный массив чисел в список диапазонов:

```go
package main

import (
	"fmt"
	"strconv"
)

func summaryRanges(nums []int) []string {
	if len(nums) == 0 {
		return []string{}
	}

	var result []string
	start := nums[0]

	for i := 1; i < len(nums); i++ {
		if nums[i] != nums[i-1]+1 {
			if start == nums[i-1] {
				result = append(result, strconv.Itoa(start))
			} else {
				result = append(result, strconv.Itoa(start)+"->"+strconv.Itoa(nums[i-1]))
			}
			start = nums[i]
		}
	}

	// Добавляем последний диапазон
	if start == nums[len(nums)-1] {
		result = append(result, strconv.Itoa(start))
	} else {
		result = append(result, strconv.Itoa(start)+"->"+strconv.Itoa(nums[len(nums)-1]))
	}

	return result
}

func main() {
	nums := []int{1, 2, 3, 5, 8, 9, 10}
	fmt.Println(summaryRanges(nums)) // Output: [1->3 5 8->10]
}
```

### Ключевые моменты решения:

1. **Проверка пустого массива** в начале
2. **Инициализация переменных**:
   - `result` для хранения итоговых строк
   - `start` для отслеживания начала текущего диапазона
3. **Основной цикл**:
   - Сравниваем текущий элемент с предыдущим
   - Если разрыв в последовательности, добавляем диапазон или одиночное число
4. **Обработка последнего диапазона** после цикла
5. **Преобразование чисел в строки** с помощью `strconv.Itoa`

### Альтернативная версия (более компактная):

```go
func summaryRanges(nums []int) []string {
    var res []string
    for i, j := 0, 0; j < len(nums); j++ {
        if j+1 < len(nums) && nums[j+1] == nums[j]+1 {
            continue
        }
        if i == j {
            res = append(res, strconv.Itoa(nums[i]))
        } else {
            res = append(res, strconv.Itoa(nums[i])+"->"+strconv.Itoa(nums[j]))
        }
        i = j + 1
    }
    return res
}
```

### Тестирование:
```go
func main() {
    tests := [][]int{
        {1, 2, 3, 5, 8, 9, 10},
        {0, 1, 2, 4, 5, 7},
        {},
        {1},
        {1, 3, 5, 7},
    }
    
    for _, nums := range tests {
        fmt.Printf("%v -> %v\n", nums, summaryRanges(nums))
    }
}
```

Вывод:
```
[1 2 3 5 8 9 10] -> [1->3 5 8->10]
[0 1 2 4 5 7] -> [0->2 4->5 7]
[] -> []
[1] -> [1]
[1 3 5 7] -> [1 3 5 7]
```

Оба решения работают за O(n) времени и используют O(1) дополнительной памяти (не считая результата). Первое решение более явное и простое для понимания, второе - более компактное.