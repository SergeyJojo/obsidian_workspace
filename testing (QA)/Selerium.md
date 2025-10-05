### **Полный гайд по работе с Selenium для автоматизации тестирования веб-приложений**

---

## **1. Введение в Selenium**
**Selenium** — это фреймворк для автоматизированного тестирования веб-приложений. Поддерживает языки: Python, Java, C#, JavaScript и др.

### **Что можно тестировать?**
- Открытие страниц, клики, ввод данных.
- Проверка элементов на странице (тексты, атрибуты).
- Работа с cookies, алертами, iframe.

### **Компоненты Selenium**:
- **Selenium WebDriver** — ядро для управления браузером.
- **Selenium Grid** — запуск тестов на нескольких машинах.
- **Selenium IDE** (устарел) — запись действий в браузере.

---

## **2. Установка и настройка**
### **Для Python**:
```bash
pip install selenium
```
### **Драйверы браузеров**:
- **Chrome**: [ChromeDriver](https://sites.google.com/chromium.org/driver/)
- **Firefox**: [GeckoDriver](https://github.com/mozilla/geckodriver)
- **Edge**: [EdgeDriver](https://developer.microsoft.com/en-us/microsoft-edge/tools/webdriver/)

Положите драйвер в `PATH` или укажите путь явно.

---

## **3. Базовые команды WebDriver**
### **Запуск браузера**:
```python
from selenium import webdriver

# Для Chrome
driver = webdriver.Chrome()  
# Для Firefox
# driver = webdriver.Firefox()

driver.get("https://example.com")  # Открыть URL
```

### **Поиск элементов**:
```python
# По ID
element = driver.find_element("id", "username")

# По CSS-селектору
button = driver.find_element("css selector", "button.submit")

# По XPath
link = driver.find_element("xpath", "//a[@href='/login']")
```

### **Действия с элементами**:
```python
element.click()          # Клик
element.send_keys("text") # Ввод текста
element.clear()          # Очистить поле
```

### **Получение данных**:
```python
text = element.text              # Текст элемента
attribute = element.get_attribute("href")  # Атрибут
is_displayed = element.is_displayed()      # Видим ли элемент?
```

---

## **4. Ожидания (Waits)**
Чтобы избежать ошибок из-за асинхронной загрузки страницы.

### **Явные ожидания** (рекомендуются):
```python
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

wait = WebDriverWait(driver, 10)  # Макс. 10 сек
element = wait.until(
    EC.presence_of_element_located(("id", "dynamic-element"))
)
```

### **Неявные ожидания** (глобальные):
```python
driver.implicitly_wait(5)  # Ждать до 5 сек при поиске элементов
```

---

## **5. Работа с окнами, вкладками и alert-ами**
### **Переключение между вкладками**:
```python
driver.switch_to.window(driver.window_handles[1])  # Вторая вкладка
```

### **Alert-ы**:
```python
alert = driver.switch_to.alert
alert.accept()      # Принять
alert.dismiss()     # Отклонить
text = alert.text   # Текст алерта
```

### **Iframe**:
```python
driver.switch_to.frame("iframe-name")  # Переключиться в iframe
driver.switch_to.default_content()     # Вернуться
```

---

## **6. Расширенные возможности**
### **Скриншоты**:
```python
driver.save_screenshot("screenshot.png")
```

### **Cookies**:
```python
driver.add_cookie({"name": "test", "value": "123"})
cookies = driver.get_cookies()
```

### **JavaScript**:
```python
driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
```

---

## **7. Пример теста для авторизации**
```python
import unittest
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

class TestLogin(unittest.TestCase):
    def setUp(self):
        self.driver = webdriver.Chrome()
        self.driver.get("https://example.com/login")

    def test_login(self):
        driver = self.driver
        driver.find_element("id", "username").send_keys("admin")
        driver.find_element("id", "password").send_keys("password")
        driver.find_element("css selector", "button[type='submit']").click()
        
        # Проверка успешного входа
        welcome = WebDriverWait(driver, 5).until(
            EC.presence_of_element_located((By.ID, "welcome"))
        )
        self.assertIn("Welcome", welcome.text)

    def tearDown(self):
        self.driver.quit()

if __name__ == "__main__":
    unittest.main()
```

---

## **8. Интеграция с Pytest**
Установите плагин:
```bash
pip install pytest pytest-selenium
```

**Пример теста**:
```python
# test_login.py
def test_login(selenium):
    selenium.get("https://example.com/login")
    selenium.find_element("id", "username").send_keys("admin")
    selenium.find_element("id", "password").send_keys("password")
    selenium.find_element("css selector", "button").click()
    assert "Welcome" in selenium.title
```

Запуск:
```bash
pytest test_login.py --driver Chrome
```

---

## **9. Лучшие практики**
1. **Page Object Model (POM)** — разделяйте код на классы (1 класс = 1 страница).
2. **Фикстуры** — используйте `setUp`/`tearDown` для подготовки данных.
3. **Параметризация** — запускайте тесты с разными данными (см. `@pytest.mark.parametrize`).
4. **Логирование** — добавляйте логи для отладки.

---

## **10. Альтернативы Selenium**
- **Playwright** — современный аналог с поддержкой мобильных браузеров.
- **Cypress** — для JavaScript (удобный, но только под Chrome).

---

### **Итог**
Selenium — мощный инструмент для автоматизации веб-тестов. Освоив базовые методы, вы сможете:
- Писать стабильные UI-тесты.
- Интегрировать их в CI/CD (Jenkins, GitHub Actions).
- Тестировать сложные сценарии (например, платежи).  

**Ресурсы для углубления**:  
- [Официальная документация](https://www.selenium.dev/documentation/)  
- [Курс на Stepik](https://stepik.org/course/575/)  
- [Примеры проектов на GitHub](https://github.com/SeleniumHQ/selenium)  

Начните с малого — автоматизируйте вход на сайт, и постепенно усложняйте задачи! 🚀