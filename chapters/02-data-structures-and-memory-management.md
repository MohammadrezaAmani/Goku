# داده‌ساختارها و مدیریت حافظه در زبان برنامه‌نویسی Go

این جزوه به بررسی داده‌ساختارهای اصلی در Go (آرایه‌ها، اسلایس‌ها، مپ‌ها، رشته‌ها، ساختارها، و رابط‌ها) و مدیریت حافظه (اشاره‌گرها، garbage collection، و بهینه‌سازی) می‌پردازد. هر بخش با توضیحات مفهومی، مثال‌های عملی، نکات پیشرفته، و خطاهای رایج همراه است.

---

## 1. داده‌ساختارها در Go

Go دارای داده‌ساختارهای داخلی قدرتمندی است که برای حل مسائل مختلف طراحی شده‌اند. در این بخش، هر داده‌ساختار به طور کامل بررسی می‌شود.

### 1.1. آرایه‌ها (Arrays)

#### تعریف
آرایه یک مجموعه با **اندازه ثابت** از عناصر با نوع یکسان است. اندازه آرایه در زمان تعریف مشخص می‌شود و قابل تغییر نیست.

#### سینتکس
```go
var arr [3]int // آرایه‌ای با 3 عدد صحیح
arr := [3]int{1, 2, 3} // مقداردهی مستقیم
```

#### ویژگی‌ها
- **اندازه ثابت**: نمی‌توان عنصر اضافه یا کم کرد.
- **صفر مقداردهی**: اگر مقداری مشخص نشود، عناصر به مقدار پیش‌فرض (مثلاً 0 برای int) تنظیم می‌شوند.
- **دسترسی با ایندکس**: از 0 شروع می‌شود.

#### مثال
```go
package main

import "fmt"

func main() {
    var arr [4]int
    arr[0] = 10
    arr[1] = 20
    fmt.Println(arr) // خروجی: [10 20 0 0]

    arr2 := [3]string{"Ali", "Bob", "Cathy"}
    for i, v := range arr2 {
        fmt.Printf("Index: %d, Value: %s\n", i, v)
    }
}
```

#### نکات پیشرفته
- **کپی آرایه**: آرایه‌ها به صورت **کپی** منتقل می‌شوند (نه ارجاع). برای تغییر آرایه در تابع، از اشاره‌گر استفاده کنید:
  ```go
  func modifyArray(arr *[3]int) {
      arr[0] = 100
  }

  func main() {
      arr := [3]int{1, 2, 3}
      modifyArray(&arr)
      fmt.Println(arr) // خروجی: [100 2 3]
  }
  ```
- **محدودیت**: به دلیل اندازه ثابت، آرایه‌ها کمتر از اسلایس‌ها استفاده می‌شوند.

#### خطاهای رایج
- دسترسی به ایندکس خارج از محدوده: باعث `panic: runtime error: index out of range` می‌شود.
- تلاش برای تغییر اندازه آرایه: غیرممکن است.

---

### 1.2. اسلایس‌ها (Slices)

#### تعریف
اسلایس یک نمای پویا از یک آرایه است که اندازه آن می‌تواند تغییر کند. اسلایس‌ها پرکاربردترین داده‌ساختار در Go هستند.

#### سینتکس
```go
var slice []int // اسلایس خالی
slice := []int{1, 2, 3} // مقداردهی مستقیم
slice := make([]int, 5, 10) // طول 5، ظرفیت 10
```

#### ساختار داخلی
اسلایس شامل سه بخش است:
1. **اشاره‌گر**: به آرایه زیرین اشاره می‌کند.
2. **طول (Length)**: تعداد عناصر فعلی.
3. **ظرفیت (Capacity)**: تعداد عناصری که آرایه زیرین می‌تواند نگه دارد.

#### عملیات اصلی
- **افزودن عنصر**:
  ```go
  slice := []int{1, 2}
  slice = append(slice, 3) // خروجی: [1 2 3]
  ```
- **برش (Slicing)**:
  ```go
  slice := []int{1, 2, 3, 4}
  sub := slice[1:3] // خروجی: [2 3]
  ```
- **کپی**:
  ```go
  src := []int{1, 2, 3}
  dst := make([]int, len(src))
  copy(dst, src) // کپی کامل
  ```

#### مثال
```go
package main

import "fmt"

func main() {
    slice := []int{1, 2, 3}
    slice = append(slice, 4, 5)
    fmt.Println("Slice:", slice) // خروجی: [1 2 3 4 5]

    fmt.Println("Length:", len(slice)) // 5
    fmt.Println("Capacity:", cap(slice)) // 5 یا بیشتر

    subSlice := slice[1:4]
    fmt.Println("Subslice:", subSlice) // [2 3 4]

    // تغییر در subSlice روی slice اصلی اثر می‌گذارد
    subSlice[0] = 99
    fmt.Println("Original after change:", slice) // [1 99 3 4 5]
}
```

