**Позднее статическое связывание (Late Static Binding - LSB)** в PHP — это механизм, который позволяет статическим методам и свойствам знать, какой класс их вызвал в контексте наследования, а не только в каком классе они определены.

## 🆚 **Проблема: Раннее статическое связывание**

Сначала посмотрим на проблему, которую решает LSB:

```php
class ParentClass {
    public static function who() {
        return __CLASS__;
    }
    
    public static function test() {
        echo self::who(); // Всегда вернет 'ParentClass'
    }
}

class ChildClass extends ParentClass {
    public static function who() {
        return __CLASS__;
    }
}

ChildClass::test(); // Выведет: ParentClass (проблема!)
```

**Почему так происходит?**
- `self::` всегда ссылается на класс, где оно написано
- `__CLASS__` всегда возвращает класс, где оно объявлено

## 🎯 **Решение: Позднее статическое связывание**

### **Ключевое слово `static::`**
```php
class ParentClass {
    public static function who() {
        return __CLASS__;
    }
    
    public static function test() {
        echo static::who(); // Будет вызывать who() из вызывающего класса
    }
}

class ChildClass extends ParentClass {
    public static function who() {
        return __CLASS__;
    }
}

ChildClass::test(); // Выведет: ChildClass (работает!)
```

## 💻 **Практические примеры**

### **Пример 1: Паттерн Singleton**
```php
class Singleton {
    private static $instances = [];
    
    protected function __construct() {}
    protected function __clone() {}
    
    public static function getInstance() {
        $class = static::class; // Получаем вызывающий класс
        
        if (!isset(self::$instances[$class])) {
            self::$instances[$class] = new static();
        }
        
        return self::$instances[$class];
    }
}

class Database extends Singleton {
    public function connect() {
        return "Connected to database";
    }
}

class Cache extends Singleton {
    public function set($key, $value) {
        return "Cache set: $key = $value";
    }
}

$db1 = Database::getInstance();
$db2 = Database::getInstance();
$cache = Cache::getInstance();

var_dump($db1 === $db2); // true (один и тот же экземпляр Database)
var_dump($db1 === $cache); // false (разные классы)
```

### **Пример 2: ActiveRecord Pattern**
```php
class Model {
    protected static $table;
    
    public static function getTable() {
        if (static::$table) {
            return static::$table;
        }
        
        // Автоматическое определение имени таблицы
        $class = static::class;
        return strtolower(str_replace('App\\Models\\', '', $class)) . 's';
    }
    
    public static function find($id) {
        $table = static::getTable();
        return "SELECT * FROM {$table} WHERE id = {$id}";
    }
}

class User extends Model {
    protected static $table = 'users';
}

class Post extends Model {
    // table будет автоматически 'posts'
}

echo User::find(1);    // SELECT * FROM users WHERE id = 1
echo Post::find(5);    // SELECT * FROM posts WHERE id = 5
```

### **Пример 3: Фабричный метод**
```php
abstract class Product {
    abstract public function getName();
    
    public static function create() {
        return new static(); // Создает объект вызывающего класса
    }
}

class Book extends Product {
    public function getName() {
        return "Book Product";
    }
}

class Electronics extends Product {
    public function getName() {
        return "Electronics Product";
    }
}

$book = Book::create();
$electronics = Electronics::create();

echo $book->getName();        // Book Product
echo $electronics->getName(); // Electronics Product
```

## 🔧 **Сравнение `self::` vs `static::` vs `parent::`**

```php
class A {
    public static function test() {
        echo "self: " . self::class . "\n";
        echo "static: " . static::class . "\n";
        echo "CLASS: " . __CLASS__ . "\n";
    }
}

class B extends A {}
class C extends B {}

C::test();
// Вывод:
// self: A
// static: C  
// CLASS: A
```

## 🚀 **Продвинутое использование**

### **Пример с конструктором:**
```php
class BaseEntity {
    protected $data = [];
    
    public function __construct($data = []) {
        $this->data = $data;
    }
    
    public static function create($data) {
        return new static($data); // Создает объект вызывающего класса
    }
    
    public function toArray() {
        return $this->data;
    }
}

class User extends BaseEntity {
    public function getEmail() {
        return $this->data['email'] ?? null;
    }
}

class Product extends BaseEntity {
    public function getPrice() {
        return $this->data['price'] ?? null;
    }
}

$user = User::create(['email' => 'test@mail.com']);
$product = Product::create(['price' => 100]);

echo get_class($user);    // User
echo get_class($product); // Product
```

### **Пример с трейтами:**
```php
trait Cacheable {
    public static function getCacheKey() {
        return 'cache_' . static::class;
    }
}

class Article {
    use Cacheable;
}

class News {
    use Cacheable;
}

echo Article::getCacheKey(); // cache_Article
echo News::getCacheKey();    // cache_News
```

## ⚠️ **Ограничения и особенности**

### **Не работает с приватными методами:**
```php
class ParentClass {
    private static function secret() {
        return "Parent secret";
    }
    
    public static function test() {
        return static::secret(); // Ошибка!
    }
}

class ChildClass extends ParentClass {
    private static function secret() {
        return "Child secret";
    }
}

ChildClass::test(); // Fatal error: Call to private method
```

### **Работает с защищенными методами:**
```php
class ParentClass {
    protected static function protectedMethod() {
        return "Parent protected";
    }
    
    public static function test() {
        return static::protectedMethod(); // Работает!
    }
}

class ChildClass extends ParentClass {
    protected static function protectedMethod() {
        return "Child protected";
    }
}

echo ChildClass::test(); // Child protected
```

## 💡 **Когда использовать LSB?**

- ✅ **Паттерн Singleton** для наследуемых классов
- ✅ **Active Record** и ORM системы
- ✅ **Фабричные методы** в абстрактных классах
- ✅ **Создание fluent interfaces**
- ✅ **Любые статические методы**, которые могут быть переопределены в потомках

## 🎯 **Итог:**

- **`self::`** — раннее связывание (compile-time)
- **`static::`** — позднее связывание (runtime)
- **LSB делает наследование статических методов полезным и предсказуемым**
- **Используйте `static::` когда хотите, чтобы метод вел себя полиморфно**

**Позднее статическое связывание** — это мощный инструмент для создания гибких и расширяемых архитектур в PHP!