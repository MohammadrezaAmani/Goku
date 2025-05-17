# برنامه‌نویسی پیشرفته در زبان Go

این جزوه به بررسی جنبه‌های پیشرفته برنامه‌نویسی در Go می‌پردازد و شامل موضوعات زیر است:
1. بسته‌ها و ماژول‌ها
2. مدیریت وابستگی‌ها
3. پکیج‌های استاندارد
4. کار با JSON و فرمت‌های داده
5. فایل‌ها و I/O
6. تست‌نویسی

هر بخش با توضیحات مفهومی، مثال‌های عملی، نکات پیشرفته، و بهترین شیوه‌ها همراه است.

---

## 1. بسته‌ها و ماژول‌ها

بسته‌ها (Packages) در Go واحدهای سازمان‌دهی کد هستند که امکان استفاده مجدد و ماژولاریتی را فراهم می‌کنند. ماژول‌ها (Modules) ساختار مدرن‌تری برای مدیریت پروژه‌ها و وابستگی‌ها ارائه می‌دهند.

### 1.1. ایجاد بسته‌های سفارشی

#### تعریف
یک بسته مجموعه‌ای از فایل‌های Go است که در یک دایرکتوری قرار دارند و نام بسته در ابتدای هر فایل مشخص می‌شود (معمولاً نام دایرکتوری).

#### مراحل ایجاد
1. **ساختار دایرکتوری**:
   ```bash
   mkdir mypackage
   cd mypackage
   touch mypackage.go
   ```
2. **نوشتن کد بسته**:
   ```go
   // mypackage/mypackage.go
   package mypackage

   import "fmt"

   // Hello یک تابع عمومی است (با حرف بزرگ شروع می‌شود)
   func Hello(name string) string {
       return fmt.Sprintf("Hello, %s!", name)
   }

   // تابع خصوصی (با حرف کوچک شروع می‌شود)
   func secret() string {
       return "This is private"
   }
   ```
3. **استفاده از بسته**:
   ```go
   // main.go
   package main

   import (
       "example.com/mypackage"
       "fmt"
   )

   func main() {
       fmt.Println(mypackage.Hello("Ali")) // خروجی: Hello, Ali!
   }
   ```

#### نکات
- **نام‌گذاری**: نام بسته باید کوتاه، معنادار، و مطابق با نام دایرکتوری باشد.
- **دسترسی**: متغیرها، توابع، و انواع با حرف بزرگ (مثل `Hello`) عمومی هستند؛ با حرف کوچک (مثل `secret`) خصوصی‌اند.
- **ابتدا ماژول**:
   ```bash
   go mod init example.com/myapp
   ```

### 1.2. مستندسازی

#### مستندات با godoc
Go از مستندات درون‌خطی پشتیبانی می‌کند که با ابزار `godoc` قابل مشاهده است.
- **نوشتن مستندات**:
  ```go
  // mypackage/mypackage.go
  package mypackage

  // Hello یک پیام خوش‌آمدگویی برای نام داده‌شده تولید می‌کند.
  func Hello(name string) string {
      return fmt.Sprintf("Hello, %s!", name)
  }
  ```
- **مشاهده مستندات**:
  ```bash
  go doc example.com/mypackage
  ```
  یا سرور مستندات:
  ```bash
  godoc -http=:6060
  ```
  سپس به `http://localhost:6060/pkg/example.com/mypackage` بروید.

#### بهترین شیوه‌ها
- مستندات را قبل از تعریف تابع یا نوع بنویسید.
- از جملات کامل و توضیحات واضح استفاده کنید.
- برای مثال‌ها از `// Example` استفاده کنید:
  ```go
  // ExampleHello یک نمونه از استفاده تابع Hello است.
  func ExampleHello() {
      fmt.Println(Hello("Ali"))
      // Output: Hello, Ali!
  }
  ```

### 1.3. انتشار بسته

#### مراحل
1. **مخزن Git**:
   - کد را در یک مخزن عمومی (مثل GitHub) قرار دهید: `github.com/username/mypackage`.
2. **تگ نسخه**:
   ```bash
   git tag v1.0.0
   git push origin v1.0.0
   ```
3. **ماژول Go**:
   - فایل `go.mod` را تنظیم کنید:
     ```go
     module github.com/username/mypackage
     go 1.20
     ```
4. **استفاده توسط دیگران**:
   ```bash
   go get github.com/username/mypackage@v1.0.0
   ```

