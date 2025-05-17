# پروژه‌های عملی در زبان برنامه‌نویسی Go

این جزوه پنج پروژه عملی در Go را با جزئیات کامل شرح می‌دهد. هر پروژه شامل توضیحات مفهومی، کد کامل، نکات پیشرفته، بهترین شیوه‌ها، و خطاهای رایج است. پروژه‌ها به گونه‌ای طراحی شده‌اند که جنبه‌های مختلف Go (مثل همزمانی، شبکه، و ابزارهای استاندارد) را به نمایش بگذارند.

---

## 1. ساخت CLI: ایجاد ابزار خط فرمان با Cobra یا Flag

### 1.1. مفهوم
یک ابزار خط فرمان (CLI) برنامه‌ای است که از طریق ترمینال اجرا می‌شود و دستورات کاربر را پردازش می‌کند. پکیج‌های `flag` (استاندارد) و `cobra` (پیشرفته) برای ساخت CLI در Go استفاده می‌شوند. در این پروژه، از **Cobra** استفاده می‌کنیم زیرا قابلیت‌های بیشتری (مثل زیرفرمان‌ها و مستندات خودکار) ارائه می‌دهد.

### 1.2. هدف پروژه
ساخت یک CLI برای مدیریت وظایف (Todo List) با دستوراتی برای افزودن، لیست کردن، و حذف وظایف.

### 1.3. نصب Cobra
```bash
go install github.com/spf13/cobra-cli@latest
```

### 1.4. پیاده‌سازی
#### ساختار پروژه
```bash
mkdir todo-cli
cd todo-cli
go mod init github.com/username/todo-cli
cobra-cli init
cobra-cli add add
cobra-cli add list
cobra-cli add delete
```

#### کد اصلی
- **main.go**:
```go
package main

import "github.com/username/todo-cli/cmd"

func main() {
    cmd.Execute()
}
```

- **cmd/root.go** (ایجادشده توسط Cobra):
```go
package cmd

import (
    "os"
    "github.com/spf13/cobra"
)

var rootCmd = &cobra.Command{
    Use:   "todo-cli",
    Short: "A simple todo list CLI",
    Long:  `A CLI tool to manage your tasks with commands to add, list, and delete tasks.`,
}

func Execute() {
    err := rootCmd.Execute()
    if err != nil {
        os.Exit(1)
    }
}

func init() {
    rootCmd.AddCommand(addCmd)
    rootCmd.AddCommand(listCmd)
    rootCmd.AddCommand(deleteCmd)
}
```

- **cmd/add.go**:
```go
package cmd

import (
    "fmt"
    "os"
    "encoding/json"
    "github.com/spf13/cobra"
)

type Task struct {
    ID      int    `json:"id"`
    Content string `json:"content"`
}

var addCmd = &cobra.Command{
    Use:   "add [task content]",
    Short: "Add a new task",
    Args:  cobra.MinimumNArgs(1),
    Run: func(cmd *cobra.Command, args []string) {
        tasks := loadTasks()
        task := Task{
            ID:      len(tasks) + 1,
            Content: args[0],
        }
        tasks = append(tasks, task)
        saveTasks(tasks)
        fmt.Printf("Added task: %s (ID: %d)\n", task.Content, task.ID)
    },
}

func loadTasks() []Task {
    data, err := os.ReadFile("tasks.json")
    if err != nil {
        return []Task{}
    }
    var tasks []Task
    json.Unmarshal(data, &tasks)
    return tasks
}

func saveTasks(tasks []Task) {
    data, err := json.MarshalIndent(tasks, "", "  ")
    if err != nil {
        fmt.Println("Error saving tasks:", err)
        return
    }
    os.WriteFile("tasks.json", data, 0644)
}
```

- **cmd/list.go**:
```go
package cmd

import (
    "fmt"
    "github.com/spf13/cobra"
)

var listCmd = &cobra.Command{
    Use:   "list",
    Short: "List all tasks",
    Run: func(cmd *cobra.Command, args []string) {
        tasks := loadTasks()
        if len(tasks) == 0 {
            fmt.Println("No tasks found.")
            return
        }
        for _, task := range tasks {
            fmt.Printf("ID: %d, Content: %s\n", task.ID, task.Content)
        }
    },
}
```

