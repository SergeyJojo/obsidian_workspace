Похожая: 1004. Max Consecutive Ones III


---

## 🟩 **Задача 7 — Авторская: «Пора в отпуск»**

🔹 **Ближайший аналог на LeetCode:** задачи на **скользящее окно (sliding window)**  
Похожая: [1004. Max Consecutive Ones III](https://leetcode.com/problems/max-consecutive-ones-iii)

---

### 📋 Условие

У нас есть список дней с количеством встреч в них:

```go
type Day struct {
	day      int // номер дня
	meetings int // сколько встреч в этот день
}
```

Также задана длина отпуска `vacationLength` и длина окна, в которое надо уместить отпуск `periodLength`.  
Нужно выбрать начало отпуска `X`, чтобы он полностью вошёл в X, X+vacationLength) в пределах `periodLength` дней, и чтобы **пропущенных встреч было как можно меньше**.

---

### 📌 Пример

```go
daysWithMeetings = []Day{
	{3, 1},
	{4, 3},
	{14, 7},
	{21, 3},
	{22, 1},
}

periodLength   := 30
vacationLength := 7

// Нужно выбрать такой X, чтобы:
// - [X, X+6] не выходило за границы [1..30]
// - сумма встреч в эти дни была минимальна
```

---

## 🧠 Идея алгоритма

1. Составим массив `meetings[1..periodLength]` — сколько встреч в каждый из дней.
    
2. Применим **скользящее окно** длиной `vacationLength`:
    
    - Находим окно с минимальной суммой встреч.
        
    - Возвращаем начало этого окна.
        

---

## ✅ Реализация на Go

```go
package main

import (
	"fmt"
)

type Day struct {
	day      int
	meetings int
}

func bestVacationStart(days []Day, periodLength, vacationLength int) (startDay int, missedMeetings int) {
	// 1. Составим массив встреч по дням
	meetingMap := make([]int, periodLength+2) // 1-based

	for _, d := range days {
		if d.day >= 1 && d.day <= periodLength {
			meetingMap[d.day] += d.meetings
		}
	}

	// 2. Sliding window
	minSum := int(1e9)
	windowSum := 0
	start := 1

	// Инициализируем окно [1, vacationLength]
	for i := 1; i <= vacationLength; i++ {
		windowSum += meetingMap[i]
	}
	minSum = windowSum

	for i := 2; i <= periodLength-vacationLength+1; i++ {
		// сдвигаем окно: -левый +правый
		windowSum = windowSum - meetingMap[i-1] + meetingMap[i+vacationLength-1]
		if windowSum < minSum {
			minSum = windowSum
			start = i
		}
	}

	return start, minSum
}

func main() {
	daysWithMeetings := []Day{
		{3, 1},
		{4, 3},
		{14, 7},
		{21, 3},
		{22, 1},
	}
	start, skipped := bestVacationStart(daysWithMeetings, 30, 7)
	fmt.Printf("Лучший день для начала отпуска: %d (пропущено встреч: %d)\n", start, skipped)
}
```

---

## 📈 Сложность

- **Время**: `O(periodLength)`
    
- **Память**: `O(periodLength)` — массив встреч
    

---

## 🧪 Edge Cases

- Встреч нет — любое окно подойдёт
    
- Встречи на границах окна
    
- Все встречи вне отпускного периода
    

---

## ✅ Следующая задача:

**8. Чемпионат по шагам** — классическая [70. Climbing Stairs](https://leetcode.com/problems/climbing-stairs)

Продолжим?