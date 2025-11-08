Отлично 💪  
Начнём с файла `exceptions_and_errors.md` — я сделаю его **структурно**, как твой учебный конспект:  
чёткие блоки → теория → под капотом → примеры → что спрашивают на собесах → практика.

---

## 📄 `exceptions_and_errors.md`

````md
# Исключения и ошибки в PHP

---

## 1. Общая идея

PHP поддерживает две системы обработки сбоев:
1. **Ошибки уровня движка (Errors)** — сбои в рантайме или синтаксисе, возникающие при исполнении кода (например, `Division by zero`, `Undefined variable`).
2. **Исключения (Exceptions)** — объектная система ошибок, добавленная с PHP 5 и развивавшаяся до PHP 8, где ошибки стали наследоваться от `Throwable`.

Начиная с **PHP 7**, и ошибки, и исключения реализованы как объекты, наследующие интерфейс `Throwable`.

---

## 2. Иерархия классов ошибок

```text
Throwable
├── Exception          ← для логических исключений
│   ├── ErrorException
│   ├── LogicException
│   ├── RuntimeException
│   └── ...
└── Error              ← для фатальных ошибок движка
    ├── TypeError
    ├── ParseError
    ├── ArithmeticError
    └── AssertionError
````

**Ключевой момент**:  
всё, что можно поймать в `catch (Throwable $e)`, включает и `Error`, и `Exception`.

---

## 3. Обработка исключений

### Пример:

```php
try {
    $result = divide(10, 0);
} catch (DivisionByZeroError $e) {
    echo "Ошибка: " . $e->getMessage();
} catch (Throwable $e) {
    echo "Что-то пошло не так: " . $e->getMessage();
} finally {
    echo "Завершение блока try.";
}

function divide($a, $b) {
    if ($b === 0) {
        throw new DivisionByZeroError("Деление на ноль запрещено");
    }
    return $a / $b;
}
```

---

## 4. Отличие `Error` и `Exception`

|Критерий|Exception|Error|
|---|---|---|
|Используется для|Логических ошибок (бизнес-ошибок)|Сбоев движка и среды исполнения|
|Наследует|`Exception`|`Error`|
|Типичные примеры|`InvalidArgumentException`, `RuntimeException`|`TypeError`, `ParseError`, `DivisionByZeroError`|
|Обрабатывается в `catch`|Да|Да, начиная с PHP 7|
|Пример|Ошибка в логике|Ошибка типов или синтаксиса|

---

## 5. Преобразование ошибок в исключения

Чтобы перехватывать обычные ошибки (warning, notice) как исключения, можно использовать:

```php
set_error_handler(function ($severity, $message, $file, $line) {
    throw new ErrorException($message, 0, $severity, $file, $line);
});
```

Это полезно для унификации обработки ошибок в больших системах.

---

## 6. Встроенные типы ошибок

|Класс|Когда возникает|
|---|---|
|`Error`|Общие фатальные ошибки|
|`TypeError`|Несовпадение типов аргументов или возвращаемого значения|
|`ParseError`|Ошибки при парсинге кода (например, eval)|
|`ArithmeticError`|Арифметические ошибки|
|`DivisionByZeroError`|Деление на ноль|
|`AssertionError`|Нарушение `assert()`|

---

## 7. Ключевые методы `Throwable`

|Метод|Описание|
|---|---|
|`$e->getMessage()`|Текст ошибки|
|`$e->getCode()`|Код ошибки|
|`$e->getFile()`|Файл|
|`$e->getLine()`|Строка|
|`$e->getTrace()`|Стек вызовов (массив)|
|`$e->getTraceAsString()`|Стек вызовов в виде строки|
|`$e->getPrevious()`|Предыдущее исключение (цепочка)|

---

## 8. Цепочка исключений (Exception chaining)

Позволяет «оборачивать» одну ошибку другой, сохраняя контекст:

```php
try {
    $data = file_get_contents('config.json');
    if ($data === false) {
        throw new Exception("Не удалось прочитать файл");
    }
} catch (Exception $e) {
    throw new RuntimeException("Ошибка при загрузке конфигурации", 0, $e);
}
```

---

## 9. Fatal errors и необрабатываемые ошибки

- Некоторые ошибки не перехватываются (например, синтаксическая ошибка в коде до выполнения).
    
- Можно зарегистрировать глобальный обработчик:
    
    ```php
    set_exception_handler(function (Throwable $e) {
        error_log("Uncaught: " . $e->getMessage());
    });
    ```
    
- Для завершения работы корректно:
    
    ```php
    register_shutdown_function(function () {
        $error = error_get_last();
        if ($error) {
            error_log("Shutdown due to: " . json_encode($error));
        }
    });
    ```
    

---

## 10. Под капотом (runtime)

- Исключения реализованы через `zend_object`.
    
- При `throw` создаётся объект `zend_exception_object`, помещается в стек исключений движка.
    
- Вызов `catch` проверяет соответствие типа (`instanceof`) и снимает объект со стека.
    
- При необработанном исключении Zend Engine вызывает глобальный хендлер (fatal error).
    
- `Error` и `Exception` не связаны напрямую с SIGSEGV — это уровень PHP, не ОС.
    

---

## 11. Что спрашивают на собеседованиях

- Чем `Error` отличается от `Exception`?
    
- Можно ли поймать `ParseError`?
    
- Что делает `set_error_handler`?
    
- Что произойдет, если `throw` внутри `finally`?
    
- Можно ли поймать `TypeError` при вызове функции с неверным типом?
    
- Что вернёт `getPrevious()` и зачем она нужна?
    
- Почему стоит ловить `Throwable`, а не `Exception`?
    

---

## 12. Типичные ловушки

- `return` внутри `finally` перезапишет исключение → **не делай этого**.
    
- Если ты бросаешь исключение в `__destruct`, оно **будет проигнорировано**.
    
- Не все ошибки движка можно поймать (`memory_limit` overflow, segfault).
    
- В старом PHP 5 код вроде `catch (Exception $e)` не ловил фатальные ошибки.
    

---

## 13. Практика

**Задача 1:**  
Реализуй функцию, которая читает JSON-файл и при ошибке выбрасывает `InvalidArgumentException`, а при проблемах чтения — `RuntimeException`.

**Задача 2:**  
Напиши глобальный обработчик ошибок, который логирует всё в `error.log`, включая `Error` и `Exception`.

**Задача 3:**  
Создай свой класс `BusinessException`, унаследуй от `Exception`, добавь поле `httpCode` и обработай его в middleware (если делаешь веб-приложение).

---

## 14. Краткий вывод

- В PHP всё, что можно поймать — это `Throwable`.
    
- Логические ошибки — через `Exception`, системные — через `Error`.
    
- `set_error_handler` превращает нативные ошибки в исключения.
    
- Для продакшена всегда делай централизованный обработчик ошибок.
    
- Для собесов важно показать понимание **иерархии и runtime поведения**.
    

```

---

Хочешь, я следующим сделаю `strict_types.md` — он логично идёт после этой темы (тоже часто спрашивают вместе с `TypeError`)?
```