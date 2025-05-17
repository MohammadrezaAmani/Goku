# بهترین شیوه‌ها و نکات پیشرفته در زبان برنامه‌نویسی Go

این جزوه به بررسی جامع بهترین شیوه‌ها و تکنیک‌های پیشرفته در Go می‌پردازد. Go به دلیل سادگی و کارایی، برای ساخت سیستم‌های مقیاس‌پذیر و قابل نگهداری بسیار مناسب است. تمام موضوعات درخواستی با جزئیات کامل، مثال‌های عملی، و نکات پیشرفته شرح داده شده‌اند.

---

## 1. طراحی کد: اصول SOLID و معماری تمیز در Go

### 1.1. مفهوم
طراحی کد با کیفیت بالا شامل استفاده از اصول طراحی (مثل SOLID) و معماری تمیز (Clean Architecture) است که کد را قابل نگهداری، تست‌پذیر، و مقیاس‌پذیر می‌کند.

### 1.2. اصول SOLID
SOLID شامل پنج اصل طراحی شیءگرا است که در Go نیز قابل اعمال هستند، اگرچه Go به طور کامل شیءگرا نیست.

#### 1.2.1. S - Single Responsibility Principle (اصل مسئولیت واحد)
هر نوع یا تابع باید تنها یک دلیل برای تغییر داشته باشد.
- **مثال**:
  ```go
  // بد: یک struct با چندین مسئولیت
  type UserManager struct {
      // مدیریت کاربر
      // ارسال ایمیل
      // لاگ‌گیری
  }

  // خوب: جداسازی مسئولیت‌ها
  type UserService struct {
      db *sql.DB
  }

  func (s *UserService) CreateUser(name string) error {
      // فقط مدیریت کاربر
      _, err := s.db.Exec("INSERT INTO users (name) VALUES ($1)", name)
      return err
  }

  type EmailService struct{}

  func (e *EmailService) SendEmail(to, msg string) error {
      // فقط ارسال ایمیل
      return nil
  }
  ```

#### 1.2.2. O - Open/Closed Principle (اصل باز/بسته)
کدها باید برای گسترش باز و برای تغییر بسته باشند.
- **مثال**: استفاده از رابط‌ها (interface) برای افزودن رفتار جدید:
  ```go
  type Logger interface {
      Log(message string)
  }

  type ConsoleLogger struct{}

  func (c ConsoleLogger) Log(message string) {
      fmt.Println("Console:", message)
  }

  type FileLogger struct{}

  func (f FileLogger) Log(message string) {
      // نوشتن در فایل
  }

  func ProcessData(data string, logger Logger) {
      logger.Log(data) // قابل گسترش بدون تغییر ProcessData
  }
  ```

#### 1.2.3. L - Liskov Substitution Principle (اصل جایگذاری لیسکف)
انواع مشتق‌شده باید بدون تغییر رفتار قابل جایگزینی با نوع پایه باشند.
- **مثال**: رابط‌ها در Go به طور ضمنی این اصل را پشتیبانی می‌کنند:
  ```go
  type Writer interface {
      Write(data string) error
  }

  type FileWriter struct{}

  func (f FileWriter) Write(data string) error {
      // نوشتن در فایل
      return nil
  }

  type BufferWriter struct{}

  func (b BufferWriter) Write(data string) error {
      // نوشتن در بافر
      return nil
  }

  func SaveData(w Writer, data string) error {
      return w.Write(data) // هر Writer بدون مشکل کار می‌کند
  }
  ```

#### 1.2.4. I - Interface Segregation Principle (اصل تفکیک رابط)
کلاینت‌ها نباید مجبور به پیاده‌سازی رابط‌های غیرضروری شوند.
- **مثال**:
  ```go
  // بد: رابط بزرگ
  type BigInterface interface {
      Read()
      Write()
      Close()
  }

  // خوب: رابط‌های کوچک
  type Reader interface {
      Read()
  }

  type Writer interface {
      Write()
  }

  type FileReader struct{}

  func (f FileReader) Read() {
      // فقط خواندن
  }
  ```

