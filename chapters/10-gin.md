# آموزش جامع، پیشرفته و کاربردی فریم‌ورک Gin در Go

این جزوه به بررسی عمیق و تخصصی **Gin**، یکی از محبوب‌ترین فریم‌ورک‌های وب برای Go، می‌پردازد. Gin به دلیل سرعت بالا، سادگی، و انعطاف‌پذیری برای ساخت APIهای مقیاس‌پذیر بسیار مناسب است. تمام موضوعات درخواستی با جزئیات کامل، مثال‌های عملی، و نکات پیشرفته شرح داده شده‌اند.

---

## 1. معماری پیشرفته برای اپلیکیشن‌های Gin

### 1.1. پیاده‌سازی Clean Architecture
Clean Architecture کد را به لایه‌های مستقل تقسیم می‌کند تا تست‌پذیری، نگهداری، و مقیاس‌پذیری بهبود یابد:
- **Entities**: مدل‌های کسب‌وکار.
- **Use Cases (Services)**: منطق کسب‌وکار.
- **Interface Adapters (Handlers)**: تعامل با دنیای خارج (مثل HTTP).
- **Frameworks/Drivers (Repositories)**: دسترسی به دیتابیس یا سیستم‌های خارجی.

#### ساختار پروژه
```bash
/api
├── /entities
│   └── user.go
├── /services
│   └── user.go
├── /handlers
│   └── user.go
├── /repositories
│   └── user.go
├── /middleware
├── /config
├── main.go
```

### 1.2. جداسازی لایه‌ها
#### Entities
```go
package entities

type User struct {
    ID    uint
    Name  string
    Email string
}
```

#### Repository
```go
package repositories

import (
    "gorm.io/gorm"
    "github.com/username/api/entities"
)

type UserRepository interface {
    Create(user *entities.User) error
    FindByID(id uint) (*entities.User, error)
}

type GORMUserRepository struct {
    db *gorm.DB
}

func NewGORMUserRepository(db *gorm.DB) *GORMUserRepository {
    return &GORMUserRepository{db: db}
}

func (r *GORMUserRepository) Create(user *entities.User) error {
    return r.db.Create(user).Error
}

func (r *GORMUserRepository) FindByID(id uint) (*entities.User, error) {
    var user entities.User
    err := r.db.First(&user, id).Error
    return &user, err
}
```

#### Service
```go
package services

import (
    "github.com/username/api/entities"
    "github.com/username/api/repositories"
)

type UserService interface {
    CreateUser(name, email string) (*entities.User, error)
    GetUser(id uint) (*entities.User, error)
}

type userService struct {
    repo repositories.UserRepository
}

func NewUserService(repo repositories.UserRepository) *userService {
    return &userService{repo: repo}
}

func (s *userService) CreateUser(name, email string) (*entities.User, error) {
    user := &entities.User{Name: name, Email: email}
    if err := s.repo.Create(user); err != nil {
        return nil, err
    }
    return user, nil
}

func (s *userService) GetUser(id uint) (*entities.User, error) {
    return s.repo.FindByID(id)
}
```

#### Handler
```go
package handlers

import (
    "net/http"
    "github.com/gin-gonic/gin"
    "github.com/username/api/services"
)

type UserHandler struct {
    service services.UserService
}

func NewUserHandler(service services.UserService) *UserHandler {
    return &UserHandler{service: service}
}

func (h *UserHandler) CreateUser(c *gin.Context) {
    var input struct {
        Name  string `json:"name" binding:"required"`
        Email string `json:"email" binding:"required,email"`
    }
    if err := c.ShouldBindJSON(&input); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    user, err := h.service.CreateUser(input.Name, input.Email)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusCreated, user)
}
```

### 1.3. استفاده از Interfaceها برای Decoupling
Interfaceها وابستگی‌ها را کاهش می‌دهند:
```go
type UserRepository interface {
    Create(user *entities.User) error
    FindByID(id uint) (*entities.User, error)
}

type UserService interface {
    CreateUser(name, email string) (*entities.User, error)
    GetUser(id uint) (*entities.User, error)
}
```

