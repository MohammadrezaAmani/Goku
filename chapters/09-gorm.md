# آموزش کامل، پیشرفته و عمیق GORM در Go

این جزوه به بررسی جامع و تخصصی **GORM**، کتابخانه ORM محبوب برای Go، می‌پردازد. GORM به دلیل انعطاف‌پذیری، پشتیبانی از دیتابیس‌های مختلف، و قابلیت‌های پیشرفته، یکی از بهترین ابزارها برای مدیریت دیتابیس در Go است. تمام موضوعات درخواستی با جزئیات کامل، مثال‌های عملی، و نکات پیشرفته شرح داده شده‌اند.

---

## 1. مفاهیم پایه GORM (به‌صورت حرفه‌ای)

### 1.1. مدل‌سازی دقیق دیتابیس
مدل‌ها در GORM به صورت struct تعریف می‌شوند و با استفاده از برچسب‌ها (tags) به جداول دیتابیس نگاشت می‌شوند.

#### مثال مدل پیشرفته
```go
package models

import (
    "time"
    "gorm.io/gorm"
)

// User مدل کاربر با فیلدهای مختلف
type User struct {
    ID        uint           `gorm:"primaryKey"`
    Name      string         `gorm:"type:varchar(100);not null"`
    Email     string         `gorm:"type:varchar(100);uniqueIndex"`
    CreatedAt time.Time      `gorm:"autoCreateTime"`
    UpdatedAt time.Time      `gorm:"autoUpdateTime"`
    DeletedAt gorm.DeletedAt `gorm:"index"` // برای Soft Delete
}
```

- **توضیحات**:
  - `primaryKey`: کلید اصلی.
  - `type:varchar(100)`: نوع ستون در دیتابیس.
  - `uniqueIndex`: ایندکس منحصربه‌فرد.
  - `autoCreateTime` و `autoUpdateTime`: به‌روزرسانی خودکار زمان.
  - `gorm.DeletedAt`: پشتیبانی از Soft Delete.

### 1.2. تعریف انواع فیلدها
GORM از انواع داده‌های سفارشی، JSON، و Enum پشتیبانی می‌کند.

#### Custom Type
```go
package models

import (
    "database/sql/driver"
    "encoding/json"
    "errors"
    "gorm.io/gorm"
)

type JSONMap map[string]interface{}

// Value برای تبدیل به دیتابیس
func (j JSONMap) Value() (driver.Value, error) {
    return json.Marshal(j)
}

// Scan برای خواندن از دیتابیس
func (j *JSONMap) Scan(value interface{}) error {
    bytes, ok := value.([]byte)
    if !ok {
        return errors.New("type assertion to []byte failed")
    }
    return json.Unmarshal(bytes, j)
}

type Profile struct {
    ID       uint     `gorm:"primaryKey"`
    UserID   uint     `gorm:"index"`
    Settings JSONMap  `gorm:"type:jsonb"` // برای Postgres
}
```

#### Enum
```go
package models

import "gorm.io/gorm"

type Role string

const (
    Admin  Role = "admin"
    User   Role = "user"
    Guest  Role = "guest"
)

type User struct {
    ID   uint `gorm:"primaryKey"`
    Role Role `gorm:"type:varchar(20);default:'user'"`
}
```

### 1.3. استفاده از برچسب‌ها
برچسب‌ها (tags) برای کنترل دقیق رفتار مدل‌ها استفاده می‌شوند.

#### مثال
```go
type Order struct {
    ID         uint   `gorm:"primaryKey"`
    UserID     uint   `gorm:"index;foreignKey:UserID;references:ID"` // کلید خارجی
    Amount     int    `gorm:"not null;check:amount >= 0"`           // شرط
    Status     string `gorm:"type:varchar(50);default:'pending'"`
    UniqueCode string `gorm:"uniqueIndex:idx_code"`                 // ایندکس منحصربه‌فرد
}
```

- **foreignKey**: تعریف کلید خارجی.
- **check**: شرط در سطح دیتابیس.
- **uniqueIndex**: ایندکس منحصربه‌فرد با نام خاص.

