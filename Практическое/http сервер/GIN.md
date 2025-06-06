
Вот **подробный пример** работы с веб-фреймворком [Gin](https://github.com/gin-gonic/gin) в Go, включая обработку запросов, middleware, валидацию и работу с JSON.

---

## **1. Установка Gin**
Добавьте Gin в проект:
```bash
go get -u github.com/gin-gonic/gin
```

---

## **2. Базовое API (Hello World)**
```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default() // Создаем роутер с дефолтными middleware (логгер и recovery)

	// Маршрут GET /ping
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"message": "pong",
		})
	})

	r.Run(":8080") // Запускаем сервер на :8080
}
```
**Проверка**:
```bash
curl http://localhost:8080/ping
# Ответ: {"message":"pong"}
```

---

## **3. Параметры запроса**
### **Path-параметры**
```go
r.GET("/user/:name", func(c *gin.Context) {
	name := c.Param("name") // Получаем параметр из URL
	c.String(http.StatusOK, "Hello, %s!", name)
})
```
**Пример**:  
`GET /user/Alice` → `Hello, Alice!`

### **Query-параметры**
```go
r.GET("/search", func(c *gin.Context) {
	query := c.Query("q")       // Обязательный параметр
	page := c.DefaultQuery("page", "1") // С дефолтным значением
	c.String(http.StatusOK, "Search: %s, Page: %s", query, page)
})
```
**Пример**:  
`GET /search?q=golang&page=2` → `Search: golang, Page: 2`

---

## **4. Работа с JSON**
### **Прием JSON (POST)**
```go
type User struct {
	Name  string `json:"name" binding:"required"`
	Email string `json:"email" binding:"required,email"`
}

r.POST("/users", func(c *gin.Context) {
	var user User
	if err := c.ShouldBindJSON(&user); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	c.JSON(http.StatusCreated, gin.H{"status": "user created", "data": user})
})
```
**Проверка**:
```bash
curl -X POST http://localhost:8080/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@example.com"}'
```

### **Отправка JSON**
```go
r.GET("/user", func(c *gin.Context) {
	user := User{
		Name:  "Bob",
		Email: "bob@example.com",
	}
	c.JSON(http.StatusOK, user) // Автоматический Content-Type: application/json
})
```

---

## **5. Middleware**
### **Кастомный middleware (логирование)**
```go
func Logger() gin.HandlerFunc {
	return func(c *gin.Context) {
		fmt.Printf("Request: %s %s\n", c.Request.Method, c.Request.URL.Path)
		c.Next() // Передаем управление следующему обработчику
	}
}

func main() {
	r := gin.New()
	r.Use(Logger()) // Подключаем middleware глобально

	r.GET("/", func(c *gin.Context) {
		c.String(http.StatusOK, "Middleware works!")
	})
}
```

### **Группы маршрутов с middleware**
```go
authGroup := r.Group("/admin", gin.BasicAuth(gin.Accounts{
	"admin": "secret",
}))

authGroup.GET("/dashboard", func(c *gin.Context) {
	c.String(http.StatusOK, "Admin dashboard!")
})
```

---

## **6. Загрузка файлов**
```go
// Загрузка одного файла
r.POST("/upload", func(c *gin.Context) {
	file, _ := c.FormFile("file") // Получаем файл из формы
	c.SaveUploadedFile(file, "./uploads/"+file.Filename)
	c.String(http.StatusOK, "File %s uploaded!", file.Filename)
})

// Множественная загрузка
r.POST("/upload-multiple", func(c *gin.Context) {
	form, _ := c.MultipartForm()
	files := form.File["files"]

	for _, file := range files {
		c.SaveUploadedFile(file, "./uploads/"+file.Filename)
	}
	c.String(http.StatusOK, "Uploaded %d files!", len(files))
})
```

---

## **7. Валидация данных**
Используйте библиотеку `validator` (встроена в Gin):
```go
type LoginForm struct {
	Username string `form:"username" binding:"required,min=3,max=20"`
	Password string `form:"password" binding:"required,alphanum"`
}

r.POST("/login", func(c *gin.Context) {
	var form LoginForm
	if err := c.ShouldBind(&form); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	c.JSON(http.StatusOK, gin.H{"status": "logged in"})
})
```

---

## **8. Статические файлы**
```go
r.Static("/assets", "./public") // Доступ к файлам из ./public по /assets
r.StaticFile("/favicon.ico", "./resources/favicon.ico") // Один файл
```

---

## **9. Полный пример API**
```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

type Product struct {
	ID    int     `json:"id"`
	Name  string  `json:"name" binding:"required"`
	Price float64 `json:"price" binding:"required,gte=0"`
}

var products = []Product{
	{1, "Laptop", 999.99},
	{2, "Phone", 699.99},
}

func main() {
	r := gin.Default()

	// Маршруты
	r.GET("/products", getProducts)
	r.POST("/products", createProduct)
	r.GET("/products/:id", getProductByID)

	r.Run(":8080")
}

func getProducts(c *gin.Context) {
	c.JSON(http.StatusOK, products)
}

func createProduct(c *gin.Context) {
	var newProduct Product
	if err := c.ShouldBindJSON(&newProduct); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	newProduct.ID = len(products) + 1
	products = append(products, newProduct)
	c.JSON(http.StatusCreated, newProduct)
}

func getProductByID(c *gin.Context) {
	id := c.Param("id")
	for _, p := range products {
		if fmt.Sprint(p.ID) == id {
			c.JSON(http.StatusOK, p)
			return
		}
	}
	c.JSON(http.StatusNotFound, gin.H{"error": "Product not found"})
}
```

---

## **10. Тестирование API**
Установите [httptest](https://pkg.go.dev/net/http/httptest) для модульного тестирования:
```go
func TestGetProducts(t *testing.T) {
	r := gin.Default()
	r.GET("/products", getProducts)

	req, _ := http.NewRequest("GET", "/products", nil)
	w := httptest.NewRecorder()
	r.ServeHTTP(w, req)

	assert.Equal(t, http.StatusOK, w.Code)
	assert.Contains(t, w.Body.String(), "Laptop")
}
```

---

### **Итог**
- **Гибкость**: Gin поддерживает REST, middleware, валидацию.
- **Производительность**: Основан на `httprouter`, один из самых быстрых фреймворков Go.
- **Документация**: [Официальный GitHub](https://github.com/gin-gonic/gin#readme).

Для продакшена добавьте:
- **Graceful shutdown**.
- **JWT-аутентификацию** (используйте [gin-jwt](https://github.com/appleboy/gin-jwt)).
- **Swagger-документацию** ([swaggo](https://github.com/swaggo/swag)).