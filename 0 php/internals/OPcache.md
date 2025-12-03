**OPcache** (Opcode Cache) — это встроенный в PHP механизм кеширования скомпилированного байт-кода, который значительно ускоряет выполнение PHP-скриптов.

## Как работает PHP без OPcache

```php
// При каждом запросе:
1. Чтение файла script.php с диска
2. Лексический анализ (разбиение на токены)
3. Парсинг (построение AST)
4. Компиляция в байт-код (opcodes)
5. Выполнение байт-кода
```

## Как работает с OPcache

```php
// При первом запросе:
1. Компиляция script.php → байт-код
2. Сохранение байт-кода в shared memory
3. Выполнение

// При последующих запросах:
1. Проверка кеша в memory
2. Немедленное выполнение готового байт-кода
```

## Настройка OPcache

### Конфигурация в php.ini:
```ini
; Включение OPcache
opcache.enable=1

; Память для хранения байт-кода (рекомендуется 128-256MB)
opcache.memory_consumption=256

; Максимальное количество файлов в кеше
opcache.max_accelerated_files=20000

; Потребление памяти для внутренних структур (в MB)
opcache.interned_strings_buffer=16

; Валидация таймстампов файлов (0 - отключить в production)
opcache.validate_timestamps=0

; Частота проверки изменений (в секундах)
opcache.revalidate_freq=2

; Быстрый shutdown (ускоряет завершение работы)
opcache.fast_shutdown=1

; Включение для CLI (полезно для composer, artisan)
opcache.enable_cli=1
```

## Проверка статуса OPcache

### Через phpinfo():
```php
<?php
phpinfo();
// Ищем раздел "OPcache"
```

### Через opcache_get_status():
```php
<?php
$status = opcache_get_status();
echo "OPcache enabled: " . ($status['opcache_enabled'] ? 'Yes' : 'No') . "\n";
echo "Memory usage: " . round($status['memory_usage']['used_memory']/1024/1024, 2) . "MB\n";
echo "Cached scripts: " . $status['opcache_statistics']['num_cached_scripts'] . "\n";
echo "Hits: " . $status['opcache_statistics']['hits'] . "\n";
echo "Misses: " . $status['opcache_statistics']['misses'] . "\n";
```

### Через консоль:
```bash
php -r "print_r(opcache_get_status());"
```

## Управление OPcache

### Сброс кеша:
```php
<?php
// Принудительный сброс всего кеша
opcache_reset();

// Сброс конкретного файла (PHP 8.0+)
opcache_invalidate('/path/to/script.php', true);
```

### Preloading (PHP 7.4+):
```ini
; В php.ini - предзагрузка часто используемых классов
opcache.preload=/path/to/preload.php
```

**preload.php:**
```php
<?php
// Предзагрузка фреймворка
opcache_compile_file('/path/to/vendor/autoload.php');
opcache_compile_file('/path/to/framework/bootstrap.php');

// Предзагрузка часто используемых классов
$classes = [
    '/path/to/app/Models/User.php',
    '/path/to/app/Controllers/HomeController.php',
];

foreach ($classes as $file) {
    if (file_exists($file)) {
        opcache_compile_file($file);
    }
}
```

## Практические примеры

### 1. **Производительность до и после**
```php
<?php
// Без OPcache
$start = microtime(true);
for ($i = 0; $i < 1000; $i++) {
    include 'heavy_script.php';
}
echo "Without OPcache: " . (microtime(true) - $start) . "s\n";

// С OPcache
$start = microtime(true);
for ($i = 0; $i < 1000; $i++) {
    include 'heavy_script.php';
}
echo "With OPcache: " . (microtime(true) - $start) . "s\n";
```

### 2. **Мониторинг в реальном времени**
```php
<?php
class OPCacheMonitor {
    public static function getStats() {
        $status = opcache_get_status();
        
        return [
            'memory_used' => round($status['memory_usage']['used_memory'] / 1024 / 1024, 2),
            'memory_free' => round($status['memory_usage']['free_memory'] / 1024 / 1024, 2),
            'memory_wasted' => round($status['memory_usage']['wasted_memory'] / 1024 / 1024, 2),
            'cached_scripts' => $status['opcache_statistics']['num_cached_scripts'],
            'hits' => $status['opcache_statistics']['hits'],
            'misses' => $status['opcache_statistics']['misses'],
            'hit_rate' => round($status['opcache_statistics']['opcache_hit_rate'], 2)
        ];
    }
    
    public static function showStatus() {
        $stats = self::getStats();
        echo "OPcache Status:\n";
        foreach ($stats as $key => $value) {
            echo "  $key: $value\n";
        }
    }
}

OPCacheMonitor::showStatus();
```

## Рекомендации для разных сред

### Development:
```ini
opcache.validate_timestamps=1
opcache.revalidate_freq=0
opcache.enable_cli=1
```

### Production:
```ini
opcache.validate_timestamps=0
opcache.revalidate_freq=0
opcache.enable_cli=0
opcache.save_comments=0
```

### High-Traffic:
```ini
opcache.memory_consumption=512
opcache.max_accelerated_files=40000
opcache.interned_strings_buffer=32
```

## Решение распространенных проблем

### 1. **Кеш не обновляется**
```php
// При изменении кода в production
opcache_reset(); // Сбросить кеш

// Или для конкретного файла
opcache_invalidate('/path/to/updated_file.php', true);
```

### 2. **Нехватка памяти**
```ini
; Увеличить memory_consumption если видим много wasted memory
opcache.memory_consumption=512
```

### 3. **Достигнут лимит файлов**
```ini
; Увеличить max_accelerated_files
opcache.max_accelerated_files=40000
```

## Итог

**OPcache дает:**
- ✅ **Ускорение в 2-5 раз** для PHP-приложений
- ✅ **Снижение нагрузки** на CPU
- ✅ **Уменьшение I/O операций** с диском
- ✅ **Лучшую масштабируемость**

**Обязательно использовать в:**
- Production-окружении
- High-load проектах
- Системах с большим количеством PHP-файлов

OPcache — это самый эффективный способ ускорить PHP-приложение без изменения кода!