### 1.4. Embedded Structs و Inheritance
GORM از structهای تو در تو برای مدل‌سازی روابط پیچیده پشتیبانی می‌کند.

#### مثال
```go
type BaseModel struct {
    CreatedAt time.Time      `gorm:"autoCreateTime"`
    UpdatedAt time.Time      `gorm:"autoUpdateTime"`
    DeletedAt gorm.DeletedAt `gorm:"index"`
}

type User struct {
    BaseModel
    ID    uint   `gorm:"primaryKey"`
    Name  string `gorm:"type:varchar(100)"`
}

type Admin struct {
    User
    Permissions JSONMap `gorm:"type:jsonb"`
}
```

- **توضیحات**:
  - `BaseModel` فیلدهای مشترک را تعریف می‌کند.
  - `Admin` از `User` ارث‌بری می‌کند و فیلدهای اضافی دارد.

### 1.5. نکات پیشرفته
- **Custom Table Names**:
  ```go
  func (User) TableName() string {
      return "app_users"
  }
  ```
- **Validation**: از پکیج `validator` برای اعتبارسنجی مدل‌ها استفاده کنید:
  ```go
  type User struct {
      Name string `gorm:"type:varchar(100)" validate:"required,min=3"`
  }
  ```
- **Polymorphic Relations**:
  ```go
  type Comment struct {
      ID           uint
      Content      string
      CommentableID uint   `gorm:"index"`
      CommentableType string `gorm:"index"`
  }
  ```

### 1.6. خطاهای رایج
- عدم تعریف ایندکس برای فیلدهای پرکاربرد.
- استفاده نادرست از برچسب‌ها که باعث ناسازگاری با دیتابیس می‌شود.

---

## 2. عملیات پیشرفته CRUD

### 2.1. Query پیشرفته با Conditions پیچیده
GORM از شرط‌های پیچیده با روش‌های مختلف پشتیبانی می‌کند.

#### مثال
```go
var users []User
db.Where("name LIKE ? AND role = ?", "%Ali%", "admin").
   Or("email LIKE ?", "%@example.com").
   Find(&users)
```

- **Map Conditions**:
  ```go
  db.Where(map[string]interface{}{
      "role": "admin",
      "age":  30,
  }).Find(&users)
  ```

- **Struct Conditions**:
  ```go
  db.Where(&User{Role: "admin"}).Find(&users)
  ```

### 2.2. Dynamic Filters و Query Builder
برای ساخت کوئری‌های پویا از `Scopes` استفاده کنید.

#### مثال
```go
func WithName(name string) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        if name != "" {
            return db.Where("name LIKE ?", "%"+name+"%")
        }
        return db
    }
}

func WithRole(role string) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        if role != "" {
            return db.Where("role = ?", role)
        }
        return db
    }
}

var users []User
db.Scopes(WithName("Ali"), WithRole("admin")).Find(&users)
```

### 2.3. کار با Preload و Joins
#### Preload
برای بارگذاری روابط:
```go
type User struct {
    ID      uint
    Name    string
    Orders  []Order `gorm:"foreignKey:UserID"`
}

var user User
db.Preload("Orders").First(&user, 1)
```

#### Joins
برای کوئری‌های پیچیده:
```go
var results []struct {
    UserName  string
    OrderID   uint
}
db.Model(&User{}).
   Select("users.name AS user_name, orders.id AS order_id").
   Joins("LEFT JOIN orders ON orders.user_id = users.id").
   Scan(&results)
```

### 2.4. Bulk Insert و Batch Processing
#### Bulk Insert
```go
users := []User{
    {Name: "Ali", Email: "ali@example.com"},
    {Name: "Bob", Email: "bob@example.com"},
}
db.CreateInBatches(users, 100) // دسته‌های 100 تایی
```

#### Batch Processing
```go
db.Where("status = ?", "pending").FindInBatches(&users, 100, func(tx *gorm.DB, batch int) error {
    for i := range users {
        users[i].Status = "processed"
    }
    return tx.Save(&users).Error
})
```

