# ابزارها و بهینه‌سازی در زبان برنامه‌نویسی Go

این جزوه به بررسی جامع ابزارها و تکنیک‌های بهینه‌سازی در Go می‌پردازد. Go با ابزارهای داخلی و خارجی قدرتمند، امکان پروفایلینگ، مدیریت حافظه، ساخت برنامه‌های کارآمد، و یکپارچه‌سازی با فرآیندهای توسعه را فراهم می‌کند. تمام موضوعات درخواستی با جزئیات کامل و مثال‌های عملی شرح داده شده‌اند.

---

## 1. پروفایلینگ و دیباگ

پروفایلینگ و دیباگ برای شناسایی گلوگاه‌های عملکرد و اشکالات برنامه ضروری هستند. Go ابزارهای داخلی مثل `pprof`، `trace`، و `go tool` را ارائه می‌دهد.

### 1.1. استفاده از pprof

#### مفهوم
`pprof` ابزاری برای پروفایلینگ برنامه‌های Go است که اطلاعاتی درباره مصرف CPU، حافظه، و goroutine‌ها ارائه می‌دهد.

#### فعال‌سازی pprof
پکیج `net/http/pprof` را به برنامه اضافه کنید:
```go
package main

import (
    "log"
    "net/http"
    _ "net/http/pprof"
)

func main() {
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
    // کد برنامه
    select {}
}
```

#### جمع‌آوری پروفایل
- **CPU**:
  ```bash
  go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
  ```
  این دستور 30 ثانیه داده جمع‌آوری کرده و وارد محیط تعاملی `pprof` می‌شود.
- **حافظه (Heap)**:
  ```bash
  go tool pprof http://localhost:6060/debug/pprof/heap
  ```
- **Goroutine‌ها**:
  ```bash
  go tool pprof http://localhost:6060/debug/pprof/goroutine
  ```

#### تحلیل
در محیط `pprof`:
- **top**: نمایش توابعی که بیشترین منابع را مصرف می‌کنند:
  ```
  (pprof) top
  ```
- **web**: تولید نمودار گرافیکی:
  ```
  (pprof) web
  ```
- **list <function>**:
  ```bash
  (pprof) list main.myFunction
  ```

#### مثال عملی
فرض کنید برنامه‌ای دارید که CPU زیادی مصرف می‌کند:
```go
package main

import (
    "net/http"
    _ "net/http/pprof"
)

func heavyComputation() {
    for i := 0; i < 1000000; i++ {
        _ = i * i
    }
}

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        heavyComputation()
        w.Write([]byte("Done"))
    })
    http.ListenAndServe(":8080", nil)
}
```
1. اجرا: `go run main.go`
2. پروفایل CPU: `go tool pprof http://localhost:6060/debug/pprof/profile`
3. تحلیل: `top` نشان می‌دهد `heavyComputation` منابع زیادی مصرف می‌کند.

### 1.2. استفاده از trace
`trace` برای تحلیل اجرای برنامه در سطح goroutine‌ها و زمان‌بندی استفاده می‌شود.

#### جمع‌آوری
```go
package main

import (
    "log"
    "os"
    "runtime/trace"
)

func main() {
    f, err := os.Create("trace.out")
    if err != nil {
        log.Fatal(err)
    }
    defer f.Close()

    if err := trace.Start(f); err != nil {
        log.Fatal(err)
    }
    defer trace.Stop()

    // کد برنامه
    for i := 0; i < 10; i++ {
        go func() {
            time.Sleep(time.Millisecond * 100)
        }()
    }
    time.Sleep(time.Second)
}
```

#### تحلیل
```bash
go tool trace trace.out
```
این دستور یک رابط وب باز می‌کند که شامل:
- **Timeline**: زمان‌بندی goroutine‌ها.
- **Network/Sync**: تحلیل بلاک شدن‌ها.
- **Syscalls**: تماس‌های سیستمی.

### 1.3. استفاده از go tool
`go tool` ابزارهای مختلفی برای دیباگ ارائه می‌دهد:
- **go tool objdump**: تحلیل کد اسمبلی:
  ```bash
  go tool objdump -s main.main mybinary
  ```
- **go tool nm**: نمایش سمبل‌ها:
  ```bash
  go tool nm mybinary
  ```

