
Вот перевод поясняющего комментария из исходников Go про работу `sync.Mutex` и его справедливость:

---

### 🔄 Справедливость `Mutex` (Mutex fairness)

```go
// Справедливость mutex'а.
//
// Mutex работает в двух режимах: нормальном и starvation (голодание).
// В нормальном режиме ожидающие (waiters) ставятся в очередь FIFO,
// но разбуженный горутина (waiter) не получает mutex сразу — он конкурирует
// с новыми goroutine'ами за владение. Новые gorутины имеют преимущество:
// они уже запущены на CPU и их может быть много, так что разбуженный
// waiter часто проигрывает. В этом случае он возвращается в начало очереди.
// Если waiter не получает mutex более 1 мс, mutex переключается в режим
// starvation.
//
// В режиме starvation владение mutex'ом передаётся напрямую от
// unlock'а goroutine'ы к waiter'у в начале очереди.
// Новые goroutine'ы даже не пытаются захватить mutex — они просто встают
// в конец очереди и не делают спин (не крутятся в ожидании).
//
// Когда waiter получает владение и видит, что либо
// (1) он — последний waiter в очереди, либо
// (2) он ждал менее 1 мс,
// — mutex возвращается в нормальный режим.
//
// Нормальный режим значительно быстрее, так как горутина может захватывать
// mutex подряд, даже если есть заблокированные waiters.
// Режим starvation важен, чтобы предотвращать *pathological tail latency*.
```

---

### 📘 Что это значит простыми словами

1. **Нормальный режим**
    
    - Запросы идут примерно FIFO, но разбуженная горутина **не гарантированно** первой получает доступ — она конкурирует с уже активными горутинами.
        
    - Это быстрее, но может создавать _нечестную_ конкуренцию.
        
2. **Когда включается starvation**
    
    - Если одна и та же горутина ждёт больше **1 миллисекунды**, она переводит mutex в режим starvation.
        
    - Теперь очередь соблюдается строже: каждый unlock **передаёт** владение следующему в очереди, без конкуренции, и новые ждут в конце.
        
3. **Возврат в нормальный режим**
    
    - Как только очередь почти пуста — или задержка ожидания была меньше миллисекунды — `Mutex` снова переходит в быстрый (нормальный) режим.
        
4. **Почему важно**
    
    - **Нормальный режим** эффективен, но может создать ситуацию, когда одна горутина всё время выигрывает.
        
    - **Starvation режим** гарантирует, что все ждут свое право, и нечестное ожидание устраняется.
        

---

### 📌 Источники

- VictoriaMetrics — подробное объяснение режимов `Mutex` в Go, включая тихий режим голодания после 1 мс ожидания ([victoriametrics.com](https://victoriametrics.com/blog/go-sync-mutex/?utm_source=chatgpt.com "Go sync.Mutex: Normal and Starvation Mode - VictoriaMetrics"), [medium.com](https://medium.com/a-journey-with-go/go-mutex-and-starvation-3f4f4e75ad50?utm_source=chatgpt.com "Go: Mutex and Starvation - Medium"))
    
- StackOverflow — комментарий, где приведён полный текст этого блока документации ([stackoverflow.com](https://stackoverflow.com/questions/57562606/why-does-sync-mutex-largely-drop-performance-when-goroutine-contention-is-more-t?utm_source=chatgpt.com "Why does sync.Mutex largely drop performance when goroutine ..."))
    

---

Да, ты совершенно прав — обсуждение нормального и starvation режимов мьютекса — это именно часть **slow path**, а не про fast/slow пути блокировки в общем контексте. В Go используется **3 уровня** работы с мьютексом:

---

## 🌟 Три режима работы `sync.Mutex`

### 1. **Fast Path**

- Это самый быстрый и простой случай.
    
- Реализован через `atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked)`.
    
- Если мьютекс свободен, он сразу берётся, и метод уходит без дополнительных затрат ([VictoriaMetrics](https://victoriametrics.com/blog/go-sync-mutex/?utm_source=chatgpt.com "Go sync.Mutex: Normal and Starvation Mode - VictoriaMetrics")).
    

### 2. **Slow Path** (средний и медленный)

Когда fast path не срабатывает (мьютекс уже занят):

#### 🔹 Mid Path (спиннинг)

- Горутина активно жёстко зацикливается (до ~120 CPU‑циклов), пытаясь заново захватить мьютекс — это называется _спиннинг_ ([VictoriaMetrics](https://victoriametrics.com/blog/go-sync-mutex/?utm_source=chatgpt.com "Go sync.Mutex: Normal and Starvation Mode - VictoriaMetrics"), [blog.devtrovert.com](https://blog.devtrovert.com/p/go-sync-mutex-normal-and-starvation?utm_source=chatgpt.com "Go EP 15: sync.Mutex - Normal and Starvation Mode - Devtrovert")).
    

#### 🔹 Stall Path (блокировка)

- Если спиннинг не помогает, горутина садится в очередь ожидания и уходит в сон — это уже настоящий «медленный путь», включающий очередь и семафор.
    

---

### 3. **Режим справедливости (Starvation Mode)**

Это _не отдельный путь запроса_, а **часть slow path’а**, когда:

- Если горутина не получает мьютекс в течение >1 мс, система переключается в режим starvation — тогда разблокировка передаёт мьютекс строго по очереди, без спиннинга и конкуренции, и новые горутины ставятся в хвост очереди ([VictoriaMetrics](https://victoriametrics.com/blog/go-sync-mutex/?utm_source=chatgpt.com "Go sync.Mutex: Normal and Starvation Mode - VictoriaMetrics"), [Stack Overflow](https://stackoverflow.com/questions/57562606/why-does-sync-mutex-largely-drop-performance-when-goroutine-contention-is-more-t?utm_source=chatgpt.com "Why does sync.Mutex largely drop performance when goroutine ...")).
    

Когда очередь почти пуста или ожидание было коротким (<1 мс), мьютекс возвращается в normal режим ([VictoriaMetrics](https://victoriametrics.com/blog/go-sync-mutex/?utm_source=chatgpt.com "Go sync.Mutex: Normal and Starvation Mode - VictoriaMetrics")).

---

## ✅ Вывод

- **Fast Path** — всё происходит мгновенно без задержек.
    
- **Slow Path** делится на два этапа:
    
    1. _Mid Path_ — спиннинг.
        
    2. _Stall Path_ — реальная блокировка и очередь + starvation logic.
        
- **Starvation Mode** — это особое поведение внутри slow path, обеспечивающее справедливость, а не отдельно выделенный режим блокировки.
    

Другими словами, все — и fast/slow/mid/starvation — являются составляющими **Lock()**, но **starvation mode** — это лишь один из подрежимов **slow path**, обеспечивающий fairness, а не отдельная фаза. Если хочется — могу показать схемы из исходников, как это выражено в `lockSlow()`.