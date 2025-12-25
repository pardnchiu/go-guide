## `runtime`
> Goroutine、記憶體、GC 與系統資訊

### 常用函數

#### CPU 與系統資訊

| 函數/常數 | 說明 | 回傳值 |
|-|-|-|
| `NumCPU()` | 回傳邏輯 CPU 核心數 | `int` |
| `GOMAXPROCS(n)` | 設定/取得最大 P（邏輯處理器）數量 | `int`（舊值） |
| `GOOS` | 作業系統（linux, darwin, windows） | `string` |
| `GOARCH` | 架構（amd64, arm64） | `string` |
| `Version()` | 取得 Go 版本字串 | `string` |
| `Compiler` | 編譯器名稱（gc, gccgo） | `string` |

```go
fmt.Printf("Go %s\n", runtime.Version())              // Go 1.25.1
fmt.Printf("%s/%s\n", runtime.GOOS, runtime.GOARCH)   // darwin/arm64
fmt.Printf("CPU: %d\n", runtime.NumCPU())             // CPU: 8


old := runtime.GOMAXPROCS(4)            // old = 8
current := runtime.GOMAXPROCS(0)        // current = 4
runtime.GOMAXPROCS(runtime.NumCPU())
```

#### 記憶體與 GC

| 函數 | 說明 | 使用時機 |
|-|-|-|
| `GC()` | 手動觸發 GC | 測試或需要立即回收記憶體 |
| `ReadMemStats(*MemStats)` | 讀取記憶體統計資訊 | 監控記憶體使用 |
| `KeepAlive(x)` | 確保物件不被 GC 回收到此點 | CGo、unsafe 指標場景 |

##### `MemStats`
記憶體統計資訊，常用欄位：

| 欄位 | 說明 |
|-|-|
| `Alloc` | 當前分配的 heap 記憶體（bytes） |
| `TotalAlloc` | 累計分配的記憶體（bytes） |
| `Sys` | 從 OS 取得的總記憶體（bytes） |
| `Mallocs` | 累計 malloc 次數 |
| `Frees` | 累計 free 次數 |
| `HeapAlloc` | heap 上已分配的記憶體 |
| `HeapSys` | heap 從 OS 取得的記憶體 |
| `HeapIdle` | heap 上閒置的記憶體 |
| `HeapInuse` | heap 上使用中的記憶體 |
| `HeapObjects` | heap 上分配的物件數 |
| `NumGC` | GC 執行次數 |
| `PauseTotalNs` | GC 暫停總時間（nanoseconds） |
| `LastGC` | 上次 GC 時間（Unix timestamp） |

- 基本使用
    ```go
    data := make([]byte, 100*1024*1024)   // 100 MB
    data = nil                            // 清空 (未釋放記憶體)
    runtime.GC()                          // 立即回收 100 MB

    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    fmt.Printf("Alloc: %d MB\n", m.Alloc/1024/1024)           // 當前分配的記憶體
    fmt.Printf("TotalAlloc: %d MB\n", m.TotalAlloc/1024/1024) // 累計分配
    fmt.Printf("Sys: %d MB\n", m.Sys/1024/1024)               // 從 OS 取得的總記憶體
    fmt.Printf("NumGC: %d\n", m.NumGC)                        // GC 執行次數
    fmt.Printf("PauseTotalNs: %d ms\n", m.PauseTotalNs/1e6)   // GC 暫停總時間
    ```
- 監控記憶體
    ```go
    func monitorMemory(interval time.Duration) {
        ticker := time.NewTicker(interval)
        defer ticker.Stop()
        
        for range ticker.C {
            var m runtime.MemStats
            runtime.ReadMemStats(&m)
            
            log.Printf("Memory: Alloc=%d MB, Sys=%d MB, NumGC=%d, Goroutines=%d",
                m.Alloc/1024/1024,
                m.Sys/1024/1024,
                m.NumGC,
                runtime.NumGoroutine())
        }
    }
    ```