### 2.5. کار با Soft Delete
GORM به طور داخلی از Soft Delete پشتیبانی می‌کند.
```go
// حذف نرم
db.Delete(&user)

// بازیابی با حذف‌شده‌ها
db.Unscoped().Find(&users)

// حذف دائم
db.Unscoped().Delete(&user)
```

### 2.6. Transactions، SavePoints و Rollback
#### Transaction
```go
err := db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&User{Name: "Ali"}).Error; err != nil {
        return err
    }
    if err := tx.Create(&Order{UserID: 1, Amount: 100}).Error; err != nil {
        return err
    }
    return nil
})
```

#### SavePoints
```go
db.Transaction(func(tx *gorm.DB) error {
    tx.Create(&User{Name: "Ali"})
    tx.SavePoint("sp1")
    tx.Create(&Order{UserID: 1, Amount: 100})
    if someCondition {
        tx.RollbackTo("sp1")
    }
    return nil
})
```

### 2.7. نکات پیشرفته
- **Smart Select**: فقط فیلدهای مورد نیاز را انتخاب کنید:
  ```go
  db.Select("id, name").Find(&users)
  ```
- **Raw SQL**: برای کوئری‌های پیچیده:
  ```go
  db.Raw("SELECT * FROM users WHERE age > ?", 30).Scan(&users)
  ```
- **Optimistic Locking**:
  ```go
  type User struct {
      ID      uint
      Version int `gorm:"default:1"`
  }
  db.Clauses(clause.Locking{Strength: "UPDATE"}).Find(&user)
  ```

### 2.8. خطاهای رایج
- عدم استفاده از `Preload` برای روابط که باعث N+1 Query Problem می‌شود.
- مدیریت نادرست تراکنش‌ها که باعث نشت منابع می‌شود.

---

## 3. معماری تمیز و استفاده از SOLID در لایه دیتابیس

### 3.1. Repository Pattern
Repository Pattern لایه‌ای بین منطق کسب‌وکار و دیتابیس ایجاد می‌کند.

#### مثال
- **Repository Interface**:
  ```go
  package repository

  import "github.com/username/app/models"

  type UserRepository interface {
      Create(user *models.User) error
      FindByID(id uint) (*models.User, error)
      FindAll() ([]models.User, error)
  }
  ```

- **GORM Repository**:
  ```go
  package repository

  import (
      "gorm.io/gorm"
      "github.com/username/app/models"
  )

  type GORMUserRepository struct {
      db *gorm.DB
  }

  func NewGORMUserRepository(db *gorm.DB) *GORMUserRepository {
      return &GORMUserRepository{db: db}
  }

  func (r *GORMUserRepository) Create(user *models.User) error {
      return r.db.Create(user).Error
  }

  func (r *GORMUserRepository) FindByID(id uint) (*models.User, error) {
      var user models.User
      err := r.db.First(&user, id).Error
      return &user, err
  }

  func (r *GORMUserRepository) FindAll() ([]models.User, error) {
      var users []models.User
      err := r.db.Find(&users).Error
      return users, err
  }
  ```

- **Service Layer**:
  ```go
  package service

  import (
      "github.com/username/app/models"
      "github.com/username/app/repository"
  )

  type UserService struct {
      repo repository.UserRepository
  }

  func NewUserService(repo repository.UserRepository) *UserService {
      return &UserService{repo: repo}
  }

  func (s *UserService) CreateUser(name string) error {
      user := &models.User{Name: name}
      return s.repo.Create(user)
  }
  ```

### 3.2. Interface Segregation و Dependency Injection
- رابط‌ها را کوچک و متمرکز نگه دارید:
  ```go
  type UserReader interface {
      FindByID(id uint) (*models.User, error)
  }

  type UserWriter interface {
      Create(user *models.User) error
  }
  ```
- تزریق وابستگی:
  ```go
  func main() {
      db, _ := gorm.Open(postgres.Open("..."), &gorm.Config{})
      repo := repository.NewGORMUserRepository(db)
      service := service.NewUserService(repo)
  }
  ```

