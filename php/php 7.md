Отличный вопрос, Сергей.  
PHP 7 — это был **революционный релиз**, вышедший в **2015 году**, и он полностью **переписал Zend Engine** (в версии 3). Это обновление не просто добавило синтаксис, а **радикально ускорило** язык и изменило его внутреннюю модель типов и памяти.

---

## ⚙️ Основные изменения в PHP 7

### 🔹 1. Новый движок — **Zend Engine 3**

Главное достижение:

- Производительность выросла **в 2–3 раза** по сравнению с PHP 5.6.
    
- Потребление памяти уменьшилось примерно на 30 %.
    
- Новый механизм **memory slots** и **copy-on-write**, оптимизированный под реальные веб-нагрузки.
    
- Более простое представление `zval` (внутреннего PHP-значения): уменьшилось количество аллокаций.
    

---

### 🔹 2. **Scalar type hints** (типизация аргументов и возвратов)

Наконец добавили строгие типы для скаляров:

```php
declare(strict_types=1);

function sum(int $a, int $b): int {
    return $a + $b;
}
```

Поддерживаются: `int`, `float`, `bool`, `string`, `array`, `callable`, `object`, `iterable`.  
⚠️ Если не включён `declare(strict_types=1)`, PHP пытается привести тип автоматически (coercive mode).

---

### 🔹 3. **Return type declarations**

Теперь можно указать тип возвращаемого значения:

```php
function getName(): string {
    return "Sergey";
}
```

---

### 🔹 4. **Null coalescing operator** `??`

Упрощает проверку на `null`:

```php
$username = $_GET['user'] ?? 'guest';
```

эквивалентно:

```php
$username = isset($_GET['user']) ? $_GET['user'] : 'guest';
```

---

### 🔹 5. **Spaceship operator** `<=>`

Упрощённый трёхсторонний оператор сравнения (возвращает -1, 0 или 1):

```php
echo 1 <=> 1; // 0
echo 1 <=> 2; // -1
echo 2 <=> 1; // 1
```

Используется, например, в `usort()` для компактных компараторов.

---

### 🔹 6. **Anonymous classes**

Теперь можно создавать классы «на лету»:

```php
$logger = new class {
    public function log($msg) {
        echo $msg;
    }
};

$logger->log('Hello');
```

Используется для DI, тестов, временных обёрток.

---

### 🔹 7. **Grouped use statements**

Импорт нескольких классов из одного namespace:

```php
use App\Controllers\{HomeController, UserController, ApiController};
```

---

### 🔹 8. **Error handling: Throwable, Error, Exception**

С PHP 7 система ошибок унифицирована:  
теперь _фатальные ошибки_ тоже можно перехватывать.

```php
try {
    nonExistentFunction();
} catch (Error $e) {
    echo "Ошибка: " . $e->getMessage();
}
```

➡️ Всё, что можно поймать в `catch`, реализует интерфейс `Throwable`.

---

### 🔹 9. **Engine Exceptions**

Теперь даже parse-time и fatal-ошибки генерируют исключения (`EngineException`), что делает обработку более предсказуемой.

---

### 🔹 10. **New operators:**

- **Null coalescing** `??`
    
- **Spaceship** `<=>`
    
- **Combined comparison** и улучшенные проверки типов
    

---

### 🔹 11. **Uniform Variable Syntax**

Более логичная трактовка сложных выражений:

До PHP 7:

```php
$foo = $bar['baz']->method;
```

в некоторых случаях трактовалось иначе, чем ожидалось.  
Теперь поведение строгое и единообразное.

---

### 🔹 12. **CSPRNG API (Cryptographically Secure PRNG)**

Добавлены:

```php
random_bytes()
random_int()
```

Безопасные источники случайных чисел (через OS entropy pool).

---

### 🔹 13. **Generator delegation (yield from)**

Теперь генераторы могут делегировать выполнение другим генераторам:

```php
function gen1() {
    yield 1;
    yield 2;
}

function gen2() {
    yield from gen1();
    yield 3;
}

foreach (gen2() as $v) echo $v; // 1 2 3
```

---

### 🔹 14. **Intdiv()**

Добавлена функция целочисленного деления:

```php
echo intdiv(10, 3); // 3
```

---

### 🔹 15. **Session options в session_start()**

Теперь можно задавать параметры прямо при запуске:

```php
session_start([
    'cookie_lifetime' => 3600,
    'read_and_close' => true,
]);
```

---

### 🔹 16. **Improved performance features**

- Оптимизация хеш-таблиц (меньше аллокаций).
    
- Улучшенный GC (garbage collector).
    
- Copy-on-write на уровне опкодов.
    
- Новый internal array API.
    
- Faster function calls (задействован inline cache для call frames).
    

---

### 🔹 17. **Removed legacy features**

- Удалён `mysql_*` API.
    
- Удалены старые конструкции типа `ext/ereg`.
    
- Удалены ASP-style теги `<% %>`.
    

---

### 🔹 18. **Существенные изменения под капотом**

- `zval` теперь всегда хранит тип и значение в одной структуре (ранее был pointer-wrapper).
    
- `HashTable` стал компактнее и быстрее.
    
- Минимизировали количество realocations при работе с массивами.
    

---

## ⚡ Результат

|Показатель|PHP 5.6|PHP 7.0|
|---|---|---|
|Скорость (req/s)|100 %|~200–300 %|
|Память|100 %|~70 %|
|Ошибки|Разрозненные|Throwable-система|
|Типизация|Ограниченная|Полноценная скалярная и возвращаемая|

---

Хочешь — могу показать **что именно изменилось в структуре `zval` и HashTable` между Zend Engine 2 и 3**, и как это ускорило интерпретатор (включая механизм reference count и COW)?  
Это поможет понять, почему PHP 7 стал быстрее в 2–3 раза.