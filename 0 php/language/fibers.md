**Файберы (Fibers)** в PHP — это механизм для управления выполнением кода на уровне одной нити выполнения (thread), позволяющий приостанавливать и возобновлять выполнение функций в любом месте. По сути, это "облегченные потоки" (green threads), реализованные на уровне PHP, а не операционной системы.

## 🎯 **Что такое файберы простыми словами**

Представьте, что ваша PHP-функция — это книга, а файбер — это закладка, которая позволяет:
- **Поставить на паузу** чтение на любом месте
- **Переключиться** на другую "книгу" (файбер)
- **Вернуться** к чтению с того же места

## 🏗️ **Базовое использование**

### **Создание и управление файбером:**
```php
<?php
$fiber = new Fiber(function(): void {
    echo "Файбер запущен\n";
    
    Fiber::suspend('Приостановлен'); // Ставим на паузу
    
    echo "Файбер возобновлен\n";
    
    Fiber::suspend('Снова приостановлен');
    
    echo "Файбер завершен\n";
    
    return 'Результат';
});

// Запускаем файбер до первого suspend
$suspendedValue = $fiber->start();
echo "Файбер вернул: $suspendedValue\n"; // "Файбер вернул: Приостановлен"

// Возобновляем выполнение
$suspendedValue = $fiber->resume();
echo "Файбер вернул: $suspendedValue\n"; // "Файбер вернул: Снова приостановлен"

// Возобновляем последний раз
$result = $fiber->resume();
echo "Файбер завершился с: $result\n"; // "Файбер завершился с: Результат"

// Вывод:
// Файбер запущен
// Файбер вернул: Приостановлен
// Файбер возобновлен
// Файбер вернул: Снова приостановлен
// Файбер завершен
// Файбер завершился с: Результат
```

## 🔄 **Практические примеры использования**

### **Пример 1: Кооперативная многозадачность**
```php
<?php
function createTask(string $name, int $steps): Fiber {
    return new Fiber(function() use ($name, $steps): void {
        for ($i = 1; $i <= $steps; $i++) {
            echo "Задача '$name': шаг $i/$steps\n";
            Fiber::suspend(); // Передаем управление другому файберу
        }
        echo "Задача '$name': ВЫПОЛНЕНО\n";
    });
}

// Создаем несколько "задач"
$tasks = [
    'A' => createTask('Фоновая загрузка', 3),
    'B' => createTask('Обработка данных', 4),
    'C' => createTask('Отправка email', 2),
];

// Запускаем все задачи
foreach ($tasks as $task) {
    $task->start();
}

// Кооперативный планировщик
do {
    $allCompleted = true;
    
    foreach ($tasks as $task) {
        if (!$task->isTerminated()) {
            $allCompleted = false;
            $task->resume();
        }
    }
} while (!$allCompleted);

// Вывод (примерный порядок):
// Задача 'Фоновая загрузка': шаг 1/3
// Задача 'Обработка данных': шаг 1/4  
// Задача 'Отправка email': шаг 1/2
// Задача 'Фоновая загрузка': шаг 2/3
// Задача 'Обработка данных': шаг 2/4
// ... и так далее
```

### **Пример 2: Асинхронные операции с возвратом значений**
```php
<?php
class AsyncOperation {
    public static function fetchData(string $url, int $delayMs): Fiber {
        return new Fiber(function() use ($url, $delayMs): string {
            // Имитируем асинхронную операцию (например, HTTP-запрос)
            echo "Начинаем загрузку: $url\n";
            
            for ($i = 1; $i <= 3; $i++) {
                usleep($delayMs * 1000 / 3); // Имитация задержки
                echo "Загрузка $url: " . ($i * 33) . "%\n";
                Fiber::suspend(['progress' => $i * 33]);
            }
            
            return "Данные с $url (размер: " . rand(100, 1000) . " KB)";
        });
    }
}

// Запускаем несколько "асинхронных" операций
$operations = [
    'api' => AsyncOperation::fetchData('https://api.example.com', 300),
    'cdn' => AsyncOperation::fetchData('https://cdn.example.com', 200),
];

// Запускаем
foreach ($operations as $op) {
    $op->start();
}

// Обрабатываем результаты по мере готовности
$results = [];
while (!empty($operations)) {
    foreach ($operations as $key => $fiber) {
        if (!$fiber->isTerminated()) {
            $progress = $fiber->resume();
            echo "Прогресс $key: {$progress['progress']}%\n";
        } else {
            $results[$key] = $fiber->getReturn();
            unset($operations[$key]);
        }
    }
}

echo "Все операции завершены:\n";
print_r($results);
```

## 🚀 **Реализация простого event loop с файберами**

