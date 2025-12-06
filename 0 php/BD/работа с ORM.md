В Laravel основной ORM — **Eloquent**. Он предоставляет мощный, выразительный и удобный способ взаимодействия с базой данных через PHP-объекты (модели). Ниже — **основные методы и подходы** работы с Eloquent ORM.

---

## 🔹 1. **Создание модели**

```bash
php artisan make:model Post
```

Модель автоматически связывается с таблицей `posts` (по правилу именования).

```php
// app/Models/Post.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    // $table = 'posts'; // можно указать явно
    // $primaryKey = 'id';
    // public $timestamps = true;
}
```

---

## 🔹 2. **Получение данных (Select)**

### Все записи
```php
$posts = Post::all();
```

### По условию
```php
Post::where('status', 'published')->get();
Post::where('views', '>', 100)->first();
```

### По ID
```php
Post::find(1);        // вернёт модель или null
Post::findOrFail(1);  // вернёт модель или 404
```

### Ограничение, сортировка, выбор полей
```php
Post::select('title', 'slug')
    ->where('active', true)
    ->orderBy('created_at', 'desc')
    ->limit(10)
    ->get();
```

### Агрегаты
```php
Post::count();
Post::max('views');
Post::avg('rating');
```

---

## 🔹 3. **Создание записей (Insert)**

### Способ 1: через `create()` (требует `$fillable`)
```php
// В модели:
protected $fillable = ['title', 'content'];

// В коде:
Post::create(['title' => 'Новый пост', 'content' => '...']);
```

### Способ 2: через экземпляр
```php
$post = new Post;
$post->title = 'Новый пост';
$post->content = '...';
$post->save();
```

---

## 🔹 4. **Обновление записей (Update)**

### Через найденную модель
```php
$post = Post::find(1);
$post->title = 'Обновлённый заголовок';
$post->save();
```

### Массовое обновление
```php
Post::where('status', 'draft')->update(['status' => 'published']);
```

> ⚠️ Массовое обновление **не вызывает** Eloquent-события (`saving`, `saved` и т.д.).

---

## 🔹 5. **Удаление записей (Delete)**

### Через модель
```php
$post = Post::find(1);
$post->delete();
```

### Массовое удаление
```php
Post::where('status', 'spam')->delete();
```

### Мягкое удаление (soft delete)
В модели:
```php
use Illuminate\Database\Eloquent\SoftDeletes;

class Post extends Model
{
    use SoftDeletes;
}
```

Теперь:
```php
Post::find(1)->delete(); // не удаляет физически, ставит deleted_at
Post::withTrashed()->get(); // получить даже удалённые
Post::onlyTrashed()->restore(); // восстановить
```

---

## 🔹 6. **Отношения (Relationships)**

### Один ко многим
```php
// User → много Post
class User extends Model
{
    public function posts()
    {
        return $this->hasMany(Post::class);
    }
}

// Использование:
$user->posts; // коллекция Post
```

### Многие к одному
```php
// Post → один User
class Post extends Model
{
    public function author()
    {
        return $this->belongsTo(User::class, 'user_id');
    }
}
```

### Многие ко многим
```php
class User extends Model
{
    public function roles()
    {
        return $this->belongsToMany(Role::class);
    }
}
```

> По умолчанию Laravel ищет таблицу `role_user` с колонками `user_id`, `role_id`.

---

## 🔹 7. **Ленивая и жадная загрузка (Eager Loading)**

### Ленивая загрузка (N+1 проблема):
```php
$posts = Post::all();
foreach ($posts as $post) {
    echo $post->author->name; // отдельный запрос на каждого автора!
}
```

### Жадная загрузка (решение):
```php
$posts = Post::with('author')->get(); // 2 запроса всего
```

Можно загружать вложенные отношения:
```php
Post::with('author.roles')->get();
```

---

## 🔹 8. **Области (Scopes)**

### Локальный scope:
```php
// В модели Post
public function scopePublished($query)
{
    return $query->where('status', 'published');
}

// Использование:
Post::published()->get();
```

### Глобальный scope:
```php
// В модели
protected static function boot()
{
    parent::boot();
    static::addGlobalScope('active', function (Builder $builder) {
        $builder->where('active', true);
    });
}
```

---

## 🔹 9. **События модели**

Eloquent генерирует события: `creating`, `created`, `updating`, `updated`, `deleting`, `deleted`, и т.д.

Можно перехватить:
```php
Post::created(function ($post) {
    // отправить уведомление, логировать и т.д.
});
```

Или в модели:
```php
protected static function booted()
{
    static::created(fn ($post) => ...);
}
```

---

## 🔹 10. **Кастомизация поведения**

- `$fillable` / `$guarded` — массовое присвоение
- `$casts` — автоматическое преобразование типов:
  ```php
  protected $casts = [
      'is_published' => 'boolean',
      'published_at' => 'datetime',
      'meta' => 'array', // JSON ↔ массив
  ];
  ```
- `$appends` — виртуальные атрибуты через аксессоры

---

## 🔹 11. **Query Builder vs Eloquent**

Eloquent **расширяет** Query Builder. Можно комбинировать:

```php
// Query Builder (без модели)
DB::table('posts')->where('views', '>', 100)->get();

// Eloquent (с моделью и поведением)
Post::where('views', '>', 10)->with('author')->get();
```

> ✅ Используй **Eloquent**, когда нужна логика модели, отношения, события.  
> ✅ Используй **Query Builder**, когда нужен простой и быстрый запрос без накладных расходов.

---

Если тебе нужно — могу привести **реальные примеры под твой кейс**: API, микросервис, обработка данных и т.д.