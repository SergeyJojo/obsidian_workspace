
Давайте разберем, что происходит в предоставленном коде, связанном с остановкой и запуском мира (stop-the-world, STW) в Go runtime, а также как это связано с GMP-моделью и сборщиком мусора (GC). Код из файла runtime/proc.go описывает функции stopTheWorldWithSema и startTheWorldWithSema, которые используются для координации остановки выполнения всех горутин (G) на потоках (M) через процессоры (P), например, для выполнения сборки мусора или других глобальных операций.

---

Контекст: Зачем нужны stopTheWorld и startTheWorld?

В Go используется конкурентный сборщик мусора (Garbage Collector, GC), который работает параллельно с выполнением программы. Однако некоторые операции (например, начало или конец фазы GC, изменение GOMAXPROCS, трассировка) требуют полной остановки всех горутин — это называется "stop-the-world" (STW). Во время STW ни одна горутина не выполняется, что позволяет безопасно обновить состояние runtime.

Для этого используются семафоры:

- gcsema: Блокирует изменение числа процессоров (GOMAXPROCS) и ожидает завершения текущего GC.
    
- worldsema: Гарантирует, что только одна операция STW выполняется одновременно.
    

---

Код и его разбор

Переменная gcsema

go

```go
var gcsema uint32 = 1
```

- Это семафор, который дает потоку (M) право блокировать GC или ждать завершения текущего цикла GC.
    
- Значение 1 означает, что семафор свободен. Когда поток захватывает его, он становится 0, блокируя другие попытки STW до освобождения.
    
- Комментарий TODO указывает, что это временное решение, пока GOMAXPROCS и трассировщик не смогут изменяться во время GC.
    

---

Функция stopTheWorldWithSema

go

```go
func stopTheWorldWithSema(reason stwReason) worldStop
```

- Назначение: Останавливает все процессоры (P) и переводит их в состояние _Pgcstop, чтобы остановить выполнение всех горутин.
    
- Аргумент: reason — причина остановки (например, GC, трассировка).
    
- Возвращает: Структура worldStop с метаинформацией о процессе остановки.
    

Шаги выполнения:

1. Трассировка (если включена):
    
    go
    
    ```go
    trace := traceAcquire()
    if trace.ok() {
        trace.STWStart(reason)
        traceRelease(trace)
    }
    ```
    
    - Если трассировка активна, записывается событие начала STW.
        
2. Проверка состояния текущей горутины:
    
    go
    
    ```go
    gp := getg()
    if gp.m.locks > 0 {
        throw("stopTheWorld: holding locks")
    }
    ```
    
    - Убеждаемся, что текущий поток (M) не держит блокировок, иначе остановка может привести к deadlock.
        
3. Блокировка планировщика:
    
    go
    
    ```go
    lock(&sched.lock)
    sched.stopwait = gomaxprocs
    sched.gcwaiting.Store(true)
    ```
    
    - Захватываем глобальную блокировку планировщика (sched.lock).
        
    - Устанавливаем sched.stopwait = gomaxprocs — количество P, которые нужно остановить.
        
    - sched.gcwaiting = true сигнализирует, что GC ждет остановки мира.
        
4. Предварительная остановка всех P:
    
    go
    
    ```go
    preemptall()
    gp.m.p.ptr().status = _Pgcstop
    sched.stopwait--
    ```
    
    - preemptall() отправляет запросы на прерывание всем P.
        
    - Текущий P (на котором выполняется эта функция) сразу переводится в _Pgcstop.
        
5. Остановка P в состоянии _Psyscall:
    
    go
    
    ```go
    for _, pp := range allp {
        s := pp.status
        if s == _Psyscall && atomic.Cas(&pp.status, s, _Pgcstop) {
            pp.syscalltick++
            pp.gcStopTime = nanotime()
            sched.stopwait--
        }
    }
    ```
    
    - Перебираем все P (allp) и пытаемся перехватить те, что находятся в системных вызовах (_Psyscall).
        
    - Используем атомарную операцию Cas для изменения состояния на _Pgcstop.
        
6. Остановка простаивающих P:
    
    go
    
    ```go
    for {
        pp, _ := pidleget(now)
        if pp == nil {
            break
        }
        pp.status = _Pgcstop
        pp.gcStopTime = nanotime()
        sched.stopwait--
    }
    ```
    
    - Извлекаем простаивающие P из очереди sched.pidle и переводим их в _Pgcstop.
        
7. Ожидание оставшихся P:
    
    go
    
    ```go
    if wait {
        for {
            if notetsleep(&sched.stopnote, 100*1000) {
                noteclear(&sched.stopnote)
                break
            }
            preemptall()
        }
    }
    ```
    
    - Если остались P, не остановленные добровольно (sched.stopwait > 0), ждем 100 мкс и повторяем прерывание через preemptall().
        
