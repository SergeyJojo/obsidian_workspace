
Отлично, Сергей! Переходим к задаче, которая проверяет знание **бэктрекинга (backtracking)** — одна из самых популярных:

---

## ✅ 5. **Letter Combinations of a Phone Number** (LeetCode №17)

**Уровень:** Medium

---

### 🧾 Условие:

Дан строковый ввод `digits`, содержащий только цифры от `'2'` до `'9'`. Каждая цифра сопоставляется с набором букв, как на кнопках телефона:

| Цифра | Буквы |     |
| ----- | ----- | --- |
| 2     | abc   |     |
| 3     | def   |     |
| 4     | ghi   |     |
| 5     | jkl   |     |
| 6     | mno   |     |
| 7     | pqrs  |     |
| 8     | tuv   |     |
| 9     | wxyz  |     |

Верни **все возможные комбинации букв**, соответствующие этим цифрам.

---

### 📌 Пример:

```text
Input: "23"
Output: ["ad","ae","af","bd","be","bf","cd","ce","cf"]
```

---

## 🧠 Идея: Рекурсия + Backtracking

- Каждая цифра → набор букв
    
- На каждом шаге — пробуем все буквы для текущей цифры
    
- Рекурсивно углубляемся, пока не соберём комбинацию длины `len(digits)`
    

---

## ✅ Go-реализация:

```go
func letterCombinations(digits string) []string {
    if len(digits) == 0 {
        return nil
    }

    phone := map[byte]string{
        '2': "abc", '3': "def",
        '4': "ghi", '5': "jkl", '6': "mno",
        '7': "pqrs", '8': "tuv", '9': "wxyz",
    }

    var res []string

    var backtrack func(index int, path string)
    backtrack = func(index int, path string) {
        if index == len(digits) {
            res = append(res, path)
            return
        }

        letters := phone[digits[index]]
        for i := 0; i < len(letters); i++ {
            backtrack(index+1, path+string(letters[i]))
        }
    }

    backtrack(0, "")
    return res
}
```

---

### 📈 Сложность:

- **Время:** `O(4ⁿ)`, где `n = len(digits)`  
    (в худшем случае каждая цифра = 4 буквы, например `'7'` или `'9'`)
    
- **Память:** `O(4ⁿ)` — из-за всех комбинаций
    

---

### 📌 Почему работает:

- Каждая рекурсивная ветка соответствует одной возможной букве для текущей цифры.
    
- Как только длина пути `path` = длине `digits` — комбинация готова.
    

---

Хочешь следующую задачу на генерацию комбинаций (например, **Generate Parentheses** или **Subsets**), или пойти в область динамики или деревьев?