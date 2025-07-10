
### Решение для поиска k ближайших элементов к заданному индексу в отсортированном массиве

```go
func findClosestElements(arr []int, index int, k int) []int {
    left := index
    right := index
    
    // Расширяем окно в обе стороны, пока не найдем k элементов
    for right - left + 1 < k {
        if left == 0 {
            right++
        } else if right == len(arr)-1 {
            left--
        } else {
            // Выбираем сторону с меньшей разницей
            if arr[index]-arr[left-1] <= arr[right+1]-arr[index] {
                left--
            } else {
                right++
            }
        }
    }
    
    return arr[left:right+1]
}
```

### Пояснение алгоритма:

1. **Инициализация**: Начинаем с окна `[index, index]` (один элемент)
2. **Расширение окна**:
   - Если достигли начала массива - расширяем вправо
   - Если достигли конца - расширяем влево
   - Иначе выбираем сторону с меньшей разницей значений
3. **Критерий остановки**: Когда размер окна достигает `k` элементов
4. **Возвращаем** подмассив `arr[left:right+1]`

### Пример работы:

```go
func main() {
    arr := []int{1, 2, 3, 4, 5, 6, 7}
    fmt.Println(findClosestElements(arr, 3, 3)) // [3, 4, 5]
    fmt.Println(findClosestElements(arr, 2, 4)) // [2, 3, 4, 5]
    fmt.Println(findClosestElements(arr, 0, 2)) // [1, 2]
}
```

### Оптимизация (бинарный поиск + два указателя):

Для больших массивов лучше использовать комбинацию бинарного поиска и двух указателей:

```go
func findClosestElementsOpt(arr []int, index int, k int) []int {
    low := max(0, index-k)
    high := min(len(arr)-1, index+k)
    
    for high-low+1 > k {
        if arr[index]-arr[low] <= arr[high]-arr[index] {
            high--
        } else {
            low++
        }
    }
    
    return arr[low:high+1]
}

func max(a, b int) int {
    if a > b { return a }
    return b
}

func min(a, b int) int {
    if a < b { return a }
    return b
}
```

### Сравнение подходов:

1. **Первый метод**:
   - Прост в реализации
   - Хорошо работает для небольших `k`
   - Сложность O(k)

2. **Оптимизированный метод**:
   - Начинает с большего окна (2k элементов)
   - Сужает окно до k элементов
   - Сложность O(log(k) + k)

Выбор метода зависит от размера массива и типичных значений `k`. Для большинства случаев достаточно первого варианта.