- 檢測記憶體洩漏
    ```go
    func detectMemoryLeak() {
        var m1, m2 runtime.MemStats
        
        runtime.GC()
        runtime.ReadMemStats(&m1)
        
        // 執行可能洩漏的操作
        
        runtime.GC()
        runtime.ReadMemStats(&m2)
        
        // 比較 GC 前後的記憶體
        leaked := int64(m2.Alloc) - int64(m1.Alloc)
        log.Printf("%d MB", leaked/1024/1024)
    }
    ```
- GC 壓力測試
    ```go
    func gcBenchmark() {
        const iterations = 1000000
        start := time.Now()
        
        var m1, m2 runtime.MemStats
        runtime.ReadMemStats(&m1)
        
        for i := 0; i < iterations; i++ {
            _ = make([]byte, 1024)
        }
        
        runtime.ReadMemStats(&m2)
        elapsed := time.Since(start)
        
        fmt.Printf("執行時間: %v\n", elapsed)
        fmt.Printf("GC 次數: %d\n", m2.NumGC-m1.NumGC)
        fmt.Printf("平均 GC 暫停: %.2f ms\n", 
            float64(m2.PauseTotalNs-m1.PauseTotalNs)/float64(m2.NumGC-m1.NumGC)/1e6)
    }
    ```
- KeepAlive (確保 CGo 指標不被回收)
    ```go
    func useCgo() {
        obj := &MyStruct{}
        ptr := unsafe.Pointer(obj)
        C.some_c_function(ptr)        // 呼叫自定義的 C 函數
        runtime.KeepAlive(obj)        // 防止 obj 在 C 函數執行期間被 GC
    }
    ```
- 計算 heap 碎片率
    ```go
    func heapFragmentation() float64 {
        var m runtime.MemStats
        runtime.ReadMemStats(&m)
        
        fragmentation := float64(m.HeapSys-m.HeapInuse) / float64(m.HeapSys) * 100
        return fragmentation
    }
    ```

#### Goroutine 控制

| 函數 | 說明 | 使用時機 |
|-|-|-|
| `Gosched()` | 讓出 CPU 給其他 goroutine | 長時間運算時主動讓出 CPU |
| `Goexit()` | 終止當前 goroutine（defer 仍會執行） | goroutine 需要提前結束 |
| `NumGoroutine()` | 回傳目前 goroutine 數量 | 監控併發數量 |
| `LockOSThread()` | 綁定 goroutine 到當前 OS thread | CGo、OpenGL、syscall 場景 |
| `UnlockOSThread()` | 解除 goroutine 與 OS thread 的綁定 | 配合 LockOSThread 使用 |

- 長時間運算時主動釋出 CPU
    ```go
    func compute() {
        for i := 0; i < 1000000; i++ {
            if i%10000 == 0 {
                runtime.Gosched() // 每 10000 次迭代讓出一次
            }
        }
    }
    ```
- 終止 goroutine
    ```go
    func worker() {
        defer fmt.Println("cleanup") // defer 時打印
        
        if someCondition {
            runtime.Goexit() // 終止 goroutine
        }
    }
    ```
- 固定 thread 的場景 (OpenGL)
    ```go
    // 會降低併發效能 (thread 無法被其他 goroutine 使用)
    func init() {
        runtime.LockOSThread()
    }
    ```

#### Stack Trace 與呼叫資訊

| 函數 | 說明 | 回傳值 |
|-|-|-|
| `Caller(skip)` | 取得呼叫者資訊 | `(pc, file, line, ok)` |
| `Callers(skip, pc)` | 取得呼叫堆疊的 PC 值 | `int`（寫入數量） |
| `CallersFrames(pc)` | 將 PC 值轉換為 Frame 資訊 | `*Frames` |
| `FuncForPC(pc)` | 從 PC 取得函數資訊 | `*Func` |
| `Stack(buf, all)` | 取得 stack trace 文字 | `int`（寫入長度） |