#### 1.2.5. D - Dependency Inversion Principle (اصل وارونگی وابستگی)
ماژول‌های سطح بالا نباید به ماژول‌های سطح پایین وابسته باشند؛ هر دو باید به abstractions وابسته باشند.
- **مثال**: تزریق وابستگی از طریق رابط:
  ```go
  type Repository interface {
      Save(data string) error
  }

  type SQLRepository struct{}

  func (s SQLRepository) Save(data string) error {
      // ذخیره در دیتابیس
      return nil
  }

  type Service struct {
      repo Repository
  }

  func NewService(repo Repository) *Service {
      return &Service{repo: repo}
  }

  func (s *Service) Process(data string) error {
      return s.repo.Save(data)
  }
  ```

### 1.3. معماری تمیز (Clean Architecture)
معماری تمیز کد را به لایه‌های مستقل تقسیم می‌کند:
- **Entities**: منطق کسب‌وکار (مثل مدل‌های داده).
- **Use Cases**: قوانین کسب‌وکار.
- **Interface Adapters**: تبدیل داده‌ها (مثل کنترلرها).
- **Frameworks/Drivers**: ابزارها (مثل دیتابیس، وب).

#### مثال
ساختار پروژه:
```bash
/user-service
├── /entities
│   └── user.go
├── /usecases
│   └── user.go
├── /adapters
│   └── http.go
├── /drivers
│   └── postgres.go
├── main.go
```

- **entities/user.go**:
  ```go
  package entities

  type User struct {
      ID   int
      Name string
  }
  ```

- **usecases/user.go**:
  ```go
  package usecases

  type UserRepository interface {
      Save(user entities.User) error
      Find(id int) (entities.User, error)
  }

  type UserService struct {
      repo UserRepository
  }

  func NewUserService(repo UserRepository) *UserService {
      return &UserService{repo: repo}
  }

  func (s *UserService) CreateUser(name string) (entities.User, error) {
      user := entities.User{ID: 1, Name: name}
      return user, s.repo.Save(user)
  }
  ```

- **drivers/postgres.go**:
  ```go
  package drivers

  import (
      "database/sql"
      "github.com/username/user-service/entities"
  )

  type PostgresRepo struct {
      db *sql.DB
  }

  func (r PostgresRepo) Save(user entities.User) error {
      _, err := r.db.Exec("INSERT INTO users (id, name) VALUES ($1, $2)", user.ID, user.Name)
      return err
  }

  func (r PostgresRepo) Find(id int) (entities.User, error) {
      var user entities.User
      err := r.db.QueryRow("SELECT id, name FROM users WHERE id = $1", id).Scan(&user.ID, &user.Name)
      return user, err
  }
  ```

- **adapters/http.go**:
  ```go
  package adapters

  import (
      "encoding/json"
      "net/http"
      "github.com/username/user-service/usecases"
  )

  type HTTPHandler struct {
      service *usecases.UserService
  }

  func NewHTTPHandler(service *usecases.UserService) *HTTPHandler {
      return &HTTPHandler{service: service}
  }

  func (h *HTTPHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
      var input struct {
          Name string `json:"name"`
      }
      if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
          http.Error(w, "Invalid input", http.StatusBadRequest)
          return
      }
      user, err := h.service.CreateUser(input.Name)
      if err != nil {
          http.Error(w, "Failed to create user", http.StatusInternalServerError)
          return
      }
      w.Header().Set("Content-Type", "application/json")
      json.NewEncoder(w).Encode(user)
  }
  ```

- **main.go**:
  ```go
  package main

  import (
      "database/sql"
      "log"
      "net/http"
      "github.com/username/user-service/adapters"
      "github.com/username/user-service/drivers"
      "github.com/username/user-service/usecases"
      _ "github.com/lib/pq"
  )

  func main() {
      db, err := sql.Open("postgres", "user=postgres password=secret dbname=mydb sslmode=disable")
      if err != nil {
          log.Fatal(err)
      }
      repo := drivers.PostgresRepo{db: db}
      service := usecases.NewUserService(&repo)
      handler := adapters.NewHTTPHandler(service)

      http.HandleFunc("/users", handler.CreateUser)
      log.Fatal(http.ListenAndServe(":8080", nil))
  }
  ```