### 1.4. نکات پیشرفته
- **پروفایلینگ در تولید**:
  - از endpointهای `pprof` با احتیاط استفاده کنید (با احراز هویت).
  - پروفایل‌های کوتاه‌مدت (مثل 10 ثانیه) جمع‌آوری کنید.
- **تحلیل Data Race**:
  ```bash
  go run -race main.go
  ```
- **دیباگ با Delve**:
  ```bash
  go get -u github.com/go-delve/delve/cmd/dlv
  dlv debug main.go
  ```

### 1.5. خطاهای رایج
- جمع‌آوری پروفایل‌های طولانی (باعث فشار روی سرور).
- نادیده گرفتن هشدارهای `-race`.

---

## 2. مدیریت حافظه

مدیریت حافظه در Go توسط **Garbage Collector (GC)** انجام می‌شود، اما تکنیک‌هایی مثل `sync.Pool` و بهینه‌سازی GC می‌توانند کارایی را بهبود دهند.

### 2.1. استفاده از sync.Pool

#### مفهوم
`sync.Pool` برای مدیریت اشیاء موقت و کاهش فشار روی GC استفاده می‌شود. مناسب برای اشیائی که مکرراً ایجاد و دور ریخته می‌شوند.

#### مثال
```go
package main

import (
    "bytes"
    "fmt"
    "sync"
)

var pool = sync.Pool{
    New: func() interface{} {
        return &bytes.Buffer{}
    },
}

func main() {
    buf := pool.Get().(*bytes.Buffer)
    buf.WriteString("Hello")
    fmt.Println(buf.String())
    buf.Reset()
    pool.Put(buf)
}
```

#### نکات
- **کاربرد**: برای bufferها، اتصال‌های موقت، یا ساختارهای پرتکرار.
- **مدیریت**: همیشه اشیاء را قبل از `Put` بازنشانی کنید (مثل `buf.Reset()`).
- **همزمانی**: `sync.Pool` thread-safe است.

### 2.2. بهینه‌سازی Garbage Collection

#### نحوه کار GC
- Go از الگوریتم **Mark-and-Sweep** با کم‌تأخیر استفاده می‌کند.
- متغیر `GOGC` رفتار GC را کنترل می‌کند (پیش‌فرض 100):
  - `GOGC=200`: GC کمتر اجرا می‌شود (مصرف حافظه بیشتر).
  - `GOGC=50`: GC مکرر اجرا می‌شود (مصرف CPU بیشتر).

#### تنظیم
```bash
export GOGC=200
go run main.go
```

#### تکنیک‌های بهینه‌سازی
- **کاهش تخصیص**:
  - از متغیرهای محلی به جای heap استفاده کنید.
  - از اسلایس‌ها با ظرفیت اولیه استفاده کنید:
    ```go
    s := make([]int, 0, 1000)
    ```
- **استفاده از struct به جای map**: برای داده‌های ثابت، structها کارآمدترند.
- **Escape Analysis**:
  ```bash
  go build -gcflags="-m"
  ```
  این دستور نشان می‌دهد کدام متغیرها به heap منتقل می‌شوند.

#### مثال
```go
package main

import (
    "fmt"
    "runtime"
)

func main() {
    var ms runtime.MemStats
    runtime.ReadMemStats(&ms)
    fmt.Printf("Heap Alloc: %v bytes\n", ms.HeapAlloc)

    // تخصیص زیاد
    for i := 0; i < 1000000; i++ {
        _ = make([]byte, 100)
    }

    runtime.ReadMemStats(&ms)
    fmt.Printf("Heap Alloc after: %v bytes\n", ms.HeapAlloc)

    runtime.GC() // اجرای دستی GC (فقط برای تست)
}
```

### 2.3. نکات پیشرفته
- **پروفایلینگ حافظه**:
  ```bash
  go tool pprof http://localhost:6060/debug/pprof/heap
  ```
- **مدیریت Pool در تولید**: اشیاء را به صورت دوره‌ای از Pool حذف کنید تا از نشت حافظه جلوگیری شود.
- **Finalizers**: برای آزادسازی منابع خاص:
  ```go
  runtime.SetFinalizer(obj, func(o *Type) {
      // آزادسازی
  })
  ```

### 2.4. خطاهای رایج
- عدم بازنشانی اشیاء در `sync.Pool`.
- تنظیم نادرست `GOGC` بدون تست.