##### `Func`

函數資訊型別（從 `FuncForPC` 取得）

| 方法 | 說明 |
|-|-|
| `Name()` | 函數完整名稱（含 package） |
| `Entry()` | 函數入口 PC 值 |
| `FileLine(pc)` | PC 值對應的檔案與行號 |

##### `Frame`

堆疊幀資訊（從 `CallersFrames` 取得）

| 欄位 | 說明 |
|-|-|
| `PC` | Program counter |
| `Func` | 函數名稱 |
| `Function` | 完整函數名稱 |
| `File` | 檔案路徑 |
| `Line` | 行號 |
| `Entry` | 函數入口位址 |

- 日誌顯示真實呼叫位置
    ```go
    func Log(msg string) {
        // skip=0: Log 函數自己
        // skip=1: 呼叫 Log 的函數
        // skip=2: 呼叫者的呼叫者
        _, file, line, _ := runtime.Caller(1)
        fmt.Printf("[%s:%d] %s\n", file, line, msg)
    }

    func main() {
        Log("test") // [/path/to/main.go:10] test
    }
    ```
- 自定義 error 型別包含完整 stack trace
    ```go
    type TraceError struct {
        msg   string
        stack []uintptr
    }

    func NewTraceError(msg string) *TraceError {
        const depth = 32
        var pcs [depth]uintptr
        n := runtime.Callers(2, pcs[:]) // skip=2: 跳過 Callers 和 NewTraceError
        return &TraceError{
            msg:   msg,
            stack: pcs[0:n],
        }
    }

    func (e *TraceError) Error() string {
        return e.msg
    }

    func (e *TraceError) StackTrace() string {
        var buf strings.Builder
        frames := runtime.CallersFrames(e.stack)
        
        for {
            frame, more := frames.Next()
            fmt.Fprintf(&buf, "%s\n\t%s:%d\n", 
                frame.Function, frame.File, frame.Line)
            if !more {
                break
            }
        }
        return buf.String()
    }

    func main() {
        err := processData()
        if err != nil {
            fmt.Println("錯誤:", err.Error())
            fmt.Println("\n堆疊追蹤:")
            fmt.Println(err.StackTrace())
        }
    }

    func processData() *TraceError {
        return validateInput()
    }

    func validateInput() *TraceError {
        return NewTraceError("test")
    }

    // 錯誤: test
    // 
    // 堆疊追蹤:
    // main.validateInput
    //     /path/to/main.go:45
    // main.processData
    //     /path/to/main.go:41
    // main.main
    //     /path/to/main.go:35
    ```
- 取得函數名稱
    ```go
    func getCurrentFuncName() string {
        pc, _, _, _ := runtime.Caller(0)
        return runtime.FuncForPC(pc).Name()
    }
    func main() {
        fmt.Println(getCurrentFuncName()) // main.getCurrentFuncName
    }
    ```
- 取得 stack trace 文字（用於 panic recovery 或日誌）
    ```go
    func recoverFromPanic() {
        if r := recover(); r != nil {
            buf := make([]byte, 4096)
            n := runtime.Stack(buf, false) // false=只取當前 goroutine
            fmt.Printf("Panic: %v\n\n堆疊:\n%s\n", r, buf[:n])
        }
    }

    func main() {
        defer recoverFromPanic()
        
        causeError()
    }

    func causeError() {
        panic("test")
    }

    // 輸出:
    // Panic: test
    // 
    // 堆疊:
    // goroutine 1 [running]:
    // main.recoverFromPanic()
    //     /path/to/main.go:3 +0x84
    // panic({0x1004a0, 0x10e0f8})
    //     /usr/local/go/src/runtime/panic.go:884 +0x212
    // main.causeError(...)
    //     /path/to/main.go:15
    // main.main()
    //     /path/to/main.go:11 +0x28
    ```