### 3.3. جداسازی لایه مدل
برای تست‌پذیری، مدل‌ها را از دیتابیس جدا کنید:
```go
package models

type User struct {
    ID   uint
    Name string
}

// Validate برای اعتبارسنجی
func (u *User) Validate() error {
    if u.Name == "" {
        return errors.New("name is required")
    }
    return nil
}
```

### 3.4. نکات پیشرفته
- **Unit of Work**: برای مدیریت چندین Repository در یک تراکنش:
  ```go
  type UnitOfWork struct {
      db      *gorm.DB
      UserRepo repository.UserRepository
  }

  func (u *UnitOfWork) Commit() error {
      return u.db.Commit().Error
  }
  ```
- **CQRS**: برای جداسازی خواندن و نوشتن:
  ```go
  type UserQueryService struct {
      db *gorm.DB
  }

  func (s *UserQueryService) FindByName(name string) ([]models.User, error) {
      var users []models.User
      return users, s.db.Where("name LIKE ?", "%"+name+"%").Find(&users).Error
  }
  ```

### 3.5. خطاهای رایج
- وابستگی مستقیم به GORM در لایه سرویس.
- عدم تعریف رابط‌های کوچک برای Repository.

---

## 4. مدیریت پیشرفته مهاجرت (Migrations)

### 4.1. AutoMigrate vs Manual Migrations
- **AutoMigrate**:
  ```go
  db.AutoMigrate(&User{}, &Order{})
  ```
  - مزایا: ساده برای پروژه‌های کوچک.
  - معایب: کنترل محدود، عدم پشتیبانی از تغییرات پیچیده.

- **Manual Migrations**:
  استفاده از ابزارهایی مثل `golang-migrate` یا `atlas`.

#### golang-migrate
- نصب:
  ```bash
  go get -u github.com/golang-migrate/migrate/v4
  ```
- فایل مهاجرت (migrations/000001_create_users.up.sql):
  ```sql
  CREATE TABLE users (
      id SERIAL PRIMARY KEY,
      name VARCHAR(100) NOT NULL,
      email VARCHAR(100) UNIQUE NOT NULL
  );
  ```
- فایل Rollback (migrations/000001_create_users.down.sql):
  ```sql
  DROP TABLE users;
  ```

- اجرا:
  ```go
  package main

  import (
      "github.com/golang-migrate/migrate/v4"
      _ "github.com/golang-migrate/migrate/v4/database/postgres"
      _ "github.com/golang-migrate/migrate/v4/source/file"
  )

  func main() {
      m, err := migrate.New("file://migrations", "postgres://user:secret@localhost:5432/mydb?sslmode=disable")
      if err != nil {
          log.Fatal(err)
      }
      if err := m.Up(); err != nil {
          log.Fatal(err)
      }
  }
  ```

### 4.2. نسخه‌بندی دیتابیس و CI/CD
- **نسخه‌بندی**:
  - هر مهاجرت را با timestamp نام‌گذاری کنید (مثل `202505170001_create_users.sql`).
  - از ابزارهایی مثل `atlas` برای بررسی تفاوت‌های اسکیما:
    ```bash
    atlas schema diff --from postgres://... --to file://schema.sql
    ```
- **CI/CD**:
  ```yaml
  name: Database Migration
  on:
    push:
      branches: [main]
  jobs:
    migrate:
      runs-on: ubuntu-latest
      steps:
      - uses: actions/checkout@v4
      - name: Run migrations
        run: |
          go install github.com/golang-migrate/migrate/v4/cmd/migrate@latest
          migrate -path migrations -database "postgres://..." up
  ```

### 4.3. نکات پیشرفته
- **Migration Testing**:
  ```go
  func TestMigration(t *testing.T) {
      m, _ := migrate.New("file://migrations", "postgres://...")
      if err := m.Up(); err != nil {
          t.Fatal(err)
      }
      defer m.Down()
      // تست مدل‌ها
  }
  ```
- **Schema Rollback**:
  ```go
  if err := m.Steps(-1); err != nil {
      log.Fatal(err)
  }
  ```

### 4.4. خطاهای رایج
- عدم تعریف فایل‌های Rollback.
- اجرای مهاجرت‌ها بدون تست در محیط توسعه.