#### نکات پیشرفته
- **مدیریت ظرفیت**: اگر ظرفیت اسلایس پر شود، `append` یک آرایه جدید با ظرفیت بیشتر (معمولاً دو برابر) ایجاد می‌کند.
- **اشتراک آرایه زیرین**: اسلایس‌های برش‌خورده به یک آرایه مشترک اشاره می‌کنند، بنابراین تغییرات در یکی ممکن است روی دیگری اثر بگذارد.
- **اسلایس‌های خالی و nil**:
  ```go
  var s1 []int // nil slice
  s2 := []int{} // اسلایس خالی
  fmt.Println(s1 == nil) // true
  fmt.Println(s2 == nil) // false
  ```

#### خطاهای رایج
- **دسترسی به ایندکس نامعتبر**: مانند آرایه‌ها باعث panic می‌شود.
- **استفاده نادرست از append**: بدون تخصیص نتیجه به متغیر (مثلاً `append(slice, 1)` به تنهایی تأثیری ندارد).

---

### 1.3. مپ‌ها (Maps)

#### تعریف
مپ یک داده‌ساختار کلید-مقدار است که برای ذخیره و بازیابی سریع داده‌ها استفاده می‌شود.

#### سینتکس
```go
var m map[string]int // مپ خالی (nil)
m := make(map[string]int) // مپ آماده
m := map[string]int{"Ali": 30, "Bob": 25} // مقداردهی مستقیم
```

#### عملیات اصلی
- **افزودن/به‌روزرسانی**:
  ```go
  m["Ali"] = 30
  ```
- **حذف**:
  ```go
  delete(m, "Ali")
  ```
- **بررسی وجود**:
  ```go
  value, exists := m["Ali"]
  ```

#### مثال
```go
package main

import "fmt"

func main() {
    m := make(map[string]int)
    m["Ali"] = 30
    m["Bob"] = 25

    fmt.Println("Map:", m) // خروجی: map[Ali:30 Bob:25]

    age, exists := m["Ali"]
    if exists {
        fmt.Println("Age of Ali:", age) // 30
    }

    delete(m, "Bob")
    fmt.Println("After delete:", m) // map[Ali:30]
}
```

#### نکات پیشرفته
- **مپ‌های nil**: مپی که مقداردهی نشده باشد (nil) قابل خواندن است، اما نوشتن در آن باعث panic می‌شود.
- **ترتیب**: مپ‌ها ترتیب مشخصی ندارند؛ برای پیمایش مرتب، کلیدها را جدا کنید:
  ```go
  keys := make([]string, 0, len(m))
  for k := range m {
      keys = append(keys, k)
  }
  sort.Strings(keys)
  ```
- **همزمانی**: مپ‌ها thread-safe نیستند. برای دسترسی همزمان، از `sync.RWMutex` استفاده کنید.

#### خطاهای رایج
- نوشتن در مپ nil.
- انتظار ترتیب در پیمایش مپ.

---

### 1.4. رشته‌ها و Rune

#### تعریف
رشته‌ها در Go دنباله‌ای از بایت‌ها هستند که معمولاً به صورت UTF-8 ذخیره می‌شوند. نوع `rune` برای نمایش کاراکترهای یونیکد استفاده می‌شود (معادل `int32`).

#### سینتکس
```go
s := "Hello, جهان"
r := rune('ج') // یک کاراکتر یونیکد
```

#### عملیات
- **دسترسی به کاراکتر**:
  ```go
  s := "Hello"
  fmt.Println(s[0]) // 72 (بایت H)
  ```
- **پیمایش با rune**:
  ```go
  for i, r := range "جهان" {
      fmt.Printf("Index: %d, Rune: %c\n", i, r)
  }
  ```
- **اتصال**:
  ```go
  s := "Hello" + " World"
  ```

#### مثال
```go
package main

import (
    "fmt"
    "unicode/utf8"
)

func main() {
    s := "Hello, جهان"
    fmt.Println("Length in bytes:", len(s)) // تعداد بایت‌ها
    fmt.Println("Length in runes:", utf8.RuneCountInString(s)) // تعداد کاراکترها

    for i, r := range s {
        fmt.Printf("Index: %d, Rune: %c\n", i, r)
    }

    // تبدیل به rune
    runes := []rune(s)
    fmt.Println("First rune:", string(runes[0])) // H
}
```

