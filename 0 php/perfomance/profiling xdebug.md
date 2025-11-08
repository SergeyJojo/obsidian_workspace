Отлично, Сергей — тема суперпрактическая. Ниже я собрал полноценный конспект-файл `profiling_xdebug_blackfire.md`, где подробно показаны:

- различия между **Xdebug** и **Blackfire**,
    
- как включать и использовать профайлинг,
    
- как анализировать результаты,
    
- и что важно для оптимизации производительности PHP-приложений.
    

---

# ⚙️ Профилирование PHP: Xdebug и Blackfire

---

## 🧩 Что такое профилирование

**Профилирование (profiling)** — это сбор информации о выполнении PHP-кода:  
время выполнения функций, количество вызовов, использование памяти.

Цель — найти “узкие места” (bottlenecks):  
медленные участки кода, избыточные запросы к БД, тяжелые циклы, рекурсии и т.д.

---

## 🚀 Основные инструменты

|Инструмент|Назначение|Когда использовать|
|---|---|---|
|🧮 **Xdebug Profiler**|Локальный инструмент для отладки и профилирования, пишет `.cachegrind` файлы|Для глубокой локальной диагностики|
|⚡ **Blackfire.io**|SaaS-платформа для продакшн-профилирования и CI интеграции|Для анализа на реальных окружениях (staging/prod)|

---

## 🧱 1️⃣ Xdebug Profiler

---

### 🔧 Установка

Xdebug уже обычно предустановлен в dev-контейнерах, но если нет:

```bash
pecl install xdebug
```

или проверь:

```bash
php -m | grep xdebug
```

---

### ⚙️ Настройка в `php.ini`

Добавь или измени секцию:

```ini
zend_extension=xdebug.so

[xdebug]
xdebug.mode=profile
xdebug.output_dir=/tmp/xdebug_profiles
xdebug.profiler_output_name=cachegrind.out.%p
```

**Пояснения:**

- `xdebug.mode=profile` — включает профайлинг (можно также: `develop,trace,debug,coverage`).
    
- `xdebug.output_dir` — куда сохраняются `.cachegrind` файлы.
    
- `%p` — PID процесса (чтобы файлы не перезаписывались).
    

---

### 📜 Генерация профиля

Запусти любой PHP-скрипт или запрос:

```bash
php index.php
```

После выполнения появится файл:

```
/tmp/xdebug_profiles/cachegrind.out.12345
```

---

### 🔍 Анализ результатов

Формат — **Cachegrind** (тот же, что у Valgrind).  
Смотреть можно через GUI:

#### 🔹 Linux / Windows:

- [QCacheGrind](https://kcachegrind.github.io/)
    
- [KCacheGrind](https://kde.org/applications/development/org.kde.kcachegrind)
    

#### 🔹 macOS:

```bash
brew install qcachegrind
```

#### 🔹 Пример данных:

```
main()
  -> include('index.php')
    -> App\Controller\UserController::list()
      -> App\Repository\UserRepository::findAll()
        -> PDOStatement::execute()
```

#### Метрики:

- **Incl. Time** — общее время с вызовами внутри (inclusive)
    
- **Self Time** — только эта функция (exclusive)
    
- **Calls** — количество вызовов
    
- **Memory** — использование памяти (если включено)
    

---

### 🧠 Полезные параметры

```ini
xdebug.start_with_request=yes
xdebug.profiler_enable_trigger=1
xdebug.profiler_output_name=cachegrind.out.%t
```

Можно активировать профилирование **по триггеру**, добавив в запрос:

```
?XDEBUG_PROFILE=1
```

или через cookie/ENV.

---

### ⚙️ Как использовать на практике

1. Включаешь профайлинг через `xdebug.profiler_enable_trigger`.
    
2. Отправляешь конкретный HTTP-запрос (не весь сайт).
    
3. Анализируешь `.cachegrind` файл в QCacheGrind.
    
4. Находишь функции с самым высоким **Inclusive Time**.
    
5. Оптимизируешь алгоритм, SQL или вызовы внешних сервисов.
    

---

## 🧠 2️⃣ Blackfire.io

---

### 🔎 Что это

**Blackfire** — инструмент от компании SensioLabs (создатели Symfony).  
Он работает как агент + расширение PHP + веб-интерфейс.

Преимущества:

- минимальная нагрузка на продакшн,
    
- профилирование в реальном окружении (FPM, CLI, HTTP),
    
- сравнение профилей “до и после” оптимизаций,
    
- интеграция с CI/CD и GitHub Actions.
    

---

### ⚙️ Установка

1. Установи расширение PHP:
    

```bash
pecl install blackfire
```

2. Установи агент:
    

```bash
curl -sL https://packages.blackfire.io/gpg.key | sudo apt-key add -
echo "deb http://packages.blackfire.io/debian any main" | sudo tee /etc/apt/sources.list.d/blackfire.list
sudo apt update
sudo apt install blackfire-agent
```

3. Настрой конфиг `/etc/blackfire/agent`:
    

```ini
server-id=your_server_id
server-token=your_server_token
```

(Токены берутся из [blackfire.io](https://blackfire.io/) в разделе "Credentials".)

---

### 🧩 Запуск профилирования

#### Через CLI

```bash
blackfire run php script.php
```

#### Через HTTP

```bash
blackfire curl https://your-app.test/
```

После выполнения Blackfire создаст профиль и откроет веб-интерфейс с визуализацией.

---

### 📊 Интерфейс Blackfire

Основные графы:

- **Call Graph** — дерево вызовов (аналог QCacheGrind)
    
- **Timeline** — распределение времени по запросу
    
- **Hot Paths** — функции с наибольшим временем
    
- **I/O** — сетевые и файловые операции
    

---

### 🧠 Сравнение профилей

Blackfire умеет сравнивать профили:

```bash
blackfire compare <profile1> <profile2>
```

или прямо в UI:

- "Before" → "After"
    
- видишь, где ускорилось / замедлилось
    

---

### ⚙️ Интеграция с CI

Blackfire поддерживает **assertions** в `.blackfire.yml`:

```yaml
tests:
  "Page loads under 100ms":
    path: "/"
    assertions:
      - "main.wall_time < 100ms"
```

Можно автоматизировать performance-тесты в pipeline:

- если страница выполняется дольше 100ms → build failed.
    

---

## 🧮 Сравнение Xdebug vs Blackfire

|Критерий|Xdebug|Blackfire|
|---|---|---|
|Установка|просто (локально)|требует регистрацию и агент|
|Производительность|тяжёлый, замедляет ×5–10|минимальная нагрузка|
|Использование|отладка и dev|staging / prod|
|Формат данных|Cachegrind|собственный web UI|
|Автоматизация|нет|есть тесты и CI интеграция|
|Подходит для|поиска медленных функций|оптимизации реального окружения|

---

## 🧩 Практическая связка

1. **Xdebug** — на локали, чтобы детально изучить отдельный запрос или кусок кода.
    
2. **Blackfire** — на staging/prod, чтобы увидеть общую картину и узкие места под реальной нагрузкой.
    

---

## ⚡ Рекомендации

- Для Laravel или Symfony профилируй **bootstrapping**, **Eloquent-запросы**, **Blade/Twig рендеринг**, **middlewares**.
    
- Используй Xdebug при refactor-е и тестировании.
    
- Blackfire — для метрик в CI/CD и регрессий по производительности.
    
- Избегай постоянного включенного Xdebug в продакшене — он резко замедляет.
    

---

## 📘 TL;DR

|Что делать|Чем профилировать|
|---|---|
|Найти медленные функции|Xdebug Profiler|
|Проверить продакшн производительность|Blackfire.io|
|Сравнить до/после оптимизации|Blackfire compare|
|Понять структуру вызовов|QCacheGrind / Blackfire Call Graph|

---

Хочешь — я следующим сделаю файл `php_performance_tuning.md`, где соберу конкретные техники оптимизации:  
оптимизация аллокаций, `opcache`, autoloader, preloading, array/object tradeoffs, string interning, кеширование и флаги компиляции.  
Это будет логичным продолжением `profiling_xdebug_blackfire.md`.