---

## 5. کار با دیتابیس‌های مختلف

### 5.1. استفاده از ویژگی‌های خاص هر دیتابیس
- **Postgres (JSONB, Full-Text Search)**:
  ```go
  db.Where("settings->>'key' = ?", "value").Find(&profiles)
  db.Raw("SELECT * FROM users WHERE to_tsvector(name) @@ to_tsquery(?)", "Ali").Scan(&users)
  ```
- **MySQL (InnoDB, Partitioning)**:
  ```go
  db.Set("gorm:table_options", "ENGINE=InnoDB").AutoMigrate(&User{})
  ```
- **SQLite**:
  ```go
  db, _ := gorm.Open(sqlite.Open("test.db"), &gorm.Config{})
  ```

### 5.2. Connection Pooling، Timeouts، و Failover
- **Connection Pooling**:
  ```go
  sqlDB, _ := db.DB()
  sqlDB.SetMaxOpenConns(100)
  sqlDB.SetMaxIdleConns(10)
  sqlDB.SetConnMaxLifetime(time.Hour)
  ```
- **Timeouts**:
  ```go
  db, _ := gorm.Open(postgres.New(postgres.Config{
      DSN: "user=postgres password=secret dbname=mydb sslmode=disable",
  }), &gorm.Config{
      QueryFields: true,
      ConnMaxLifetime: 30 * time.Minute,
  })
  ```
- **Failover**:
  استفاده از ابزارهای خارجی مثل `PgBouncer` یا تنظیمات Read Replica:
  ```go
  db, _ := gorm.Open(postgres.New(postgres.Config{
      DSN: "host=primary,secondary user=postgres password=secret dbname=mydb",
  }))
  ```

### 5.3. پشتیبانی از Replication و Sharding
- **Replication**:
  ```go
  readDB, _ := gorm.Open(postgres.New(postgres.Config{DSN: "host=read-replica ..."}))
  writeDB, _ := gorm.Open(postgres.New(postgres.Config{DSN: "host=primary ..."}))
  ```
- **Sharding**:
  با استفاده از `TableName` پویا:
  ```go
  func (User) TableName() string {
      return fmt.Sprintf("users_shard_%d", userID%10)
  }
  ```

### 5.4. نکات پیشرفته
- **Multi-Tenancy**:
  ```go
  db.Set("gorm:table_options", "SCHEMA tenant_1").AutoMigrate(&User{})
  ```
- **Connection Health Check**:
  ```go
  sqlDB, _ := db.DB()
  if err := sqlDB.Ping(); err != nil {
      log.Fatal("Database unreachable")
  }
  ```

### 5.5. خطاهای رایج
- تنظیم نادرست Connection Pool که باعث اتمام اتصال‌ها می‌شود.
- عدم پشتیبانی از ویژگی‌های خاص دیتابیس.

---

## 6. بهینه‌سازی و Performance Tuning

### 6.1. Index Optimization
- ایندکس برای فیلدهای پرجستجو:
  ```go
  type User struct {
      ID    uint   `gorm:"primaryKey"`
      Email string `gorm:"uniqueIndex"`
      Name  string `gorm:"index:idx_name"`
  }
  ```
- ایندکس مرکب:
  ```go
  type Order struct {
      UserID uint `gorm:"index:idx_user_status,priority:1"`
      Status string `gorm:"index:idx_user_status,priority:2"`
  }
  ```

### 6.2. Explain Analyze روی Queryهای GORM
- فعال‌سازی Logger برای بررسی کوئری‌ها:
  ```go
  db, _ := gorm.Open(postgres.Open("..."), &gorm.Config{
      Logger: logger.Default.LogMode(logger.Info),
  })
  ```
- تحلیل با EXPLAIN:
  ```go
  var explain string
  db.Raw("EXPLAIN ANALYZE SELECT * FROM users WHERE name = ?", "Ali").Scan(&explain)
  fmt.Println(explain)
  ```

