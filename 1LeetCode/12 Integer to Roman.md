
Вот решение задачи LeetCode **"12. Integer to Roman"** на Go с пошаговыми пояснениями.

---

## 🧩 Задача: Integer to Roman

Преобразовать целое число `int` (1 ≤ num ≤ 3999) в **римское число**.

---

## 🔠 Римские символы

|Символ|Значение|
|---|---|
|M|1000|
|CM|900|
|D|500|
|CD|400|
|C|100|
|XC|90|
|L|50|
|XL|40|
|X|10|
|IX|9|
|V|5|
|IV|4|
|I|1|

---

## ✅ Решение на Go

```go
package main

import "fmt"

func intToRoman(num int) string {
	// От большего к меньшему
	values := []int{1000, 900, 500, 400, 100, 90, 50, 40, 10, 9, 5, 4, 1}
	symbols := []string{"M", "CM", "D", "CD", "C", "XC", "L", "XL", "X", "IX", "V", "IV", "I"}

	result := ""

	for i := 0; i < len(values); i++ {
		for num >= values[i] {
			num -= values[i]
			result += symbols[i]
		}
	}

	return result
}

func main() {
	fmt.Println(intToRoman(58))   // LVIII
	fmt.Println(intToRoman(1994)) // MCMXCIV
}
```

---

## 🧠 Пояснение

- Мы идём по списку значений **от большего к меньшему**.
    
- Пока `num >= value[i]`, мы:
    
    - уменьшаем `num` на это значение,
        
    - добавляем соответствующий символ к результату.
        
- Всё происходит по **жадному алгоритму**, начиная с самых крупных римских чисел.
    

---

Хочешь — могу дать и решение в обратную сторону (Roman → Integer) или реализовать тесты для этого решения.