### 1.4. Dependency Injection حرفه‌ای
```go
package main

import (
    "github.com/gin-gonic/gin"
    "github.com/username/api/handlers"
    "github.com/username/api/repositories"
    "github.com/username/api/services"
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

func main() {
    db, _ := gorm.Open(postgres.Open("host=localhost user=postgres password=secret dbname=api"), &gorm.Config{})
    userRepo := repositories.NewGORMUserRepository(db)
    userService := services.NewUserService(userRepo)
    userHandler := handlers.NewUserHandler(userService)

    r := gin.Default()
    r.POST("/users", userHandler.CreateUser)
    r.Run(":8080")
}
```

### 1.5. نکات پیشرفته
- **Module Injection**: از ابزارهایی مثل `wire` برای DI خودکار استفاده کنید:
  ```bash
  go get github.com/google/wire
  ```
- **Layer Validation**: اعتبارسنجی را در لایه سرویس انجام دهید:
  ```go
  func (s *userService) CreateUser(name, email string) (*entities.User, error) {
      if !strings.Contains(email, "@") {
          return nil, errors.New("invalid email")
      }
      // ادامه
  }
  ```

### 1.6. خطاهای رایج
- ترکیب لایه‌ها (مثل دسترسی مستقیم به دیتابیس در Handler).
- عدم استفاده از Interface برای تست‌پذیری.

---

## 2. روتینگ و Middlewareهای حرفه‌ای

### 2.1. Groupهای تو در تو برای Versioning
```go
r := gin.Default()

// API نسخه 1
v1 := r.Group("/api/v1")
{
    v1.GET("/users", userHandler.GetUsers)
    v1.POST("/users", userHandler.CreateUser)
}

// API نسخه 2
v2 := r.Group("/api/v2")
{
    v2.GET("/users", userHandler.GetUsersV2)
}
```

### 2.2. Middlewareهای سفارشی
```go
func LoggingMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        c.Next()
        duration := time.Since(start)
        log.Printf("Request: %s %s, Duration: %v, Status: %d", c.Request.Method, c.Request.URL.Path, duration, c.Writer.Status())
    }
}

func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "missing token"})
            return
        }
        c.Next()
    }
}

r.Use(LoggingMiddleware())
r.GET("/protected", AuthMiddleware(), func(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{"message": "protected endpoint"})
})
```

### 2.3. مدیریت Context
- **Set/Get**:
  ```go
  c.Set("userID", 123)
  userID, _ := c.Get("userID")
  ```
- **Copy**:
  ```go
  cCopy := c.Copy()
  go func() {
      // استفاده از cCopy در goroutine
  }()
  ```
- **Abort/Next**:
  ```go
  func RestrictToAdmins() gin.HandlerFunc {
      return func(c *gin.Context) {
          role, _ := c.Get("role")
          if role != "admin" {
              c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "admin only"})
              return
          }
          c.Next()
      }
  }
  ```

### 2.4. نکات پیشرفته
- **Context Timeout**:
  ```go
  func TimeoutMiddleware() gin.HandlerFunc {
      return func(c *gin.Context) {
          ctx, cancel := context.WithTimeout(c.Request.Context(), 5*time.Second)
          defer cancel()
          c.Request = c.Request.WithContext(ctx)
          c.Next()
      }
  }
  ```
- **Rate Limiting**:
  ```go
  import "github.com/gin-contrib/ratelimit"
  r.Use(ratelimit.RequestsIn(100, time.Minute, 10)) // 100 درخواست در 10 دقیقه
  ```

### 2.5. خطاهای رایج
- استفاده نادرست از `c.Abort()` که باعث ادامه اجرای Handlerها می‌شود.
- ذخیره داده‌های بزرگ در Context که باعث مصرف حافظه می‌شود.

---

## 3. مدیریت پیشرفته خطا

### 3.1. استانداردسازی پاسخ‌های خطا
```go
type APIError struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
    Details string `json:"details,omitempty"`
}

func NewAPIError(code int, message, details string) *APIError {
    return &APIError{Code: code, Message: message, Details: details}
}
```

### 3.2. Custom Error Structs
```go
type ValidationError struct {
    Field string
    Error string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on %s: %s", e.Field, e.Error)
}

func validateUser(user entities.User) error {
    if user.Name == "" {
        return &ValidationError{Field: "name", Error: "required"}
    }
    return nil
}
```