### 6.3. Logger سفارشی
```go
type CustomLogger struct {
    logger.Logger
}

func (l *CustomLogger) LogMode(level logger.LogLevel) logger.Interface {
    newLogger := *l
    newLogger.Logger = l.Logger.LogMode(level)
    return &newLogger
}

func (l *CustomLogger) Info(ctx context.Context, msg string, data ...interface{}) {
    log.Printf("[GORM] %s %v", msg, data)
}

db, _ := gorm.Open(postgres.Open("..."), &gorm.Config{
    Logger: &CustomLogger{},
})
```

### 6.4. کش‌کردن Queryها
- استفاده از Redis:
  ```go
  import "github.com/go-redis/redis/v8"

  func GetUser(ctx context.Context, db *gorm.DB, client *redis.Client, id uint) (*User, error) {
      cacheKey := fmt.Sprintf("user:%d", id)
      cached, err := client.Get(ctx, cacheKey).Result()
      if err == nil {
          var user User
          json.Unmarshal([]byte(cached), &user)
          return &user, nil
      }
      var user User
      if err := db.First(&user, id).Error; err != nil {
          return nil, err
      }
      data, _ := json.Marshal(user)
      client.Set(ctx, cacheKey, data, 10*time.Minute)
      return &user, nil
  }
  ```

### 6.5. Query Profiling و Monitoring
- استفاده از Prometheus:
  ```go
  import "github.com/prometheus/client_golang/prometheus"

  var queryDuration = prometheus.NewHistogram(prometheus.HistogramOpts{
      Name: "gorm_query_duration_seconds",
      Help: "Duration of GORM queries",
  })

  func init() {
      prometheus.MustRegister(queryDuration)
  }

  db.Callback().Query().After("gorm:query").Register("prometheus", func(d *gorm.DB) {
      queryDuration.Observe(float64(d.Statement.SQL.String()))
  })
  ```

### 6.6. نکات پیشرفته
- **Batch Size Optimization**:
  ```go
  db.CreateInBatches(users, 500) // تست برای یافتن اندازه بهینه
  ```
- **Avoid N+1**: همیشه از `Preload` یا `Joins` استفاده کنید.
- **Query Hints**:
  ```go
  db.Set("gorm:query_option", "INDEX (idx_name)").Find(&users)
  ```

### 6.7. خطاهای رایج
- عدم تعریف ایندکس برای کوئری‌های سنگین.
- کش کردن داده‌های متغیر که باعث ناسازگاری می‌شود.

---

## 7. امنیت دیتابیس در GORM

### 7.1. جلوگیری از SQL Injection
همیشه از پارامترهای باندشده استفاده کنید:
```go
db.Where("name = ?", userInput).Find(&users) // ایمن
db.Raw("SELECT * FROM users WHERE name = '" + userInput + "'") // ناایمن
```

### 7.2. دسترسی‌ها و Authorization
- محدود کردن کوئری‌ها بر اساس کاربر:
  ```go
  func GetUserData(db *gorm.DB, userID uint) ([]Data, error) {
      var data []Data
      return data, db.Where("user_id = ?", userID).Find(&data).Error
  }
  ```
- استفاده از Row-Level Security در Postgres:
  ```sql
  ALTER TABLE users ENABLE ROW LEVEL SECURITY;
  CREATE POLICY user_access ON users USING (id = current_user_id());
  ```

### 7.3. Audit Trail و History Logs
```go
type AuditLog struct {
    ID        uint
    TableName string
    RecordID  uint
    Action    string
    Data      JSONMap `gorm:"type:jsonb"`
    CreatedAt time.Time
}

func LogAudit(db *gorm.DB, table string, recordID uint, action string, data interface{}) error {
    logEntry := AuditLog{
        TableName: table,
        RecordID:  recordID,
        Action:    action,
        Data:      JSONMap(data),
        CreatedAt: time.Now(),
    }
    return db.Create(&logEntry).Error
}
```

### 7.4. نکات پیشرفته
- **Encryption**:
  ```go
  type User struct {
      ID           uint
      EncryptedData string `gorm:"type:text"`
  }

  func Encrypt(data string) string {
      // استفاده از پکیج crypto
      return encrypted
  }
  ```
