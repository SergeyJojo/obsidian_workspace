В PHP **уровни видимости (visibility)** определяют, **где можно обращаться к свойствам и методам класса**. Они работают **только в контексте ООП** (классов) и применяются к:

- **Свойствам** (`$property`)
- **Методам** (`function method()`)
- **Константам класса** (`const NAME = ...`)

---

## 🔑 Три уровня видимости

| Уровень | Где доступен |
|--------|--------------|
| **`public`** | Везде: внутри класса, в наследниках, извне |
| **`protected`** | Только внутри класса и его **наследников** |
| **`private`** | **Только внутри самого класса** (даже наследники не видят) |

---

### 📌 Пример
```php
class ParentClass {
    public    $publicProp    = 'public';
    protected $protectedProp = 'protected';
    private   $privateProp   = 'private';

    public function showAll() {
        echo $this->publicProp;    // ✅
        echo $this->protectedProp; // ✅
        echo $this->privateProp;   // ✅
    }
}

class ChildClass extends ParentClass {
    public function showInherited() {
        echo $this->publicProp;    // ✅
        echo $this->protectedProp; // ✅
        // echo $this->privateProp; // ❌ Fatal error!
    }
}

$obj = new ChildClass();
echo $obj->publicProp;    // ✅
// echo $obj->protectedProp; // ❌ Fatal error!
// echo $obj->privateProp;   // ❌ Fatal error!
```

---

## 🧱 Подробнее по каждому уровню

### 1. `public`
- Доступен **абсолютно везде**.
- Используется для **публичного API класса**.
- По умолчанию (если не указано иное) — **всё `public`** (но это плохая практика!).

```php
class ApiClient {
    public function sendRequest() { } // публичный метод
}
```

---

### 2. `protected`
- Доступен **внутри класса и всех его потомков**.
- Используется для **внутренней логики, которую должны наследовать дочерние классы**.

```php
abstract class Vehicle {
    protected function startEngine() {
        echo "Engine started";
    }
}

class Car extends Vehicle {
    public function drive() {
        $this->startEngine(); // ✅ можно вызвать
    }
}
```

---

### 3. `private`
- Доступен **только внутри того класса, где объявлен**.
- Даже **класс-наследник не имеет доступа**.
- Используется для **инкапсуляции деталей реализации**.

```php
class Database {
    private $connection;

    private function connect() {
        $this->connection = new PDO(...);
    }

    public function query($sql) {
        if (!$this->connection) $this->connect(); // ✅
        // ...
    }
}
```

> 💡 Если наследник попытается вызвать `connect()` — будет **ошибка времени выполнения**.

---

## ⚠️ Важные нюансы

### 1. **Конструкторы и деструкторы**
Могут быть `private` или `protected` — это используется в **паттернах**:
- `private __construct()` → **Singleton**
- `protected __construct()` → **Factory**

```php
class Singleton {
    private static ?self $instance = null;
    private function __construct() { } // запрещаем new

    public static function getInstance(): self {
        if (self::$instance === null) {
            self::$instance = new self();
        }
        return self::$instance;
    }
}
```

---

### 2. **Константы класса**
Тоже поддерживают видимость (с PHP 7.1+):

```php
class Config {
    public    const PUBLIC_API = 'https://api.example.com';
    protected const INTERNAL_KEY = 'secret123';
    private   const DB_PASSWORD = 'supersecret';
}
```

---

### 3. **Статические свойства/методы**
Видимость работает **точно так же**:

```php
class Counter {
    private static int $count = 0;

    public static function increment(): void {
        self::$count++; // ✅
    }
}
```

---

### 4. **Интерфейсы и абстрактные классы**
- В **интерфейсах** всё **автоматически `public`** (нельзя указать `protected`/`private`).
- В **абстрактных классах** — можно использовать все уровни.

---

## 🆚 Сравнение с другими языками

| Язык | `private` | `protected` | Заметки |
|------|----------|------------|--------|
| **PHP** | Только в своём классе | Класс + наследники | Точно как в Java/C# |
| **JavaScript** | `#field` (новый синтаксис) | Нет нативной поддержки | Через замыкания или WeakMap |
| **Python** | `_field` (соглашение), `__field` (name mangling) | То же | Нет настоящей приватности |

---

## ✅ Рекомендации (best practices)

1. **По умолчанию делай всё `private`** — расширяй видимость только при необходимости.
2. Используй `protected`, если **точно знаешь**, что наследники должны иметь доступ.
3. `public` — только для **публичного интерфейса** класса.
4. Избегай `public` свойств — лучше использовать **геттеры/сеттеры**.

```php
// ПЛОХО:
class User {
    public $name;
}

// ХОРОШО:
class User {
    private string $name;
    public function getName(): string { return $this->name; }
    public function setName(string $name): void { $this->name = $name; }
}
```

---

## 💡 На собеседовании спросят:
- В чём разница между `protected` и `private`?
- Может ли наследник получить доступ к `private` методу родителя? → **Нет**.
- Что будет, если не указать уровень видимости? → **`public`** (но это устаревшее поведение — всегда указывай явно!).

---

Готово! Теперь ты знаешь всё про уровни видимости в PHP — от синтаксиса до best practices. 🚀