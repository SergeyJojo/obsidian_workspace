
Отлично, Сергей. Продолжаем.

---

## 🟩 **Задача 2 — Аналога на LeetCode нет**

### 🔹 Авторская: «Собери команду»

---

### 📋 Условие

У тебя есть 4 отсортированных массива:

- backend
    
- frontend
    
- qa
    
- design
    

Нужно выбрать **по одному элементу из каждого массива**, чтобы **разница между самым большим и самым маленьким** значением была **минимальной**.

---

### 📌 Пример 1:

```go
backend  = [1, 2, 2, 3]
frontend = [1, 3]
qa       = [3, 4, 4]
design   = [2, 3]

// Один из возможных оптимальных ответов:
[3, 3, 3, 3]
```

---

## 🧠 Идея алгоритма

Это вариация задачи **"K pointers"** или «K-way merge».

### Подход:

1. Инициализируем **K указателей** — по одному на каждый массив.
    
2. На каждом шаге:
    
    - Берём текущие значения по указателям.
        
    - Ищем `min` и `max` среди них.
        
    - Обновляем текущую лучшую команду, если `max - min` меньше, чем предыдущая разница.
        
    - **Двигаем тот указатель, у кого значение минимальное** (чтобы попытаться сузить диапазон).
        
3. Останавливаемся, когда **хотя бы один массив заканчивается**.
    

---

## ✅ Реализация на Go

```go
package main

import (
	"fmt"
)

func min(a []int) (int, int) {
	minVal := a[0]
	idx := 0
	for i, v := range a {
		if v < minVal {
			minVal = v
			idx = i
		}
	}
	return minVal, idx
}

func max(a []int) int {
	maxVal := a[0]
	for _, v := range a {
		if v > maxVal {
			maxVal = v
		}
	}
	return maxVal
}

func teamSelection(groups [][]int) []int {
	k := len(groups)
	ptrs := make([]int, k)
	bestRange := int(1e9)
	result := make([]int, k)

	for {
		curVals := make([]int, k)
		active := true

		for i := 0; i < k; i++ {
			if ptrs[i] >= len(groups[i]) {
				active = false
				break
			}
			curVals[i] = groups[i][ptrs[i]]
		}

		if !active {
			break
		}

		curMin, minIdx := min(curVals)
		curMax := max(curVals)
		curRange := curMax - curMin

		if curRange < bestRange {
			bestRange = curRange
			copy(result, curVals)
		}

		ptrs[minIdx]++ // сдвигаем того, у кого минимум
	}

	return result
}

func main() {
	engineers := [][]int{
		{1, 2, 2, 3}, // backend
		{1, 3},       // frontend
		{3, 4, 4},    // qa
		{2, 3},       // design
	}

	fmt.Println("Best team:", teamSelection(engineers)) // Примерный результат: [3 3 3 3]
}
```

---

## 📈 Сложность

- **Время**: `O(N)` — в худшем случае суммарная длина всех массивов.
    
- **Память**: `O(K)` — где K — число ролей (4 здесь).
    

---

## ⚠️ Особенности:

- Работает **только если массивы отсортированы**.
    
- Можно обобщить для произвольного K.
    

---

## ✅ Следующая задача:

**3. [215. Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array)**  
Хочешь продолжить?