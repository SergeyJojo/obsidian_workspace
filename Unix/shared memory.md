
### **Устройство Shared Memory в Linux**

**Shared Memory (Общая память)** — это механизм **меж процессного взаимодействия (IPC)**, позволяющий нескольким процессам обращаться к одной области оперативной памяти. Это самый быстрый способ обмена данными между процессами, так как не требует копирования через ядро (в отличие от pipes, sockets и т. д.).

---

## **1. Типы Shared Memory в Linux**
В Linux есть **два основных механизма** общей памяти:

### **1.1. System V Shared Memory (устаревший, но до сих пор поддерживается)**
- Основан на API System V IPC (`shmget`, `shmat`, `shmdt`).
- Управляется через **ключи (`key_t`)** и **идентификаторы (`shmid`)**.
- Не интегрирован с файловой системой (использует отдельные системные вызовы).

### **1.2. POSIX Shared Memory (современный стандарт)**
- Использует **файлоподобный интерфейс** (`shm_open`, `mmap`).
- Интегрирован с виртуальной файловой системой (`/dev/shm`).
- Более удобен и безопасен.

---

## **2. Как работает POSIX Shared Memory?** (рекомендуемый способ)
Рассмотрим на примере:

### **Шаг 1: Создание Shared Memory**
```c
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>

int main() {
    // 1. Открываем (или создаём) shared memory объект
    int fd = shm_open("/my_shared_mem", O_CREAT | O_RDWR, 0666);
    
    // 2. Устанавливаем размер
    ftruncate(fd, 4096);  // 4 KB
    
    // 3. Маппируем в адресное пространство процесса
    void *ptr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    
    // 4. Используем как обычную память
    sprintf(ptr, "Hello from Process 1!");
    
    // 5. В другом процессе можно прочитать эти данные
    return 0;
}
```

### **Шаг 2: Доступ из другого процесса**
```c
int main() {
    // 1. Открываем существующий shared memory
    int fd = shm_open("/my_shared_mem", O_RDWR, 0666);
    
    // 2. Маппируем
    void *ptr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    
    // 3. Читаем данные
    printf("Received: %s\n", (char*)ptr);
    
    // 4. Удаляем (опционально)
    shm_unlink("/my_shared_mem");
    return 0;
}
```

---

## **3. Как устроена Shared Memory внутри ядра?**
### **3.1. Структуры данных**
- **`struct shmid_kernel`** (для System V) или **`struct file`** (для POSIX) — хранят метаданные общей памяти.
- **Виртуальная файловая система (`tmpfs`)** — `/dev/shm` (для POSIX Shared Memory).

### **3.2. Маппинг в адресное пространство процесса**
- Когда процесс вызывает `mmap()` для shared memory, ядро:
  1. Находит или создаёт **объект shared memory**.
  2. Добавляет запись в **Page Table** процесса, указывающую на **физическую память**.
  3. При обращении к этой памяти CPU использует **TLB** для трансляции адресов.

### **3.3. Синхронизация**
- **Shared Memory не предоставляет встроенной синхронизации** (нужны мьютексы/семафоры).
- Обычно используется **POSIX Semaphores** или **Futex** (Fast Userspace Mutex).

---

## **4. Пример с синхронизацией (через семафоры)**
```c
#include <semaphore.h>

int main() {
    // 1. Создаём shared memory
    int fd = shm_open("/shared_mem", O_CREAT | O_RDWR, 0666);
    ftruncate(fd, 4096);
    void *ptr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

    // 2. Создаём семафор в shared memory
    sem_t *sem = (sem_t*)ptr;
    sem_init(sem, 1, 1);  // 1 = shared между процессами, 1 = начальное значение

    // 3. Используем семафор для синхронизации
    sem_wait(sem);  // Блокируем
    sprintf((char*)(sem + 1), "Safe write!");
    sem_post(sem);  // Разблокируем

    return 0;
}
```

---

## **5. Управление Shared Memory в Linux**
### **5.1. Просмотр существующих сегментов**
```bash
ipcs -m  # System V Shared Memory
ls /dev/shm  # POSIX Shared Memory
```

### **5.2. Удаление Shared Memory**
```bash
ipcrm -m <shmid>  # System V
rm /dev/shm/my_shared_mem  # POSIX
```

---

## **6. Плюсы и минусы Shared Memory**
| **Преимущества**               | **Недостатки**                     |
|--------------------------------|-----------------------------------|
| ⚡ **Самая быстрая IPC-методика** | 🔒 **Нужна синхронизация** (Race Conditions) |
| 📦 **Нет копирования данных**   | 🧩 **Сложнее в отладке**          |
| 🔄 **Подходит для больших данных** | ⚠ **Уязвимости безопасности** (если не настроены права) |

---

## **Вывод**
1. **POSIX Shared Memory (`shm_open` + `mmap`)** — современный и удобный способ.
2. **System V Shared Memory (`shmget`)** — устарел, но иногда встречается.
3. **Общая память маппится в адресное пространство процессов** через Page Tables.
4. **Синхронизация обязательна** (семафоры, мьютексы).

# **Устройство работы с памятью: MMU, struct, shared memory и другие механизмы**

В Linux работа с памятью — это сложная система, включающая аппаратные (MMU) и программные (структуры ядра, shared memory) компоненты. Разберём ключевые элементы.

---

## **1. Memory Management Unit (MMU) — аппаратный уровень**
**Функции MMU**:
- **Трансляция виртуальных адресов** в физические (через таблицы страниц).
- **Защита памяти**: изоляция процессов, контроль прав (R/W/X).
- **Кэширование TLB** (Translation Lookaside Buffer) для ускорения трансляции.

