
### Функция `Zip` для слайсов `[]int` разной длины на Go

Вот простая реализация функции `Zip` для слайсов целых чисел (`[]int`), которая работает со слайсами разной длины:

```go
// ZipInts объединяет несколько слайсов int в слайс кортежей
// Останавливается на самом коротком слайсе
func ZipInts(slices ...[]int) [][]int {
	if len(slices) == 0 {
		return nil
	}

	// Находим минимальную длину
	minLen := len(slices[0])
	for _, s := range slices[1:] {
		if len(s) < minLen {
			minLen = len(s)
		}
	}

	// Создаем результат
	result := make([][]int, minLen)
	for i := 0; i < minLen; i++ {
		tuple := make([]int, len(slices))
		for j, s := range slices {
			tuple[j] = s[i]
		}
		result[i] = tuple
	}

	return result
}
```

### Пример использования:
```go
func main() {
	a := []int{1, 2, 3, 4}
	b := []int{10, 20, 30}
	c := []int{100, 200, 300, 400, 500}

	zipped := ZipInts(a, b, c)
	fmt.Println(zipped)
	// Output: [[1 10 100] [2 20 200] [3 30 300]]
}
```

### Особенности реализации:
1. **Простота** - работает только с `[]int`
2. **Разные длины** - берется по минимальной длине слайса
3. **Эффективность** - один проход для нахождения minLen, затем сборка результата

### Альтернативная версия (с заполнением нулями):
Если нужно, чтобы все слайсы обрабатывались до максимальной длины (заполняя недостающие элементы нулями):

```go
func ZipIntsFill(slices ...[]int) [][]int {
	if len(slices) == 0 {
		return nil
	}

	// Находим максимальную длину
	maxLen := 0
	for _, s := range slices {
		if len(s) > maxLen {
			maxLen = len(s)
		}
	}

	// Создаем результат
	result := make([][]int, maxLen)
	for i := 0; i < maxLen; i++ {
		tuple := make([]int, len(slices))
		for j, s := range slices {
			if i < len(s) {
				tuple[j] = s[i]
			} else {
				tuple[j] = 0 // заполняем нулями
			}
		}
		result[i] = tuple
	}

	return result
}
```

Пример использования:
```go
func main() {
	a := []int{1, 2}
	b := []int{10, 20, 30}
	
	zipped := ZipIntsFill(a, b)
	fmt.Println(zipped)
	// Output: [[1 10] [2 20] [0 30]]
}
```

Выберите подходящий вариант в зависимости от ваших требований к обработке слайсов разной длины.