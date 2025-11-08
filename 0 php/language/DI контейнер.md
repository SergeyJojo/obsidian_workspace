**DI-контейнер (Dependency Injection Container)** — это "умный" менеджер зависимостей, который автоматически создает и связывает объекты вашего приложения. Давайте разберем на простых примерах.

## 🎯 Что решает DI-контейнер?

### Проблема без DI:
```php
class OrderController {
    private $paymentService;
    
    public function __construct() {
        // Жесткая зависимость - плохо!
        $this->paymentService = new PayPalPayment();
    }
}
```

### Решение с DI:
```php
class OrderController {
    public function __construct(private PaymentInterface $paymentService) {
        // Контейнер сам подставит нужную реализацию!
    }
}
```

---

## 🏗️ Как работает DI-контейнер

### Базовый принцип:
```php
// 1. Регистрируем зависимости
$container->bind(PaymentInterface::class, PayPalPayment::class);

// 2. Запрашиваем объект
$controller = $container->make(OrderController::class);

// 3. Контейнер автоматически:
//    - Создает PayPalPayment
//    - Создает OrderController, передавая PayPalPayment в конструктор
```

---

## 📝 Типы внедрения зависимостей

### 1. Constructor Injection (самый частый)
```php
class UserService {
    public function __construct(
        private UserRepository $repository,
        private EmailService $email
    ) {}
}
```

### 2. Method Injection
```php
class OrderService {
    public function processOrder(Order $order, LoggerInterface $logger) {
        $logger->info('Processing order');
    }
}
```

### 3. Property Injection (редко)
```php
class ReportService {
    #[Inject]
    public FormatterInterface $formatter;
}
```

---

## 🔧 Реализация простого DI-контейнера

```php
class Container {
    private $bindings = [];
    
    // Регистрация зависимости
    public function bind(string $abstract, Closure|string $concrete): void {
        $this->bindings[$abstract] = $concrete;
    }
    
    // Создание объекта
    public function make(string $abstract): mixed {
        // Если есть привязка - используем ее
        if (isset($this->bindings[$abstract])) {
            $concrete = $this->bindings[$abstract];
            
            if ($concrete instanceof Closure) {
                return $concrete($this);
            }
            
            return $this->make($concrete);
        }
        
        // Автоматическое создание через рефлексию
        return $this->resolve($abstract);
    }
    
    private function resolve(string $class): object {
        $reflector = new ReflectionClass($class);
        
        // Если нет конструктора - просто создаем
        if (!$constructor = $reflector->getConstructor()) {
            return new $class();
        }
        
        // Анализируем параметры конструктора
        $parameters = $constructor->getParameters();
        $dependencies = [];
        
        foreach ($parameters as $parameter) {
            $type = $parameter->getType();
            
            if ($type && !$type->isBuiltin()) {
                // Рекурсивно создаем зависимости
                $dependencies[] = $this->make($type->getName());
            } else {
                throw new Exception("Cannot resolve {$parameter->getName()}");
            }
        }
        
        return $reflector->newInstanceArgs($dependencies);
    }
}
```

---

## 🚀 Использование контейнера

### Регистрация зависимостей:
```php
$container = new Container();

// Биндинг интерфейса к реализации
$container->bind(
    PaymentInterface::class, 
    PayPalPayment::class
);

// Биндинг с замыканием для сложной логики
$container->bind('database', function($container) {
    return new Database(
        host: 'localhost',
        user: 'root', 
        pass: 'password'
    );
});

// Синглтон (один экземпляр на все приложение)
$container->singleton(LoggerInterface::class, FileLogger::class);
```

### Автоматическое создание:
```php
// Контейнер рекурсивно создает всю цепочку зависимостей
$orderService = $container->make(OrderService::class);

// Это создаст:
// OrderService → 
//   PaymentInterface (PayPalPayment) →
//     HttpClient →
//   LoggerInterface (FileLogger) →
//     FileSystem
```

---

## 💡 Real-world пример (Laravel-style)

```php
// Сервисы
interface PaymentGateway {
    public function charge($amount);
}

class StripePayment implements PaymentGateway {
    public function __construct(private string $apiKey) {}
    public function charge($amount) { /* ... */ }
}

class OrderProcessor {
    public function __construct(
        private PaymentGateway $payment,
        private EmailService $email
    ) {}
    
    public function process(Order $order) {
        $this->payment->charge($order->total);
        $this->email->sendReceipt($order);
    }
}

// Настройка контейнера
$container->bind(PaymentGateway::class, function($container) {
    return new StripePayment(config('stripe.key'));
});

$container->bind(EmailService::class, SmtpEmailService::class);

// Использование
$processor = $container->make(OrderProcessor::class);
$processor->process($order);
```

