Отлично! Переходим ко второй фундаментальной теме:

---

## 🎨 **2. Блочная модель CSS (Box Model)**

### 📚 Теория
**Блочная модель (Box Model)** - это фундаментальная концепция CSS, которая описывает, как браузер рассчитывает размеры и расположение каждого элемента на странице.

**Компоненты блочной модели:**
```
┌─────────────────────────────────────┐
│             margin (внешний отступ) │
│  ┌───────────────────────────────┐  │
│  │         border (граница)      │  │
│  │  ┌─────────────────────────┐  │  │
│  │  │     padding (внутренний │  │  │
│  │  │          отступ)        │  │  │
│  │  │  ┌───────────────────┐  │  │  │
│  │  │  │   content         │  │  │  │
│  │  │  │   (содержимое)    │  │  │  │
│  │  │  └───────────────────┘  │  │  │
│  │  └─────────────────────────┘  │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

### 💻 Синтаксис и свойства

#### Базовые свойства блочной модели:
```css
.element {
    /* Размер содержимого */
    width: 300px;
    height: 200px;
    
    /* Внутренние отступы */
    padding: 20px;
    
    /* Границы */
    border: 2px solid #333;
    
    /* Внешние отступы */
    margin: 15px;
    
    /* Фон для наглядности */
    background-color: #f0f0f0;
}
```

#### Детальная настройка каждого направления:
```css
.box {
    /* Padding - по отдельности */
    padding-top: 10px;
    padding-right: 20px;
    padding-bottom: 10px;
    padding-left: 20px;
    
    /* Или сокращенная запись (по часовой стрелке: верх право низ лево) */
    padding: 10px 20px 10px 20px;
    
    /* Еще короче: верх/низ право/лево */
    padding: 10px 20px;
    
    /* Margin - аналогично */
    margin: 20px 15px 20px 15px;
    margin: 20px 15px; /* верх/низ право/лево */
    
    /* Border - варианты */
    border: 2px solid #000; /* все стороны */
    border-top: 1px dashed red;
    border-right: 3px dotted blue;
    border-bottom: 2px solid green;
    border-left: 1px double purple;
}
```

### 🎯 Практическое применение

#### Пример карточки товара:
```html
<!DOCTYPE html>
<html>
<head>
<style>
.product-card {
    width: 250px;
    padding: 20px;
    margin: 15px;
    border: 1px solid #ddd;
    border-radius: 8px;
    background-color: white;
    box-shadow: 0 2px 5px rgba(0,0,0,0.1);
}

.product-image {
    width: 100%;
    height: 150px;
    background-color: #f5f5f5;
    margin-bottom: 15px;
    border-radius: 4px;
}

.product-title {
    font-size: 18px;
    margin-bottom: 10px;
    padding-bottom: 10px;
    border-bottom: 1px solid #eee;
}

.product-price {
    font-size: 20px;
    font-weight: bold;
    color: #e74c3c;
    margin-top: 10px;
}
</style>
</head>
<body>

<div class="product-card">
    <div class="product-image"></div>
    <h3 class="product-title">Название товара</h3>
    <p>Описание товара здесь...</p>
    <div class="product-price">$199.99</div>
</div>

</body>
</html>
```

### 🔧 Важные концепции

#### 1. **box-sizing: border-box vs content-box**
```css
/* STANDARD (content-box) - устаревший */
.standard-box {
    box-sizing: content-box; /* по умолчанию в некоторых браузерах */
    width: 300px;
    padding: 20px;
    border: 5px solid black;
    /* Фактическая ширина: 300 + 20*2 + 5*2 = 350px */
}

/* MODERN (border-box) - РЕКОМЕНДУЕТСЯ */
.modern-box {
    box-sizing: border-box; /* современный стандарт */
    width: 300px;
    padding: 20px;
    border: 5px solid black;
    /* Фактическая ширина: 300px (padding и border ВКЛЮЧЕНЫ в width) */
}

/* Глобальная установка для всех элементов */
* {
    box-sizing: border-box;
}
```

#### 2. **Схлопывание margin (Margin Collapse)**
```css
.block-a {
    margin-bottom: 30px;
    background: lightblue;
}

.block-b {
    margin-top: 20px;
    background: lightcoral;
}

/* Расстояние между блоками будет 30px (max), а не 50px (30+20) */
```

#### 3. **auto значения для margin**
```css
.center-block {
    width: 300px;
    margin-left: auto;
    margin-right: auto;
    /* Блок будет центрирован по горизонтали */
}

.push-right {
    margin-left: auto;
    /* Блок прижмется к правому краю */
}
```

### ⚠️ Частые ошибки

1. **Непонимание итоговой ширины элемента**
   ```css
   /* Ожидание: 200px, Реальность: 240px */
   .confusing-box {
       width: 200px;
       padding: 20px;
       /* Итог: 200 + 20 + 20 = 240px */
   }
   
   /* Решение: используем border-box */
   .clear-box {
       box-sizing: border-box;
       width: 200px;
       padding: 20px;
       /* Итог: 200px (включая padding) */
   }
   ```

2. **Неожиданное схлопывание margin**
   ```css
   /* Между этими div будет 30px, а не 50px */
   .div1 { margin-bottom: 30px; }
   .div2 { margin-top: 20px; }
   ```

3. **Margin на инлайн-элементах**
   ```css
   /* НЕ РАБОТАЕТ сверху/снизу */
   span {
       margin-top: 10px;    /* Игнорируется */
       margin-bottom: 10px; /* Игнорируется */
   }
   ```

### 🎮 Упражнение для закрепления

**Задача:** Создать профиль пользователя с правильной блочной моделью
```html
<div class="user-profile">
    <div class="avatar"></div>
    <h2 class="username">John Doe</h2>
    <p class="bio">Frontend developer with 3 years of experience...</p>
    <div class="stats">
        <span class="stat">Posts: 156</span>
        <span class="stat">Followers: 1.2K</span>
    </div>
</div>
```

```css
.user-profile {
    /* Ваш CSS здесь */
}

/* Подсказка: используйте border-box, padding, margin, border-radius */
```

### 🔗 Связанные темы
- [[Позиционирование в CSS]]
- [[Flexbox полное руководство]] 
- [[CSS Grid система]]

---

**Теги:** `#css-база #блочная-модель #box-model #верстка`

**Следующая тема:** ▶️ **"Позиционирование в CSS (static, relative, absolute...)"**

---
Переходим к позиционированию? Или хотите практические задания к блочной модели?