### 3.3. استفاده از errors.Join, errors.Is, errors.As
```go
import "errors"

func processUser(c *gin.Context, user entities.User) error {
    var errs []error
    if user.Name == "" {
        errs = append(errs, &ValidationError{Field: "name", Error: "required"})
    }
    if user.Email == "" {
        errs = append(errs, &ValidationError{Field: "email", Error: "required"})
    }
    if len(errs) > 0 {
        return errors.Join(errs...)
    }
    return nil
}

func handleUser(c *gin.Context) {
    err := processUser(c, entities.User{})
    if err != nil {
        var valErr *ValidationError
        if errors.As(err, &valErr) {
            c.JSON(http.StatusBadRequest, NewAPIError(1001, valErr.Error(), ""))
            return
        }
        c.JSON(http.StatusInternalServerError, NewAPIError(1000, "internal error", err.Error()))
    }
}
```

### 3.4. Middleware برای Error Handler مرکزی
```go
func ErrorHandler() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()
        if len(c.Errors) > 0 {
            for _, e := range c.Errors {
                switch e.Type {
                case gin.ErrorTypeBind:
                    c.JSON(http.StatusBadRequest, NewAPIError(1001, "validation failed", e.Error()))
                default:
                    c.JSON(http.StatusInternalServerError, NewAPIError(1000, "internal error", e.Error()))
                }
            }
        }
    }
}

r := gin.Default()
r.Use(ErrorHandler())
```

### 3.5. نکات پیشرفته
- **Stack Trace**:
  ```go
  import "github.com/pkg/errors"
  return errors.WithStack(fmt.Errorf("failed to process user: %w", err))
  ```
- **Error Logging**:
  ```go
  log.Printf("Error: %+v", errors.WithStack(err))
  ```

### 3.6. خطاهای رایج
- عدم استانداردسازی پاسخ‌های خطا.
- نادیده گرفتن خطاهای چندگانه با `errors.Join`.

---

## 4. اعتبارسنجی و بایندینگ داده‌ها

### 4.1. استفاده از binding و validator.v10
```go
import "github.com/go-playground/validator/v10"

type CreateUserInput struct {
    Name  string `json:"name" binding:"required,min=3"`
    Email string `json:"email" binding:"required,email"`
    Age   int    `json:"age" binding:"gte=18"`
}

func (h *UserHandler) CreateUser(c *gin.Context) {
    var input CreateUserInput
    if err := c.ShouldBindJSON(&input); err != nil {
        c.JSON(http.StatusBadRequest, NewAPIError(1001, "validation failed", err.Error()))
        return
    }
    // ادامه
}
```

### 4.2. Ruleهای سفارشی
```go
import "github.com/go-playground/validator/v10"

func RegisterCustomValidations(v *validator.Validate) {
    v.RegisterValidation("strongpassword", func(fl validator.FieldLevel) bool {
        password := fl.Field().String()
        return len(password) >= 8 && strings.ContainsAny(password, "!@#$%")
    })
}

type UserInput struct {
    Password string `json:"password" binding:"required,strongpassword"`
}

func main() {
    v := validator.New()
    RegisterCustomValidations(v)
    r := gin.Default()
    r.POST("/users", func(c *gin.Context) {
        var input UserInput
        if err := c.ShouldBindJSON(&input); err != nil {
            c.JSON(http.StatusBadRequest, NewAPIError(1001, "validation failed", err.Error()))
            return
        }
        c.JSON(http.StatusOK, gin.H{"message": "valid"})
    })
}
```

### 4.3. کار با فرم، JSON، XML و Multipart
```go
r.POST("/upload", func(c *gin.Context) {
    form, _ := c.MultipartForm()
    files := form.File["files"]
    for _, file := range files {
        c.SaveUploadedFile(file, "./uploads/"+file.Filename)
    }
    c.JSON(http.StatusOK, gin.H{"message": "uploaded"})
})

r.POST("/xml", func(c *gin.Context) {
    var input struct {
        Name string `xml:"name" binding:"required"`
    }
    if err := c.ShouldBindXML(&input); err != nil {
        c.JSON(http.StatusBadRequest, NewAPIError(1001, "validation failed", err.Error()))
        return
    }
    c.JSON(http.StatusOK, input)
})
```

### 4.4. Struct-level Validation
```go
type RegisterInput struct {
    Password        string `json:"password" binding:"required"`
    ConfirmPassword string `json:"confirm_password" binding:"required"`
}

func (r RegisterInput) Validate(sl validator.StructLevel) {
    if r.Password != r.ConfirmPassword {
        sl.ReportError(r.ConfirmPassword, "confirm_password", "ConfirmPassword", "eqfield", "password")
    }
}

func main() {
    v := validator.New()
    v.RegisterStructValidation(RegisterInput{}.Validate, RegisterInput{})
}
```

