# برنامه‌نویسی شبکه و HTTP در زبان Go

این جزوه به بررسی جامع برنامه‌نویسی شبکه و پروتکل HTTP در Go می‌پردازد. Go به دلیل کتابخانه استاندارد قدرتمند (`net/http`) و سادگی در مدیریت همزمانی، برای ساخت سرورهای وب و برنامه‌های شبکه‌ای بسیار مناسب است. تمام موضوعات درخواستی با جزئیات کامل و مثال‌های عملی شرح داده شده‌اند.

---

## 1. سرور HTTP پایه

### 1.1. مفهوم
پکیج `net/http` ابزارهای لازم برای ساخت سرورهای HTTP را فراهم می‌کند. یک سرور HTTP درخواست‌ها را دریافت کرده، پردازش می‌کند و پاسخ مناسب را ارسال می‌کند.

### 1.2. استفاده از net/http
#### ایجاد سرور ساده
```go
package main

import (
    "fmt"
    "net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, World!")
}

func main() {
    http.HandleFunc("/", handler)
    fmt.Println("Server running on :8080")
    http.ListenAndServe(":8080", nil)
}
```
- **اجرا**: کد را اجرا کنید و به `http://localhost:8080` بروید تا خروجی `Hello, World!` را ببینید.
- **توضیحات**:
  - `http.HandleFunc`: یک مسیر (route) را به یک تابع هندلر متصل می‌کند.
  - `http.ResponseWriter`: برای نوشتن پاسخ به کلاینت استفاده می‌شود.
  - `http.Request`: اطلاعات درخواست (مثل مسیر، هدرها) را نگه می‌دارد.
  - `ListenAndServe`: سرور را روی پورت مشخص راه‌اندازی می‌کند.

### 1.3. هندلرها
هندلرها توابعی هستند که درخواست‌های HTTP را پردازش می‌کنند. دو نوع اصلی:
- **HandleFunc**: برای توابع ساده.
- **Handler**: برای پیاده‌سازی رابط `http.Handler`:
  ```go
  type CustomHandler struct{}

  func (h CustomHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
      w.Write([]byte("Custom Handler"))
  }

  func main() {
      http.Handle("/", CustomHandler{})
      http.ListenAndServe(":8080", nil)
  }
  ```

### 1.4. پاسخ به درخواست‌ها
- **تنظیم هدرها**:
  ```go
  w.Header().Set("Content-Type", "text/plain")
  ```
- **کد وضعیت**:
  ```go
  w.WriteHeader(http.StatusOK) // 200
  ```
- **ارسال خطا**:
  ```go
  http.Error(w, "Not Found", http.StatusNotFound) // 404
  ```

### 1.5. نکات پیشرفته
- **مدیریت همزمانی**: هر درخواست در یک goroutine جداگانه پردازش می‌شود.
- **مهلت زمانی سرور**:
  ```go
  server := &http.Server{
      Addr:         ":8080",
      ReadTimeout:  10 * time.Second,
      WriteTimeout: 10 * time.Second,
  }
  server.ListenAndServe()
  ```
- **خاموشی منظم**:
  ```go
  ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
  defer cancel()
  server.Shutdown(ctx)
  ```

### 1.6. خطاهای رایج
- عدم بررسی خطاهای `ListenAndServe`.
- تنظیم هدرها پس از نوشتن پاسخ (باعث خطا می‌شود).

---

## 2. مدیریت درخواست‌ها

### 2.1. مفهوم
مدیریت درخواست‌ها شامل استخراج اطلاعات از درخواست (مثل Query Parameters، Form Data، و Method‌ها) و پاسخ مناسب است.

### 2.2. کار با Query Parameters
Query Parameters در URL (مثل `?name=Ali`) قرار دارند.
```go
func handler(w http.ResponseWriter, r *http.Request) {
    name := r.URL.Query().Get("name")
    if name == "" {
        name = "Guest"
    }
    fmt.Fprintf(w, "Hello, %s!", name)
}
```

### 2.3. کار با Form Data
برای درخواست‌های POST یا PUT که داده‌های فرم دارند:
```go
func handler(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        return
    }
    r.ParseForm()
    username := r.Form.Get("username")
    fmt.Fprintf(w, "Welcome, %s!", username)
}
```
- **نکته**: برای فایل‌ها از `r.ParseMultipartForm` استفاده کنید.

### 2.4. مدیریت Method‌ها
```go
func handler(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodGet:
        fmt.Fprintf(w, "GET request received")
    case http.MethodPost:
        fmt.Fprintf(w, "POST request received")
    default:
        http.Error(w, "Method not supported", http.StatusMethodNotAllowed)
    }
}
```

### 2.5. نکات پیشرفته
- **استخراج مسیر پویا**:
  ```go
  path := r.URL.Path // مثلاً /users/123
  parts := strings.Split(path, "/")
  userID := parts[len(parts)-1]
  ```
