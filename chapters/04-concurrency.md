# همزمانی (Concurrency) در زبان برنامه‌نویسی Go

همزمانی یکی از نقاط قوت اصلی Go است که آن را برای ساخت سیستم‌های مقیاس‌پذیر و کارآمد ایده‌آل می‌کند. Go از **goroutines** (واحدهای سبک اجرای موازی) و **channels** (ابزارهای ارتباطی بین goroutine‌ها) برای مدیریت همزمانی استفاده می‌کند. این جزوه تمام جنبه‌های همزمانی در Go را با جزئیات کامل پوشش می‌دهد.

---

## 1. Goroutines

### 1.1. مفهوم
Goroutine یک تابع است که به صورت موازی و مستقل از تابع اصلی اجرا می‌شود. برخلاف thread‌های سیستم‌عامل که سنگین هستند، goroutine‌ها سبک‌وزن‌اند (چند کیلوبایت حافظه) و توسط runtime Go مدیریت می‌شوند.

### 1.2. ایجاد Goroutine
برای ایجاد یک goroutine، کافی است کلمه کلیدی `go` را قبل از فراخوانی تابع قرار دهید:
```go
package main

import (
    "fmt"
    "time"
)

func sayHello() {
    for i := 0; i < 3; i++ {
        fmt.Println("Hello from goroutine")
        time.Sleep(100 * time.Millisecond)
    }
}

func main() {
    go sayHello() // اجرای موازی
    for i := 0; i < 3; i++ {
        fmt.Println("Hello from main")
        time.Sleep(100 * time.Millisecond)
    }
    time.Sleep(1 * time.Second) // منتظر اتمام goroutine
}
```
- خروجی ممکن است به صورت درهم باشد، زیرا `sayHello` و `main` به صورت موازی اجرا می‌شوند.

### 1.3. مدیریت Goroutine‌ها
- **بدون هماهنگی**: Goroutine‌ها به طور پیش‌فرض مستقل‌اند و ممکن است قبل از اتمام، برنامه خاتمه یابد.
- **هماهنگی**: برای مدیریت، از ابزارهایی مثل `sync.WaitGroup` یا `channels` استفاده می‌شود (در بخش‌های بعدی توضیح داده می‌شود).
- **خاتمه زودهنگام**: اگر تابع `main` پایان یابد، تمام goroutine‌ها متوقف می‌شوند.

### 1.4. نکات پیشرفته
- **هزینه کم**: می‌توانید هزاران goroutine ایجاد کنید بدون فشار زیاد روی سیستم.
- **زمان‌بندی**: زمان‌بندی goroutine‌ها توسط runtime Go مدیریت می‌شود و وابسته به CPU نیست.
- **پروفایلینگ**: از ابزار `pprof` برای بررسی تعداد goroutine‌ها استفاده کنید:
  ```bash
  go tool pprof http://localhost:6060/debug/pprof/goroutine
  ```

### 1.5. خطاهای رایج
- **خاتمه زودهنگام**: عدم انتظار برای اتمام goroutine‌ها (مثل مثال بالا بدون `time.Sleep`).
- **ایجاد بیش از حد**: ایجاد تعداد زیاد goroutine بدون مدیریت منابع.

---

## 2. Channels

### 2.1. مفهوم
کانال‌ها (Channels) ابزار اصلی برای ارتباط و هماهنگی بین goroutine‌ها هستند. آن‌ها امکان ارسال و دریافت داده‌ها را به صورت ایمن فراهم می‌کنند.

### 2.2. انواع کانال
- **Unbuffered Channel**: بدون ظرفیت، ارسال و دریافت باید همزمان باشند.
  ```go
  ch := make(chan int)
  ```
- **Buffered Channel**: دارای ظرفیت مشخص، امکان ارسال بدون دریافت فوری.
  ```go
  ch := make(chan int, 5) // ظرفیت 5
  ```

### 2.3. ارسال/دریافت داده
- **ارسال**: `ch <- value`
- **دریافت**: `value := <-ch`
- **بستن کانال**: `close(ch)`