- **cmd/delete.go**:
```go
package cmd

import (
    "fmt"
    "strconv"
    "github.com/spf13/cobra"
)

var deleteCmd = &cobra.Command{
    Use:   "delete [task ID]",
    Short: "Delete a task by ID",
    Args:  cobra.ExactArgs(1),
    Run: func(cmd *cobra.Command, args []string) {
        id, err := strconv.Atoi(args[0])
        if err != nil {
            fmt.Println("Invalid ID")
            return
        }
        tasks := loadTasks()
        for i, task := range tasks {
            if task.ID == id {
                tasks = append(tasks[:i], tasks[i+1:]...)
                saveTasks(tasks)
                fmt.Printf("Deleted task ID: %d\n", id)
                return
            }
        }
        fmt.Println("Task not found")
    },
}
```

#### اجرا
```bash
go build -o todo-cli
./todo-cli add "Buy groceries"
./todo-cli list
./todo-cli delete 1
```

### 1.5. نکات پیشرفته
- **پرچم‌ها (Flags)**:
  ```go
  var priority int
  addCmd.Flags().IntVarP(&priority, "priority", "p", 1, "Task priority")
  ```
- **مستندات خودکار**:
  ```bash
  ./todo-cli --help
  ```
- **ذخیره‌سازی پیشرفته**: به جای فایل JSON، از دیتابیس (مثل SQLite) استفاده کنید.
- **تست**:
  ```go
  func TestAddTask(t *testing.T) {
      tasks := loadTasks()
      initialLen := len(tasks)
      addCmd.Run(addCmd, []string{"Test task"})
      tasks = loadTasks()
      if len(tasks) != initialLen+1 {
          t.Errorf("Expected %d tasks, got %d", initialLen+1, len(tasks))
      }
  }
  ```

### 1.6. خطاهای رایج
- عدم بررسی خطاهای فایل (مثل `os.ReadFile`).
- استفاده از IDهای تکراری در وظایف.

---

## 2. پیاده‌سازی REST API کامل با دیتابیس و احراز هویت

### 2.1. مفهوم
یک REST API کامل برای مدیریت کاربران با قابلیت CRUD (ایجاد، خواندن، به‌روزرسانی، حذف) و احراز هویت مبتنی بر JWT.

### 2.2. هدف پروژه
ساخت API برای مدیریت کاربران با PostgreSQL و احراز هویت JWT.

### 2.3. پیش‌نیازها
- PostgreSQL
- پکیج‌ها:
  ```bash
  go get -u github.com/lib/pq
  go get -u github.com/dgrijalva/jwt-go
  go get -u golang.org/x/crypto/bcrypt
  ```

### 2.4. پیاده‌سازی
#### ساختار پروژه
```bash
mkdir user-api
cd user-api
go mod init github.com/username/user-api
mkdir handlers models middleware
```

#### دیتابیس
جدول کاربران در PostgreSQL:
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    password VARCHAR(100) NOT NULL
);
```

#### کد اصلی
- **models/user.go**:
```go
package models

type User struct {
    ID       int    `json:"id"`
    Username string `json:"username"`
    Password string `json:"-"` // مخفی در JSON
}
```

- **main.go**:
```go
package main

import (
    "database/sql"
    "log"
    "net/http"
    "github.com/username/user-api/handlers"
    _ "github.com/lib/pq"
)