- **Body درخواست**:
  ```go
  body, err := io.ReadAll(r.Body)
  if err != nil {
      http.Error(w, "Bad request", http.StatusBadRequest)
      return
  }
  fmt.Fprintf(w, "Body: %s", body)
  ```

### 2.6. خطاهای رایج
- عدم بررسی Method درخواست.
- نادیده گرفتن خطاهای `ParseForm` یا `ReadAll`.

---

## 3. کار با JSON در HTTP

### 3.1. مفهوم
JSON فرمت رایج برای تبادل داده در برنامه‌های وب است. Go با پکیج `encoding/json` از آن پشتیبانی می‌کند.

### 3.2. ارسال داده‌های JSON
```go
type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

func handler(w http.ResponseWriter, r *http.Request) {
    user := User{ID: 1, Name: "Ali"}
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}
```
- خروجی: `{"id":1,"name":"Ali"}`

### 3.3. دریافت داده‌های JSON
```go
func handler(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        return
    }
    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }
    fmt.Fprintf(w, "Received: %+v", user)
}
```

### 3.4. Middleware
Middleware توابعی هستند که قبل یا بعد از هندلر اجرا می‌شوند (مثل لاگ‌گیری، احراز هویت).
```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        fmt.Printf("%s %s %v\n", r.Method, r.URL.Path, time.Since(start))
    })
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello!")
    })
    http.ListenAndServe(":8080", loggingMiddleware(mux))
}
```

### 3.5. نکات پیشرفته
- **Streaming JSON**:
  ```go
  enc := json.NewEncoder(w)
  for _, user := range users {
      enc.Encode(user)
  }
  ```
- **مدیریت خطا**:
  ```go
  type ErrorResponse struct {
      Error string `json:"error"`
  }

  func handler(w http.ResponseWriter, r *http.Request) {
      w.Header().Set("Content-Type", "application/json")
      w.WriteHeader(http.StatusBadRequest)
      json.NewEncoder(w).Encode(ErrorResponse{Error: "Invalid input"})
  }
  ```

### 3.6. خطاهای رایج
- عدم تنظیم `Content-Type`.
- نادیده گرفتن خطاهای `Decode` یا `Encode`.

---

## 4. فریم‌ورک‌های وب

### 4.1. مفهوم
فریم‌ورک‌های وب ابزارهایی هستند که توسعه برنامه‌های وب را ساده‌تر می‌کنند. سه فریم‌ورک محبوب در Go عبارتند از **Gin**، **Echo**، و **Fiber**.

### 4.2. Gin
#### نصب
```bash
go get -u github.com/gin-gonic/gin
```
#### مثال
```go
package main

import "github.com/gin-gonic/gin"

func main() {
    r := gin.Default()
    r.GET("/ping", func(c *gin.Context) {
        c.JSON(200, gin.H{
            "message": "pong",
        })
    })
    r.Run(":8080")
}
```
- **ویژگی‌ها**: سریع، middleware داخلی، و پشتیبانی از JSON.

### 4.3. Echo
#### نصب
```bash
go get -u github.com/labstack/echo/v4
```
#### مثال
```go
package main

import (
    "net/http"
    "github.com/labstack/echo/v4"
)

func main() {
    e := echo.New()
    e.GET("/ping", func(c echo.Context) error {
        return c.JSON(http.StatusOK, map[string]string{"message": "pong"})
    })
    e.Start(":8080")
}
```
- **ویژگی‌ها**: ساده، انعطاف‌پذیر، و مناسب برای API‌ها.

### 4.4. Fiber
#### نصب
```bash
go get -u github.com/gofiber/fiber/v2
```
#### مثال
```go
package main

import "github.com/gofiber/fiber/v2"

func main() {
    app := fiber.New()
    app.Get("/ping", func(c *fiber.Ctx) error {
        return c.JSON(fiber.Map{"message": "pong"})
    })
    app.Listen(":8080")
}
```
- **ویژگی‌ها**: الهام گرفته از Express، بسیار سریع.

### 4.5. نکات پیشرفته
- **مقایسه**:
  - Gin: تعادل بین سرعت و ویژگی‌ها.
  - Echo: مناسب برای پروژه‌های ساده.
  - Fiber: سریع‌ترین، اما ممکن است ناپایدار باشد.
- **Middleware**: همه فریم‌ورک‌ها از middleware پشتیبانی می‌کنند (مثل لاگ‌گیری، احراز هویت).
- **انتخاب**: برای پروژه‌های کوچک، `net/http` کافی است؛ برای پروژه‌های بزرگ، Gin یا Echo توصیه می‌شود.

### 4.6. خطاهای رایج
- استفاده از فریم‌ورک بدون نیاز (افزایش پیچیدگی).
- عدم مدیریت خطاها در middleware.