### 1.4. نکات پیشرفته
- **رابط‌های کوچک**: رابط‌های Go را کوچک و متمرکز نگه دارید.
- **تست‌پذیری**: معماری تمیز تست‌پذیری را افزایش می‌دهد:
  ```go
  func TestUserService(t *testing.T) {
      repo := &MockRepo{}
      service := usecases.NewUserService(repo)
      user, err := service.CreateUser("Ali")
      if err != nil || user.Name != "Ali" {
          t.Errorf("Expected user Ali, got %v", user)
      }
  }
  ```
- **تزریق وابستگی**: از سازنده‌ها (constructor) برای تزریق وابستگی استفاده کنید.

### 1.5. خطاهای رایج
- ترکیب لایه‌ها (مثل دسترسی مستقیم به دیتابیس در usecase).
- رابط‌های بیش از حد بزرگ.

---

## 2. مدیریت خطاها پیشرفته

### 2.1. مفهوم
مدیریت خطا در Go با استفاده از نوع `error` انجام می‌شود. پکیج `pkg/errors` ابزارهای پیشرفته‌ای برای wrapping و تحلیل خطاها ارائه می‌دهد.

### 2.2. استفاده از pkg/errors
#### نصب
```bash
go get github.com/pkg/errors
```

#### wrapping خطاها
Wrapping خطاها اطلاعات زمینه‌ای (context) اضافه می‌کند:
```go
package main

import (
    "database/sql"
    "fmt"
    "github.com/pkg/errors"
)

func queryUser(db *sql.DB, id int) error {
    var name string
    err := db.QueryRow("SELECT name FROM users WHERE id = $1", id).Scan(&name)
    if err == sql.ErrNoRows {
        return errors.Wrapf(err, "user with id %d not found", id)
    }
    if err != nil {
        return errors Wrap(err, "failed to query user")
    }
    return nil
}

func main() {
    db, _ := sql.Open("postgres", "user=postgres password=secret dbname=mydb sslmode=disable")
    err := queryUser(db, 999)
    if err != nil {
        fmt.Printf("Error: %+v\n", err) // چاپ stack trace
    }
}
```

#### بررسی خطاها
- **Is**: بررسی نوع خطا:
  ```go
  if errors.Is(err, sql.ErrNoRows) {
      fmt.Println("No rows found")
  }
  ```
- **As**: استخراج خطای خاص:
  ```go
  var dbErr *sql.DBError
  if errors.As(err, &dbErr) {
      fmt.Println("Database error:", dbErr)
  }
  ```

### 2.3. تعریف خطاهای سفارشی
```go
type NotFoundError struct {
    Resource string
    ID       int
}

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s with ID %d not found", e.Resource, e.ID)
}

func findUser(id int) error {
    return &NotFoundError{Resource: "user", ID: id}
}

func main() {
    err := findUser(999)
    if err != nil {
        if e, ok := err.(*NotFoundError); ok {
            fmt.Printf("Not found: %s, ID: %d\n", e.Resource, e.ID)
        }
    }
}
```

### 2.4. نکات پیشرفته
- **Stack Trace**: از `errors.WithStack` برای افزودن stack trace استفاده کنید:
  ```go
  return errors.WithStack(fmt.Errorf("something went wrong"))
  ```
- **Sentinel Errors**: خطاهای ثابت برای بررسی ساده‌تر:
  ```go
  var ErrNotFound = errors.New("resource not found")
  ```
- **Deferred Recovery**: برای مدیریت panic:
  ```go
  defer func() {
      if r := recover(); r != nil {
          log.Printf("Recovered: %v", r)
      }
  }()
  ```

### 2.5. خطاهای رایج
- Wrapping بیش از حد که خوانایی stack trace را کاهش می‌دهد.
- نادیده گرفتن بررسی خطاها.

---

## 3. الگوهای طراحی: Factory، Singleton، و Observer

