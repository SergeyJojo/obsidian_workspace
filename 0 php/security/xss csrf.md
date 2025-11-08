Отлично, Сергей — разберём XSS и CSRF глубоко и практично: как они работают, примеры уязвимостей, и конкретные меры в PHP + фронт/сервер, которые реально защищают приложение. В конце — чеклист для продакшена и готовые сниппеты.

# XSS (Cross‑Site Scripting)

## Что это и почему опасно

XSS — внедрение произвольного JavaScript/HTML в страницы, которые видят другие пользователи. Атакующий может украсть сессии, выполнить действия от имени пользователя, изменить интерфейс, фишить форму ввода и т.д.

## Виды XSS

1. **Stored (persistent)** — вредоносный скрипт сохраняется в базе (комментарий, профиль) и показывается всем.
    
2. **Reflected (non‑persistent)** — вредоносный payload отправлен в URL/форме и отражён в ответе (обычно через поисковую строку).
    
3. **DOM‑based** — уязвимость в клиентском JS: данные из location/hash/document.write и т.п. используются напрямую.
    

## Принцип защиты — _never trust input; always encode on output_

Основная идея: никогда не подставлять сырые данные в HTML без контекстного экранирования.

### Контекстное экранирование (важнейшее)

Разные контексты требуют разного экранирования:

- **HTML body text**: используем `htmlspecialchars($s, ENT_QUOTES|ENT_SUBSTITUTE, 'UTF-8')`.
    
- **HTML attribute**: тоже `htmlspecialchars`, но избегаем вставки атрибутов без кавычек.
    
- **Inside JavaScript**: не просто `htmlspecialchars` — либо JSON‑энкодьте данные и вставляйте в JS через `json_encode` на сервере, либо используйте безопасные шаблоны.
    
- **URL (href/src)**: `rawurlencode` для частей, `filter_var($url, FILTER_VALIDATE_URL)` для валидации.
    
- **CSS context / style attribute**: избегать прямой вставки пользовательских строк в CSS; если необходимо — тщательно валидация и whitelist.
    

### PHP — helper для безопасного вывода

```php
function e($s): string {
    return htmlspecialchars((string)$s, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
}
```

Примеры:

```php
// В теле HTML
echo '<p>' . e($user['bio']) . '</p>';

// В атрибуте
echo '<input value="' . e($value) . '">';

// В JS через JSON:
echo '<script>window.__DATA__ = ' . json_encode($data, JSON_HEX_TAG|JSON_HEX_AMP|JSON_HEX_APOS|JSON_HEX_QUOT) . ';</script>';
```

### Предотвращение DOM‑XSS

- Не присваивайте `innerHTML` пользовательским данным — используйте `textContent` или `innerText`.
    
- Если нужно вставить HTML — используйте доверенные шаблоны или очищайте через библиотеку (HTMLPurifier).
    
- Не формируйте DOM из `location.hash`/`location.search` без валидации.
    

### Content Security Policy (CSP)

CSP — сильный инструмент: блокирует выполнение неподписанного inline‑JS и запросы к неразрешённым источникам. Пример базовой политики (в header):

```php
header("Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'self';");
```

Замечание: впроваджение CSP требует тестирования (Report‑only режим).

### HttpOnly и Secure куки

Установите для сессионных куки флаги `HttpOnly; Secure; SameSite=Strict/Lax` — уменьшает риск кражи сессионной куки через XSS/CSRF.

### Санитизация HTML

Если вы принимаете HTML от пользователя (редактор WYSIWYG), используйте проверенные библиотеки (в PHP — например, HTMLPurifier) и whitelist тэгов/атрибутов. Не пытайтесь писать свою санитизацию.

---

# CSRF (Cross‑Site Request Forgery)

## Что это и почему опасно

CSRF — атака, при которой злоумышленник вынуждает аутентифицированного пользователя выполнить нежелательное действие на доверенном сайте (например, сменить e‑mail, перевести деньги), отправив запрос с его браузера.

Ключ: браузер автоматически отправляет куки/credentials к доверенному домену при запросе, поэтому внешний сайт может инициировать запрос.

## Как это работает (упрощённо)

1. Пользователь залогинен на `bank.com` (есть сессионная кука).
    
2. Пользователь посещает `evil.com`.
    
3. `evil.com` содержит форму/скрипт, который отправляет POST на `https://bank.com/transfer` — браузер прикрепляет куки и действие выполнится от имени пользователя.
    

## Надёжные способы защиты

### 1) CSRF‑токены (synchronizer token pattern) — стандарт

- На стороне сервера при рендеринге формы генерируйте крипто‑стойкий токен, сохраняйте в сессии и вставляйте в скрытое поле формы.
    
- При обработке запроса проверяйте токен (равен ли тому, что в сессии).
    

Пример:

```php
// При формировании формы
session_start();
if (!isset($_SESSION['csrf_token'])) {
    $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
}
$token = $_SESSION['csrf_token'];
// В форме:
echo '<input type="hidden" name="csrf_token" value="' . htmlspecialchars($token, ENT_QUOTES, 'UTF-8') . '">';
```

Проверка:

```php
session_start();
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $token = $_POST['csrf_token'] ?? '';
    if (!hash_equals($_SESSION['csrf_token'] ?? '', $token)) {
        http_response_code(403);
        echo 'CSRF validation failed';
        exit;
    }
    // OK — выполняем действие
}
```

`hash_equals` — чтобы избежать timing‑attacks.

