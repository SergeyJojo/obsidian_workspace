# Cheatsheet: REST API, работа с БД и события в Битрикс

## 🚀 **REST API в Битрикс24 (облако и коробка)**

### Общая структура вызова
```javascript
// Вебхук-URL: https://ваш-портал.bitrix24.ru/rest/ID_пользователя/ВЕБХУК_КЛЮЧ/
// Методы: crm.lead.add, crm.deal.list, crm.contact.update и т.д.
const BX24 = require('bx24-api'); // или прямой fetch

// Пример: получить список сделок
const result = await BX24.callMethod('crm.deal.list', {
    order: {"DATE_CREATE": "DESC"},
    filter: {">OPPORTUNITY": 5000},
    select: ["ID", "TITLE", "OPPORTUNITY", "STAGE_ID"]
});
```

### Основные методы CRM

| Сущность           | Методы                                                | Пример использования                                               |
| ------------------ | ----------------------------------------------------- | ------------------------------------------------------------------ |
| **Лиды**           | `crm.lead.add`, `.list`, `.get`, `.update`, `.delete` | `crm.lead.list({filter: {STATUS_ID: "NEW"}})`                      |
| **Сделки**         | `crm.deal.*`                                          | `crm.deal.add({fields: {TITLE: "Новая сделка", STAGE_ID: "NEW"}})` |
| **Контакты**       | `crm.contact.*`                                       | `crm.contact.get(123)`                                             |
| **Компании**       | `crm.company.*`                                       | `crm.company.list({select: ["*", "UF_*"]})`                        |
| **Смарт-процессы** | `crm.item.*`                                          | `crm.item.list(entityTypeId: 128)`                                 |

**Важно**: В облачной версии доступен только REST API. В коробочной можно использовать как REST, так и внутренний D7 API.

## 🗄️ **Работа с БД в D7 (ORM)**

### 1. **Базовые CRUD-операции**
```php
use Bitrix\Iblock\ElementTable;
use Bitrix\Crm\LeadTable;

// SELECT
$items = ElementTable::getList([
    'select' => ['ID', 'NAME', 'IBLOCK_ID'],
    'filter' => ['=ACTIVE' => 'Y', '>ID' => 100],
    'order' => ['ID' => 'DESC'],
    'limit' => 50,
    'cache' => ['ttl' => 3600] // Кеширование на 1 час
])->fetchAll();

// INSERT
$result = ElementTable::add([
    'NAME' => 'Новый элемент',
    'IBLOCK_ID' => 2,
    'ACTIVE' => 'Y'
]);
$newId = $result->getId();

// UPDATE
ElementTable::update($id, [
    'NAME' => 'Обновленное имя',
    'PREVIEW_TEXT' => 'Новый текст'
]);

// DELETE
ElementTable::delete($id);
```

### 2. **Оптимизация: избегаем N+1 проблем**
```php
// ❌ ПЛОХО: N+1 запросов
$leads = LeadTable::getList()->fetchAll();
foreach ($leads as $lead) {
    $contact = ContactTable::getById($lead['CONTACT_ID'])->fetch(); // +1 запрос!
}

// ✅ ХОРОШО: 1 запрос с JOIN
$leads = LeadTable::getList([
    'select' => [
        'ID', 'TITLE',
        'CONTACT_NAME' => 'CONTACT.NAME',
        'CONTACT_LAST_NAME' => 'CONTACT.LAST_NAME'
    ],
    'runtime' => [
        new \Bitrix\Main\Entity\ReferenceField(
            'CONTACT',
            '\Bitrix\Crm\ContactTable',
            ['=this.CONTACT_ID' => 'ref.ID']
        )
    ]
])->fetchAll();
```

### 3. **Работа со свойствами инфоблоков**
```php
// Получение элементов со свойствами
$elements = \Bitrix\Iblock\ElementTable::getList([
    'select' => [
        'ID', 'NAME',
        'PROPERTY_PRICE_VALUE',
        'PROPERTY_COLOR_VALUE'
    ],
    'filter' => ['=IBLOCK_ID' => 2]
])->fetchAll();

// Массовое обновление свойств
\Bitrix\Iblock\ElementTable::updateMultiple([
    [123, ['PROPERTY_VALUES' => ['COLOR' => 'Красный']]],
    [124, ['PROPERTY_VALUES' => ['COLOR' => 'Синий']]]
]);
```