### 3.1. مفهوم
الگوهای طراحی راه‌حل‌های استاندارد برای مسائل رایج هستند. در Go، به دلیل سادگی زبان، برخی الگوها ساده‌تر پیاده‌سازی می‌شوند.

### 3.2. Factory
#### مفهوم
Factory برای ایجاد اشیاء بدون مشخص کردن نوع دقیق استفاده می‌شود.
- **مثال**:
  ```go
  type Database interface {
      Query() string
  }

  type PostgresDB struct{}

  func (p PostgresDB) Query() string {
      return "Postgres query"
  }

  type MongoDB struct{}

  func (m MongoDB) Query() string {
      return "MongoDB query"
  }

  func NewDatabase(dbType string) (Database, error) {
      switch dbType {
      case "postgres":
          return PostgresDB{}, nil
      case "mongo":
          return MongoDB{}, nil
      default:
          return nil, fmt.Errorf("unsupported db: %s", dbType)
      }
  }

  func main() {
      db, err := NewDatabase("postgres")
      if err != nil {
          log.Fatal(err)
      }
      fmt.Println(db.Query()) // Postgres query
  }
  ```

### 3.3. Singleton
#### مفهوم
Singleton تضمین می‌کند که تنها یک نمونه از یک نوع وجود داشته باشد.
- **مثال**:
  ```go
  package main

  import (
      "fmt"
      "sync"
  )

  type Config struct {
      setting string
  }

  var (
      instance *Config
      once     sync.Once
  )

  func GetConfig() *Config {
      once.Do(func() {
          instance = &Config{setting: "default"}
      })
      return instance
  }

  func main() {
      cfg1 := GetConfig()
      cfg2 := GetConfig()
      fmt.Println(cfg1 == cfg2) // true
  }
  ```

### 3.4. Observer
#### مفهوم
Observer به اشیاء اجازه می‌دهد تا از تغییرات یکدیگر مطلع شوند.
- **مثال**:
  ```go
  package main

  import "fmt"

  type Subscriber interface {
      Update(message string)
  }

  type Publisher struct {
      subscribers []Subscriber
  }

  func (p *Publisher) Subscribe(s Subscriber) {
      p.subscribers = append(p.subscribers, s)
  }

  func (p *Publisher) Notify(message string) {
      for _, s := range p.subscribers {
          s.Update(message)
      }
  }

  type EmailSubscriber struct {
      email string
  }

  func (e EmailSubscriber) Update(message string) {
      fmt.Printf("Email to %s: %s\n", e.email, message)
  }

  func main() {
      pub := Publisher{}
      sub1 := EmailSubscriber{email: "user1@example.com"}
      sub2 := EmailSubscriber{email: "user2@example.com"}
      pub.Subscribe(sub1)
      pub.Subscribe(sub2)
      pub.Notify("New update!") // Email to user1@example.com: New update!
  }
  ```

### 3.5. نکات پیشرفته
- **Factory**: برای ایجاد اشیاء پیچیده، از Builder به همراه Factory استفاده کنید.
- **Singleton**: در Go کمتر استفاده می‌شود؛ به جای آن از بسته‌های global یا تزریق وابستگی استفاده کنید.
- **Observer**: برای سیستم‌های بزرگ، از کانال‌ها (channels) برای پیاده‌سازی الگوی مشابه استفاده کنید:
  ```go
  messages := make(chan string)
  go func() {
      for msg := range messages {
          fmt.Println("Received:", msg)
      }
  }()
  ```

### 3.6. خطاهای رایج
- استفاده بیش از حد از Singleton که تست‌پذیری را کاهش می‌دهد.
- پیچیدگی غیرضروری در پیاده‌سازی الگوها.

---

## 4. کارایی و مقیاس‌پذیری

### 4.1. مفهوم
کارایی (Performance) به سرعت اجرای برنامه و مقیاس‌پذیری (Scalability) به توانایی مدیریت بار زیاد اشاره دارد.

### 4.2. بهینه‌سازی کارایی
- **کاهش تخصیص حافظه**:
  - از اسلایس‌ها با ظرفیت اولیه استفاده کنید:
    ```go
    s := make([]int, 0, 1000)
    ```
  - از `sync.Pool` برای اشیاء موقت استفاده کنید:
    ```go
    var pool = sync.Pool{New: func() interface{} { return &bytes.Buffer{} }}
    ```