### 2) SameSite куки

Установите сессионной куке `SameSite=Lax` или `Strict`. Современные браузеры по умолчанию применяют `Lax`, что уже блокирует многие CSRF через GET/unsafe navigations. Для чувствительных операций рекомендуется `Strict` или дополнительная проверка токена.

Пример установки (PHP 7.3+):

```php
session_set_cookie_params([
    'lifetime' => 0,
    'path' => '/',
    'domain' => 'example.com',
    'secure' => true,
    'httponly' => true,
    'samesite' => 'Lax',
]);
session_start();
```

### 3) Double Submit Cookie (альтернатива)

- На логине сервер ставит CSRF cookie с случайным значением.
    
- JS читает cookie и добавляет значение в заголовок/поле формы.
    
- Сервер сравнивает значение в cookie и в теле запроса. Это безопасно, если cookie метится `HttpOnly=false` (чтобы JS мог прочитать) и все остальные риски учтены.
    

### 4) Проверка `Origin`/`Referer`

- Для чувствительных POST/PUT/DELETE запросов проверяйте `Origin` или `Referer` (Origin предпочтительней). `Origin` содержит схему+хост+порт. Если не совпадает с вашим, отклоняйте запрос.
    
- Подходит для API и форм, но не всегда доступно (старые браузеры).
    

Пример:

```php
$origin = $_SERVER['HTTP_ORIGIN'] ?? $_SERVER['HTTP_REFERER'] ?? null;
if ($origin) {
    $allowed = ['https://example.com'];
    $host = parse_url($origin, PHP_URL_SCHEME) . '://' . parse_url($origin, PHP_URL_HOST);
    if (!in_array($host, $allowed, true)) {
        http_response_code(403);
        exit('Invalid origin');
    }
}
```

### 5) Используйте CORS корректно для API

- Если API должен принимать запросы только с вашего frontend, настройте CORS (Access‑Control‑Allow‑Origin) и не разрешайте `*`.
    
- Для cookie‑based auth помните: кросс‑доменные запросы требуют `withCredentials=true` и строгой CORS политики.
    

### 6) Для AJAX — передавайте токен в заголовке

JS:

```js
fetch('/action', {
  method: 'POST',
  headers: { 'X-CSRF-Token': csrfToken, 'Content-Type': 'application/json' },
  body: JSON.stringify(data),
  credentials: 'include'
});
```

Сервер читает из заголовка `X-CSRF-Token`.

---

# Дополнительные рекомендации (общие)

- **POST/PUT/DELETE**: защищать CSRF токенами; для idempotent GET — обычно не критично, но избегайте работать побочками.
    
- **JWT**: если храните JWT в localStorage — уязвимы к XSS. Лучше хранить в cookie с `HttpOnly; Secure; SameSite`.
    
- **Минимизируйте attack surface**: отключайте inline‑JS/inline‑styles, используйте CSP.
    
- **Шаблонизаторы**: пользуйтесь авто‑экранирующими шаблонизаторами (Twig, Blade и т.д.) — они по умолчанию экранируют вывод.
    
- **Логирование и alerting**: логируйте 403 на CSRF ошибки и подозрительные Origin/Referer.
    
- **Тестирование**: включайте в CI‑suite сканы и ручное тестирование XSS/CSRF (проверить все формы, AJAX endpoints, старые урлы).
    

---

# Быстрый чеклист для продакшена (минимум)

1. Все выводы в HTML — через `htmlspecialchars` / автоэкранирование.
    
2. Для HTML‑входа используйте проверенные санитайзеры (HTMLPurifier).
    
3. Установите CSP (report‑only сначала).
    
4. Сессионные куки: `Secure; HttpOnly; SameSite=Lax/Strict`.
    
5. Все state‑changing запросы (POST/PUT/DELETE) — защищены CSRF‑токеном.
    
6. Проверяйте `Origin/Referer` для критичных операций.
    
7. Composer autoload — безопасные шаблоны/фреймворк middleware (Laravel/ Symfony уже имеют CSRF middleware).
    
8. Логи 403 и мониторинг CSP отчетов.
    

---

# Полезные сниппеты (ещё раз — PHP)

**Генерация/вставка CSRF в форму**

```php
session_start();
if (!isset($_SESSION['csrf'])) $_SESSION['csrf'] = bin2hex(random_bytes(32));
function csrf_field() {
    $t = $_SESSION['csrf'];
    return '<input type="hidden" name="csrf" value="'.htmlspecialchars($t, ENT_QUOTES, 'UTF-8').'">';
}
```

**Проверка CSRF**

```php
session_start();
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $token = $_POST['csrf'] ?? '';
    if (!hash_equals($_SESSION['csrf'] ?? '', $token)) {
        http_response_code(403);
        exit('CSRF fail');
    }
}
```

**CSP (пример заголовка)**

```php
header("Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-{$nonce}'; object-src 'none'; connect-src 'self'; frame-ancestors 'none'; base-uri 'self';");
```

(используйте nonce при необходимости inline‑скриптов)

---

Если хочешь, могу:

- Сделать полный **middleware**‑пример на чистом PHP (routing + CSRF middleware + XSS helper).
    
- Подготовить таблицу «контекст → функция экранирования» для вставки в README проекта.
    
- Рассмотреть твой реальный код/страницу и показать, где конкретно слабые места по XSS/CSRF (пришли фрагменты).
    

Что предпочитаешь дальше?