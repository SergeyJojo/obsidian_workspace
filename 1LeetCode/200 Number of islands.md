
### Задача "Number of Islands" (Количество островов) на Go

**Условие задачи:**
Дана двумерная матрица `grid` из символов `'1'` (суша) и `'0'` (вода). Остров — это группа `'1'`, соединённых по вертикали или горизонтали. Нужно найти количество островов.

**Пример:**
```go
grid := [][]byte{
    {'1','1','0','0','0'},
    {'1','1','0','0','0'},
    {'0','0','1','0','0'},
    {'0','0','0','1','1'},
}
// Ответ: 3 острова
```

### Решение на Go (DFS подход)

```go
func numIslands(grid [][]byte) int {
    if len(grid) == 0 || len(grid[0]) == 0 {
        return 0
    }

    rows, cols := len(grid), len(grid[0])
    count := 0

    for i := 0; i < rows; i++ {
        for j := 0; j < cols; j++ {
            if grid[i][j] == '1' {
                count++
                dfs(grid, i, j, rows, cols)
            }
        }
    }

    return count
}

func dfs(grid [][]byte, i, j, rows, cols int) {
    // Проверка границ и что это земля
    if i < 0 || j < 0 || i >= rows || j >= cols || grid[i][j] != '1' {
        return
    }

    // Помечаем как посещённую
    grid[i][j] = '0'

    // Рекурсивно проверяем соседей
    dfs(grid, i+1, j, rows, cols) // вниз
    dfs(grid, i-1, j, rows, cols) // вверх
    dfs(grid, i, j+1, rows, cols) // вправо
    dfs(grid, i, j-1, rows, cols) // влево
}
```

### Ключевые моменты:
1. **Обход матрицы**: Проходим по всем клеткам матрицы.
2. **Обнаружение острова**: Когда находим `'1'`, увеличиваем счётчик и запускаем DFS.
3. **DFS (поиск в глубину)**: 
   - Помечаем посещённые клетки как `'0'` (чтобы не считать их повторно)
   - Рекурсивно проверяем 4 соседние клетки (вверх, вниз, влево, вправо)
4. **Сложность**: O(M×N), где M и N — размеры матрицы.

### Альтернативное решение (BFS)
```go
func numIslands(grid [][]byte) int {
    if len(grid) == 0 {
        return 0
    }

    rows, cols := len(grid), len(grid[0])
    count := 0
    directions := [][]int{{1,0}, {-1,0}, {0,1}, {0,-1}}

    for i := 0; i < rows; i++ {
        for j := 0; j < cols; j++ {
            if grid[i][j] == '1' {
                count++
                queue := [][]int{{i,j}}
                grid[i][j] = '0'

                for len(queue) > 0 {
                    cell := queue[0]
                    queue = queue[1:]
                    for _, dir := range directions {
                        r, c := cell[0]+dir[0], cell[1]+dir[1]
                        if r >= 0 && c >= 0 && r < rows && c < cols && grid[r][c] == '1' {
                            queue = append(queue, []int{r,c})
                            grid[r][c] = '0'
                        }
                    }
                }
            }
        }
    }

    return count
}
```

### Тестирование
```go
func main() {
    grid1 := [][]byte{
        {'1','1','0','0','0'},
        {'1','1','0','0','0'},
        {'0','0','1','0','0'},
        {'0','0','0','1','1'},
    }
    fmt.Println(numIslands(grid1)) // 3

    grid2 := [][]byte{
        {'1','1','1','1','0'},
        {'1','1','0','1','0'},
        {'1','1','0','0','0'},
        {'0','0','0','0','0'},
    }
    fmt.Println(numIslands(grid2)) // 1
}
```

Эта задача (LeetCode #200) отлично демонстрирует применение DFS/BFS для работы с матрицами.