#### مثال
```go
package main

import "fmt"

func main() {
    ch := make(chan string)

    go func() {
        ch <- "Hello from goroutine"
        close(ch)
    }()

    msg := <-ch
    fmt.Println(msg) // Hello from goroutine
}
```

### 2.4. نکات پیشرفته
- **بستن کانال**: فقط فرستنده باید کانال را ببندد. دریافت از کانال بسته مقدار پیش‌فرض (مثل 0 برای int) برمی‌گرداند.
- **بررسی بسته بودن**:
  ```go
  value, ok := <-ch
  if !ok {
      fmt.Println("Channel closed")
  }
  ```
- **محدودیت جهت**: می‌توانید جهت کانال را محدود کنید:
  ```go
  func send(ch chan<- int) { // فقط ارسال
      ch <- 42
  }
  ```

### 2.5. خطاهای رایج
- **ارسال به کانال بسته**: باعث panic می‌شود.
- **Deadlock**: وقتی همه goroutine‌ها منتظر دریافت/ارسال باشند:
  ```go
  ch := make(chan int)
  ch <- 1 // Deadlock: هیچ دریافت‌کننده‌ای نیست
  ```

---

## 3. Select

### 3.1. مفهوم
دستور `select` برای مدیریت چندین کانال به صورت همزمان استفاده می‌شود. شبیه `switch` است، اما برای عملیات کانال‌ها.

### 3.2. سینتکس
```go
select {
case v := <-ch1:
    fmt.Println("Received from ch1:", v)
case ch2 <- x:
    fmt.Println("Sent to ch2")
default:
    fmt.Println("No operation")
}
```

### 3.3. الگوهای تایم‌اوت و لغو
- **تایم‌اوت**:
  ```go
  select {
  case v := <-ch:
      fmt.Println(v)
  case <-time.After(2 * time.Second):
      fmt.Println("Timeout")
  }
  ```
- **لغو**:
  ```go
  done := make(chan struct{})
  go func() {
      time.Sleep(1 * time.Second)
      close(done)
  }()

  select {
  case <-done:
      fmt.Println("Done")
  case <-ch:
      fmt.Println("Received")
  }
  ```

### 3.4. مثال
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)

    go func() {
        time.Sleep(1 * time.Second)
        ch1 <- "from ch1"
    }()
    go func() {
        time.Sleep(2 * time.Second)
        ch2 <- "from ch2"
    }()

    for i := 0; i < 2; i++ {
        select {
        case msg1 := <-ch1:
            fmt.Println(msg1)
        case msg2 := <-ch2:
            fmt.Println(msg2)
        }
    }
}
```

### 3.5. نکات پیشرفته
- **انتخاب تصادفی**: اگر چند عملیات آماده باشند، یکی به صورت تصادفی انتخاب می‌شود.
- **default**: برای جلوگیری از بلاک شدن استفاده می‌شود، اما باید با احتیاط استفاده شود تا عملیات مهم از دست نروند.

### 3.6. خطاهای رایج
- **Deadlock در select**: اگر هیچ عملیاتی آماده نباشد و `default` وجود نداشته باشد.
- **استفاده نادرست از default**: ممکن است عملیات‌های مهم را نادیده بگیرد.

---

## 4. الگوهای Concurrency

Go از الگوهای مختلفی برای حل مسائل همزمانی استفاده می‌کند. در ادامه، الگوهای مهم شرح داده می‌شوند.

### 4.1. Worker Pool

#### مفهوم
Worker Pool مجموعه‌ای از goroutine‌ها (کارگران) است که وظایف را از یک کانال دریافت کرده و پردازش می‌کنند.

#### مثال
```go
package main

import (
    "fmt"
    "sync"
)