### 4.5. نکات پیشرفته
- **Custom Error Messages**:
  ```go
  import "github.com/go-playground/validator/v10"

  func TranslateErrors(err error) map[string]string {
      errors := make(map[string]string)
      for _, e := range err.(validator.ValidationErrors) {
          errors[e.Field()] = fmt.Sprintf("failed on %s", e.Tag())
      }
      return errors
  }
  ```
- **Dynamic Binding**:
  ```go
  var input interface{}
  if c.ContentType() == "application/xml" {
      input = &XMLInput{}
  } else {
      input = &JSONInput{}
  }
  c.ShouldBind(input)
  ```

### 4.6. خطاهای رایج
- عدم ترجمه خطاهای اعتبارسنجی برای کاربر.
- استفاده نادرست از `ShouldBind` که باعث نادیده گرفتن خطاها می‌شود.

---

## 5. احراز هویت و مجوزها

### 5.1. استفاده از JWT
```go
import "github.com/golang-jwt/jwt/v4"

type Claims struct {
    UserID uint   `json:"user_id"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

func GenerateJWT(userID uint, role string) (string, error) {
    claims := Claims{
        UserID: userID,
        Role:   role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
        },
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte("secret"))
}

func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        tokenStr := c.GetHeader("Authorization")
        tokenStr = strings.TrimPrefix(tokenStr, "Bearer ")
        claims := &Claims{}
        token, err := jwt.ParseWithClaims(tokenStr, claims, func(token *jwt.Token) (interface{}, error) {
            return []byte("secret"), nil
        })
        if err != nil || !token.Valid {
            c.AbortWithStatusJSON(http.StatusUnauthorized, NewAPIError(1002, "invalid token", err.Error()))
            return
        }
        c.Set("userID", claims.UserID)
        c.Set("role", claims.Role)
        c.Next()
    }
}
```

### 5.2. Middleware برای بررسی سطح دسترسی
```go
func RoleMiddleware(requiredRole string) gin.HandlerFunc {
    return func(c *gin.Context) {
        role, exists := c.Get("role")
        if !exists || role != requiredRole {
            c.AbortWithStatusJSON(http.StatusForbidden, NewAPIError(1003, "insufficient permissions", ""))
            return
        }
        c.Next()
    }
}

r.GET("/admin", AuthMiddleware(), RoleMiddleware("admin"), func(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{"message": "admin access"})
})
```

### 5.3. پیاده‌سازی RBAC و ACL
```go
type ACL struct {
    RolePermissions map[string][]string
}

func NewACL() *ACL {
    return &ACL{
        RolePermissions: map[string][]string{
            "admin": {"create_user", "delete_user"},
            "user":  {"read_user"},
        },
    }
}

func (a *ACL) Authorize(role, permission string) bool {
    perms, ok := a.RolePermissions[role]
    if !ok {
        return false
    }
    for _, p := range perms {
        if p == permission {
            return true
        }
    }
    return false
}

func PermissionMiddleware(acl *ACL, permission string) gin.HandlerFunc {
    return func(c *gin.Context) {
        role, _ := c.Get("role")
        if !acl.Authorize(role.(string), permission) {
            c.AbortWithStatusJSON(http.StatusForbidden, NewAPIError(1003, "permission denied", ""))
            return
        }
        c.Next()
    }
}
```

### 5.4. Refresh Tokens و Secure Cookies
```go
func GenerateRefreshToken(userID uint) (string, error) {
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
        "user_id": userID,
        "exp":     time.Now().Add(30 * 24 * time.Hour).Unix(),
    })
    return token.SignedString([]byte("refresh_secret"))
}

