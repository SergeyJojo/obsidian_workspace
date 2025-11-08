Отличный вопрос, Сергей 👏  
Тема **PSR (PHP Standards Recommendations)** — ключевая для понимания современного PHP-кода, потому что **весь современный PHP-фреймворк-мир (Laravel, Symfony, Slim, PHPUnit и т.д.) опирается именно на них.**

---

# 🧩 Что такое PSR

**PSR (PHP Standards Recommendation)** — это **набор стандартов**, утверждённых **PHP-FIG (PHP Framework Interop Group)**.

Цель — чтобы разные фреймворки и библиотеки **были совместимы между собой**, а код — **одинаково структурирован и читаем**.

> 📖 Иными словами, PSR — это “единые правила игры” для архитектуры PHP.

---

# ⚙️ Кто делает PSR

**PHP-FIG (Framework Interop Group)** — это объединение представителей крупных PHP-фреймворков и библиотек: Laravel, Symfony, Drupal, WordPress, Composer, Slim и др.  
Они договариваются: «давайте писать код по общим правилам».

---

# 📚 Основные группы PSR

Сейчас PSR много (40+), но реально активно применяются **около 10–12 ключевых**.  
Разделим их по категориям 👇

---

## 🧱 1. **Базовые стандарты кода**

### 🧩 PSR-1 — _Basic Coding Standard_

Определяет минимальные базовые правила:

- Код должен быть в UTF-8.
    
- Каждый класс — в отдельном файле.
    
- Пространства имён (namespace) обязательны.
    
- Имена классов — `StudlyCaps`.
    
- Константы — `UPPER_CASE_WITH_UNDERSCORES`.
    
- Методы — `camelCase`.
    

📄 Пример:

```php
namespace App\Controller;

class UserController {
    public function getUser() {}
}
```

---

### 🧩 PSR-12 — _Extended Coding Style Guide_

Продолжение PSR-1.  
Это **детальный стиль кодирования** (аналог ESLint в JS или PEP8 в Python):

- Отступ — 4 пробела.
    
- Одна строка = одно выражение.
    
- Ключевые слова — в нижнем регистре (`if`, `while`).
    
- Объявления классов и методов отделяются пустыми строками.
    
- Скобки и фигурные скобки строго формализованы.
    

📄 Пример (валидный PSR-12 код):

```php
namespace App\Service;

use App\Repository\UserRepository;

class UserService
{
    private UserRepository $repo;

    public function __construct(UserRepository $repo)
    {
        $this->repo = $repo;
    }

    public function findUser(int $id): ?User
    {
        return $this->repo->find($id);
    }
}
```

---

## ⚙️ 2. **Автозагрузка**

### 🧩 PSR-4 — _Autoloading Standard_

Определяет **как классы автоматически загружаются** (вместо ручных `require`).

Карта в `composer.json`:

```json
"autoload": {
  "psr-4": {
    "App\\": "src/"
  }
}
```

→ класс `App\Controller\UserController` должен лежать по пути `src/Controller/UserController.php`.

Это то, что использует **Composer**.

📌 Ранее существовал PSR-0 — устарел, заменён PSR-4.

---

## 🔌 3. **Интерфейсы инфраструктуры**

### 🧩 PSR-3 — _Logger Interface_

Определяет **единый интерфейс логирования**:

```php
interface LoggerInterface {
    public function emergency($message, array $context = []);
    public function alert($message, array $context = []);
    public function critical($message, array $context = []);
    public function error($message, array $context = []);
    public function warning($message, array $context = []);
    public function notice($message, array $context = []);
    public function info($message, array $context = []);
    public function debug($message, array $context = []);
    public function log($level, $message, array $context = []);
}
```

📦 Monolog, Symfony Logger, Laravel Logger — **все реализуют PSR-3**, поэтому можно их подменять без изменения кода.

---

### 🧩 PSR-6 — _Caching Interface_

Задаёт интерфейсы для **кеширования**:

```php
$cacheItem = $cachePool->getItem('user_1');
if (!$cacheItem->isHit()) {
    $cacheItem->set($userData);
    $cachePool->save($cacheItem);
}
```

→ Любой кэш (Redis, Filesystem, Memcached) можно подставить, если он реализует `CacheItemPoolInterface`.

---

### 🧩 PSR-7 — _HTTP Message Interface_

Стандарт описания **HTTP-запросов и ответов**.  
Он определяет `RequestInterface`, `ResponseInterface`, `StreamInterface` и т.д.