```php
<?php
class SimpleEventLoop {
    private array $fibers = [];
    private array $timers = [];
    
    public function add(Fiber $fiber): void {
        $this->fibers[] = $fiber;
    }
    
    public function setTimeout(callable $callback, int $delayMs): void {
        $this->timers[] = [
            'execute_at' => microtime(true) + ($delayMs / 1000),
            'callback' => $callback
        ];
    }
    
    public function run(): void {
        while (!empty($this->fibers) || !empty($this->timers)) {
            // Выполняем файберы
            foreach ($this->fibers as $i => $fiber) {
                if (!$fiber->isStarted()) {
                    $fiber->start();
                } elseif (!$fiber->isTerminated()) {
                    $fiber->resume();
                } else {
                    unset($this->fibers[$i]);
                }
            }
            
            // Проверяем таймеры
            $now = microtime(true);
            foreach ($this->timers as $i => $timer) {
                if ($now >= $timer['execute_at']) {
                    $timer['callback']();
                    unset($this->timers[$i]);
                }
            }
            
            usleep(1000); // Небольшая пауза
        }
    }
}

// Использование
$loop = new SimpleEventLoop();

// Добавляем файберы
$loop->add(new Fiber(function(): void {
    for ($i = 0; $i < 5; $i++) {
        echo "Файбер 1: итерация $i\n";
        Fiber::suspend();
    }
}));

$loop->add(new Fiber(function(): void {
    for ($i = 0; $i < 3; $i++) {
        echo "Файбер 2: работа $i\n";
        Fiber::suspend();
    }
}));

// Добавляем таймер
$loop->setTimeout(function() {
    echo "ТАЙМЕР: Прошло 100ms!\n";
}, 100);

$loop->run();
```

## ⚡ **Файберы vs Генераторы**

### **Сходства:**
- Оба могут приостанавливать выполнение
- Оба сохраняют состояние между вызовами

### **Отличия:**
```php
<?php
// Генератор (более ограниченный)
function simpleGenerator(): Generator {
    yield 'value1';
    yield 'value2';
    return 'final';
}

$gen = simpleGenerator();
foreach ($gen as $value) {
    echo $value . "\n";
}
echo $gen->getReturn(); // final

// Файбер (полный контроль)
$fiber = new Fiber(function(): string {
    Fiber::suspend('value1');
    Fiber::suspend('value2'); 
    return 'final';
});

$fiber->start();
echo $fiber->resume() . "\n"; // value1
echo $fiber->resume() . "\n"; // value2
echo $fiber->resume() . "\n"; // final
```

## 🛡️ **Обработка ошибок в файберах**

```php
<?php
try {
    $fiber = new Fiber(function(): void {
        echo "Начало выполнения\n";
        
        // Имитация ошибки
        if (rand(0, 1)) {
            throw new RuntimeException('Что-то пошло не так!');
        }
        
        Fiber::suspend('Успех');
        echo "Это не выполнится при ошибке\n";
    });
    
    $fiber->start();
    $result = $fiber->resume();
    echo "Результат: $result\n";
    
} catch (Throwable $e) {
    echo "Поймана ошибка: " . $e->getMessage() . "\n";
}

// Проверка состояния файбера
if ($fiber->isTerminated()) {
    echo "Файбер завершен\n";
} elseif ($fiber->isSuspended()) {
    echo "Файбер приостановлен\n";
} elseif ($fiber->isStarted()) {
    echo "Файбер запущен\n";
}
```

## 💡 **Практические применения файберов**

### **1. Асинхронные HTTP-запросы**
```php
<?php
class AsyncHttp {
    public static function get(string $url): Fiber {
        return new Fiber(function() use ($url): string {
            // Имитация асинхронного запроса
            for ($i = 0; $i < 3; $i++) {
                Fiber::suspend(['status' => 'pending', 'progress' => $i * 33]);
            }
            return "Ответ от $url: {данные}";
        });
    }
}
```

### **2. Игровые циклы**
```php
<?php
class GameEntity {
    public function __construct(private string $name) {}
    
    public function createAi(): Fiber {
        return new Fiber(function(): void {
            while (true) {
                echo "{$this->name}: Думаю...\n";
                Fiber::suspend();
                
                echo "{$this->name}: Действую\n";
                Fiber::suspend();
            }
        });
    }
}
```

## ✅ **Преимущества файберов:**

- **🚀 Полный контроль** над выполнением
- **💡 Кооперативная многозадачность** без многопоточности
- **🎯 Более читаемый код** для асинхронных операций
- **⚡ Эффективное использование ресурсов**

## ⚠️ **Ограничения:**

- **Только в PHP 8.1+**
- **Не настоящая параллельность** - один поток выполнения
- **Требует ручного управления** переключениями

**Файберы** — это мощный инструмент для создания кооперативной многозадачности, асинхронных операций и сложных flow-процессов в PHP!