---

## 5. REST API

### 5.1. مفهوم
REST (Representational State Transfer) یک معماری برای طراحی API‌های وب است که از متدهای HTTP (GET، POST، PUT، DELETE) استفاده می‌کند.

### 5.2. طراحی REST API
- **منابع**: هر منبع (مثل کاربر) با یک URL مشخص می‌شود (مثل `/users`).
- **متدها**:
  - GET: دریافت داده.
  - POST: ایجاد داده.
  - PUT/PATCH: به‌روزرسانی.
  - DELETE: حذف.
- **ساختار URL**:
  - `/users`: لیست کاربران.
  - `/users/:id`: کاربر خاص.

### 5.3. پیاده‌سازی
```go
package main

import (
    "encoding/json"
    "net/http"
    "strconv"
)

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

var users = map[int]User{
    1: {ID: 1, Name: "Ali"},
}

func main() {
    mux := http.NewServeMux()

    // دریافت همه کاربران
    mux.HandleFunc("GET /users", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(users)
    })

    // دریافت کاربر خاص
    mux.HandleFunc("GET /users/", func(w http.ResponseWriter, r *http.Request) {
        idStr := r.URL.Path[len("/users/"):]
        id, _ := strconv.Atoi(idStr)
        user, ok := users[id]
        if !ok {
            http.Error(w, "User not found", http.StatusNotFound)
            return
        }
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(user)
    })

    // ایجاد کاربر
    mux.HandleFunc("POST /users", func(w http.ResponseWriter, r *http.Request) {
        var user User
        if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
            http.Error(w, "Invalid input", http.StatusBadRequest)
            return
        }
        user.ID = len(users) + 1
        users[user.ID] = user
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusCreated)
        json.NewEncoder(w).Encode(user)
    })

    http.ListenAndServe(":8080", mux)
}
```

### 5.4. بهترین شیوه‌ها
- **کدهای وضعیت مناسب**:
  - 200: موفقیت.
  - 201: ایجاد موفقیت‌آمیز.
  - 400: ورودی نامعتبر.
  - 404: منبع یافت نشد.
- **نسخه‌بندی**: از `/v1/users` برای نسخه‌بندی API استفاده کنید.
- **HATEOAS**: لینک‌های مرتبط را در پاسخ‌ها قرار دهید:
  ```json
  {
      "id": 1,
      "name": "Ali",
      "links": {
          "self": "/users/1"
      }
  }
  ```
- **مدیریت خطا**:
  ```json
  {
      "error": "Invalid input",
      "details": "Name is required"
  }
  ```

### 5.5. نکات پیشرفته
- **احراز هویت**: از JWT یا OAuth2 استفاده کنید.
- **Rate Limiting**: برای جلوگیری از سوءاستفاده:
  ```go
  func rateLimit(next http.Handler) http.Handler {
      // پیاده‌سازی محدودیت
  }
  ```
- **Pagination**:
  ```go
  limit := r.URL.Query().Get("limit")
  offset := r.URL.Query().Get("offset")
  ```

### 5.6. خطاهای رایج
- عدم استفاده از کدهای وضعیت مناسب.
- نادیده گرفتن اعتبارسنجی ورودی.

---

## 6. WebSocket

### 6.1. مفهوم
WebSocket پروتکلی برای ارتباط دوطرفه و بلادرنگ بین سرور و کلاینت است. پکیج `github.com/gorilla/websocket` ابزار محبوبی در Go است.

### 6.2. نصب
```bash
go get -u github.com/gorilla/websocket
```

