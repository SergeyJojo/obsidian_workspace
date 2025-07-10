Задача **LeetCode #40 — Combination Sum II**

> 🔢 Сложная, но очень важная задача на **backtracking** с учётом **повторяющихся элементов**.

---

## 🧩 Условие

> Дан массив `candidates`, содержащий целые числа (возможно с повторами), и целевое число `target`.  
> Найдите **все уникальные комбинации** кандидатов, сумма которых равна `target`.  
> Каждое число может использоваться **только один раз** в комбинации. Результат не должен содержать дубликатов.

---

### Пример

```text
Input:  candidates = [10,1,2,7,6,1,5], target = 8
Output: 
[
  [1,1,6],
  [1,2,5],
  [1,7],
  [2,6]
]
```

---

## 📚 Подход (Backtracking + Пропуск дубликатов)

1. Отсортировать входной массив — это **ключ к устранению дубликатов**.
    
2. Использовать **бэктрекинг**:
    
    - Проходим по массиву от текущего индекса.
        
    - Пропускаем элемент, если он равен предыдущему (`nums[i] == nums[i-1]`) и мы **на том же уровне рекурсии**.
        
3. Следим за оставшейся суммой (target - sum).
    
4. Когда сумма достигает нуля — сохраняем копию текущего пути.
    

---

## ✅ Решение на Go

```go
func combinationSum2(candidates []int, target int) [][]int {
    sort.Ints(candidates) // сортировка для устранения дубликатов
    var result [][]int

    var backtrack func(start int, target int, path []int)
    backtrack = func(start int, target int, path []int) {
        if target == 0 {
            // нашли комбинацию
            comb := make([]int, len(path))
            copy(comb, path)
            result = append(result, comb)
            return
        }

        for i := start; i < len(candidates); i++ {
            // Пропускаем дубликаты на одном уровне
            if i > start && candidates[i] == candidates[i-1] {
                continue
            }

            if candidates[i] > target {
                break // дальше смысла нет
            }

            path = append(path, candidates[i])
            backtrack(i+1, target - candidates[i], path) // i+1, потому что нельзя переиспользовать
            path = path[:len(path)-1] // откат
        }
    }

    backtrack(0, target, []int{})
    return result
}
```

---

## 🧠 Пояснение ключевых моментов

|Строка|Объяснение|
|---|---|
|`sort.Ints(candidates)`|Помогает удобно сравнивать соседние элементы и пропускать дубликаты|
|`if i > start && candidates[i] == candidates[i-1]`|Пропускаем повторяющиеся элементы на том же уровне|
|`backtrack(i+1, ...)`|Переходим к следующему элементу — нельзя использовать текущий дважды|
|`path = path[:len(path)-1]`|Откат к предыдущему состоянию (backtrack)|

---

## ⏱ Сложность

- **Время**: В худшем случае — `O(2^n)`, где `n = len(candidates)`
    
- **Память**: До `O(n)` стек вызовов + хранение результатов
    

---

## 🧪 Пример запуска

```go
candidates := []int{10,1,2,7,6,1,5}
target := 8
fmt.Println(combinationSum2(candidates, target))
```

Вернёт:

```go
[[1 1 6] [1 2 5] [1 7] [2 6]]
```

---

Хочешь — могу показать аналогичное решение на Python или дать шаблон для бэктрекинга.