- **همزمانی بهینه**:
  - از Worker Pool برای محدود کردن goroutine‌ها استفاده کنید:
    ```go
    jobs := make(chan int, 100)
    for w := 0; w < 4; w++ {
        go worker(jobs)
    }
    ```
- **پروفایلینگ**:
  ```bash
  go tool pprof http://localhost:6060/debug/pprof/profile
  ```

### 4.3. مقیاس‌پذیری
- **تقسیم بار**:
  - از Load Balancer برای توزیع درخواست‌ها استفاده کنید.
  - از Redis یا Kafka برای مدیریت صف‌ها:
    ```go
    import "github.com/go-redis/redis/v8"
    client := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
    ```
- **دیتابیس مقیاس‌پذیر**:
  - از شاردینگ یا replication در PostgreSQL/MongoDB استفاده کنید.
  - از اتصال‌های پایدار با `sql.DB`:
    ```go
    db, _ := sql.Open("postgres", connStr)
    db.SetMaxOpenConns(100)
    ```
- **Caching**:
  - از `groupcache` یا Redis برای کش کردن داده‌ها:
    ```go
    import "github.com/golang/groupcache"
    cache := groupcache.NewGroup("users", 64<<20, groupcache.GetterFunc(
        func(ctx context.Context, key string, dest groupcache.Sink) error {
            // دریافت داده
            return nil
        },
    ))
    ```

### 4.4. نکات پیشرفته
- **Rate Limiting**:
  ```go
  import "golang.org/x/time/rate"
  limiter := rate.NewLimiter(10, 1) // 10 درخواست در ثانیه
  ```
- **Graceful Shutdown**:
  ```go
  server := &http.Server{Addr: ":8080"}
  ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
  defer cancel()
  server.Shutdown(ctx)
  ```
- **مانیتورینگ**:
  - از Prometheus و Grafana برای مانیتورینگ استفاده کنید:
    ```go
    import "github.com/prometheus/client_golang/prometheus/promhttp"
    http.Handle("/metrics", promhttp.Handler())
    ```

### 4.5. خطاهای رایج
- تخصیص بیش از حد حافظه در حلقه‌ها.
- عدم مدیریت تعداد goroutine‌ها.

---

## 5. مستندسازی و نگهداری

### 5.1. مفهوم
مستندسازی و نگهداری کد برای درک، استفاده، و توسعه آینده ضروری است. Go با ابزارهایی مثل `godoc` مستندسازی را ساده می‌کند.

### 5.2. نوشتن مستندات با godoc
- **مستندات درون‌کد**:
  ```go
  // Package user manages user-related operations.
  package user

  // UserService provides methods to manage users.
  type UserService struct{}

  // CreateUser creates a new user with the given name.
  // Returns an error if the user already exists.
  func (s *UserService) CreateUser(name string) error {
      return nil
  }
  ```
- **مشاهده**:
  ```bash
  go doc github.com/username/user
  ```

#### مستندات مثال
```go
func ExampleUserService_CreateUser() {
    s := &UserService{}
    err := s.CreateUser("Ali")
    fmt.Println(err)
    // Output: <nil>
}
```

### 5.3. مستندات پروژه
- **README**:
  - شامل توضیح پروژه، نصب، استفاده، و مثال‌ها.
  - مثال:
    ```markdown
    # User Service
    A Go service for managing users.

    ## Installation
    ```bash
    go get github.com/username/user
    ```

    ## Usage
    ```go
    s := user.NewUserService()
    s.CreateUser("Ali")
    ```
    ```
- **CHANGELOG**: تغییرات نسخه‌ها را مستند کنید.
- **API Docs**: از Swagger برای API‌ها استفاده کنید:
  ```bash
  go get github.com/swaggo/swag/cmd/swag
  swag init
  ```