- **Prepared Statements**:
  ```go
  db.Set("gorm:prepare_stmt", true).Find(&users)
  ```

### 7.5. خطاهای رایج
- استفاده از Raw SQL بدون پارامترهای باندشده.
- عدم محدود کردن دسترسی‌ها در کوئری‌ها.

---

## 8. تست‌نویسی و Mock کردن دیتابیس

### 8.1. Unit Test با Mock GORM
استفاده از پکیج `go-sqlmock`:
```go
package repository

import (
    "testing"
    "github.com/DATA-DOG/go-sqlmock"
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

func TestGORMUserRepository_FindByID(t *testing.T) {
    db, mock, _ := sqlmock.New()
    gormDB, _ := gorm.Open(postgres.New(postgres.Config{Conn: db}), &gorm.Config{})
    repo := NewGORMUserRepository(gormDB)

    mock.ExpectQuery(`SELECT * FROM "users" WHERE id = \$1`).
        WithArgs(1).
        WillReturnRows(sqlmock.NewRows([]string{"id", "name"}).AddRow(1, "Ali"))

    user, err := repo.FindByID(1)
    if err != nil || user.Name != "Ali" {
        t.Errorf("Expected user Ali, got %v", user)
    }
}
```

### 8.2. Integration Test با دیتابیس واقعی و Docker
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
      db.AutoMigrate(&User{})
      repo := NewGORMUserRepository(db)

      user := &User{Name: "Ali"}
      repo.Create(user)

      found, _ := repo.FindByID(user.ID)
      if found.Name != "Ali" {
          t.Errorf("Expected Ali, got %s", found.Name)
      }
  }
  ```

### 8.3. تست Migration
```go
func TestMigration(t *testing.T) {
    db, _ := gorm.Open(sqlite.Open("file::memory:?cache=shared"), &gorm.Config{})
    m, _ := migrate.New("file://migrations", "sqlite://test.db")
    m.Up()
    db.AutoMigrate(&User{})
    // تست مدل‌ها
}
```

### 8.4. نکات پیشرفته
- **Test Fixtures**:
  ```go
  func SetupTestDB(t *testing.T) *gorm.DB {
      db, _ := gorm.Open(sqlite.Open("file::memory:"), &gorm.Config{})
      db.Create(&User{Name: "Ali"})
      return db
  }
  ```
- **Parallel Tests**:
  ```go
  t.Parallel()
  ```

### 8.5. خطاهای رایج
- عدم پاکسازی دیتابیس پس از تست.
- Mock نادرست کوئری‌ها.

---

## 9. مستندسازی و نگهداری

### 9.1. نوشتن کامنت‌های دقیق
```go
// Package models defines database models for the application.
// All models include standard fields for auditing and soft deletion.
package models

// User represents a user in the system.
// It includes fields for identification, authentication, and auditing.
type User struct {
    ID        uint           `gorm:"primaryKey" json:"id"`
    Name      string         `gorm:"type:varchar(100);not null" json:"name"`
    Email     string         `gorm:"type:varchar(100);uniqueIndex" json:"email"`
    CreatedAt time.Time      `gorm:"autoCreateTime" json:"created_at"`
    UpdatedAt time.Time      `gorm:"autoUpdateTime" json:"updated_at"`
    DeletedAt gorm.DeletedAt `gorm:"index" json:"-"` // Hidden in JSON
}
```

### 9.2. استفاده از godoc
- تولید مستندات:
  ```bash
  godoc -http=:6060
  ```
- مشاهده در `http://localhost:6060/pkg/github.com/username/app/models/`.

### 9.3. تولید ERD
استفاده از ابزارهایی مثل `dbml` یا `pgAdmin`:
- **DBML**:
  ```dbml
  Table users {
    id integer [primary key]
    name varchar(100)
    email varchar(100) [unique]
  }
  ```
- تبدیل به SVG:
  ```bash
  dbml2svg schema.dbml > schema.svg
  ```