r.POST("/refresh", func(c *gin.Context) {
    refreshToken, err := c.Cookie("refresh_token")
    if err != nil {
        c.JSON(http.StatusUnauthorized, NewAPIError(1002, "missing refresh token", ""))
        return
    }
    claims := jwt.MapClaims{}
    token, err := jwt.ParseWithClaims(refreshToken, &claims, func(token *jwt.Token) (interface{}, error) {
        return []byte("refresh_secret"), nil
    })
    if err != nil || !token.Valid {
        c.JSON(http.StatusUnauthorized, NewAPIError(1002, "invalid refresh token", ""))
        return
    }
    userID := uint(claims["user_id"].(float64))
    newToken, _ := GenerateJWT(userID, "user")
    c.JSON(http.StatusOK, gin.H{"token": newToken})
})
```

### 5.5. نکات پیشرفته
- **Secure Cookies**:
  ```go
  c.SetCookie("refresh_token", refreshToken, 30*24*3600, "/", "localhost", true, true)
  ```
- **OAuth2**:
  ```go
  import "golang.org/x/oauth2"
  config := &oauth2.Config{
      ClientID:     "client_id",
      ClientSecret: "client_secret",
      RedirectURL:  "http://localhost:8080/callback",
      Scopes:       []string{"email"},
      Endpoint:     google.Endpoint,
  }
  ```

### 5.6. خطاهای رایج
- ذخیره توکن‌های حساس در مکان‌های ناامن.
- عدم بررسی تاریخ انقضای توکن‌ها.

---

## 6. Performance و Scalability

### 6.1. Sync Pool و Zero-Allocation
```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func (h *UserHandler) GetUsers(c *gin.Context) {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer bufferPool.Put(buf)
    buf.Reset()
    // استفاده از buf برای رندر JSON
}
```

### 6.2. بهینه‌سازی Memory Allocation در JSON Encoding
```go
import "github.com/goccy/go-json"

r := gin.Default()
r.JSONEncoder = &json.Encoder{} // استفاده از go-json به جای encoding/json
```

### 6.3. Benchmark و Profiling
- **Benchmark**:
  ```go
  func BenchmarkCreateUser(b *testing.B) {
      r := gin.Default()
      h := NewUserHandler(mockService{})
      r.POST("/users", h.CreateUser)
      req, _ := http.NewRequest("POST", "/users", strings.NewReader(`{"name":"Ali","email":"ali@example.com"}`))
      w := httptest.NewRecorder()
      b.ResetTimer()
      for i := 0; i < b.N; i++ {
          r.ServeHTTP(w, req)
      }
  }
  ```
  ```bash
  go test -bench=. -benchmem
  ```

- **Profiling**:
  ```go
  r.GET("/debug/pprof/", gin.WrapH(http.HandlerFunc(pprof.Index)))
  ```
  ```bash
  go tool pprof http://localhost:8080/debug/pprof/profile
  ```

- **Trace**:
  ```bash
  go tool trace trace.out
  ```

### 6.4. Load Balancing
- **Nginx**:
  ```nginx
  upstream api {
      server localhost:8080;
      server localhost:8081;
  }
  server {
      listen 80;
      location / {
          proxy_pass http://api;
      }
  }
  ```

### 6.5. نکات پیشرفته
- **Graceful Shutdown**:
  ```go
  server := &http.Server{Addr: ":8080", Handler: r}
  go server.ListenAndServe()
  quit := make(chan os.Signal, 1)
  signal.Notify(quit, os.Interrupt)
  <-quit
  ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
  defer cancel()
  server.Shutdown(ctx)
  ```
- **Connection Pooling**:
  ```go
  sqlDB, _ := db.DB()
  sqlDB.SetMaxOpenConns(100)
  ```

### 6.6. خطاهای رایج
- تخصیص غیرضروری حافظه در حلقه‌ها.
- عدم استفاده از `sync.Pool` برای اشیاء موقت.

---

## 7. کار با WebSocket و SSE

### 7.1. WebSocket
```go
import "github.com/gorilla/websocket"

var upgrader = websocket.Upgrader{
    CheckOrigin: func(r *http.Request) bool { return true },
}

type Hub struct {
    clients    map[*websocket.Conn]string
    broadcast  chan Message
    register   chan *websocket.Conn
    unregister chan *websocket.Conn
    mu         sync.RWMutex
}

type Message struct {
    UserID  uint   `json:"user_id"`
    Content string `json:"content"`
}

func NewHub() *Hub {
    return &Hub{
        clients:    make(map[*websocket.Conn]string),
        broadcast:  make(chan Message),
        register:   make(chan *websocket.Conn),
        unregister: make(chan *websocket.Conn),
    }
}

func (h *Hub) Run() {
    for {
        select {
        case conn := <-h.register:
            h.mu.Lock()
            h.clients[conn] = ""
            h.mu.Unlock()
        case conn := <-h.unregister:
            h.mu.Lock()
            delete(h.clients, conn)
            h.mu.Unlock()
        case msg := <-h.broadcast:
            h.mu.RLock()
            for conn := range h.clients {
                conn.WriteJSON(msg)
            }
            h.mu.RUnlock()
        }
    }
}