#### نکات پیشرفته
- **UTF-8**: رشته‌ها در Go به صورت UTF-8 ذخیره می‌شوند، بنابراین کاراکترهای غیرلاتین (مثل فارسی) چند بایت اشغال می‌کنند.
- **تغییرناپذیری**: رشته‌ها immutable هستند. برای تغییرات، به `[]rune` تبدیل کنید.
- **کارایی**: برای اتصال تعداد زیادی رشته، از `strings.Builder` استفاده کنید:
  ```go
  var builder strings.Builder
  for i := 0; i < 1000; i++ {
      builder.WriteString("x")
  }
  result := builder.String()
  ```

#### خطاهای رایج
- دسترسی مستقیم به کاراکترها بدون در نظر گرفتن UTF-8.
- استفاده از `len` برای شمارش کاراکترها (بایت‌ها را می‌شمارد).

---

### 1.5. ساختارها (Structs)

#### تعریف
ساختارها برای تعریف داده‌های پیچیده با چندین فیلد استفاده می‌شوند.

#### سینتکس
```go
type Person struct {
    Name string
    Age  int
}
```

#### عملیات
- **ایجاد**:
  ```go
  p := Person{Name: "Ali", Age: 30}
  ```
- **دسترسی**:
  ```go
  fmt.Println(p.Name) // Ali
  ```
- **متدها**:
  ```go
  func (p Person) Greet() string {
      return "Hello, " + p.Name
  }
  ```

#### مثال
```go
package main

import "fmt"

type Person struct {
    Name string
    Age  int
}

func (p Person) Greet() string {
    return fmt.Sprintf("Hello, %s! You are %d years old.", p.Name, p.Age)
}

func main() {
    p := Person{Name: "Ali", Age: 30}
    fmt.Println(p.Greet()) // Hello, Ali! You are 30 years old.

    // اشاره‌گر به struct
    pp := &p
    pp.Age = 31
    fmt.Println(p.Age) // 31
}
```

#### نکات پیشرفته
- **Embedding**:
  ```go
  type Employee struct {
      Person
      ID int
  }
  e := Employee{Person: Person{Name: "Ali"}, ID: 123}
  fmt.Println(e.Name) // Ali
  ```
- **اشاره‌گر در متدها**: برای تغییر struct، از اشاره‌گر استفاده کنید:
  ```go
  func (p *Person) Birthday() {
      p.Age++
  }
  ```

#### خطاهای رایج
- فراموش کردن اشاره‌گر در متدهای تغییردهنده.
- مقداردهی ناقص فیلدها.

---

### 1.6. رابط‌ها (Interfaces)

#### تعریف
رابط‌ها مجموعه‌ای از متدها را تعریف می‌کنند که یک نوع باید پیاده‌سازی کند.

#### سینتکس
```go
type Shape interface {
    Area() float64
}
```

#### پیاده‌سازی
هر نوع که متدهای رابط را پیاده‌سازی کند، به طور خودکار آن رابط را ارضا می‌کند.

#### مثال
```go
package main

import (
    "fmt"
    "math"
)

type Shape interface {
    Area() float64
}

type Circle struct {
    Radius float64
}

func (c Circle) Area() float64 {
    return math.Pi * c.Radius * c.Radius
}

type Rectangle struct {
    Width, Height float64
}

func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

func printArea(s Shape) {
    fmt.Printf("Area: %.2f\n", s.Area())
}

func main() {
    c := Circle{Radius: 5}
    r := Rectangle{Width: 4, Height: 6}

    printArea(c) // Area: 78.54
    printArea(r) // Area: 24.00
}
```

#### نکات پیشرفته
- **رابط خالی (`interface{}`)**: می‌تواند هر مقداری را نگه دارد (مشابه `any` در Go 1.18+).
  ```go
  var i interface{}
  i = 42
  i = "hello"
  ```
- **Type Assertion**:
  ```go
  if v, ok := i.(string); ok {
      fmt.Println("String:", v)
  }
  ```
- **Type Switch**:
  ```go
  switch v := i.(type) {
  case int:
      fmt.Println("Int:", v)
  case string:
      fmt.Println("String:", v)
  }
  ```

#### خطاهای رایج
- عدم پیاده‌سازی تمام متدهای رابط.
- استفاده نادرست از type assertion.

---

## 2. مدیریت حافظه در Go

Go دارای سیستم مدیریت حافظه خودکار (garbage collection) است، اما درک اشاره‌گرها و بهینه‌سازی حافظه برای نوشتن برنامه‌های کارآمد ضروری است.

### 2.1. اشاره‌گرها (Pointers)

#### تعریف
اشاره‌گر متغیری است که آدرس حافظه یک متغیر دیگر را ذخیره می‌کند.

#### سینتکس
```go
var p *int // اشاره‌گر به int
x := 42
p = &x // آدرس x
*p = 43 // تغییر مقدار
```

#### مثال
```go
package main

import "fmt"

func increment(p *int) {
    *p++
}

func main() {
    x := 42
    increment(&x)
    fmt.Println(x) // 43
}
```