---

## 🎯 Преимущества DI-контейнера

### 1. **Слабая связанность**
```php
// Легко поменять реализацию
$container->bind(PaymentInterface::class, StripePayment::class);
// Или
$container->bind(PaymentInterface::class, PayPalPayment::class);
```

### 2. **Тестируемость**
```php
class OrderServiceTest {
    public function test_order_processing() {
        // Подменяем реальную зависимость на mock
        $mockPayment = $this->createMock(PaymentInterface::class);
        $service = new OrderService($mockPayment);
        
        // Тестируем только логику OrderService
        $service->processOrder($testOrder);
    }
}
```

### 3. **Переиспользование кода**
```php
// Один биндинг - много использований
$container->bind(LoggerInterface::class, FileLogger::class);

// Все эти сервисы получат один и тот же Logger
$userService = $container->make(UserService::class);     // ✓
$orderService = $container->make(OrderService::class);   // ✓  
$reportService = $container->make(ReportService::class); // ✓
```

### 4. **Управление жизненным циклом**
```php
// Синглтон - один экземпляр
$container->singleton(Database::class, MySQLDatabase::class);

// Фабрика - новый экземпляр каждый раз  
$container->bind(EmailService::class, SmtpEmailService::class);
```

---

## 🔧 Продвинутые возможности

### 1. **Автоматическое связывание (Autowiring)**
```php
// Контейнер сам определяет зависимости по type-hints
class ComplexService {
    public function __construct(
        RepositoryInterface $repo,
        LoggerInterface $logger,
        CacheInterface $cache
    ) {}
    // Контейнер найдет все реализации автоматически!
}
```

### 2. **Контекстное связывание (Contextual Binding)**
```php
// Разные реализации для разных классов
$container->when(OrderController::class)
          ->needs(PaymentInterface::class)
          ->give(PayPalPayment::class);

$container->when(SubscriptionController::class)  
          ->needs(PaymentInterface::class)
          ->give(StripePayment::class);
```

### 3. **Тегирование**
```php
// Группировка сервисов
$container->tag([LoggerA::class, LoggerB::class], 'loggers');

// Получение всех сервисов по тегу
$loggers = $container->tagged('loggers');
```

---

## 🛠️ DI-контейнеры в популярных фреймворках

### Laravel:
```php
// Service Provider
class AppServiceProvider extends ServiceProvider {
    public function register() {
        $this->app->bind(PaymentGateway::class, function($app) {
            return new StripePayment(config('services.stripe.key'));
        });
    }
}

// Использование
$payment = app(PaymentGateway::class);
```

### Symfony:
```yaml
# services.yaml
services:
    App\Payment\StripePayment:
        arguments:
            $apiKey: '%env(STRIPE_KEY)%'
```

### Yii2:
```php
Yii::$container->set('app\components\PaymentInterface', 'app\components\StripePayment');
```

---

## 💡 Best Practices

### 1. **Интерфейсы, а не реализации**
```php
// Хорошо
$container->bind(PaymentInterface::class, StripePayment::class);

// Плохо  
$container->bind(StripePayment::class, StripePayment::class);
```

### 2. **Регистрируйте в провайдерах**
```php
class PaymentServiceProvider {
    public function register(Container $container) {
        $container->bind(PaymentInterface::class, StripePayment::class);
    }
}
```

### 3. **Используйте для всех сервисов**
```php
// Вместо new
$service = new OrderService(new StripePayment(), new Logger());

// Используйте контейнер
$service = $container->make(OrderService::class);
```

---

## 💎 Итог

**DI-контейнер** — это мозг вашего приложения, который:

- ✅ **Автоматически создает объекты** и их зависимости
- ✅ **Уменьшает связанность** между компонентами
- ✅ **Делает код тестируемым**
- ✅ **Упрощает замену реализаций**
- ✅ **Управляет жизненным циклом** объектов

**Главный принцип:** "Не вызывай нас — мы тебя вызовем". Ваши классы просто объявляют зависимости, а контейнер заботится об их создании и связывании!

Это фундаментальная концепция современных PHP-фреймворков, которая делает код чище, гибче и поддерживаемее. 🚀