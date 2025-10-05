
### **Краткое содержание статьи: Google OAuth2 Login в Go (минимальное и полное руководство)**  

#### **🔹 Основные шаги для реализации Google OAuth2 в Go:**  
1. **Создание OAuth2 клиента в Google Cloud Console**  
   - Зайти в [Google Cloud Console](https://console.cloud.google.com/).  
   - Создать новый проект (или выбрать существующий).  
   - Перейти в **APIs & Services → Credentials**.  
   - Нажать **"Create Credentials" → OAuth Client ID**.  
   - Выбрать тип приложения (**Web Application**).  
   - Указать **Authorized Redirect URI** (например, `http://localhost:8080/auth/google/callback`).  

2. **Установка необходимых библиотек**  
   ```bash
   go get golang.org/x/oauth2
   go get golang.org/x/oauth2/google
   ```

3. **Настройка OAuth2 конфигурации в Go**  
   ```go
   var googleOauthConfig = &oauth2.Config{
       RedirectURL:  "http://localhost:8080/auth/google/callback",
       ClientID:     "ваш-client-id.apps.googleusercontent.com",
       ClientSecret: "ваш-client-secret",
       Scopes:       []string{"https://www.googleapis.com/auth/userinfo.email"},
       Endpoint:     google.Endpoint,
   }
   ```

4. **Роутинг и обработка запросов**  
   - **Шаг 1:** Перенаправление пользователя на Google OAuth2 страницу.  
     ```go
     func handleGoogleLogin(w http.ResponseWriter, r *http.Request) {
         url := googleOauthConfig.AuthCodeURL("random-state")
         http.Redirect(w, r, url, http.StatusTemporaryRedirect)
     }
     ```
   - **Шаг 2:** Обработка callback от Google.  
     ```go
     func handleGoogleCallback(w http.ResponseWriter, r *http.Request) {
         // Проверка state-параметра (CSRF-защита)
         if r.FormValue("state") != "random-state" {
             http.Error(w, "Invalid state", http.StatusBadRequest)
             return
         }
         
         // Получение токена
         token, err := googleOauthConfig.Exchange(context.Background(), r.FormValue("code"))
         if err != nil {
             http.Error(w, "Failed to exchange token", http.StatusInternalServerError)
             return
         }
         
         // Получение данных пользователя
         resp, err := http.Get("https://www.googleapis.com/oauth2/v2/userinfo?access_token=" + token.AccessToken)
         if err != nil {
             http.Error(w, "Failed to get user info", http.StatusInternalServerError)
             return
         }
         defer resp.Body.Close()
         
         // Парсинг JSON-ответа
         var userInfo struct {
             Email string `json:"email"`
             Name  string `json:"name"`
         }
         json.NewDecoder(resp.Body).Decode(&userInfo)
         
         fmt.Fprintf(w, "Hello, %s (%s)!", userInfo.Name, userInfo.Email)
     }
     ```

5. **Запуск сервера**  
   ```go
   func main() {
       http.HandleFunc("/", handleHome)
       http.HandleFunc("/login", handleGoogleLogin)
       http.HandleFunc("/auth/google/callback", handleGoogleCallback)
       
       fmt.Println("Server running on :8080")
       http.ListenAndServe(":8080", nil)
   }
   ```

---

### **🔹 Важные моменты:**  
✅ **`state`-параметр** — защищает от CSRF-атак.  
✅ **Scopes** — определяют, какие данные запрашиваются (например, `userinfo.email`).  
✅ **Токен** — обменивается на код (`code`) после аутентификации.  
✅ **UserInfo Endpoint** — `https://www.googleapis.com/oauth2/v2/userinfo` возвращает email и имя.  

---

### **🔹 Итог:**  
Статья даёт **готовый минимальный код** для реализации Google OAuth2 в Go.  
- Подходит для быстрого старта.  
- Можно расширить (например, добавить сессии через JWT или Cookies).  
- Полный код есть в [оригинальной статье](https://medium.com/@aynacialiriza/google-oauth2-login-in-go-a-minimal-and-complete-guide-0e9af75908de).  

🚀 **Теперь вы можете добавить Google-логин в своё Go-приложение!**