## 🔔 **События (Events) в D7**

### 1. **Подписка на события**
```php
use \Bitrix\Main\EventManager;

$eventManager = EventManager::getInstance();

// Подписка на событие
$eventManager->addEventHandler(
    'iblock', // Модуль
    'OnAfterIBlockElementAdd', // Событие
    function($fields) {
        // $fields - данные элемента
        if ($fields['IBLOCK_ID'] == 2) {
            // Логика обработки
            \Bitrix\Main\Diag\Debug::writeToFile(
                "Добавлен элемент ID: " . $fields['ID'],
                'events.log'
            );
        }
    }
);
```

### 2. **Создание собственных событий**
```php
// Отправка события
use \Bitrix\Main\Event;

$event = new Event('my.module', 'onCustomAction', [
    'itemId' => $id,
    'data' => $someData
]);
$event->send();

// Обработка события в другом месте
EventManager::getInstance()->addEventHandler(
    'my.module',
    'onCustomAction',
    function(Event $event) {
        $params = $event->getParameters();
        // Обработка данных из $params
    }
);
```

### 3. **Полезные события CRM**

| Событие | Описание | Когда использовать |
|---------|----------|-------------------|
| `OnAfterCrmLeadAdd` | После добавления лида | Автоматизация, нотификации |
| `OnBeforeCrmDealUpdate` | Перед обновлением сделки | Валидация, логирование |
| `OnAfterCrmContactDelete` | После удаления контакта | Очистка связанных данных |
| `OnCrmEntityProcess` | Обработка любой сущности CRM | Универсальные обработчики |

## 📊 **Сравнение REST API и D7 ORM**

| Критерий | REST API | D7 ORM |
|----------|----------|---------|
| **Доступность** | Облако + Коробка | Только Коробка |
| **Скорость** | Медленнее (HTTP) | Быстрее (прямой доступ) |
| **Безопасность** | OAuth/Вебхуки | Права пользователя PHP |
| **Сложность** | Проще для интеграций | Сложнее, но мощнее |
| **Кеширование** | На уровне HTTP | Встроенное в ORM |

## 💡 **Быстрые советы**

1. **Всегда используйте кеширование**:
```php
'cache' => [
    'ttl' => 3600,
    'cache_joins' => true  // Для запросов с JOIN
]
```

2. **Для массовых операций используйте пакетную обработку**:
```php
// Вместо цикла с update() внутри
ElementTable::updateMulti($updates);
```

3. **В REST API всегда проверяйте лимиты**:
```javascript
// Битрикс24 имеет лимиты на запросы
const deals = await BX24.callBatch({
    'deal1': ['crm.deal.list', {filter: {}}],
    'deal2': ['crm.deal.list', {filter: {}}]
});
```

4. **Используйте runtime-поля для сложной логики**:
```php
'runtime' => [
    new \Bitrix\Main\Entity\ExpressionField(
        'FULL_NAME',
        "CONCAT(%s, ' ', %s)",
        ['LAST_NAME', 'NAME']
    )
]
```

5. **Обрабатывайте ошибки в событиях**:
```php
try {
    // Код обработчика
} catch (\Exception $e) {
    AddMessage2Log("Ошибка в обработчике: " . $e->getMessage());
    // НЕ бросайте исключение дальше - сломаете основной процесс
}
```

## 🚨 **Чего избегать**

1. **N+1 запросов** в циклах
2. **Прямых SQL-запросов** (используйте ORM)
3. **Отправки REST-запросов** из PHP коробочной версии (используйте D7 API)
4. **Блокирующих операций** в обработчиках событий
5. **Игнорирования кеширования** для часто используемых данных

Этот читшит покрывает 90% повседневных задач. Для специфичных случаев обращайтесь к [официальной документации](https://dev.1c-bitrix.ru/rest_help/) и [документации по D7](https://dev.1c-bitrix.ru/api_d7/).