#### نکات پیشرفته
- **نسخه‌بندی معنایی**: از فرمت `vX.Y.Z` استفاده کنید.
- **ماژول‌های چندبخشی**: برای بسته‌های بزرگ، دایرکتوری‌هایی مثل `internal` و زیرماژول‌ها ایجاد کنید.
- **انتشار مستندات**: از `pkg.go.dev` برای نمایش خودکار مستندات استفاده کنید.

#### خطاهای رایج
- عدم تگ نسخه: باعث خطای `no matching versions` می‌شود.
- نام‌گذاری ناسازگار بین `go.mod` و مسیر مخزن.

---

## 2. مدیریت وابستگی‌ها

Go از سیستم ماژول‌ها برای مدیریت وابستگی‌ها استفاده می‌کند که با `go mod` کنترل می‌شود.

### 2.1. استفاده از go mod

#### ابتدایی کردن ماژول
```bash
go mod init example.com/myapp
```
این دستور فایل `go.mod` را ایجاد می‌کند:
```go
module example.com/myapp
go 1.20
```

#### افزودن وابستگی
برای استفاده از یک بسته خارجی:
```bash
go get github.com/some/package@v1.2.3
```
این کار وابستگی را به `go.mod` اضافه می‌کند:
```go
require github.com/some/package v1.2.3
```

#### به‌روزرسانی
```bash
go get -u github.com/some/package
```

#### حذف وابستگی‌های استفاده‌نشده
```bash
go mod tidy
```

### 2.2. Vendoring

#### تعریف
Vendoring کپی کدهای وابستگی‌ها را در پروژه ذخیره می‌کند تا بدون دسترسی به اینترنت قابل ساخت باشد.

#### فعال‌سازی
```bash
go mod vendor
```
این کار دایرکتوری `vendor` را ایجاد می‌کند.

#### ساخت با vendor
```bash
go build -mod=vendor
```

#### نکات
- Vendoring برای پروژه‌های قدیمی یا محیط‌های آفلاین مفید است.
- در Go مدرن کمتر استفاده می‌شود، زیرا `go mod` بهینه‌تر است.

### 2.3. نسخه‌بندی

#### نسخه‌بندی معنایی
- Go از نسخه‌بندی معنایی (Semantic Versioning) پشتیبانی می‌کند: `vX.Y.Z`.
- **عمده (X)**: تغییرات ناسازگار.
- **جزئی (Y)**: ویژگی‌های جدید.
- **رفع باگ (Z)**: اصلاحات.

#### مدیریت نسخه‌ها
- مشخص کردن حداقل نسخه:
  ```go
  require github.com/some/package v1.2.0
  ```
- استفاده از نسخه خاص:
  ```bash
  go get github.com/some/package@v1.2.3
  ```

#### نکات پیشرفته
- **ماژول‌های جایگزین**:
  ```go
  replace github.com/some/package => ../local/package
  ```
- **پروکسی ماژول**: از `GOPROXY` برای دانلود سریع‌تر استفاده کنید:
  ```bash
  export GOPROXY=https://proxy.golang.org
  ```

#### خطاهای رایج
- عدم به‌روزرسانی `go.mod` با `go mod tidy`.
- استفاده از نسخه‌های ناسازگار.

---

## 3. پکیج‌های استاندارد

کتابخانه استاندارد Go شامل پکیج‌های قدرتمندی است که برای کارهای روزمره استفاده می‌شوند.

### 3.1. پکیج fmt
برای فرمت‌دهی و چاپ استفاده می‌شود.
- **چاپ**:
  ```go
  fmt.Println("Hello") // چاپ ساده
  fmt.Printf("Name: %s, Age: %d\n", "Ali", 30) // فرمت‌دهی
  ```
- **خواندن ورودی**:
  ```go
  var name string
  fmt.Scan(&name)
  ```

### 3.2. پکیج strings
برای عملیات روی رشته‌ها:
- **جستجو**:
  ```go
  s := "Hello, World"
  fmt.Println(strings.Contains(s, "World")) // true
  ```
- **جایگزینی**:
  ```go
  fmt.Println(strings.ReplaceAll(s, "World", "Go")) // Hello, Go
  ```

### 3.3. پکیج time
برای کار با زمان و تاریخ:
- **زمان فعلی**:
  ```go
  now := time.Now()
  fmt.Println(now) // 2025-05-17 16:53:00 +0000 UTC
  ```
- **تأخیر**:
  ```go
  time.Sleep(2 * time.Second)
  ```
- **فرمت‌دهی**:
  ```go
  fmt.Println(now.Format("2006-01-02")) // 2025-05-17
  ```

### 3.4. پکیج math
برای عملیات ریاضی:
- **توابع پایه**:
  ```go
  fmt.Println(math.Sqrt(16)) // 4
  fmt.Println(math.Pow(2, 3)) // 8
  ```
