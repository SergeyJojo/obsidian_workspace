# Функция Zip для слайсов int на Go

Вот упрощенная версия функции `Zip` только для слайсов типа `int`:

```go
package main

import "fmt"

// ZipInt объединяет несколько слайсов int в один слайс слайсов
// Длина результата равна длине самого короткого слайса
func ZipInt(slices ...[]int) [][]int {
	if len(slices) == 0 {
		return nil
	}

	// Находим минимальную длину
	minLength := len(slices[0])
	for _, slice := range slices[1:] {
		if len(slice) < minLength {
			minLength = len(slice)
		}
	}

	// Создаем результат
	result := make([][]int, minLength)
	for i := 0; i < minLength; i++ {
		tuple := make([]int, len(slices))
		for j, slice := range slices {
			tuple[j] = slice[i]
		}
		result[i] = tuple
	}

	return result
}

func main() {
	// Пример использования
	a := []int{1, 2, 3, 4}
	b := []int{10, 20, 30}
	c := []int{100, 200, 300, 400}

	zipped := ZipInt(a, b, c)

	for _, tuple := range zipped {
		fmt.Println(tuple)
	}
}
```

### Особенности этой реализации:
1. Работает только со слайсами `int` (без использования `interface{}` и reflection)
2. Более эффективна, так как не использует reflection
3. Проще и типобезопасна
4. Возвращает `[][]int` вместо `[][]interface{}`

### Пример вывода:
```
[1 10 100]
[2 20 200]
[3 30 300]
```

Если вам нужно работать с другими типами, можно либо создать аналогичные функции для каждого типа (`ZipString`, `ZipFloat64` и т.д.), либо использовать generics из Go 1.18+.