---

## 3. کامپایل و بیلد

Go فرآیند ساخت ساده‌ای دارد، اما تکنیک‌هایی برای بهینه‌سازی اندازه باینری و cross-compilation وجود دارد.

### 3.1. ساخت باینری
#### دستور پایه
```bash
go build -o myapp main.go
```
- خروجی: فایل اجرایی `myapp`.

#### بهینه‌سازی
- **حذف اطلاعات دیباگ**:
  ```bash
  go build -ldflags="-s -w" -o myapp
  ```
  - `-s`: حذف جدول سمبل‌ها.
  - `-w`: حذف اطلاعات DWARF.
- **فشرده‌سازی با UPX**:
  ```bash
  upx --brute myapp
  ```

### 3.2. Cross-Compilation
برای ساخت باینری برای سیستم‌عامل‌ها و معماری‌های دیگر:
```bash
GOOS=linux GOARCH=amd64 go build -o myapp-linux main.go
GOOS=windows GOARCH=amd64 go build -o myapp.exe main.go
```
- **GOOS**: سیستم‌عامل (linux، windows، darwin).
- **GOARCH**: معماری (amd64، arm، 386).

#### مثال
ساخت برای لینوکس از مک:
```bash
GOOS=linux GOARCH=amd64 go build -o myapp-linux main.go
```

### 3.3. بهینه‌سازی اندازه
- **حذف وابستگی‌های غیرضروری**:
  ```bash
  go mod tidy
  ```
- **استفاده از CGO=0**:
  ```bash
  CGO_ENABLED=0 go build -o myapp
  ```
  - غیرفعال کردن CGO برای باینری‌های مستقل.
- **Minimal Imports**: فقط پکیج‌های مورد نیاز را وارد کنید.

### 3.4. نکات پیشرفته
- **Static Linking**:
  ```bash
  CGO_ENABLED=0 GOOS=linux go build -a -ldflags="-extldflags '-static'" -o myapp
  ```
- **Build Tags**:
  ```go
  // +build prod
  package main
  ```
  ```bash
  go build -tags prod
  ```
- **Version Embedding**:
  ```bash
  go build -ldflags="-X main.version=1.0.0" -o myapp
  ```
  ```go
  var version string
  fmt.Println("Version:", version)
  ```

### 3.5. خطاهای رایج
- عدم تنظیم `GOOS`/`GOARCH` برای cross-compilation.
- استفاده از CGO در محیط‌هایی که وابستگی‌های C ندارند.

---

## 4. لینت و فرمت کد

لینت و فرمت کد برای حفظ کیفیت و خوانایی کد ضروری هستند.

### 4.1. استفاده از gofmt
`gofmt` ابزار استاندارد برای فرمت کد است.
```bash
gofmt -w main.go
```
- **ویژگی‌ها**: تنظیم فاصله‌ها، تورفتگی، و ساختار کد.
- **اتوماسیون**: اکثر IDEها (مثل VS Code) به طور خودکار `gofmt` را اجرا می‌کنند.

### 4.2. استفاده از golint
`golint` پیشنهادهایی برای بهبود سبک کد ارائه می‌دهد.
#### نصب
```bash
go install golang.org/x/lint/golint@latest
```
#### اجرا
```bash
golint ./...
```
- **مثال خروجی**:
  ```
  main.go:10:1: exported function Foo should have comment or be unexported
  ```

### 4.3. استفاده از staticcheck
`staticcheck` ابزار پیشرفته‌تری برای تحلیل کد است.
#### نصب
```bash
go install honnef.co/go/tools/cmd/staticcheck@latest
```
#### اجرا
```bash
staticcheck ./...
```
- **ویژگی‌ها**:
  - تشخیص کدهای مرده.
  - بررسی خطاهای رایج.
  - پیشنهاد بهینه‌سازی.

#### مثال
```go
package main

func main() {
    x := 1
    // x استفاده نمی‌شود
}
```
```bash
staticcheck main.go
main.go:3:6: x is unused (U1000)
```

### 4.4. نکات پیشرفته
- **اتوماسیون**:
  - از `pre-commit` برای اجرای `gofmt` و `staticcheck` استفاده کنید:
    ```yaml
    repos:
    - repo: local
      hooks:
      - id: gofmt
        name: gofmt
        entry: gofmt -w
        files: \.go$
        language: system
    ```