- **ثابت‌ها**:
  ```go
  fmt.Println(math.Pi) // 3.141592653589793
  ```

### 3.5. مثال ترکیبی
```go
package main

import (
    "fmt"
    "math"
    "strings"
    "time"
)

func main() {
    name := "Ali"
    fmt.Printf("Hello, %s!\n", strings.ToUpper(name))

    t := time.Now()
    fmt.Println("Current time:", t.Format("2006-01-02 15:04"))

    fmt.Println("Square root of 16:", math.Sqrt(16))
}
```

#### نکات پیشرفته
- **کارایی**: از `strings.Builder` به جای `strings.Replace` برای تغییرات مکرر استفاده کنید.
- **دقت زمان**: برای اندازه‌گیری عملکرد از `time.Since` استفاده کنید:
  ```go
  start := time.Now()
  // عملیات
  fmt.Println(time.Since(start))
  ```

#### خطاهای رایج
- فرمت نادرست زمان (باید از `2006-01-02` استفاده شود).
- استفاده از `fmt.Scan` بدون مدیریت خطا.

---

## 4. کار با JSON و فرمت‌های داده

Go از فرمت‌های داده مانند JSON، YAML، و XML پشتیبانی می‌کند.

### 4.1. JSON

#### Marshal/Unmarshal
- **Marshal** (تبدیل struct به JSON):
  ```go
  type Person struct {
      Name string `json:"name"`
      Age  int    `json:"age"`
  }

  p := Person{Name: "Ali", Age: 30}
  data, err := json.Marshal(p)
  if err != nil {
      log.Fatal(err)
  }
  fmt.Println(string(data)) // {"name":"Ali","age":30}
  ```
- **Unmarshal** (تبدیل JSON به struct):
  ```go
  jsonStr := `{"name":"Ali","age":30}`
  var p Person
  err := json.Unmarshal([]byte(jsonStr), &p)
  if err != nil {
      log.Fatal(err)
  }
  fmt.Println(p) // {Ali 30}
  ```

#### نکات
- تگ‌های `json` برای کنترل نام فیلدها و رفتار (مثل `omitempty`) استفاده می‌شوند:
  ```go
  type Person struct {
      Name string `json:"name,omitempty"`
      Age  int    `json:"age"`
  }
  ```
- برای JSON‌های پویا از `map[string]interface{}` یا `[]interface{}` استفاده کنید.

### 4.2. YAML
برای YAML نیاز به بسته خارجی (مثل `gopkg.in/yaml.v3`) دارید:
```bash
go get gopkg.in/yaml.v3
```
- **مثال**:
  ```go
  type Config struct {
      Name string `yaml:"name"`
      Age  int    `yaml:"age"`
  }

  yamlStr := "name: Ali\nage: 30"
  var c Config
  err := yaml.Unmarshal([]byte(yamlStr), &c)
  if err != nil {
      log.Fatal(err)
  }
  fmt.Println(c) // {Ali 30}
  ```

### 4.3. XML
مشابه JSON، با پکیج `encoding/xml`:
```go
type Person struct {
    XMLName xml.Name `xml:"person"`
    Name    string   `xml:"name"`
    Age     int      `xml:"age"`
}

xmlStr := `<person><name>Ali</name><age>30</age></person>`
var p Person
err := xml.Unmarshal([]byte(xmlStr), &p)
fmt.Println(p) // {Ali 30}
```

#### نکات پیشرفته
- **Streaming**: برای فایل‌های JSON/XML بزرگ، از `json.Decoder` و `xml.Decoder` استفاده کنید:
  ```go
  dec := json.NewDecoder(strings.NewReader(jsonStr))
  var p Person
  dec.Decode(&p)
  ```
- **مدیریت خطا**: همیشه خطاهای Marshal/Unmarshal را بررسی کنید.

#### خطاهای رایج
- عدم تطابق تگ‌ها با ساختار داده.
- استفاده از اشاره‌گر نادرست در Unmarshal.

---

## 5. فایل‌ها و I/O

Go ابزارهای قدرتمندی برای کار با فایل‌ها و عملیات ورودی/خروجی ارائه می‌دهد.

### 5.1. خواندن/نوشتن فایل

#### نوشتن
```go
data := []byte("Hello, Go!")
err := os.WriteFile("output.txt", data, 0644)
if err != nil {
    log.Fatal(err)
}
```

#### خواندن
```go
data, err := os.ReadFile("output.txt")
if err != nil {
    log.Fatal(err)
}
fmt.Println(string(data)) // Hello, Go!
```