func main() {
    connStr := "user=postgres password=secret dbname=mydb sslmode=disable"
    db, err := sql.Open("postgres", connStr)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    mux := http.NewServeMux()
    handlers.SetupRoutes(mux, db)
    log.Println("Server running on :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

- **handlers/handlers.go**:
```go
package handlers

import (
    "database/sql"
    "encoding/json"
    "net/http"
    "github.com/username/user-api/middleware"
    "github.com/username/user-api/models"
    "golang.org/x/crypto/bcrypt"
    "github.com/dgrijalva/jwt-go"
)

func SetupRoutes(mux *http.ServeMux, db *sql.DB) {
    mux.HandleFunc("POST /register", registerHandler(db))
    mux.HandleFunc("POST /login", loginHandler(db))
    mux.Handle("/users", middleware.AuthMiddleware(http.HandlerFunc(usersHandler(db))))
}

func registerHandler(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var user models.User
        if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
            http.Error(w, "Invalid input", http.StatusBadRequest)
            return
        }
        hashedPassword, _ := bcrypt.GenerateFromPassword([]byte(user.Password), bcrypt.DefaultCost)
        _, err := db.Exec("INSERT INTO users (username, password) VALUES ($1, $2)", user.Username, string(hashedPassword))
        if err != nil {
            http.Error(w, "User already exists", http.StatusConflict)
            return
        }
        w.WriteHeader(http.StatusCreated)
    }
}

func loginHandler(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var user models.User
        if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
            http.Error(w, "Invalid input", http.StatusBadRequest)
            return
        }
        var storedPassword string
        err := db.QueryRow("SELECT password FROM users WHERE username = $1", user.Username).Scan(&storedPassword)
        if err != nil {
            http.Error(w, "Invalid credentials", http.StatusUnauthorized)
            return
        }
        if err := bcrypt.CompareHashAndPassword([]byte(storedPassword), []byte(user.Password)); err != nil {
            http.Error(w, "Invalid credentials", http.StatusUnauthorized)
            return
        }
        token := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
            "username": user.Username,
        })
        tokenString, _ := token.SignedString([]byte("secret"))
        json.NewEncoder(w).Encode(map[string]string{"token": tokenString})
    }
}

func usersHandler(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        rows, err := db.Query("SELECT id, username FROM users")
        if err != nil {
            http.Error(w, "Database error", http.StatusInternalServerError)
            return
        }
        defer rows.Close()
        var users []models.User
        for rows.Next() {
            var user models.User
            rows.Scan(&user.ID, &user.Username)
            users = append(users, user)
        }
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(users)
    }
}
```

- **middleware/auth.go**:
```go
package middleware

import (
    "net/http"
    "strings"
    "github.com/dgrijalva/jwt-go"
)

func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            http.Error(w, "Authorization header required", http.StatusUnauthorized)
            return
        }
        tokenStr := strings.TrimPrefix(authHeader, "Bearer ")
        token, err := jwt.Parse(tokenStr, func(token *jwt.Token) (interface{}, error) {
            return []byte("secret"), nil
        })
        if err != nil || !token.Valid {
            http.Error(w, "Invalid token", http.StatusUnauthorized)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

#### اجرا
1. PostgreSQL را راه‌اندازی کنید.
2. برنامه را اجرا کنید: `go run main.go`
3. تست با `curl`:
   ```bash
   curl -X POST http://localhost:8080/register -d '{"username":"ali","password":"pass123"}'
   curl -X POST http://localhost:8080/login -d '{"username":"ali","password":"pass123"}'
   curl -H "Authorization: Bearer <token>" http://localhost:8080/users
   ```

### 2.5. نکات پیشرفته
- **امنیت**:
  - از رمزنگاری قوی‌تر برای JWT استفاده کنید.
  - از HTTPS برای جلوگیری از شنود استفاده کنید.
- **دیتابیس**:
  - از اتصال‌های پایدار با `sql.DB` استفاده کنید.
  - از migrationها (مثل `golang-migrate`) برای مدیریت اسکیما استفاده کنید.
- **تست**:
  ```go
  func TestRegister(t *testing.T) {
      req := httptest.NewRequest(http.MethodPost, "/register", strings.NewReader(`{"username":"test","password":"test"}`))
      w := httptest.NewRecorder()
      registerHandler(db)(w, req)
      if w.Code != http.StatusCreated {
          t.Errorf("Expected status 201, got %d", w.Code)
      }
  }
  ```

### 2.6. خطاهای رایج
- عدم بستن اتصال‌های دیتابیس.
- نادیده گرفتن خطاهای JWT.

---

## 3. برنامه Concurrent: سیستم پردازش داده موازی

### 3.1. مفهوم
یک سیستم پردازش داده موازی با استفاده از goroutine‌ها و Worker Pool برای پردازش وظایف سنگین.

### 3.2. هدف پروژه
پردازش موازی فایل‌های متنی برای شمارش کلمات.

### 3.3. پیاده‌سازی
#### ساختار پروژه
```bash
mkdir word-counter
cd word-counter
go mod init github.com/username/word-counter
```

#### کد اصلی
```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "strings"
    "sync"
)

type Result struct {
    FileName string
    WordCount int
}

func countWords(fileName string) (int, error) {
    file, err := os.Open(fileName)
    if err != nil {
        return 0, err
    }
    defer file.Close()

    count := 0
    scanner := bufio.NewScanner(file)
    for scanner.Scan() {
        words := strings.Fields(scanner.Text())
        count += len(words)
    }
    return count, scanner.Err()
}

func worker(files <-chan string, results chan<- Result, wg *sync.WaitGroup) {
    defer wg.Done()
    for file := range files {
        count, err := countWords(file)
        if err != nil {
            fmt.Printf("Error processing %s: %v\n", file, err)
            continue
        }
        results <- Result{FileName: file, WordCount: count}
    }
}

func main() {
    files := []string{"file1.txt", "file2.txt", "file3.txt"}
    numWorkers := 2
    filesChan := make(chan string, len(files))
    resultsChan := make(chan Result, len(files))
    var wg sync.WaitGroup

    // راه‌اندازی کارگران
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go worker(filesChan, resultsChan, &wg)
    }

    // ارسال فایل‌ها
    for _, file := range files {
        filesChan <- file
    }
    close(filesChan)

    // جمع‌آوری نتایج
    go func() {
        wg.Wait()
        close(resultsChan)
    }()

    total := 0
    for result := range resultsChan {
        fmt.Printf("%s: %d words\n", result.FileName, result.WordCount)
        total += result.WordCount
    }
    fmt.Printf("Total words: %d\n", total)
}
```

#### ایجاد فایل‌های نمونه
```bash
echo "This is file one" > file1.txt
echo "File two has more words" > file2.txt
echo "Third file" > file3.txt
```

#### اجرا
```bash
go run main.go
```

### 3.4. نکات پیشرفته
- **بهینه‌سازی**:
  - تعداد کارگران را بر اساس `runtime.NumCPU()` تنظیم کنید.
  - از `sync.Pool` برای bufferها استفاده کنید:
    ```go
    var pool = sync.Pool{New: func() interface{} { return &bytes.Buffer{} }}
    ```
- **مدیریت خطا**:
  - خطاها را در یک کانال جداگانه جمع‌آوری کنید.
- **تست**:
  ```go
  func TestCountWords(t *testing.T) {
      f, _ := os.CreateTemp("", "test.txt")
      f.WriteString("hello world")
      count, err := countWords(f.Name())
      if err != nil || count != 2 {
          t.Errorf("Expected 2 words, got %d", count)
      }
  }
  ```

### 3.5. خطاهای رایج
- عدم بستن کانال‌ها.
- نادیده گرفتن خطاهای فایل.

---

## 4. میکروسرویس‌ها: طراحی و پیاده‌سازی با Go و gRPC

### 4.1. مفهوم
میکروسرویس‌ها برنامه‌هایی هستند که به سرویس‌های کوچک و مستقل تقسیم شده‌اند. gRPC یک فریم‌ورک RPC با کارایی بالا است که در Go بسیار محبوب است.

### 4.2. هدف پروژه
ساخت یک میکروسرویس برای مدیریت محصولات با gRPC.

### 4.3. پیش‌نیازها
- نصب protoc:
  ```bash
  sudo apt install protobuf-compiler
  go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
  go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
  ```

### 4.4. پیاده‌سازی
#### ساختار پروژه
```bash
mkdir product-service
cd product-service
go mod init github.com/username/product-service
mkdir proto server client
```

#### تعریف پروتکل (proto/product.proto)
```proto
syntax = "proto3";