func worker(id int, jobs <-chan int, results chan<- int) {
    for j := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, j)
        results <- j * 2
    }
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)
    var wg sync.WaitGroup

    // راه‌اندازی 3 کارگر
    for w := 1; w <= 3; w++ {
        wg.Add(1)
        go func(wid int) {
            defer wg.Done()
            worker(wid, jobs, results)
        }(w)
    }

    // ارسال کارها
    for j := 1; j <= 5; j++ {
        jobs <- j
    }
    close(jobs)

    // جمع‌آوری نتایج
    go func() {
        wg.Wait()
        close(results)
    }()

    for r := range results {
        fmt.Println("Result:", r)
    }
}
```

#### نکات
- **مزایا**: کنترل تعداد goroutine‌ها و جلوگیری از فشار روی سیستم.
- **بهینه‌سازی**: تعداد کارگران را بر اساس CPU تنظیم کنید (مثل `runtime.NumCPU()`).

### 4.2. Fan-out/Fan-in

#### مفهوم
- **Fan-out**: تقسیم یک کار بزرگ به چندین goroutine.
- **Fan-in**: جمع‌آوری نتایج از چندین goroutine.

#### مثال
```go
package main

import (
    "fmt"
    "sync"
)

func producer(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * n
        }
    }()
    return out
}

func merge(cs ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    wg.Add(len(cs))
    for _, c := range cs {
        go func(ch <-chan int) {
            defer wg.Done()
            for n := range ch {
                out <- n
            }
        }(c)
    }
    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}

func main() {
    in := producer(1, 2, 3, 4)
    c1 := square(in)
    c2 := square(in) // Fan-out: دو goroutine برای محاسبه مربع
    out := merge(c1, c2) // Fan-in: جمع‌آوری نتایج
    for n := range out {
        fmt.Println(n)
    }
}
```

#### نکات
- **کارایی**: Fan-out برای وظایف سنگین مناسب است.
- **مدیریت منابع**: تعداد goroutine‌ها را محدود کنید.

### 4.3. Pipeline

#### مفهوم
Pipeline مجموعه‌ای از مراحل پردازش است که هر مرحله خروجی خود را به مرحله بعدی ارسال می‌کند.

#### مثال
```go
package main

import "fmt"

func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

func main() {
    for n := range square(gen(2, 3, 4)) {
        fmt.Println(n) // 4, 9, 16
    }
}
```

#### نکات
- **ماژولاریتی**: هر مرحله مستقل است و می‌تواند جداگانه تست شود.
- **مقیاس‌پذیری**: برای پردازش داده‌های بزرگ مناسب است.

### 4.4. Generator

#### مفهوم
Generator یک goroutine است که مقادیر را به صورت تدریجی تولید می‌کند.

#### مثال
```go
package main

import (
    "fmt"
    "time"
)

func generator() <-chan int {
    out := make(chan int)
    go func() {
        for i := 1; ; i++ {
            out <- i
            time.Sleep(500 * time.Millisecond)
        }
    }()
    return out
}

func main() {
    nums := generator()
    for i := 0; i < 5; i++ {
        fmt.Println(<-nums)
    }
}
```

#### نکات
- **انعطاف‌پذیری**: برای تولید داده‌های بی‌پایان (مثل جریان داده) مناسب است.
- **لغو**: از Context برای توقف generator استفاده کنید (بخش بعدی).

---

## 5. مدیریت منابع

### 5.1. پکیج sync

#### WaitGroup
برای انتظار اتمام مجموعه‌ای از goroutine‌ها:
```go
var wg sync.WaitGroup
wg.Add(1)
go func() {
    defer wg.Done()
    // کار
}()
wg.Wait()
```

#### Mutex
برای جلوگیری از دسترسی همزمان به منابع مشترک:
```go
var (
    counter int
    mu      sync.Mutex
)

func increment() {
    mu.Lock()
    defer mu.Unlock()
    counter++
}
```

#### RWMutex
برای خواندن/نوشتن همزمان:
```go
var (
    data  map[string]int
    rwmu  sync.RWMutex
)

