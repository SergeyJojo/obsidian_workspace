Это называется **Array Callable** или **Callable Array**. В PHP это один из способов представления вызываемого объекта (callable).

## 🎯 Что такое Array Callable?

```php
// Класс и метод
class Calculator {
    public function add($a, $b) {
        return $a + $b;
    }
    
    public static function multiply($a, $b) {
        return $a * $b;
    }
}

// Array callable для НЕстатического метода
$calculator = new Calculator();
$callable = [$calculator, 'add'];
$result = $callable(5, 3); // 8

// Array callable для СТАТИЧЕСКОГО метода  
$staticCallable = ['Calculator', 'multiply'];
$result = $staticCallable(5, 3); // 15

// Или с использованием ::
$staticCallable2 = [Calculator::class, 'multiply'];
```

---

## 📝 Синтаксисы

### 1. **Нестатические методы**
```php
[$object, 'methodName']
```

### 2. **Статические методы**
```php
['ClassName', 'methodName']
[ClassName::class, 'methodName']
```

### 3. **Родительские методы**
```php
[$this, 'parent::methodName']
```

---

## 🚀 Использование в практике

### С функциями, ожидающими callable:
```php
class UserService {
    public function processUsers(array $users, callable $filter) {
        return array_filter($users, $filter);
    }
}

class UserFilters {
    public function isActive(User $user) {
        return $user->isActive();
    }
}

$service = new UserService();
$filter = new UserFilters();

// Использование array callable
$activeUsers = $service->processUsers(
    $users, 
    [$filter, 'isActive']  // ← Array callable
);
```

### В обработчиках событий:
```php
class EventDispatcher {
    private $listeners = [];
    
    public function addListener(string $event, callable $listener) {
        $this->listeners[$event][] = $listener;
    }
    
    public function dispatch(string $event, $data) {
        foreach ($this->listeners[$event] ?? [] as $listener) {
            $listener($data);
        }
    }
}

class EmailService {
    public function sendWelcomeEmail(User $user) {
        echo "Welcome email sent to: " . $user->email;
    }
}

$dispatcher = new EventDispatcher();
$emailService = new EmailService();

// Регистрируем обработчик как array callable
$dispatcher->addListener(
    'user.registered', 
    [$emailService, 'sendWelcomeEmail']  // ← Array callable
);
```

---

## 🔧 Проверка callable

```php
$callable = [$object, 'methodName'];

// Проверяем, можно ли вызвать
if (is_callable($callable)) {
    $callable($arg1, $arg2);
} else {
    throw new Exception('Not callable');
}

// Или с method_exists для большей надежности
if (method_exists($object, 'methodName')) {
    $callable($arg1, $arg2);
}
```

---

## 🆚 Сравнение с другими callable типами

### 1. **Array Callable**
```php
[$object, 'method']      // Нестатический метод
['Class', 'method']      // Статический метод
```

### 2. **String Callable** (только для статических)
```php
'ClassName::methodName'
```

### 3. **Closure**
```php
function($arg) { return $arg * 2; }
```

### 4. **Invokable Object**
```php
class InvokableClass {
    public function __invoke($arg) {
        return $arg * 2;
    }
}
$callable = new InvokableClass();
```

---

## 💡 Практические примеры

### Laravel Routes:
```php
// Array callable в маршрутах
Route::get('/users', [UserController::class, 'index']);
Route::post('/users', [UserController::class, 'store']);
```

### Event Listeners:
```php
// В Laravel Event Service Provider
protected $listen = [
    'UserRegistered' => [
        [SendWelcomeEmail::class, 'handle'],
        [CreateUserProfile::class, 'handle'],
    ],
];
```

### Middleware:
```php
// В Laravel HTTP Kernel
protected $middleware = [
    \App\Http\Middleware\TrustProxies::class,
    \App\Http\Middleware\CheckForMaintenanceMode::class,
];
```

---

## ⚠️ Важные нюансы

### 1. **Видимость методов**
```php
class Example {
    public function publicMethod() {}
    protected function protectedMethod() {}  // ❌ Нельзя вызвать извне
    private function privateMethod() {}      // ❌ Нельзя вызвать извне
}

$callable = [$example, 'protectedMethod']; // Будет ошибка при вызове
```

### 2. **Статические vs Нестатические**
```php
class Math {
    public static function staticAdd($a, $b) { return $a + $b; }
    public function instanceAdd($a, $b) { return $a + $b; }
}

// Статический - не требует создания объекта
$staticCallable = ['Math', 'staticAdd'];
$staticCallable(2, 3); // 5

// Нестатический - требует объект
$math = new Math();
$instanceCallable = [$math, 'instanceAdd']; 
$instanceCallable(2, 3); // 5
```

### 3. **Наследование**
```php
class ParentClass {
    public function parentMethod() {}
}

class ChildClass extends ParentClass {
    public function childMethod() {}
}

$child = new ChildClass();

// Можно вызывать родительские методы
$callable = [$child, 'parentMethod']; // ✓ Работает
```

---

## 🎯 Modern PHP (8.0+)

### С union types:
```php
class Processor {
    public function process(array|callable $data) {
        if (is_callable($data)) {
            return $data();
        }
        return $data;
    }
}

$processor = new Processor();
$result = $processor->process([$service, 'methodName']);
```

### First-class callable syntax (PHP 8.1+):
```php
// Новый синтаксис - альтернатива array callable
$callable = $object->methodName(...);  // Вместо [$object, 'methodName']
$callable = ClassName::staticMethod(...);
```

---

## 💎 Итог

**Array Callable** - это мощный механизм PHP, который позволяет:
- ✅ Передавать методы объектов как callable
- ✅ Использовать в функциях типа `array_map`, `usort`, etc
- ✅ Создавать гибкие системы событий и middleware
- ✅ Работать с DI-контейнерами и маршрутизацией

**Синтаксис:** `[$object, 'method']` или `['ClassName', 'staticMethod']`

Это фундаментальная концепция, которую активно используют все современные PHP-фреймворки! 🚀