- **ابزارهای دیگر**:
  - `revive`: جایگزین سریع‌تر برای `golint`.
  - `golangci-lint`: ترکیبی از چندین ابزار لینت:
    ```bash
    go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
    golangci-lint run
    ```

### 4.5. خطاهای رایج
- نادیده گرفتن هشدارهای لینت.
- عدم اجرای `gofmt` قبل از commit.

---

## 5. CI/CD برای Go

CI/CD (Continuous Integration/Continuous Deployment) فرآیند خودکارسازی تست، بیلد، و استقرار است.

### 5.1. تنظیم GitHub Actions
GitHub Actions برای اجرای خودکار وظایف CI/CD استفاده می‌شود.

#### مثال Workflow
```yaml
name: Go CI

on:
  push:
    branches: [main]
  pull_request:
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
    - name: Install dependencies
      run: go mod download
    - name: Run tests
      run: go test -v ./...
    - name: Run linters
      uses: golangci/golangci-lint-action@v6
      with:
        version: latest
    - name: Build
      run: go build -o myapp
```

#### توضیحات
- **Trigger**: روی push و pull request اجرا می‌شود.
- **Jobs**:
  - نصب Go.
  - دانلود وابستگی‌ها.
  - اجرای تست‌ها.
  - لینت با `golangci-lint`.
  - ساخت باینری.

### 5.2. داکرایز کردن برنامه‌ها
داکر برای بسته‌بندی و استقرار برنامه‌ها استفاده می‌شود.

#### Dockerfile
```dockerfile
FROM golang:1.20 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o myapp

FROM alpine:latest
WORKDIR /root/
COPY --from=builder /app/myapp .
EXPOSE 8080
CMD ["./myapp"]
```

#### ساخت و اجرا
```bash
docker build -t myapp .
docker run -p 8080:8080 myapp
```

#### Multi-Stage Build
- **مرحله اول**: ساخت باینری با تصویر Go.
- **مرحله دوم**: استفاده از تصویر سبک Alpine برای کاهش اندازه.

### 5.3. نکات پیشرفته
- **Caching در CI**:
  ```yaml
  - name: Cache Go modules
    uses: actions/cache@v4
    with:
      path: ~/go/pkg/mod
      key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
  ```
- **استقرار**:
  - استقرار روی Kubernetes:
    ```bash
    kubectl apply -f deployment.yaml
    ```
  - استقرار روی AWS/GCP با داکر.
- **تست‌های پیشرفته**:
  - تست‌های یکپارچگی:
    ```bash
    go test -tags=integration ./...
    ```
  - Coverage:
    ```bash
    go test -coverprofile=coverage.out
    go tool cover -html=coverage.out
    ```

### 5.4. خطاهای رایج
- عدم تنظیم نسخه Go در CI.
- تصاویر داکر بزرگ به دلیل عدم استفاده از multi-stage.

---

## 6. بهترین شیوه‌ها و نکات

- **پروفایلینگ منظم**: به طور دوره‌ای پروفایل CPU و حافظه جمع‌آوری کنید.
- **مدیریت حافظه**: از `sync.Pool` برای اشیاء پرتکرار و ظرفیت اولیه برای اسلایس‌ها استفاده کنید.
- **ساخت بهینه**: همیشه از `-ldflags="-s -w"` و `CGO_ENABLED=0` برای باینری‌های تولید استفاده کنید.
- **کیفیت کد**: لینت و فرمت را در CI اجباری کنید.
- **CI/CD**: تست‌ها، لینت، و بیلد را در هر commit اجرا کنید.

---

## 7. نتیجه‌گیری

این جزوه تمام جنبه‌های ابزارها و بهینه‌سازی در Go را با جزئیات کامل پوشش داد. از پروفایلینگ با `pprof` و `trace` تا مدیریت حافظه با `sync.Pool`، ساخت باینری‌های بهینه، لینت کد، و تنظیم CI/CD، هر بخش با مثال‌های عملی و نکات پیشرفته ارائه شد. برای یادگیری عمیق‌تر:
- مستندات رسمی: `https://golang.org/doc/`
- کتاب "Go in Action"
- تمرین با پروژه‌های واقعی مثل ساخت API یا ابزار CLI