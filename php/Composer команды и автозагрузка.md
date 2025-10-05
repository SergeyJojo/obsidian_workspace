Идеально! **Composer** — это сердце современной PHP-экосистемы, ваш аналог `go mod`. Давайте разберем его так, чтобы было понятно с позиции Go-разработчика.

---

### 1. Composer: Основные команды (Аналоги `go mod`)

| Команда Composer | Аналог в Go | Что делает |
| :--- | :--- | :--- |
| **`composer init`** | `go mod init` | Создает `composer.json` в текущей директории |
| **`composer install`** | `go mod tidy` + установка | Читает `composer.lock` и устанавливает ВСЕ зависимости |
| **`composer require vendor/package`** | `go get package` | Добавляет пакет в `composer.json` и УСТАНАВЛИВАЕТ его |
| **`composer update`** | `go get -u` | Обновляет ВСЕ зависимости до последних версий по правилам в `composer.json` |
| **`composer dump-autoload`** | (нет аналога) | **Перегенерирует автозагрузчик** после ручного добавления классов |
| **`composer.json`** | `go.mod` | Файл с объявлением зависимостей и прочей конфигурацией |
| **`composer.lock`** | `go.sum` | Файл с **точными** версиями установленных зависимостей |

**Ключевое отличие:** В Go зависимости устанавливаются сразу в общее пространство. Composer по умолчанию устанавливает зависимости в папку `vendor/` вашего проекта.

---

### 2. Autoloading (Самая важная концепция)

В Go вы явно импортируете пакеты. В PHP до автозагрузки нужно было подключать каждый файл с классом вручную через `require_once`.

**Autoloading (автозагрузка)** — это механизм, который **автоматически подключает файл с классом, когда он вам нужен**.

#### Как это подключить?

В вашем `composer.json` есть секция `autoload`:

```json
{
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    }
}
```

**Расшифровка для Go-разработчика:**
*   `"App\\": "src/"` означает: "Все классы в пространстве имен `App\` ищи в директории `src/`".
*   Это **аналог того, как Go ищет пакеты в вашем модуле**.

#### Пример преобразования пути:

| Пространство имен + Класс | Преобразуется в путь | Файл |
| :--- | :--- | :--- |
| `App\Controllers\UserController` | `src/Controllers/UserController.php` | `src/Controllers/UserController.php` |
| `App\Services\PaymentService` | `src/Services/PaymentService.php` | `src/Services/PaymentService.php` |

**Ваше правило:** Структура папок должна **точно повторять структуру пространств имен**.

---

### 3. Практический пример "С нуля"

Давайте создадим проект так, как вы это делаете в Go.

**1. Инициализация проекта**
```bash
mkdir my-php-app && cd my-php-app
composer init
# (ответьте на вопросы, создастся composer.json)
```

**2. Создадим структуру PSR-4**
```bash
mkdir src
```

**3. Настроим автозагрузку в `composer.json`**
```json
{
    "name": "your-name/my-php-app",
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    }
}
```

**4. Создадим наш первый класс**
Файл: `src/Controllers/UserController.php`

```php
<?php
// Объявляем пространство имен, которое соответствует структуре папок
namespace App\Controllers;

class UserController 
{
    public function index(): string
    {
        return "Getting all users";
    }
}
```

**5. Генерируем автозагрузчик**
```bash
composer dump-autoload
```
*Эта команда создаст папку `vendor/` и сгенерирует там все необходимые файлы для автозагрузки.*

**6. Используем наш класс (index.php)**
```php
<?php
// Подключаем автозагрузчик Composer (аналог импортов в Go)
require_once __DIR__ . '/vendor/autoload.php';

// Используем класс - файл подключится АВТОМАТИЧЕСКИ!
use App\Controllers\UserController;

$controller = new UserController();
echo $controller->index(); // "Getting all users"
```

---

### 4. Что добавить в Obsidian:

**[[Composer Commands Cheatsheet]]**
```
init - создать новый проект
require - добавить зависимость
install - установить все зависимости из lock-файла
update - обновить зависимости
dump-autoload - обновить автозагрузчик
```

**[[PSR-4 Autoloading Rules]]**
```
Пространство имен: App\Controllers\User
Преобразование: src/Controllers/User.php
Правило: App\ => src/
```

**[[Типичная структура проекта]]**
```
my-project/
├── composer.json
├── composer.lock
├── vendor/ (автогенерируемая)
├── src/ (ваш код)
│   └── Controllers/
│       └── UserController.php
└── index.php (точка входа)
```

**Главный вывод для вас:** 
Composer + PSR-4 Autoloading — это **стандартный и единственно верный способ** структурировать современный PHP-проект. После настройки `composer.json` вы просто создаете классы в правильных папках с правильными пространствами имен, и они автоматически становятся доступны. **Это избавляет от адского количества `require_once`**, которые были в старом PHP.