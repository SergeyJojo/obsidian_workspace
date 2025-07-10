
Отлично, Сергей. Переходим к следующей классической и часто встречающейся задаче:

---

## ✅ 3. **Longest Substring Without Repeating Characters** (LeetCode №3)

**Уровень:** Medium

---

### 🧾 Условие:

Дана строка `s`. Найди длину **самой длинной подстроки без повторяющихся символов**.

---

### 📌 Примеры:

```text
Input: "abcabcbb"  → Output: 3  ("abc")
Input: "bbbbb"     → Output: 1  ("b")
Input: "pwwkew"     → Output: 3  ("wke")
```

---

## 🧠 Идея: Sliding Window (скользящее окно)

- Мы двигаем два указателя `left` и `right`, поддерживая **текущее окно без повторов**.
    
- Используем map/rune set, чтобы отслеживать встреченные символы.
    
- Если встретили повтор — сдвигаем `left` до следующей позиции после дубликата.
    

---

## ✅ Go-реализация:

```go
func lengthOfLongestSubstring(s string) int {
    seen := make(map[byte]int)
    maxLen := 0
    left := 0

    for right := 0; right < len(s); right++ {
        if idx, ok := seen[s[right]]; ok && idx >= left {
            // повтор найден: сдвигаем левую границу окна
            left = idx + 1
        }

        seen[s[right]] = right
        maxLen = max(maxLen, right - left + 1)
    }

    return maxLen
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}
```

---

## 🧪 Пример:

```go
s := "abcabcbb"
fmt.Println(lengthOfLongestSubstring(s)) // 3
```

---

## 📈 Сложность:

- **Время:** `O(n)` — каждый символ обрабатывается максимум дважды
    
- **Память:** `O(k)`, где `k` — количество уникальных символов
    

---

### 🔍 Почему работает:

- Каждую букву мы либо добавляем в map, либо сдвигаем окно мимо неё.
    
- Нет вложенных циклов, потому что сдвиг указателя `left` всегда идёт строго вперёд.
    

---

Хочешь аналог на Python или двигаемся к следующей классике — например, **Valid Parentheses**, **Container With Most Water**, **Longest Palindromic Substring**, **Merge Intervals** или ты выберешь?