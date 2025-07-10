
Отлично, Сергей. Разбираем одну из самых популярных задач уровня **Medium** — **Three Sum** (LeetCode №15) — на языке Go.

---

## ✅ 2. **Three Sum (№15)** — Уровень: Medium

### 🧾 Условие:

Дан массив целых чисел `nums`, найти все **уникальные триплеты** `(a, b, c)`, такие что:

```
a + b + c == 0
```

Нельзя использовать один и тот же элемент более одного раза в одном триплете. Ответ не должен содержать дубликатов.

---

### 🧠 Идея:

1. Отсортировать массив.
    
2. Для каждого `i` (фиксируем `nums[i]`) использовать **двухуказательную технику** на подотрезке справа от `i`.
    
3. Пропускать дубликаты при необходимости (для `i`, `left`, `right`).
    

---

### ✅ Реализация на Go:

```go
import "sort"

func threeSum(nums []int) [][]int {
    sort.Ints(nums)
    var res [][]int

    for i := 0; i < len(nums)-2; i++ {
        if i > 0 && nums[i] == nums[i-1] {
            continue // Пропускаем дубликаты
        }

        left, right := i+1, len(nums)-1
        for left < right {
            sum := nums[i] + nums[left] + nums[right]

            if sum == 0 {
                res = append(res, []int{nums[i], nums[left], nums[right]})

                // Пропустить дубликаты
                for left < right && nums[left] == nums[left+1] {
                    left++
                }
                for left < right && nums[right] == nums[right-1] {
                    right--
                }

                left++
                right--
            } else if sum < 0 {
                left++
            } else {
                right--
            }
        }
    }

    return res
}
```

---

### 📈 Сложность:

- **Время:** `O(n²)` — внешний цикл `n`, внутренний двухуказательный — тоже в пределах `n`.
    
- **Память:** `O(1)` (если не считать память на результат)
    

---

### 🧪 Пример:

```go
nums := []int{-1, 0, 1, 2, -1, -4}
fmt.Println(threeSum(nums))
// Output: [[-1 -1 2] [-1 0 1]]
```

---

Если хочешь, покажу аналогичную реализацию на **Python**, либо можно перейти к следующей классической задаче, например:

- Longest Substring Without Repeating Characters
    
- Merge Intervals
    
- Valid Parentheses
    
- Container With Most Water
    
- или твою по выбору.