func read(key string) int {
    rwmu.RLock()
    defer rwmu.RUnlock()
    return data[key]
}
```

### 5.2. پکیج atomic
برای عملیات اتمیک بدون قفل:
```go
var counter int64
atomic.AddInt64(&counter, 1)
fmt.Println(atomic.LoadInt64(&counter)) // 1
```

### 5.3. نکات
- **Mutex یا atomic؟**: برای عملیات ساده (مثل افزایش计数گر) از `atomic` استفاده کنید.
- **مدیریت منابع**: همیشه قفل‌ها را آزاد کنید (با `defer`).

### 5.4. خطاهای رایج
- **Deadlock در Mutex**: قفل کردن بدون آزادسازی.
- **استفاده نادرست از RWMutex**: خواندن/نوشتن بدون قفل مناسب.

---

## 6. Context

### 6.1. مفهوم
پکیج `context` برای مدیریت لغو، مهلت زمانی، و انتقال داده بین goroutine‌ها استفاده می‌شود.

### 6.2. انواع Context
- **context.Background()**: برای ریشه استفاده می‌شود.
- **context.WithTimeout**:
  ```go
  ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
  defer cancel()
  ```
- **context.WithCancel**:
  ```go
  ctx, cancel := context.WithCancel(context.Background())
  defer cancel()
  ```

### 6.3. انتقال داده
```go
ctx = context.WithValue(ctx, "userID", 123)
userID := ctx.Value("userID").(int)
```

### 6.4. مثال
```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    go func() {
        select {
        case <-time.After(3 * time.Second):
            fmt.Println("Work completed")
        case <-ctx.Done():
            fmt.Println("Work cancelled:", ctx.Err())
        }
    }()

    time.Sleep(3 * time.Second)
}
```

### 6.5. نکات پیشرفته
- **زنجیره Context**: از Contextهای تو در تو برای مدیریت سلسله‌مراتبی استفاده کنید.
- **لغو سریع**: همیشه `cancel()` را فراخوانی کنید تا منابع آزاد شوند.

### 6.6. خطاهای رایج
- **عدم لغو Context**: باعث نشت منابع می‌شود.
- **استفاده نادرست از Value**: فقط برای داده‌های مرتبط با درخواست استفاده کنید.

---

## 7. نکات پیشرفته Concurrency

### 7.1. جلوگیری از Data Race
- **تشخیص**:
  ```bash
  go run -race main.go
  ```
- **راه‌حل**:
  - استفاده از `sync.Mutex` یا `sync.RWMutex`.
  - استفاده از `atomic` برای عملیات ساده.
  - استفاده از کانال‌ها برای هماهنگی.

### 7.2. پروفایلینگ
- **تعداد Goroutine‌ها**:
  ```go
  fmt.Println(runtime.NumGoroutine())
  ```
- **ابزار pprof**:
  ```bash
  go tool pprof http://localhost:6060/debug/pprof/profile
  ```

### 7.3. بهینه‌سازی
- **کاهش Goroutine‌ها**: از Worker Pool برای محدود کردن تعداد استفاده کنید.
- **مدیریت حافظه**: از `sync.Pool` برای اشیاء موقت استفاده کنید.
- **تایم‌اوت‌ها**: همیشه از Context برای جلوگیری از عملیات طولانی استفاده کنید.

### 7.4. خطاهای رایج
- **Data Race**: دسترسی همزمان به متغیرها بدون قفل.
- **Goroutine Leak**: ایجاد goroutine بدون امکان خاتمه.

---

## 8. نتیجه‌گیری

این جزوه تمام جنبه‌های همزمانی در Go را با جزئیات کامل پوشش داد. از مفاهیم پایه مثل goroutine‌ها و کانال‌ها تا الگوهای پیشرفته مثل Worker Pool و Pipeline، و ابزارهای مدیریت منابع مثل Context و sync، هر بخش با مثال‌های عملی و نکات پیشرفته ارائه شد. برای یادگیری عمیق‌تر:
- مستندات رسمی: `https://golang.org/doc/`
- کتاب "Concurrency in Go" نوشته Katherine Cox-Buday
- تمرین با پروژه‌های واقعی مثل ساخت سیستم‌های پردازش داده موازی