package product;
option go_package = "github.com/username/product-service/proto";

service ProductService {
    rpc AddProduct(AddProductRequest) returns (AddProductResponse);
    rpc GetProduct(GetProductRequest) returns (GetProductResponse);
}

message AddProductRequest {
    string name = 1;
    float price = 2;
}

message AddProductResponse {
    int32 id = 1;
}

message GetProductRequest {
    int32 id = 1;
}

message GetProductResponse {
    int32 id = 1;
    string name = 2;
    float price = 3;
}
```

#### تولید کد
```bash
protoc --go_out=. --go-grpc_out=. proto/product.proto
```

#### سرور (server/main.go)
```go
package main

import (
    "context"
    "log"
    "net"
    "sync"
    "github.com/username/product-service/proto"
    "google.golang.org/grpc"
)

type Product struct {
    ID    int32
    Name  string
    Price float32
}

type server struct {
    proto.UnimplementedProductServiceServer
    products map[int32]Product
    mu       sync.Mutex
}

func (s *server) AddProduct(ctx context.Context, req *proto.AddProductRequest) (*proto.AddProductResponse, error) {
    s.mu.Lock()
    defer s.mu.Unlock()
    id := int32(len(s.products) + 1)
    s.products[id] = Product{ID: id, Name: req.Name, Price: req.Price}
    return &proto.AddProductResponse{Id: id}, nil
}