func WebSocketHandler(hub *Hub) gin.HandlerFunc {
    return func(c *gin.Context) {
        conn, err := upgrader.Upgrade(c.Writer, c.Request, nil)
        if err != nil {
            c.JSON(http.StatusInternalServerError, NewAPIError(1000, "websocket upgrade failed", err.Error()))
            return
        }
        hub.register <- conn
        defer func() { hub.unregister <- conn }()
        for {
            var msg Message
            err := conn.ReadJSON(&msg)
            if err != nil {
                return
            }
            hub.broadcast <- msg
        }
    }
}

func main() {
    hub := NewHub()
    go hub.Run()
    r := gin.Default()
    r.GET("/ws", WebSocketHandler(hub))
}
```

### 7.2. SSE (Server-Sent Events)
```go
func SSEHandler() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Writer.Header().Set("Content-Type", "text/event-stream")
        c.Writer.Header().Set("Cache-Control", "no-cache")
        c.Writer.Header().Set("Connection", "keep-alive")

        ticker := time.NewTicker(2 * time.Second)
        defer ticker.Stop()

        c.Stream(func(w io.Writer) bool {
            select {
            case <-ticker.C:
                msg := fmt.Sprintf("data: %s\n\n", time.Now().String())
                fmt.Fprint(w, msg)
                return true
            case <-c.Request.Context().Done():
                return false
            }
        })
    }
}

r.GET("/events", SSEHandler())
```

### 7.3. نکات پیشرفته
- **Heartbeat**:
  ```go
  go func() {
      for range time.Tick(30 * time.Second) {
          conn.WriteMessage(websocket.PingMessage, []byte{})
      }
  }()
  ```
- **Concurrent-Safe**:
  ```go
  h.mu.Lock()
  defer h.mu.Unlock()
  ```

### 7.4. خطاهای رایج
- عدم بستن اتصال‌های WebSocket.
- مدیریت نادرست همزمانی در Hub.

---

## 8. تست‌نویسی پیشرفته

### 8.1. Unit Test با httptest
```go
func TestCreateUser(t *testing.T) {
    r := gin.Default()
    service := &mockUserService{}
    handler := NewUserHandler(service)
    r.POST("/users", handler.CreateUser)

    req, _ := http.NewRequest("POST", "/users", strings.NewReader(`{"name":"Ali","email":"ali@example.com"}`))
    req.Header.Set("Content-Type", "application/json")
    w := httptest.NewRecorder()

    r.ServeHTTP(w, req)

    if w.Code != http.StatusCreated {
        t.Errorf("Expected status 201, got %d", w.Code)
    }
}
```

### 8.2. Mock کردن Context و Middleware
```go
type mockUserService struct{}

func (s *mockUserService) CreateUser(name, email string) (*entities.User, error) {
    return &entities.User{Name: name, Email: email}, nil
}

func TestAuthMiddleware(t *testing.T) {
    r := gin.Default()
    r.Use(AuthMiddleware())
    r.GET("/protected", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"message": "ok"})
    })

    req, _ := http.NewRequest("GET", "/protected", nil)
    req.Header.Set("Authorization", "Bearer valid_token")
    w := httptest.NewRecorder()

    r.ServeHTTP(w, req)

    if w.Code != http.StatusOK {
        t.Errorf("Expected status 200, got %d", w.Code)
    }
}
```

### 8.3. Integration Test با Docker
- **Docker Compose**:
  ```yaml
  version: '3'
  services:
    postgres:
      image: postgres:14
      environment:
        POSTGRES_USER: test
        POSTGRES_PASSWORD: test
        POSTGRES_DB: test
      ports:
        - "5432:5432"
  ```
- **تست**:
  ```go
  func TestIntegration(t *testing.T) {
      db, _ := gorm.Open(postgres.Open("host=localhost user=test password=test dbname=test"), &gorm.Config{})
      repo := repositories.NewGORMUserRepository(db)
      service := services.NewUserService(repo)
      handler := handlers.NewUserHandler(service)

      r := gin.Default()
      r.POST("/users", handler.CreateUser)

      req, _ := http.NewRequest("POST", "/users", strings.NewReader(`{"name":"Ali","email":"ali@example.com"}`))
      w := httptest.NewRecorder()

      r.ServeHTTP(w, req)

      if w.Code != http.StatusCreated {
          t.Errorf("Expected status 201, got %d", w.Code)
      }
  }
  ```

### 8.4. Coverage حرفه‌ای
```bash
go test -coverpkg=./... -coverprofile=coverage.out
go tool cover -html=coverage.out
```

### 8.5. نکات پیشرفته
- **Test Fixtures**:
  ```go
  func SetupTestRouter(t *testing.T) *gin.Engine {
      r := gin.Default()
      db, _ := gorm.Open(sqlite.Open("file::memory:"), &gorm.Config{})
      repo := repositories.NewGORMUserRepository(db)
      service := services.NewUserService(repo)
      handler := handlers.NewUserHandler(service)
      r.POST("/users", handler.CreateUser)
      return r
  }
  ```
- **Parallel Tests**:
  ```go
  t.Parallel()
  ```

### 8.6. خطاهای رایج
- عدم Mock کردن وابستگی‌ها.
- تست‌های غیرقابل تکرار به دلیل دیتابیس مشترک.

---

## 9. مستندسازی و OpenAPI

### 9.1. تولید مستندات Swagger
```go
// @title User API
// @version 1.0
// @description API for managing users
// @host localhost:8080
// @BasePath /api/v1
package main