📦 Используется во всех фреймворках (Slim, Guzzle, Symfony HttpFoundation).

```php
$response = $response->withStatus(200)
                     ->withHeader('Content-Type', 'application/json')
                     ->withBody($body);
```

---

### 🧩 PSR-11 — _Container Interface_

Определяет, как должен работать **контейнер зависимостей (DI container)**.

```php
interface ContainerInterface {
    public function get(string $id);
    public function has(string $id): bool;
}
```

📦 Laravel Container, Symfony Container, PHP-DI — все поддерживают PSR-11.

---

### 🧩 PSR-15 — _HTTP Server Request Handlers / Middleware_

Определяет **как должны выглядеть middleware** в веб-приложениях.

```php
interface MiddlewareInterface {
    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface;
}
```

📦 Slim, Mezzio, Symfony HttpKernel — используют PSR-15.

---

### 🧩 PSR-17 — _HTTP Factories_

Фабрики для создания PSR-7 объектов:

```php
$response = $responseFactory->createResponse(200)
                            ->withHeader('Content-Type', 'text/plain');
```

---

### 🧩 PSR-18 — _HTTP Client Interface_

Задаёт общий интерфейс для HTTP-клиентов:

```php
$response = $client->sendRequest($request);
```

📦 Guzzle, Symfony HttpClient, Buzz — все реализуют PSR-18 → можно менять реализацию без переписывания кода.

---

# 🧠 Иерархия смыслов

|Категория|Стандарты|Для чего|
|---|---|---|
|Код и стиль|PSR-1, PSR-12|Единый стиль написания|
|Автозагрузка|PSR-4|Где искать классы|
|Инфраструктура|PSR-3, PSR-6, PSR-7, PSR-11, PSR-15, PSR-17, PSR-18|Совместимость библиотек|
|Совместимость фреймворков|Все выше|Чтобы Monolog, Guzzle, Symfony, Laravel могли работать вместе|

---

# 💬 Пример в реальном проекте

```php
use Psr\Log\LoggerInterface;
use Psr\Http\Message\RequestInterface;

class UserController
{
    private LoggerInterface $logger;

    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    public function handle(RequestInterface $request)
    {
        $this->logger->info('Request received', ['uri' => (string) $request->getUri()]);
    }
}
```

Здесь:

- `LoggerInterface` — из **PSR-3**
    
- `RequestInterface` — из **PSR-7**  
    И код будет совместим **с любыми** библиотеками, реализующими эти стандарты.
    

---

# ⚡ Резюме

|PSR|Назначение|Пример|
|---|---|---|
|PSR-1|Базовые правила кода|UTF-8, классы, методы|
|PSR-4|Автозагрузка|`App\\` → `src/`|
|PSR-3|Логирование|Monolog|
|PSR-6|Кэш|Symfony Cache|
|PSR-7|HTTP Request/Response|Guzzle, Slim|
|PSR-11|DI Container|Laravel, PHP-DI|
|PSR-12|Стиль кода|форматирование|
|PSR-15|Middleware|Slim, Symfony|
|PSR-17|HTTP Factory|PSR-7 фабрики|
|PSR-18|HTTP Client|Guzzle интерфейс|

---

Хочешь, я покажу **реальный пример мини-приложения**, где в одном коде участвуют PSR-3, PSR-7 и PSR-15 (логгер, middleware и HTTP-запрос), чтобы увидеть, как стандарты делают код модульным и переиспользуемым?

Вот все принятые **PSR** (PHP Standards Recommendations) с кратким описанием каждого:

## 📋 Актуальные PSR стандарты

### **PSR-1: Basic Coding Standard**
*Основной стандарт кодирования*
```php
<?php
// Открывающий тег должен быть только <?php
namespace Vendor\Package;

class ClassName
{
    const VERSION = '1.0';
    
    public function methodName($arg)
    {
        // код
    }
}
```

### **PSR-2: Coding Style Guide**
*Расширенный стандарт оформления кода*
- Использование 4 пробелов для отступов
- Строка не должна превышать 120 символов
- Открывающая фигурная скобка на той же строке
- Ключевые слова в нижнем регистре

### **PSR-4: Autoloading Standard**
*Стандарт автозагрузки классов*
```json
{
    "autoload": {
        "psr-4": {
            "App\\": "src/",
            "Vendor\\Package\\": "lib/"
        }
    }
}
```