#### نکات پیشرفته
- **اشاره‌گرهای nil**: اشاره‌گر بدون مقداردهی nil است و استفاده از آن باعث panic می‌شود.
- **عدم وجود عملیات اشاره‌گر**: برخلاف C، نمی‌توانید اشاره‌گرها را جابجا کنید (مثلاً `p++`).
- **اشاره‌گر در struct**: برای تغییر struct در متدها، از اشاره‌گر استفاده کنید.

#### خطاهای رایج
- استفاده از اشاره‌گر nil.
- کپی اشاره‌گر بدون درک تأثیرات.

---

### 2.2. Garbage Collection

#### تعریف
Go از یک garbage collector (GC) برای مدیریت حافظه استفاده می‌کند که اشیاء غیرقابل دسترس را آزاد می‌کند.

#### نحوه کار
- **Mark-and-Sweep**: GC ابتدا اشیاء قابل دسترس را علامت‌گذاری کرده و سپس اشیاء غیرقابل دسترس را حذف می‌کند.
- **توقف کوتاه**: GC در Go کم‌تأخیر است و برای برنامه‌های بلادرنگ مناسب است.

#### بهینه‌سازی
- **کاهش تخصیص حافظه**: از متغیرهای محلی و مقادیر کوچک استفاده کنید.
- **استفاده از sync.Pool**: برای اشیاء پرتکرار:
  ```go
  var pool = sync.Pool{
      New: func() interface{} {
          return &bytes.Buffer{}
      },
  }

  func main() {
      buf := pool.Get().(*bytes.Buffer)
      defer pool.Put(buf)
      buf.WriteString("test")
  }
  ```

#### نکات پیشرفته
- **پروفایلینگ**: از ابزار `pprof` برای بررسی مصرف حافظه استفاده کنید:
  ```bash
  go tool pprof http://localhost:6060/debug/pprof/heap
  ```
- **مدیریت GC**: متغیر `GOGC` برای تنظیم رفتار GC:
  ```bash
  GOGC=200 go run main.go
  ```

#### خطاهای رایج
- نگه داشتن ارجاعات غیرضروری که مانع جمع‌آوری زباله می‌شود.
- تخصیص بیش از حد حافظه در حلقه‌ها.

---

### 2.3. بهینه‌سازی حافظه

#### تکنیک‌ها
- **کاهش کپی**: از اسلایس‌ها به جای آرایه‌ها استفاده کنید تا کپی غیرضروری کاهش یابد.
- **استفاده از struct به جای map**: برای داده‌های ثابت، struct‌ها کارآمدتر هستند.
- **مدیریت رشته‌ها**: از `strings.Builder` برای اتصال رشته‌ها استفاده کنید.

#### مثال
```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    var builder strings.Builder
    for i := 0; i < 1000; i++ {
        builder.WriteString("x")
    }
    fmt.Println("Length:", builder.Len())
}
```

#### نکات پیشرفته
- **Escape Analysis**: Go بررسی می‌کند که آیا متغیرها باید در heap تخصیص یابند:
  ```bash
  go build -gcflags="-m"
  ```
- **کاهش فشار GC**: از متغیرهای محلی و اشاره‌گرهای مناسب استفاده کنید.

---

## 3. بهترین شیوه‌ها و نکات

- **نام‌گذاری واضح**: برای فیلدهای struct و کلیدهای map از نام‌های معنادار استفاده کنید.
- **اجتناب از پیچیدگی غیرضروری**: داده‌ساختار مناسب (مثلاً slice به جای map برای لیست‌ها) انتخاب کنید.
- **تست کارایی**: از بنچمارک‌ها برای مقایسه داده‌ساختارها استفاده کنید:
  ```go
  func BenchmarkSliceAppend(b *testing.B) {
      s := []int{}
      for i := 0; i < b.N; i++ {
          s = append(s, i)
      }
  }
  ```
- **مدیریت همزمانی**: برای دسترسی همزمان به map و slice، از قفل‌ها استفاده کنید.

---

## 4. نتیجه‌گیری

این جزوه داده‌ساختارهای اصلی Go (آرایه‌ها، اسلایس‌ها، مپ‌ها، رشته‌ها، ساختارها، و رابط‌ها) و مدیریت حافظه (اشاره‌گرها، garbage collection، و بهینه‌سازی) را با جزئیات کامل پوشش داد. هر بخش با مثال‌های عملی، نکات پیشرفته، و خطاهای رایج همراه بود تا درک عمیقی از این مفاهیم فراهم شود.

برای یادگیری بیشتر:
- مستندات رسمی: `https://golang.org/doc/`
- کتاب "The Go Programming Language"
- تمرین با پروژه‌های واقعی مثل ساخت ابزارهای خط فرمان یا API