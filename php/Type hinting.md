**Type hinting** (в PHP с 7 версии — точнее называется **type declarations**) — это механизм, позволяющий явно указывать **ожидаемые типы данных** для аргументов функций, возвращаемых значений и свойств классов.

Он помогает сделать код **более предсказуемым, читаемым и безопасным**, так как PHP выполняет **проверку типов во время выполнения**.

---

### 🔹 Пример без type hinting

```php
function add($a, $b) {
    return $a + $b;
}

echo add("3", 5); // PHP автоматически приведёт "3" к числу, выведет 8
```

➡️ Код сработает, но возможны скрытые ошибки — PHP попытается "угадать" типы.

---

### 🔹 Пример с type hinting

```php
function add(int $a, int $b): int {
    return $a + $b;
}

echo add(3, 5); // ✅ OK
echo add("3", 5); // ❌ Fatal error: Argument must be of type int
```

➡️ Теперь PHP **строго проверяет** типы.

---

### 🔹 Где можно указывать типы

|Место|Пример|С версии|
|---|---|---|
|Аргументы функций|`function f(int $x)`|PHP 7|
|Возвращаемое значение|`function f(): string`|PHP 7|
|Свойства класса|`public string $name;`|PHP 7.4|
|Псевдотипы (`array`, `callable`, `object`, `iterable`, `mixed`, `never`)|`function f(array $a)`|PHP 7–8|
|Union types (несколько вариантов)|`function f(int|string $x)`|
|Intersection types (все интерфейсы сразу)|`function f(A&B $obj)`|PHP 8.1|
|Nullable types|`?int $id` или `function f(): ?string`|PHP 7.1|

---

### 🔹 Пример: Nullable и Union types

```php
function greet(?string $name): string {
    return $name ? "Hello, $name" : "Hello, stranger";
}

function parseValue(int|string $value): string {
    return (string)$value;
}
```

---

### 🔹 Класс и объекты

```php
class User {}

function register(User $u): void {
    echo "Registered user";
}

register(new User()); // ✅ OK
register("test");     // ❌ TypeError
```

---

### 🔹 Зачем использовать type hinting

- ✅ Раннее обнаружение ошибок
    
- ✅ Автодополнение и проверка типов в IDE
    
- ✅ Предсказуемость поведения функций
    
- ✅ Упрощает рефакторинг и поддержку
    

---

Хочешь — я покажу, **как PHP реализует type hinting под капотом (в Zend Engine)**, то есть как он хранит типы и проверяет их при вызове функции?