8. Проверка корректности остановки:
    
    go
    
    ```go
    if sched.stopwait != 0 {
        bad = "stopTheWorld: not stopped (stopwait != 0)"
    } else {
        for _, pp := range allp {
            if pp.status != _Pgcstop {
                bad = "stopTheWorld: not stopped (status != _Pgcstop)"
            }
            stoppingCPUTime += finish - pp.gcStopTime
        }
    }
    if bad != "" {
        throw(bad)
    }
    ```
    
    - Проверяем, что все P действительно остановлены (_Pgcstop).
        
    - Подсчитываем время остановки каждого P для статистики.
        
9. Возврат результата:
    
    go
    
    ```go
    return worldStop{
        reason:           reason,
        startedStopping:  start,
        finishedStopping: finish,
        stoppingCPUTime:  stoppingCPUTime,
    }
    ```
    
    - Возвращаем структуру с данными об остановке.
        

---

Функция startTheWorldWithSema

go

```go
func startTheWorldWithSema(now int64, w worldStop) int64
```

- Назначение: Возобновляет выполнение мира после STW.
    
- Аргументы:
    
    - now — текущее время (или 0 для замера внутри функции).
        
    - w — структура worldStop из stopTheWorldWithSema.
        
- Возвращает: Время возобновления мира.
    

Шаги выполнения:

1. Обработка сетевых операций:
    
    go
    
    ```go
    if netpollinited() {
        list, delta := netpoll(0) // non-blocking
        injectglist(&list)
        netpollAdjustWaiters(delta)
    }
    ```
    
    - Проверяем завершенные сетевые операции (netpoller) и добавляем готовые G в очередь.
        
2. Обновление числа процессоров:
    
    go
    
    ```go
    procs := gomaxprocs
    if newprocs != 0 {
        procs = newprocs
        newprocs = 0
    }
    p1 := procresize(procs)
    ```
    
    - Если было запрошено изменение GOMAXPROCS (newprocs), применяем его через procresize.
        
3. Снятие флага ожидания GC:
    
    go
    
    ```go
    sched.gcwaiting.Store(false)
    ```
    
    - Указываем, что GC больше не ждет остановки.
        
4. Запуск простаивающих M:
    
    go
    
    ```go
    for p1 != nil {
        p := p1
        p1 = p1.link.ptr()
        if p.m != 0 {
            mp := p.m.ptr()
            mp.nextp.set(p)
            notewakeup(&mp.park)
        } else {
            newm(nil, p, -1)
        }
    }
    ```
    
    - Для каждого P из списка p1:
        
        - Если у P есть связанный M, будим его через notewakeup.
            
        - Если M нет, создаем новый через newm.
            
5. Статистика и трассировка:
    
    go
    
    ```go
    totalTime := now - w.startedStopping
    if w.reason.isGC() {
        sched.stwTotalTimeGC.record(totalTime)
    } else {
        sched.stwTotalTimeOther.record(totalTime)
    }
    ```
    
    - Записываем общее время остановки в зависимости от причины (GC или другое).
        
6. Пробуждение дополнительного P:
    
    go
    
    ```go
    wakep()
    ```
    
    - Активируем дополнительный P, если есть избыточная работа в очередях.
        

---

Что тут происходит в целом?

1. stopTheWorldWithSema:
    
    - Захватывает все P, переводя их в состояние _Pgcstop.
        
    - Обрабатывает три категории P: текущий, в системных вызовах (_Psyscall), и простаивающие (_Pidle).
        
    - Ждет добровольной остановки остальных P через прерывания.
        
    - Используется для GC или других операций, требующих полной остановки.
        
2. startTheWorldWithSema:
    
    - Возобновляет работу P, связывая их с M.
        
    - Применяет изменения GOMAXPROCS, если они были запрошены.
        
    - Обрабатывает завершенные сетевые операции и запускает выполнение G.
        
3. Синхронизация:
    
    - worldsema обеспечивает, что только один поток выполняет STW.
        
    - gcsema предотвращает изменение GOMAXPROCS во время GC.
        

---

Связь с GMP-моделью

- G (Goroutines): Останавливаются, когда их P переходит в _Pgcstop.
    
- M (Machines): Потоки либо паркуются, либо переключаются на выполнение системных задач (например, GC).
    
- P (Processors): Управляют очередями G и являются целью остановки/запуска.
    

STW — это механизм синхронизации всей GMP-модели, чтобы runtime мог безопасно выполнить глобальные операции.

---

Итог

Этот код реализует:

- Остановку мира: Прерывает выполнение всех P, чтобы остановить G на M.
    
- Запуск мира: Возобновляет выполнение, распределяя P по M и активируя G.
    

Это критическая часть runtime Go, обеспечивающая работу GC и других системных функций. Если хотите углубиться в конкретный аспект (например, как работает preemptall или netpoll), дайте знать!