func (s *server) GetProduct(ctx context.Context, req *proto.GetProductRequest) (*proto.GetProductResponse, error) {
    s.mu.Lock()
    defer s.mu.Unlock()
    p, ok := s.products[req.Id]
    if !ok {
        return nil, fmt.Errorf("product not found")
    }
    return &proto.GetProductResponse{Id: p.ID, Name: p.Name, Price: p.Price}, nil
}

func main() {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatal(err)
    }
    s := grpc.NewServer()
    proto.RegisterProductServiceServer(s, &server{products: make(map[int32]Product)})
    log.Println("Server running on :50051")
    log.Fatal(s.Serve(lis))
}
```

#### کلاینت (client/main.go)
```go
package main

import (
    "context"
    "fmt"
    "log"
    "github.com/username/product-service/proto"
    "google.golang.org/grpc"
)

func main() {
    conn, err := grpc.Dial("localhost:50051", grpc.WithInsecure())
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()
    client := proto.NewProductServiceClient(conn)

    // افزودن محصول
    res, err := client.AddProduct(context.Background(), &proto.AddProductRequest{
        Name:  "Laptop",
        Price: 999.99,
    })
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Added product ID: %d\n", res.Id)

    // دریافت محصول
    product, err := client.GetProduct(context.Background(), &proto.GetProductRequest{Id: res.Id})
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Product: %+v\n", product)
}
```

#### اجرا
1. سرور: `go run server/main.go`
2. کلاینت: `go run client/main.go`

### 4.5. نکات پیشرفته
- **مدیریت خطا**:
  ```go
  import "google.golang.org/grpc/status"
  return nil, status.Error(codes.NotFound, "product not found")
  ```
- **همزمانی**: از `sync.Mutex` برای دسترسی ایمن به داده‌ها استفاده کنید.
- **دیتابیس**: به جای نقشه، از دیتابیس (مثل PostgreSQL) استفاده کنید.
- **تست**:
  ```go
  func TestAddProduct(t *testing.T) {
      s := &server{products: make(map[int32]Product)}
      res, err := s.AddProduct(context.Background(), &proto.AddProductRequest{Name: "Test", Price: 10})
      if err != nil || res.Id == 0 {
          t.Errorf("Expected valid ID, got %d", res.Id)
      }
  }
  ```

### 4.6. خطاهای رایج
- عدم تولید فایل‌های proto.
- نادیده گرفتن خطاهای اتصال gRPC.

---

## 5. برنامه بلادرنگ: ساخت چت با WebSocket

### 5.1. مفهوم
یک برنامه چت بلادرنگ که کاربران می‌توانند پیام‌ها را از طریق WebSocket ارسال و دریافت کنند.

### 5.2. هدف پروژه
ساخت سرور چت WebSocket با امکان پخش پیام به همه کلاینت‌ها.

### 5.3. پیش‌نیازها
```bash
go get -u github.com/gorilla/websocket
```

### 5.4. پیاده‌سازی
#### ساختار پروژه
```bash
mkdir chat-app
cd chat-app
go mod init github.com/username/chat-app
mkdir static
```

#### کد اصلی
- **main.go**:
```go
package main

import (
    "log"
    "net/http"
    "sync"
    "github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
    CheckOrigin: func(r *http.Request) bool { return true },
}

type Hub struct {
    clients    map[*websocket.Conn]string
    broadcast  chan Message
    register   chan *websocket.Conn
    unregister chan *websocket.Conn
    mu         sync.Mutex
}

