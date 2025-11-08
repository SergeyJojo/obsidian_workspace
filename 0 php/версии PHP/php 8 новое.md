Хорошо, Сергей — давай разберём **по существу и глубоко**, что реально **нового и важного** появилось в PHP 8.0, 8.1, 8.2 и 8.3.  
PHP 8 — это не просто косметические улучшения, а серьёзное обновление движка **Zend Engine 4**, где появились _JIT_, улучшенная типизация и более строгая модель исполнения.

---

## ⚙️ PHP 8.0 (релиз: 2020 г.)

### 🔹 1. JIT — Just-In-Time Compiler

- Реализован в Zend Engine 4.
    
- Компилирует часто вызываемый байткод PHP в нативный машинный код (x86/ARM).
    
- Даёт выигрыш не столько для веб-запросов, сколько для вычислительно тяжёлых задач (image processing, ML, парсеры).
    
- Два режима:
    
    - `function` — компилирует отдельные функции;
        
    - `tracing` — оптимизирует “горячие пути”.
        

---

### 🔹 2. Union types

Позволяют указывать несколько допустимых типов:

```php
function parse(int|string $value) {}
```

---

### 🔹 3. Constructor property promotion

Сокращает шаблонный код в классах:

```php
class User {
    public function __construct(
        public string $name,
        private int $age
    ) {}
}
```

---

### 🔹 4. Named arguments

Позволяют вызывать функцию по имени параметров:

```php
setcookie(name: "id", value: "123", httponly: true);
```

---

### 🔹 5. Match expression (аналог switch, но безопасный)

```php
$result = match($status) {
    200, 201 => 'OK',
    404      => 'Not found',
    default  => 'Error',
};
```

- Возвращает значение.
    
- Проверяет тип строго (без приведения типов, как `switch`).
    

---

### 🔹 6. Nullsafe operator

```php
$username = $user?->profile?->name;
```

Не бросает ошибку, если `$user` или `$profile` — `null`.

---

### 🔹 7. Строгие сравнения с `throw` в выражениях

```php
$value = $data['key'] ?? throw new Exception('Missing key');
```

---

### 🔹 8. Атрибуты (аннотации нового поколения)

```php
#[Route("/home", methods: ["GET"])]
function home() {}
```

- Замена комментариев `@Annotation`.
    
- Используется фреймворками: Symfony, Laravel, Doctrine.
    

---

## ⚙️ PHP 8.1 (2021 г.)

### 🔹 1. Enums (перечисления)

```php
enum Status: string {
    case Active = 'active';
    case Inactive = 'inactive';
}
```

---

### 🔹 2. Fibers

Низкоуровневый механизм для кооперативной многозадачности — позволяет строить асинхронные фреймворки (Amp, ReactPHP).

```php
$fiber = new Fiber(fn() => "ok");
echo $fiber->start(); // "ok"
```

---

### 🔹 3. Readonly properties

```php
class Point {
    public readonly int $x;
    public function __construct($x) { $this->x = $x; }
}
```

После инициализации больше изменить нельзя.

---

### 🔹 4. Intersection types

```php
function f(A&B $obj) {}
```

---

### 🔹 5. `never` return type

Функция не возвращает ничего, а только завершает выполнение (через `exit`, `throw` и т.д.).

```php
function fail(string $msg): never {
    throw new Exception($msg);
}
```

---

## ⚙️ PHP 8.2 (2022 г.)

### 🔹 1. Readonly classes

```php
readonly class Config {
    public function __construct(public string $name) {}
}
```

Все свойства становятся `readonly` по умолчанию.

---

### 🔹 2. Disjunctive Normal Form types (DNF)

Поддержка сложных объединений:

```php
(A&B) | (C&D)
```

---

### 🔹 3. `true` as standalone type

Можно явно указывать `true`:

```php
function isReady(): true { return true; }
```

---

### 🔹 4. Constants in traits

Теперь можно определять константы прямо в трейтах.

---

### 🔹 5. Deprecations

- Удалены динамические свойства (теперь требуют `#[\AllowDynamicProperties]`).
    
- Deprecated `utf8_encode`, `utf8_decode`.
    

---

## ⚙️ PHP 8.3 (2023 г.)

### 🔹 1. Typed class constants

```php
class C {
    public const int ID = 42;
}
```

---

### 🔹 2. Deep cloning of readonly properties

`clone` теперь корректно работает с `readonly`.

---

### 🔹 3. New `json_validate()` function

Проверяет JSON без его парсинга:

```php
if (json_validate($data)) { ... }
```

---

### 🔹 4. Dynamic class constant fetch

```php
$class = Foo::class;
echo $class::BAR;
```

---

### 🔹 5. `Randomizer` API (новый движок случайных чисел)

```php
$r = new Random\Randomizer();
echo $r->getInt(1, 100);
```

---

### 🔹 6. Performance

- Оптимизация JIT.
    
- Ускорено сравнение массивов и вызов методов.
    
- Уменьшено потребление памяти при итерации.
    

---

## ⚙️ PHP 8.4 (предстоящий, конец 2024 г.)

> Уже в RFC: Property hooks (`get`/`set` внутри класса, как в C#), и улучшения JIT в Zend Engine 5.

---

Хочешь — я покажу **что реально меняется внутри Zend Engine между PHP 7 и 8** (например, новый `zend_type`, оптимизация HashTable, улучшение GC и copy-on-write механизма)?  
Это даст представление, почему PHP 8 быстрее и безопаснее на низком уровне.