### 9.4. نکات پیشرفته
- **Swagger برای API**:
  ```go
  // @Summary Create a new user
  // @Description Creates a user with name and email
  // @Tags Users
  // @Accept json
  // @Produce json
  // @Param user body User true "User data"
  // @Success 201 {object} User
  // @Router /users [post]
  func CreateUser(w http.ResponseWriter, r *http.Request) {}
  ```
- **Versioned Docs**:
  ```bash
  git tag v1.0.0
  go list -m github.com/username/app@v1.0.0
  ```

### 9.5. خطاهای رایج
- عدم به‌روزرسانی مستندات پس از تغییر مدل‌ها.
- مستندات ناقص برای روابط پیچیده.

---

## 10. انجمن و منابع پیشرفته یادگیری

### 10.1. مشارکت در پروژه‌های متن‌باز
- **GORM**:
  - مخزن: `https://github.com/go-gorm/gorm`
  - مشارکت: رفع باگ، افزودن ویژگی، یا بهبود مستندات.
- **پروژه‌های مرتبط**:
  - `go-gormigrate`: ابزار مهاجرت برای GORM.
  - `gorm-gen`: تولید کد خودکار برای GORM.
  - `entgo`: جایگزین ORM با رویکرد تولید کد.

### 10.2. مطالعه کد پروژه‌های معروف
- **gorm-gen**:
  ```go
  gen := gormgen.NewGenerator(gormgen.Config{
      OutPath: "./dal",
      Mode:    gormgen.FieldSignable,
  })
  gen.UseDB(db)
  gen.GenerateModel("users")
  ```
- **entgo**:
  ```go
  client, _ := ent.Open("postgres", "...")
  client.User.Create().SetName("Ali").Save(context.Background())
  ```

### 10.3. منابع یادگیری
- **مستندات رسمی**:
  - `https://gorm.io/docs/`
- **کتاب‌ها**:
  - "Building RESTful Web Services with Go"
- **انجمن‌ها**:
  - GORM Issues: `https://github.com/go-gorm/gorm/issues`
  - Gophers Slack: کانال `#gorm`
- **پروژه‌های عملی**:
  - ساخت API با GORM و Postgres.
  - پیاده‌سازی سیستم Audit Log.

### 10.4. نکات پیشرفته
- **Contribution Guide**:
  - مطالعه `CONTRIBUTING.md` پروژه.
  - نوشتن تست برای هر تغییر.
- **Blogging**: تجربیات خود را در مورد GORM منتشر کنید.

### 10.5. خطاهای رایج
- نادیده گرفتن راهنمای مشارکت پروژه.
- عدم تست تغییرات در محیط‌های مختلف.

---

## 11. بهترین شیوه‌ها و نکات کلی

- **سادگی**: از پیچیدگی غیرضروری در مدل‌ها و کوئری‌ها اجتناب کنید.
- **تست‌پذیری**: همیشه Repository Pattern را برای جداسازی منطق استفاده کنید.
- **بهینه‌سازی**:
  - ایندکس‌ها را برای کوئری‌های پرکاربرد تعریف کنید.
  - از کش برای کاهش بار دیتابیس استفاده کنید.
- **امنیت**:
  - همیشه از پارامترهای باندشده استفاده کنید.
  - دسترسی‌ها را در سطح کوئری محدود کنید.
- **مستندسازی**: مدل‌ها و کوئری‌ها را با کامنت‌های دقیق مستند کنید.

---

## 12. نتیجه‌گیری

این جزوه تمام جنبه‌های GORM را به صورت پیشرفته و عمیق پوشش داد. از مدل‌سازی حرفه‌ای و عملیات CRUD پیشرفته تا معماری تمیز، مهاجرت‌ها، بهینه‌سازی، امنیت، تست‌نویسی، مستندسازی، و مشارکت در انجمن، هر بخش با مثال‌های عملی و نکات تخصصی ارائه شد. برای یادگیری عمیق‌تر:
- مستندات رسمی: `https://gorm.io/docs/`
- مطالعه کد GORM: `https://github.com/go-gorm/gorm`
- تمرین با پروژه‌های واقعی مثل ساخت API یا سیستم مدیریت داده.