type Message struct {
    Username string
    Content  string
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
            h.clients[conn] = "Anonymous"
            h.mu.Unlock()
        case conn := <-h.unregister:
            h.mu.Lock()
            delete(h.clients, conn)
            h.mu.Unlock()
        case msg := <-h.broadcast:
            h.mu.Lock()
            for conn := range h.clients {
                err := conn.WriteJSON(msg)
                if err != nil {
                    conn.Close()
                    delete(h.clients, conn)
                }
            }
            h.mu.Unlock()
        }
    }
}

func handleWebSocket(hub *Hub) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        conn, err := upgrader.Upgrade(w, r, nil)
        if err != nil {
            log.Println(err)
            return
        }
        hub.register <- conn
        defer func() { hub.unregister <- conn }()

        for {
            var msg Message
            err := conn.ReadJSON(&msg)
            if err != nil {
                log.Println(err)
                return
            }
            hub.broadcast <- msg
        }
    }
}

func main() {
    hub := NewHub()
    go hub.Run()

    http.HandleFunc("/ws", handleWebSocket(hub))
    http.Handle("/", http.FileServer(http.Dir("static")))
    log.Println("Server running on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

- **static/index.html**:
```html
<!DOCTYPE html>
<html>
<head>
    <title>Chat App</title>
</head>
<body>
    <h1>Chat App</h1>
    <input id="username" placeholder="Username" value="Anonymous">
    <input id="message" placeholder="Type a message">
    <button onclick="sendMessage()">Send</button>
    <ul id="messages"></ul>
    <script>
        const ws = new WebSocket("ws://localhost:8080/ws");
        ws.onmessage = function(event) {
            const msg = JSON.parse(event.data);
            const li = document.createElement("li");
            li.textContent = `${msg.username}: ${msg.content}`;
            document.getElementById("messages").appendChild(li);
        };
        function sendMessage() {
            const username = document.getElementById("username").value;
            const content = document.getElementById("message").value;
            ws.send(JSON.stringify({username, content}));
            document.getElementById("message").value = "";
        }
    </script>
</body>
</html>
```

#### اجرا
```bash
go run main.go
```
- به `http://localhost:8080` بروید و چت کنید.

### 5.5. نکات پیشرفته
- **امنیت**:
  - `CheckOrigin` را در تولید محدود کنید.
  - از HTTPS/WSS استفاده کنید.
- **Heartbeat**:
  ```go
  go func() {
      for range time.Tick(30 * time.Second) {
          conn.WriteMessage(websocket.PingMessage, []byte{})
      }
  }()
  ```
- **تست**:
  ```go
  func TestHub(t *testing.T) {
      hub := NewHub()
      go hub.Run()
      conn, _, _ := websocket.DefaultDialer.Dial("ws://localhost:8080/ws", nil)
      hub.broadcast <- Message{Username: "test", Content: "hello"}
      var msg Message
      conn.ReadJSON(&msg)
      if msg.Content != "hello" {
          t.Errorf("Expected 'hello', got %s", msg.Content)
      }
  }
  ```

### 5.6. خطاهای رایج
- عدم بستن اتصال‌های WebSocket.
- نادیده گرفتن خطاهای JSON.

---

## 6. بهترین شیوه‌ها و نکات

- **ساختار پروژه**: هر پروژه را با دایرکتوری‌های مشخص (مثل `cmd`, `handlers`, `models`) سازمان‌دهی کنید.
- **مدیریت خطا**: همیشه خطاها را بررسی و به کاربر گزارش دهید.
- **تست‌نویسی**: تست‌های واحد و یکپارچگی برای هر پروژه بنویسید.
- **مستندسازی**: از مستندات واضح (مثل README) و نظرات درون‌کد استفاده کنید.
- **بهینه‌سازی**:
  - از همزمانی بهینه (مثل Worker Pool) استفاده کنید.
  - منابع (مثل فایل‌ها و اتصال‌ها) را به درستی آزاد کنید.

---

## 7. نتیجه‌گیری

این جزوه پنج پروژه عملی در Go را با جزئیات کامل پوشش داد. از ساخت CLI با Cobra تا پیاده‌سازی REST API، برنامه موازی، میکروسرویس با gRPC، و چت بلادرنگ با WebSocket، هر پروژه با کد کامل، نکات پیشرفته، و بهترین شیوه‌ها ارائه شد. برای یادگیری عمیق‌تر:
- مستندات رسمی: `https://golang.org/doc/`
- کتاب "Building Microservices with Go"
- تمرینව