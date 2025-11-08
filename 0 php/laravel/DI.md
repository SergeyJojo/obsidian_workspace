**DI (Dependency Injection) в Laravel** — это одна из самых мощных и элегантных возможностей фреймворка. Расскажу по полочкам, как это устроено.

## 🎯 Основной принцип

Laravel автоматически разрешает зависимости через **отражение (reflection)** и **контейнер служб**.

```php
// Вы просто type-hint зависимость в конструкторе
class UserController {
    public function __construct(
        private UserRepository $users,
        private MailService $mailer
    ) {}
    
    // Laravel сам создаст и подставит зависимости!
}
```

---

## 🏗️ Как работает контейнер

### 1. **Автоматическое разрешение**
```php
// Laravel анализирует тип параметра и создает объект
class OrderService {
    public function __construct(PaymentGateway $payment) {
        // Контейнер автоматически создаст PaymentGateway
    }
}

// Где-то в коде:
$service = app(OrderService::class); // Всё работает!
```

### 2. **Регистрация связей**
```php
// В Service Provider
public function register() {
    $this->app->bind(PaymentInterface::class, function ($app) {
        return new StripePayment(config('services.stripe.key'));
    });
    
    // Синглтон (один экземпляр)
    $this->app->singleton(LoggerInterface::class, FileLogger::class);
}
```

---

## 📝 Типы внедрения зависимостей

### 1. **Constructor Injection** (самый частый)
```php
class CheckoutController {
    public function __construct(
        private PaymentGateway $payment,
        private OrderRepository $orders
    ) {}
}
```

### 2. **Method Injection** (в методах)
```php
class ReportService {
    public function generate(ExportFormatter $formatter, Report $report) {
        // $formatter автоматически внедряется
    }
}

// В роуте:
Route::post('/report', [ReportService::class, 'generate']);
```

### 3. **Property Injection** (редко, через аннотации)
```php
class SomeClass {
    #[Inject]
    private SomeService $service;
}
```

---

## 🔧 Конкретные примеры

### Пример 1: **Сервис с зависимостями**
```php
// Сервис
class EmailNotification {
    public function __construct(
        private Mailer $mailer,
        private TemplateEngine $templates
    ) {}
    
    public function sendWelcome(User $user) {
        $html = $this->templates->render('welcome', $user);
        $this->mailer->send($user->email, $html);
    }
}

// Использование (Laravel сам всё разрешит)
$notification = app(EmailNotification::class);
$notification->sendWelcome($user);
```

### Пример 2: **Интерфейсы и реализации**
```php
// Интерфейс
interface PaymentGateway {
    public function charge($amount);
}

// Реализация
class StripePayment implements PaymentGateway {
    public function charge($amount) { /* ... */ }
}

// Регистрация в Service Provider
$this->app->bind(PaymentGateway::class, StripePayment::class);

// Использование в контроллере
class PaymentController {
    public function pay(PaymentGateway $payment) {
        // Автоматически получит StripePayment!
        $payment->charge(100);
    }
}
```

---

## 🛠️ Service Providers — сердце DI

### Типичный Service Provider:
```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use App\Contracts\PaymentGateway;
use App\Services\StripePayment;

class AppServiceProvider extends ServiceProvider {
    public function register() {
        // Биндинг интерфейса к реализации
        $this->app->bind(PaymentGateway::class, StripePayment::class);
        
        // Синглтон
        $this->app->singleton('geoip', function ($app) {
            return new GeoIP($app['config']['services.geoip']);
        });
        
        // Конкретный экземпляр
        $this->app->instance('version', '1.0.0');
    }
    
    public function boot() {
        // Выполняется после регистрации всех провайдеров
    }
}
```

---

## 🎯 Продвинутые возможности

### 1. **Контекстное связывание**
```php
// Разные реализации для разных классов
$this->app->when(OrderController::class)
          ->needs(PaymentGateway::class)
          ->give(StripePayment::class);

$this->app->when(SubscriptionController::class)
          ->needs(PaymentGateway::class)
          ->give(PayPalPayment::class);
```

### 2. **Тегирование**
```php
// Группировка сервисов
$this->app->tag([StripePayment::class, PayPalPayment::class], 'payments');

// Получение всех сервисов по тегу
$payments = $this->app->tagged('payments');
```

### 3. **События контейнера**
```php
// Событие при разрешении зависимости
$this->app->resolving(function ($object, $app) {
    // Вызывается при создании любого объекта
});

$this->app->resolving(PaymentGateway::class, function ($gateway, $app) {
    // Только для PaymentGateway
});
```

---

## 💡 Практические паттерны

### Паттерн **Repository**:
```php
// Интерфейс
interface UserRepository {
    public function find($id);
    public function create(array $data);
}

// Реализация
class EloquentUserRepository implements UserRepository {
    public function find($id) {
        return User::find($id);
    }
    // ...
}

// Регистрация
$this->app->bind(UserRepository::class, EloquentUserRepository::class);

// Использование в контроллере
class UserController {
    public function __construct(private UserRepository $users) {}
    
    public function show($id) {
        return $this->users->find($id);
    }
}
```

### Паттерн **Factory**:
```php
// Фабрика
class NotificationFactory {
    public function __construct(
        private Container $container
    ) {}
    
    public function make($type) {
        return $this->container->make("notification.{$type}");
    }
}

// Регистрация
$this->app->bind('notification.email', EmailNotification::class);
$this->app->bind('notification.sms', SmsNotification::class);
```

---

## 🔍 Как это работает внутри

### Процесс разрешения зависиностей:
```
1. Laravel видит: UserController(UserRepository $users)
2. Ищет биндинг для UserRepository
3. Если нет → пытается создать через reflection
4. Анализирует конструктор UserRepository
5. Рекурсивно создает все зависимости
6. Возвращает готовый объект
```

### Код псевдо-реализации:
```php
class Container {
    public function make($abstract) {
        // Если есть биндинг - используем его
        if (isset($this->bindings[$abstract])) {
            return $this->bindings[$abstract]($this);
        }
        
        // Иначе - автоматическое создание
        $reflector = new ReflectionClass($abstract);
        $constructor = $reflector->getConstructor();
        
        if (is_null($constructor)) {
            return new $abstract;
        }
        
        $dependencies = [];
        foreach ($constructor->getParameters() as $parameter) {
            $dependencies[] = $this->make($parameter->getType()->getName());
        }
        
        return $reflector->newInstanceArgs($dependencies);
    }
}
```

---

## 🚀 Использование в реальных проектах

### **Typical Laravel app structure:**
```
app/
├── Providers/
│   ├── AppServiceProvider.php
│   └── PaymentServiceProvider.php
├── Contracts/
│   └── PaymentGateway.php
├── Services/
│   ├── StripePayment.php
│   └── EmailService.php
└── Repositories/
    └── UserRepository.php
```

### **Конфиг в bootstrap/app.php:**
```php
$app->register(App\Providers\AppServiceProvider::class);
$app->register(App\Providers\PaymentServiceProvider::class);
```

---

## 💎 Итог

**DI в Laravel — это магия, которая работает так:**

1. ✅ **Автоматическое разрешение** через reflection
2. ✅ **Гибкие биндинги** через Service Providers  
3. ✅ **Интерфейсы → реализации**
4. ✅ **Синглтоны, инстансы, контекстные биндинги**
5. ✅ **Всё это "из коробки"**

**Преимущества:**
- Тестируемость (легко мокать зависимости)
- Гибкость (легко менять реализации)
- Чистая архитектура
- Автоматизация рутинных задач

Laravel делает DI настолько простым, что вы часто даже не замечаете, как используете его! 🎯