### 6.3. پیاده‌سازی
```go
package main

import (
    "log"
    "net/http"
    "github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
    CheckOrigin: func(r *http.Request) bool {
        return true // در تولید محدود کنید
    },
}

func handler(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Println(err)
        return
    }
    defer conn.Close()

    for {
        msgType, msg, err := conn.ReadMessage()
        if err != nil {
            log.Println(err)
            return
        }
        log.Printf("Received: %s", msg)
        err = conn.WriteMessage(msgType, msg)
        if err != nil {
            log.Println(err)
            return
        }
    }
}

func main() {
    http.HandleFunc("/ws", handler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### 6.4. نکات پیشرفته
- **مدیریت اتصال‌ها**: از نقشه‌ای برای ذخیره اتصال‌های فعال استفاده کنید:
  ```go
  var clients = make(map[*websocket.Conn]bool)
  var mutex = &sync.Mutex{}
  ```
- **پخش پیام**:
  ```go
  func broadcast(msg []byte) {
      mutex.Lock()
      defer mutex.Unlock()
      for client := range clients {
          client.WriteMessage(websocket.TextMessage, msg)
      }
  }
  ```
- **Heartbeat**: برای بررسی زنده بودن کلاینت‌ها از ping/pong استفاده کنید.

### 6.5. خطاهای رایج
- عدم بستن اتصال‌ها.
- محدود نکردن `CheckOrigin` در تولید.

---

## 7. کلاینت HTTP

### 7.1. مفهوم
کلاینت HTTP برای ارسال درخواست‌ها به سرورهای دیگر استفاده می‌شود.

### 7.2. ارسال درخواست‌ها
#### GET
```go
resp, err := http.Get("https://api.example.com/data")
if err != nil {
    log.Fatal(err)
}
defer resp.Body.Close()
body, err := io.ReadAll(resp.Body)
fmt.Println(string(body))
```

#### POST
```go
data := []byte(`{"name":"Ali"}`)
resp, err := http.Post("https://api.example.com/users", "application/json", bytes.NewBuffer(data))
if err != nil {
    log.Fatal(err)
}
defer resp.Body.Close()
```

### 7.3. کلاینت سفارشی
```go
client := &http.Client{
    Timeout: 10 * time.Second,
}
req, err := http.NewRequest("GET", "https://api.example.com/data", nil)
if err != nil {
    log.Fatal(err)
}
req.Header.Set("Authorization", "Bearer token")
resp, err := client.Do(req)
```

### 7.4. نکات پیشرفته
- **مدیریت خطا**:
  ```go
  if resp.StatusCode != http.StatusOK {
      log.Printf("Unexpected status: %s", resp.Status)
  }
  ```
- **Retry**:
  ```go
  for i := 0; i < 3; i++ {
      resp, err := client.Do(req)
      if err == nil && resp.StatusCode == http.StatusOK {
          return resp, nil
      }
      time.Sleep(time.Second * time.Duration(i+1))
  }
  ```

### 7.5. خطاهای رایج
- عدم بستن `resp.Body`.
- نادیده گرفتن خطاهای شبکه.

---

## 8. امنیت در HTTP

### 8.1. CORS
برای اجازه دسترسی از دامنه‌های دیگر:
```go
func corsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        if r.Method == http.MethodOptions {
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

### 8.2. CSRF
برای جلوگیری از حملات CSRF:
- از توکن‌های CSRF استفاده کنید:
  ```go
  token := generateCSRFToken()
  http.SetCookie(w, &http.Cookie{Name: "csrf_token", Value: token})
  ```
- بررسی توکن در درخواست‌های POST:
  ```go
  if r.FormValue("csrf_token") != expectedToken {
      http.Error(w, "Invalid CSRF token", http.StatusForbidden)
  }
  ```

### 8.3. TLS
برای فعال‌سازی HTTPS:
```go
http.ListenAndServeTLS(":443", "cert.pem", "key.pem", nil)
```
- **ایجاد گواهی**:
  ```bash
  openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
  ```

### 8.4. نکات پیشرفته
- **HSTS**: برای اجبار HTTPS:
  ```go
  w.Header().Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
  ```
- **Content Security Policy (CSP)**:
  ```go
  w.Header().Set("Content-Security-Policy", "default-src 'self'")
  ```
- **OWASP**: از استانداردهای OWASP برای امنیت API استفاده کنید.

### 8.5. خطاهای رایج
- اجازه CORS بیش از حد باز.
- عدم استفاده از TLS در تولید.

---

## 9. بهترین شیوه‌ها و نکات

- **ساختار پروژه**:
  ```
  /myapp
  ├── /handlers
  ├── /models
  ├── /middleware
  ├── main.go
  ```
- **مدیریت خطا**: همیشه خطاها را به صورت استاندارد برگردانید.
- **پروفایلینگ**: از `pprof` برای بررسی عملکرد سرور استفاده کنید.
- **تست**: تست‌های واحد و یکپارچگی برای API‌ها بنویسید:
  ```go
  func TestHandler(t *testing.T) {
      req := httptest.NewRequest(http.MethodGet, "/", nil)
      w := httptest.NewRecorder()
      handler(w, req)
      if w.Code != http.StatusOK {
          t.Errorf("Expected status 200, got %d", w.Code)
      }
  }
  ```
- **مستندسازی**: از ابزارهایی مثل Swagger برای مستندسازی API استفاده کنید.

---

## 10. نتیجه‌گیری

این جزوه تمام جنبه‌های برنامه‌نویسی شبکه و HTTP در Go را با جزئیات کامل پوشش داد. از ساخت سرورهای ساده تا پیاده‌سازی REST API، WebSocket، و امنیت، هر بخش با مثال‌های عملی و نکات پیشرفته ارائه شد. برای یادگیری عمیق‌تر:
- مستندات رسمی: `https://golang.org/doc/`
- کتاب "Building RESTful Web Services with Go"
- تمرین با پروژه‌های واقعی مثل ساخت API یا برنامه‌های بلادرنگ