**Как работает**:
1. Процесс обращается к виртуальному адресу (например, `0x7ffeeb5a9000`).
2. MMU проверяет TLB → если нет записи, ищет в **таблицах страниц** (PGD → PUD → PMD → PTE).
3. Преобразует адрес в физический (`0x12345000`) или генерирует **page fault**, если страницы нет в RAM.

---

## **2. Структуры ядра Linux для управления памятью**
### **A. `struct mm_struct`**
Главная структура, описывающая память процесса (включается в `struct task_struct`).  
**Ключевые поля**:
```c
struct mm_struct {
    struct vm_area_struct *mmap;  // Список VMA (областей памяти)
    pgd_t *pgd;                   // Указатель на таблицу страниц
    atomic_t mm_users;            // Счётчик пользователей (потоков)
    atomic_t mm_count;            // Счётчик ссылок
    unsigned long start_code;     // Начало кода программы
    unsigned long end_data;       // Конец данных
    // ...
};
```
**Пример использования**:
- При `fork()` создаётся копия `mm_struct` (с Copy-On-Write для страниц).
- При `execve()` загружается новый `mm_struct`.

### **B. `struct vm_area_struct` (VMA)**
Описывает непрерывную область виртуальной памяти процесса.  
**Примеры VMA**:
- Сегмент кода (`.text`).
- Динамические библиотеки (`.so`).
- Отображённые файлы (`mmap`).
- Стек и куча (heap).

**Поля**:
```c
struct vm_area_struct {
    unsigned long vm_start;     // Начальный адрес
    unsigned long vm_end;       // Конечный адрес
    struct file *vm_file;       // Связанный файл (для mmap)
    pgprot_t vm_page_prot;      // Права (R/W/X)
    // ...
};
```

---

## **3. Shared Memory (разделяемая память)**
### **A. Механизмы в Linux**
1. **System V IPC (`shmget`/`shmat`)**  
   ```c
   key_t key = ftok("/tmp", 'A');
   int shmid = shmget(key, size, IPC_CREAT | 0666);
   void *ptr = shmat(shmid, NULL, 0);
   ```
   - Устаревший, но поддерживается.
   - Управляется через `ipcs`.

2. **POSIX Shared Memory (`shm_open`)**  
   ```c
   int fd = shm_open("/my_shm", O_CREAT | O_RDWR, 0666);
   ftruncate(fd, size);
   void *ptr = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
   ```
   - Более современный аналог.
   - Файлы в `/dev/shm`.

3. **Анонимное отображение (`mmap`)**  
   ```c
   void *ptr = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED | MAP_ANONYMOUS, -1, 0);
   ```
   - Общая память между родителем и потомком после `fork()`.

### **B. Как это работает на уровне ядра**
- **Создание**:  
  Ядро выделяет область в физической памяти и отображает её в виртуальное пространство процессов.
- **Синхронизация**:  
  Нужны мьютексы/семафоры (например, `pthread_mutex_t` с атрибутом `PTHREAD_PROCESS_SHARED`).

---

## **4. Работа с памятью из пользовательского пространства**
### **A. Выделение памяти**
1. **`malloc()` / `free()`**  
   - Использует `brk()` (для малых блоков) и `mmap()` (для больших).
   - Реализовано в glibc через arenas и bins.

2. **`mmap()`**  
   ```c
   void *ptr = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
   ```
   - Прямое отображение страниц (минуя кучу).

### **B. Пример: shared memory между процессами**
```c
// Процесс 1 (создаёт и пишет)
int fd = shm_open("/test", O_CREAT | O_RDWR, 0666);
ftruncate(fd, 4096);
int *ptr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
*ptr = 42;

// Процесс 2 (читает)
int fd = shm_open("/test", O_RDWR, 0);
int *ptr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
printf("%d\n", *ptr);  // 42
```

---

## **5. Управление памятью в ядре**
### **A. Slab Allocator**
- Оптимизирует выделение мелких объектов (например, `struct task_struct`).
- **Кэши** (`kmem_cache`): заранее созданные "пулы" объектов.

### **B. Buddy Allocator**
- Управляет физическими страницами (обычно по 4 КБ).
- Объединяет/разделяет блоки размером `2^n` страниц.

---

## **6. Инструменты для анализа**
1. **`pmap`**  
   Показывает карту памяти процесса:  
   ```bash
   pmap -x <PID>
   ```

2. **`/proc/<PID>/maps`**  
   ```bash
   cat /proc/self/maps
   ```
   Вывод:
   ```
   55f1c1b40000-55f1c1b41000 r-xp 00000000 08:01 123456 /bin/cat  # Код
   7ffeeb5a9000-7ffeeb5ca000 rw-p 00000000 00:00 0       [stack]  # Стек
   ```

3. **Valgrind**  
   Поиск утечек:  
   ```bash
   valgrind --leak-check=full ./program
   ```

---

## **Итог**
- **MMU** занимается трансляцией адресов и защитой.
- **`mm_struct` и VMA** управляют виртуальной памятью процесса.
- **Shared Memory** (`mmap`, `shm_open`) — обмен данными между процессами.
- **Ядро** использует Slab и Buddy аллокаторы для эффективного управления.

Для глубокого понимания рекомендуется изучать:
1. Исходники ядра (`mm/` в Linux).
2. Документацию по `mmap` и POSIX IPC.
3. **Книгу** "Understanding the Linux Kernel" (O’Reilly).