import (
    "github.com/gin-gonic/gin"
    "github.com/swaggo/gin-swagger"
    "github.com/swaggo/files"
    _ "github.com/username/api/docs"
)

// @Summary Create a new user
// @Description Creates a user with name and email
// @Tags Users
// @Accept json
// @Produce json
// @Param user body handlers.CreateUserInput true "User data"
// @Success 201 {object} entities.User
// @Failure 400 {object} handlers.APIError
// @Router /users [post]
func (h *UserHandler) CreateUser(c *gin.Context) {}

func main() {
    r := gin.Default()
    r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
    r.Run()
}
```

- تولید:
  ```bash
  go get github.com/swaggo/swag/cmd/swag
  swag init
  ```

### 9.2. تعریف دقیق مدل‌ها و Errorها
```go
// APIError represents an error response
// @Schema
type APIError struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
    Details string `json:"details,omitempty"`
}
```

### 9.3. تولید Client SDK
```bash
swag init -o docs
docker run --rm -v $(pwd):/local swaggerapi/swagger-codegen-cli generate \
    -i /local/docs/swagger.json \
    -l go \
    -o /local/client
```

### 9.4. همگام‌سازی اتوماتیک
```yaml
name: Update Swagger Docs
on:
  push:
    branches: [main]
jobs:
  docs:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Generate Swagger
      run: |
        go install github.com/swaggo/swag/cmd/swag@latest
        swag init
    - name: Commit changes
      run: |
        git config user.name "GitHub Action"
        git config user.email "action@github.com"
        git add docs/
        git commit -m "Update Swagger docs"
        git push