### 5.2. کار با دایرکتوری‌ها
- **ایجاد**:
  ```go
  err := os.Mkdir("mydir", 0755)
  ```
- **لیست کردن**:
  ```go
  entries, err := os.ReadDir("mydir")
  for _, entry := range entries {
      fmt.Println(entry.Name())
  }
  ```

### 5.3. Buffering
برای بهبود کارایی از `bufio` استفاده کنید:
- **خواندن**:
  ```go
  file, err := os.Open("input.txt")
  if err != nil {
      log.Fatal(err)
  }
  defer file.Close()

  scanner := bufio.NewScanner(file)
  for scanner.Scan() {
      fmt.Println(scanner.Text())
  }
  ```
- **نوشتن**:
  ```go
  file, err := os.Create("output.txt")
  writer := bufio.NewWriter(file)
  writer.WriteString("Hello, Go!")
  writer.Flush() // اطمینان از نوشتن
  ```

#### نکات پیشرفته
- **مدیریت منابع**: همیشه از `defer file.Close()` استفاده کنید.
- **فایل‌های بزرگ**: برای خواندن/نوشتن تدریجی از `io.Reader` و `io.Writer` استفاده کنید.
- **همزمانی**: برای عملیات I/O سنگین، از goroutine‌ها استفاده کنید.

#### خطاهای رایج
- عدم بستن فایل‌ها.
- نادیده گرفتن خطاهای I/O.

---

## 6. تست‌نویسی

Go ابزارهای داخلی قدرتمندی برای تست‌نویسی ارائه می‌دهد.

### 6.1. استفاده از پکیج testing
- **تست ساده**:
  ```go
  // math.go
  package math

  func Add(a, b int) int {
      return a + b
  }

  // math_test.go
  package math

  import "testing"

  func TestAdd(t *testing.T) {
      result := Add(2, 3)
      if result != 5 {
          t.Errorf("Add(2, 3) = %d; want 5", result)
      }
  }
  ```
- اجرا:
  ```bash
  go test
  ```

### 6.2. تست‌های جدولی
برای تست چندین سناریو:
```go
func TestAdd(t *testing.T) {
    tests := []struct {
        a, b, want int
    }{
        {2, 3, 5},
        {0, 0, 0},
        {-1, 1, 0},
    }

    for _, tt := range tests {
        t.Run(fmt.Sprintf("%d+%d", tt.a, tt.b), func(t *testing.T) {
            if got := Add(tt.a, tt.b); got != tt.want {
                t.Errorf("Add(%d, %d) = %d; want %d", tt.a, tt.b, got, tt.want)
            }
        })
    }
}
```

### 6.3. بنچمارک
برای اندازه‌گیری عملکرد:
```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(2, 3)
    }
}
```
اجرا:
```bash
go test -bench=.
```

#### نکات پیشرفته
- **Mocking**: از بسته‌های مثل `testify` برای mock استفاده کنید:
  ```bash
  go get github.com/stretchr/testify
  ```
- **Coverage**:
  ```bash
  go test -cover
  ```
- **تست‌های همزمانی**: از `sync.WaitGroup` برای هماهنگی استفاده کنید.

#### خطاهای رایج
- نادیده گرفتن خطاها در تست‌ها.
- نوشتن تست‌های غیرقابل نگهداری.

---

## 7. بهترین شیوه‌ها و نکات

- **سازمان‌دهی کد**: بسته‌ها را بر اساس مسئولیت تقسیم کنید (مثل `api`, `db`, `utils`).
- **مدیریت وابستگی‌ها**: همیشه `go mod tidy` را اجرا کنید.
- **استفاده بهینه از کتابخانه استاندارد**: به جای بسته‌های خارجی، ابتدا پکیج‌های استاندارد را بررسی کنید.
- **تست‌نویسی منظم**: برای هر تغییر، تست جدید بنویسید.
- **مستندسازی**: مستندات را به‌روز نگه دارید و از مثال‌های قابل اجرا استفاده کنید.

---

## 8. نتیجه‌گیری

این جزوه تمام جنبه‌های برنامه‌نویسی پیشرفته در Go را با جزئیات کامل پوشش داد. از ایجاد بسته‌ها و مدیریت وابستگی‌ها تا کار با فرمت‌های داده، فایل‌ها، و تست‌نویسی، هر بخش با مثال‌های عملی و نکات پیشرفته ارائه شد. برای یادگیری عمیق‌تر:
- مستندات رسمی: `https://golang.org/doc/`
- کتاب "The Go Programming Language"
- تمرین با پروژه‌های واقعی مثل ساخت API یا ابزار CLI