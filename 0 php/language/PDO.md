**PDO (PHP Data Objects)** — это современный, безопасный и универсальный интерфейс для работы с базами данных в PHP. Это не просто "еще один способ", а **рекомендуемый подход** вместо устаревших функций типа `mysql_*`, `mysqli_*` или `pg_*`.

---

### Простая аналогия

Представьте, что PDO — это **универсальный переводчик** между PHP и разными базами данных:

- Без PDO: Вам нужен отдельный переводчик для MySQL, другой для PostgreSQL, третий для SQLite...
- С PDO: Один универсальный переводчик, который знает все "языки" баз данных.

---

### Ключевые преимущества PDO

#### 1. **Универсальность**
Один код работает с разными СУБД:
```php
// Подключение к разным базам с минимальными изменениями
$pdo_mysql = new PDO('mysql:host=localhost;dbname=test', 'user', 'pass');
$pdo_sqlite = new PDO('sqlite:/path/to/database.sqlite');
$pdo_pgsql = new PDO('pgsql:host=localhost;dbname=test');
```

#### 2. **Защита от SQL-инъекций через подготовленные выражения**
```php
// НЕПРАВИЛЬНО (уязвимо к инъекциям)
$sql = "SELECT * FROM users WHERE email = '{$_POST['email']}'";

// ПРАВИЛЬНО (безопасно с PDO)
$stmt = $pdo->prepare("SELECT * FROM users WHERE email = :email");
$stmt->execute(['email' => $_POST['email']]);
$user = $stmt->fetch();
```

#### 3. **Объектно-ориентированный интерфейс**
```php
// Современный ООП стиль
$stmt = $pdo->prepare("INSERT INTO users (name, email) VALUES (?, ?)");
$stmt->execute(['Иван', 'ivan@mail.ru']);
```

---

### Основное использование PDO

#### Подключение к базе данных
```php
try {
    $pdo = new PDO(
        'mysql:host=localhost;dbname=my_database;charset=utf8mb4',
        'username',
        'password',
        [
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION, // Важно: исключения при ошибках
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC // Получать ассоциативные массивы
        ]
    );
} catch (PDOException $e) {
    die("Ошибка подключения: " . $e->getMessage());
}
```

#### CRUD операции с PDO

**CREATE (Создание)**
```php
$stmt = $pdo->prepare("INSERT INTO users (name, email, age) VALUES (?, ?, ?)");
$stmt->execute(['Анна', 'anna@example.com', 25]);
$newUserId = $pdo->lastInsertId(); // Получить ID новой записи
```

**READ (Чтение)**
```php
// Получить одну запись
$stmt = $pdo->prepare("SELECT * FROM users WHERE id = ?");
$stmt->execute([1]);
$user = $stmt->fetch();

// Получить все записи
$stmt = $pdo->prepare("SELECT * FROM users WHERE age > ?");
$stmt->execute([18]);
$users = $stmt->fetchAll();

// Получить значение одного столбца
$stmt = $pdo->prepare("SELECT COUNT(*) FROM users");
$stmt->execute();
$count = $stmt->fetchColumn();
```

**UPDATE (Обновление)**
```php
$stmt = $pdo->prepare("UPDATE users SET email = ?, age = ? WHERE id = ?");
$stmt->execute(['new_email@example.com', 30, 1]);
$affectedRows = $stmt->rowCount(); // Количество измененных строк
```

**DELETE (Удаление)**
```php
$stmt = $pdo->prepare("DELETE FROM users WHERE id = ?");
$stmt->execute([5]);
```

---

### Подготовленные выражения (Prepared Statements)

#### Два стиля использования:

**Именованные плейсхолдеры:**
```php
$stmt = $pdo->prepare("
    INSERT INTO products (name, price, category) 
    VALUES (:name, :price, :category)
");
$stmt->execute([
    'name' => 'Ноутбук',
    'price' => 50000,
    'category' => 'electronics'
]);
```

**Позиционные плейсхолдеры:**
```php
$stmt = $pdo->prepare("
    UPDATE products SET price = ?, stock = ? WHERE id = ?
");
$stmt->execute([45000, 15, 42]);
```

#### Множественное выполнение:
```php
$stmt = $pdo->prepare("INSERT INTO users (name, email) VALUES (?, ?)");

$users = [
    ['Алексей', 'alex@mail.ru'],
    ['Мария', 'maria@mail.ru'],
    ['Петр', 'petr@mail.ru']
];

foreach ($users as $user) {
    $stmt->execute($user);
}
```

---

### Обработка ошибок

**Правильный подход:**
```php
try {
    $pdo->beginTransaction(); // Начало транзакции
    
    $stmt1 = $pdo->prepare("UPDATE accounts SET balance = balance - ? WHERE id = ?");
    $stmt1->execute([1000, 1]);
    
    $stmt2 = $pdo->prepare("UPDATE accounts SET balance = balance + ? WHERE id = ?");
    $stmt2->execute([1000, 2]);
    
    $pdo->commit(); // Подтверждение транзакции
} catch (PDOException $e) {
    $pdo->rollBack(); // Откат при ошибке
    echo "Ошибка: " . $e->getMessage();
}
```

---

### Сравнение PDO с mysqli

| Критерий | PDO | MySQLi |
|----------|-----|--------|
| **Поддержка СУБД** | MySQL, PostgreSQL, SQLite, Oracle... | Только MySQL |
| **API стиль** | Объектно-ориентированный | Процедурный и ООП |
| **Подготовленные выражения** | Единый синтаксис для всех СУБД | Только для MySQL |
| **Именованные плейсхолдеры** | ✅ Поддерживаются | ❌ Не поддерживаются |

---

### Практический пример класса-обертки для PDO

```php
class Database {
    private $pdo;
    
    public function __construct($dsn, $username, $password) {
        $this->pdo = new PDO($dsn, $username, $password, [
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC
        ]);
    }
    
    public function query($sql, $params = []) {
        $stmt = $this->pdo->prepare($sql);
        $stmt->execute($params);
        return $stmt;
    }
    
    public function fetchAll($sql, $params = []) {
        return $this->query($sql, $params)->fetchAll();
    }
    
    public function fetchOne($sql, $params = []) {
        return $this->query($sql, $params)->fetch();
    }
}

// Использование
$db = new Database('mysql:host=localhost;dbname=test', 'user', 'pass');
$users = $db->fetchAll("SELECT * FROM users WHERE age > ?", [18]);
```

---

### Итог

**PDO — это:**
- ✅ **Безопасно** (защита от SQL-инъекций)
- ✅ **Универсально** (работа с разными СУБД)
- ✅ **Современно** (ООП подход)
- ✅ **Удобно** (подготовленные выражения, транзакции)

**Когда использовать:**
- Для всех новых проектов
- Когда нужна поддержка разных баз данных
- Когда важна безопасность
- Для совместимости с современными стандартами PHP

Это стандарт де-факто для современной PHP-разработки.