### **PSR-3: Logger Interface**
*Стандарт для логгеров*
```php
$logger->emergency('Система не работает', ['user' => $user]);
$logger->error('Ошибка', ['code' => 500]);
$logger->info('Пользователь залогинился', ['user_id' => 123]);
```

### **PSR-6: Caching Interface**
*Интерфейс для кэширования*
```php
$pool = new CachePool();
$item = $pool->getItem('cache_key');
$item->set('data')->expiresAfter(3600);
$pool->save($item);
```

### **PSR-7: HTTP Message Interface**
*Интерфейс HTTP сообщений*
```php
$response = new Response(200);
$response = $response->withHeader('Content-Type', 'application/json');
$response->getBody()->write(json_encode($data));
```

### **PSR-11: Container Interface**
*Интерфейс контейнера зависимостей*
```php
interface ContainerInterface
{
    public function get($id);
    public function has($id);
}
```

### **PSR-12: Extended Coding Style**
*Расширенное руководство по стилю кода*
- Уточнения к PSR-2
- Более строгие правила форматирования

### **PSR-13: Hypermedia Links**
*Определение ссылок в гипермедиа*
```php
$link = new Link('author', 'https://example.com/author');
```

### **PSR-14: Event Dispatcher**
*Интерфейс диспетчера событий*
```php
$dispatcher = new EventDispatcher();
$dispatcher->addListener('user.created', function (UserCreatedEvent $event) {
    // обработка
});
```

### **PSR-15: HTTP Server Request Handlers**
*Обработчики HTTP запросов*
```php
class AuthMiddleware implements MiddlewareInterface
{
    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        // проверка аутентификации
        return $handler->handle($request);
    }
}
```

### **PSR-16: Simple Cache**
*Простой интерфейс кэширования*
```php
$cache = new SimpleCache();
$cache->set('key', 'value', 3600);
$value = $cache->get('key');
```

### **PSR-17: HTTP Factories**
*Фабрики для PSR-7*
```php
$factory = new ResponseFactory();
$response = $factory->createResponse(200, 'OK');
```

### **PSR-18: HTTP Client**
*Интерфейс HTTP клиента*
```php
$client = new HttpClient();
$request = $requestFactory->createRequest('GET', 'https://example.com');
$response = $client->sendRequest($request);
```

---

## 🔄 Устаревшие PSR

### **PSR-0: Autoloading Standard (Устарел)**
*Предшественник PSR-4*
```json
{
    "autoload": {
        "psr-0": {
            "Vendor\\Package\\": "src/"
        }
    }
}
```

### **PSR-5/PSR-19: PHPDoc (Не приняты)**
*Стандарты для документации - были отозваны*

---

## 🛠️ Практическое использование

### Composer + PSR-4
```json
{
    "autoload": {
        "psr-4": {
            "App\\": "src/",
            "Tests\\": "tests/"
        }
    }
}
```

### PSR-7 в действии
```php
// Создание response
$response = new Response();
return $response
    ->withStatus(200)
    ->withHeader('Content-Type', 'application/json')
    ->withBody(new StringStreamBody(json_encode($data)));
```

### PSR-11 Контейнер
```php
$container = new Container();
$container->set(Database::class, function () {
    return new Database();
});

$db = $container->get(Database::class);
```

### PSR-14 События
```php
class UserRegisteredEvent
{
    public function __construct(public User $user) {}
}

$dispatcher->dispatch(new UserRegisteredEvent($user));
```

---

## 📚 Где применяются

- **Laravel**: PSR-4, PSR-7, PSR-11, PSR-14
- **Symfony**: PSR-4, PSR-6, PSR-7, PSR-11, PSR-14, PSR-15, PSR-18
- **Zend/Laminas**: Все основные PSR
- **Slim**: PSR-7, PSR-11, PSR-15

---

## 💡 Ключевые принципы

1. **Interfaces, not implementations** - Стандарты определяют интерфейсы
2. **Framework interoperability** - Совместимость между фреймворками
3. **Modern PHP features** - Использование возможностей современных версий PHP

## 🎯 Самые важные PSR

1. **PSR-4** - Автозагрузка классов (обязателен)
2. **PSR-12** - Стиль кодирования (для читаемости)
3. **PSR-7** - HTTP сообщения (для веб-разработки)
4. **PSR-11** - Контейнер зависимостей (для архитектуры)
5. **PSR-14** - События (для расширяемости)

Эти стандарты сделали PHP экосистему более согласованной и профессиональной! 🚀