### 5.4. نکات پیشرفته
- **مستندات خودکار**:
  - از `pkg.go.dev` برای انتشار مستندات استفاده کنید.
  - تگ نسخه بزنید:
    ```bash
    git tag v1.0.0
    git push origin v1.0.0
    ```
- **تست‌های مستند**:
  ```go
  func TestCreateUser(t *testing.T) {
      // این تست به عنوان مستندات نیز عمل می‌کند
  }
  ```
- **Code Review**: از نظرات واضح در pull requestها استفاده کنید.

### 5.5. خطاهای رایج
- عدم به‌روزرسانی مستندات پس از تغییر کد.
- مستندات مبهم یا ناقص.

---

## 6. انجمن و منابع

### 6.1. مفهوم
مشارکت در انجمن Go و استفاده از منابع یادگیری برای بهبود مهارت‌ها ضروری است.

### 6.2. مشارکت در پروژه‌های متن‌باز
- **یافتن پروژه**:
  - GitHub: جستجوی پروژه‌های Go با برچسب `good first issue`.
  - پروژه‌های محبوب: Kubernetes، Hugo، Prometheus.
- **مشارکت**:
  - Fork کنید، تغییر دهید، و Pull Request بفرستید.
  - از راهنمای مشارکت پروژه (CONTRIBUTING.md) پیروی کنید.
- **مثال**:
  - افزودن یک ویژگی به یک CLI:
    ```go
    // اضافه کردن پرچم جدید
    cmd.Flags().StringVar(&output, "output", "", "Output file")
    ```

### 6.3. منابع یادگیری
- **مستندات رسمی**:
  - `https://golang.org/doc/`
  - `https://go.dev/tour`
- **کتاب‌ها**:
  - "The Go Programming Language" (Donovan & Kernighan)
  - "Concurrency in Go" (Katherine Cox-Buday)
- **دوره‌های آنلاین**:
  - Go by Example: `https://gobyexample.com`
  - Udemy: "Learn Go Programming"
- **انجمن‌ها**:
  - Reddit: `r/golang`
  - Slack: Gophers Slack (`https://gophers.slack.com`)
  - Stack Overflow: تگ `go`
- **کنفرانس‌ها**:
  - GopherCon
  - GoLab
- **پروژه‌های عملی**:
  - ساخت CLI، API، یا میکروسرویس.
  - مشارکت در پروژه‌های متن‌باز.

### 6.4. نکات پیشرفته
- **Blogging**: تجربیات خود را در وبلاگ یا Medium منتشر کنید.
- **Mentorship**: در انجمن‌ها به سوالات پاسخ دهید.
- **Hackathons**: در رویدادهای برنامه‌نویسی Go شرکت کنید.

### 6.5. خطاهای رایج
- عدم مطالعه راهنمای مشارکت پروژه.
- نادیده گرفتن بازخورد در Pull Requestها.

---

## 7. بهترین شیوه‌ها و نکات کلی

- **سادگی**: کد را ساده و خوانا نگه دارید؛ از پیچیدگی غیرضروری اجتناب کنید.
- **تست‌نویسی**: برای هر تغییر، تست بنویسید:
  ```go
  go test -cover
  ```
- **پروفایلینگ منظم**: از `pprof` برای شناسایی گلوگاه‌ها استفاده کنید.
- **مستندسازی مداوم**: مستندات را هم‌زمان با کد به‌روزرسانی کنید.
- **انجمن**: فعالانه در انجمن مشارکت کنید و از بازخورد دیگران یاد بگیرید.

---

## 8. نتیجه‌گیری

این جزوه تمام جنبه‌های بهترین شیوه‌ها و نکات پیشرفته در Go را با جزئیات کامل پوشش داد. از طراحی کد با SOLID و معماری تمیز تا مدیریت خطاها، الگوهای طراحی، بهینه‌سازی، مستندسازی، و مشارکت در انجمن، هر بخش با مثال‌های عملی و نکات پیشرفته ارائه شد. برای یادگیری عمیق‌تر:
- مستندات رسمی: `https://golang.org/doc/`
- کتاب "The Go Programming Language"
- مشارکت در پروژه‌های متن‌باز و مطالعه کد دیگران