
### Решение задачи "Roman to Integer" на Go (LeetCode 13)

Задача: преобразовать римское число в целое. Римские цифры представлены семью символами: I, V, X, L, C, D, M.

```go
package main

import "fmt"

func romanToInt(s string) int {
    // Создаем карту соответствия римских цифр и их значений
    romanMap := map[byte]int{
        'I': 1,
        'V': 5,
        'X': 10,
        'L': 50,
        'C': 100,
        'D': 500,
        'M': 1000,
    }
    
    total := 0
    prevValue := 0
    
    // Идем по строке справа налево
    for i := len(s) - 1; i >= 0; i-- {
        currentValue := romanMap[s[i]]
        
        // Если текущее значение меньше предыдущего, вычитаем его
        if currentValue < prevValue {
            total -= currentValue
        } else {
            total += currentValue
        }
        
        prevValue = currentValue
    }
    
    return total
}

func main() {
    testCases := []string{"III", "IV", "IX", "LVIII", "MCMXCIV"}
    for _, tc := range testCases {
        fmt.Printf("%s -> %d\n", tc, romanToInt(tc))
    }
}
```

### Ключевые моменты решения:

1. **Создание карты значений** для быстрого доступа к числовым значениям римских цифр
2. **Обработка строки справа налево** - это упрощает обработку случаев вычитания (IV, IX и т.д.)
3. **Логика вычитания**: если текущая цифра меньше предыдущей, вычитаем её значение
4. **Сложность**: O(n), где n - длина строки

### Примеры работы:
```
III -> 3
IV -> 4
IX -> 9
LVIII -> 58
MCMXCIV -> 1994
```

### Альтернативное решение (слева направо):

```go
func romanToInt(s string) int {
    romanMap := map[byte]int{
        'I': 1, 'V': 5, 'X': 10, 'L': 50,
        'C': 100, 'D': 500, 'M': 1000,
    }
    
    total := 0
    for i := 0; i < len(s); i++ {
        if i < len(s)-1 && romanMap[s[i]] < romanMap[s[i+1]] {
            total -= romanMap[s[i]]
        } else {
            total += romanMap[s[i]]
        }
    }
    return total
}
```

Оба решения эффективны, но первый вариант (справа налево) может быть немного проще для понимания.