
🧙🏾‍♂️ Отлично! Ты хочешь познакомиться с методами из стандартного **пакета `net/http`** в Go. Ниже — обзор **ключевых типов и их методов**, с которыми чаще всего работают при создании и использовании HTTP-серверов и клиентов.

---

## 📦 Основные типы и методы пакета `net/http`

---

### 🔹 `http.Request` — Входящий HTTP-запрос (на сервер)

```go
type Request struct {
    Method string
    URL *url.URL
    Header Header
    Body io.ReadCloser
    Form url.Values
    PostForm url.Values
    ...
}
```

🛠 Полезные методы и поля:

- `r.Method` — метод запроса (GET, POST, ...)
    
- `r.URL` — URL запроса
    
- `r.Header` — HTTP-заголовки
    
- `r.FormValue("name")` — значение поля формы
    
- `r.ParseForm()` — парсинг query и form
    
- `r.Body` — тело запроса (надо закрывать)
    
- `r.Context()` — контекст запроса (например, для отмены)
    

---

### 🔹 `http.ResponseWriter` — ответ, отправляемый клиенту

Интерфейс:

```go
type ResponseWriter interface {
    Header() Header
    Write([]byte) (int, error)
    WriteHeader(statusCode int)
}
```

🛠 Полезные методы:

- `w.Write([]byte("OK"))` — тело ответа
    
- `w.WriteHeader(404)` — статус ответа (по умолчанию 200)
    
- `w.Header().Set("Content-Type", "application/json")`
    

---

### 🔹 `http.HandleFunc` и `http.ListenAndServe`

```go
http.HandleFunc("/", handlerFunc)
http.ListenAndServe(":8080", nil)
```

- `http.HandleFunc` — регистрирует хендлер
    
- `http.ListenAndServe` — запускает HTTP-сервер
    

---

### 🔹 `http.Client` — HTTP-клиент (для отправки запросов)

```go
client := http.Client{}
resp, err := client.Get("https://example.com")
defer resp.Body.Close()
```

🛠 Методы:

- `client.Get(url)`
    
- `client.Post(url, contentType, body)`
    
- `client.Do(req *http.Request)` — гибкий способ отправки запроса
    

---

### 🔹 `http.NewRequest(method, url string, body io.Reader)` — ручное создание запроса

```go
req, _ := http.NewRequest("POST", "https://example.com", bytes.NewBuffer([]byte(`{"foo":"bar"}`)))
req.Header.Set("Content-Type", "application/json")
client.Do(req)
```

---

### 🔹 `http.ServeMux` — маршрутизатор

```go
mux := http.NewServeMux()
mux.HandleFunc("/hello", helloHandler)
http.ListenAndServe(":8080", mux)
```

---

Хочешь, я соберу пример мини-сервера, использующего эти методы, или покажу клиент с кастомным заголовком?