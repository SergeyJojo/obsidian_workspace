Отличный вопрос — тип **`callable`** в PHP один из самых интересных и гибких.  
Он обозначает **«вызваемый объект»**, то есть **всё, что можно вызвать как функцию**.

---

## 🔹 Что такое `callable`

Тип `callable` — это **специальный тип аргумента** (и возвращаемого значения), который говорит PHP:

> «Сюда нужно передать что-то, что можно вызвать как функцию».

---

## 🔹 Что может быть `callable`

`callable` охватывает несколько форм:

| Форма                                  | Пример                                                     | Комментарий                   |
| -------------------------------------- | ---------------------------------------------------------- | ----------------------------- |
| **Имя функции (строка)**               | `'strlen'`                                                 | Любая глобальная функция      |
| **Анонимная функция (Closure)**        | `function($x) { return $x * 2; }`                          | Лямбда                        |
| **Метод объекта (массив)**             | `[$obj, 'methodName']`                                     | Экземпляр и имя метода        |
| **Статический метод класса (массив)**  | `['ClassName', 'staticMethod']`                            | Класс и метод                 |
| **Строка с указанием класса и метода** | `'ClassName::staticMethod'`                                | Альтернатива массиву          |
| **Объект, реализующий `__invoke()`**   | `$obj`, если `class Obj { public function __invoke() {} }` | Тогда можно вызывать `$obj()` |

---

## 🔹 Пример использования `callable`

```php
function apply(callable $func, array $values): array {
    $result = [];
    foreach ($values as $v) {
        $result[] = $func($v);
    }
    return $result;
}

$double = function($x) { return $x * 2; };

print_r(apply($double, [1, 2, 3])); // [2, 4, 6]
```

---

## 🔹 Использование встроенных функций с `callable`

Многие функции PHP принимают `callable`:

- `array_map(callable $callback, array $array)`
    
- `array_filter(callable $callback, array $array)`
    
- `usort(array &$array, callable $callback)`
    

Пример:

```php
$nums = [3, 1, 4];
usort($nums, function($a, $b) { return $a <=> $b; });
print_r($nums); // [1, 3, 4]
```

---

## 🔹 Проверка на callable

Ты можешь проверить, можно ли что-то вызвать:

```php
if (is_callable($x)) {
    $x(); // безопасно вызывать
}
```

---

## 🔹 Пример с методом класса

```php
class Greeter {
    public static function sayHi($name) {
        echo "Hi, $name!\n";
    }
}

function greet(callable $callback) {
    $callback("Сергей");
}

greet(['Greeter', 'sayHi']); // Hi, Сергей!
```

---

## 🔹 Пример с `__invoke`

```php
class Multiplier {
    private int $factor;
    public function __construct($factor) { $this->factor = $factor; }
    public function __invoke($x) { return $x * $this->factor; }
}

$triple = new Multiplier(3);

echo $triple(10); // 30
```

`$triple` — это объект, но его можно вызывать как функцию, значит он `callable`.

---

## 🔹 Почему это важно

- Позволяет **писать гибкие и переиспользуемые функции** (функциональный стиль).
    
- Используется во **фреймворках**, **middleware**, **hooks**, **event listeners** и **замыканиях**.
    
- Позволяет инкапсулировать поведение и передавать его как данные.
    

---

Хочешь, я покажу **как PHP хранит и вызывает `callable` внутри Zend Engine** — т.е. как он определяет, можно ли что-то вызвать и какие структуры данных при этом используются (например, `zend_fcall_info`)?