```

### 9.5. نکات پیشرفته
- **Custom Templates**:
  ```bash
  swag init --templateDir custom_templates
  ```
- **Multi-Version Docs**:
  ```go
  r.GET("/v1/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler, url.PathPrefix("/v1")))
  ```

### 9.6. خطاهای رایج
- عدم به‌روزرسانی مستندات پس از تغییر API.
- تعریف ناقص مدل‌ها در Swagger.

---

## 10. Deployment و مانیتورینگ

### 10.1. تنظیمات Production
```go
r := gin.New()
r.Use(gin.Recovery())
r.MaxMultipartMemory = 8 << 20 // 8 MiB
r.Use(gzip.Gzip(gzip.DefaultCompression))
r.Use(timeout.New(timeout.WithTimeout(5*time.Second)))
```

### 10.2. مانیتورینگ با Prometheus و Grafana
```go
import "github.com/gin-contrib/promeheus"

r.Use(promeheus.Metrics())
r.GET("/metrics", gin.WrapH(promhttp.Handler()))
```

### 10.3. Tracing با Jaeger
```go
import "github.com/opentracing-contrib/go-gin"

r.Use(ginopentracing.OpenTracing())
```

### 10.4. CI/CD
```yaml
name: Deploy API
on:
  push:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Set up Go
      uses: actions/setup-go@v5
      with:
        go-version: '1.20'
    - name: Build
      run: go build -o api
    - name: Test
      run: go test -v ./...
    - name: Deploy
      run: |
        scp api user@server:/app/
        ssh user@server "systemctl restart api"
```

### 10.5. نکات پیشرفته
- **Canary Deployment**:
  ```yaml
  deploy:
    canary:
      percentage: 10
      target: new_version
  ```
- **Health Checks**:
  ```go
  r.GET("/health", func(c *gin.Context) {
      c.JSON(http.StatusOK, gin.H{"status": "healthy"})
  })
  ```

### 10.6. خطاهای رایج
- عدم تنظیم Timeout در تولید.
- مانیتورینگ ناکافی که باعث شناسایی دیرهنگام مشکلات می‌شود.

---

## 11. انجمن، ابزارها و منابع یادگیری

### 11.1. مشارکت در پروژه‌های اپن‌سورس
- **Gin**:
  - مخزن: `https://github.com/gin-gonic/gin`
  - مشارکت: افزودن Middleware، بهبود مستندات.
- **gin-contrib**:
  - مخزن: `https://github.com/gin-contrib`
  - ابزارهای مفید: `gin-cors`, `gin-jwt`, `gin-prometheus`.

### 11.2. ابزارهای کمکی
- **gin-cors**:
  ```go
  import "github.com/gin-contrib/cors"
  r.Use(cors.Default())
  ```
- **gin-jwt**:
  ```go
  import "github.com/appleboy/gin-jwt/v2"
  jwtMiddleware, _ := jwt.New(&jwt.GinJWTMiddleware{
      Realm:       "test zone",
      Key:         []byte("secret"),
      Timeout:     time.Hour,
      MaxRefresh:  time.Hour * 24,
  })
  r.Use(jwtMiddleware.MiddlewareFunc())
  ```

### 11.3. مطالعه پروژه‌های بزرگ
- **Real-World Examples**:
  - `https://github.com/go-gin-example`
  - `https://github.com/restaurant-api`
- **مطالعه کد**:
  - بررسی Handlerها و Middlewareها در پروژه‌های بزرگ.

### 11.4. منابع یادگیری
- **مستندات رسمی**:
  - `https://gin-gonic.com/docs/`
- **کتاب‌ها**:
  - "Building RESTful Web Services with Go"
- **انجمن‌ها**:
  - Reddit: `r/golang`
  - Gophers Slack: کانال `#gin`
- **پروژه‌های عملی**:
  - ساخت API با Gin و Postgres.
  - پیاده‌سازی WebSocket Chat.

### 11.5. نکات پیشرفته
- **Blogging**: تجربیات خود را در مورد Gin منتشر کنید.
- **Hackathons**: در رویدادهای برنامه‌نویسی شرکت کنید.

### 11.6. خطاهای رایج
- عدم مطالعه مستندات ابزارهای gin-contrib.
- کپی‌برداری بدون درک از پروژه‌های نمونه.

---

## 12. بهترین شیوه‌ها و نکات کلی

- **سادگی**: کد را ساده و خوانا نگه دارید.
- **تست‌پذیری**: از Clean Architecture برای جداسازی لایه‌ها استفاده کنید.
- **بهینه‌سازی**:
  - از `sync.Pool` برای کاهش تخصیص حافظه استفاده کنید.
  - کوئری‌ها و پاسخ‌های JSON را بهینه کنید.
- **امنیت**:
  - توکن‌ها را در مکان‌های امن ذخیره کنید.
  - از HTTPS و CORS محدود استفاده کنید.
- **مستندسازی**: مستندات Swagger را همیشه به‌روز نگه دارید.
- **مانیتورینگ**: از Prometheus و Jaeger برای شناسایی مشکلات استفاده کنید.

---

## 13. نتیجه‌گیری

این جزوه تمام جنبه‌های فریم‌ورک Gin را به صورت جامع و پیشرفته پوشش داد. از معماری تمیز و روتینگ حرفه‌ای تا مدیریت خطا، اعتبارسنجی، احراز هویت، بهینه‌سازی، WebSocket، تست‌نویسی، مستندسازی، و استقرار، هر بخش با مثال‌های عملی و نکات تخصصی ارائه شد. برای یادگیری عمیق‌تر:
- مستندات رسمی: `https://gin-gonic.com/docs/`
- مطالعه کد Gin: `https://github.com/gin-gonic/gin`
- تمرین با پروژه‌های واقعی مثل ساخت API یا سیستم چت بلادرنگ.