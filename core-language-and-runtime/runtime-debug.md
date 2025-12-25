### `runtime/debug`
> Build 資訊、GC 調校、Stack dump

#### 常用函數

| 函數 | 說明 | 使用時機 |
|-|-|-|
| `ReadBuildInfo()` | 取得編譯時的模組資訊 | 顯示版本、依賴資訊 |
| `SetGCPercent(n)` | 設定 GC 目標百分比（預設 100） | 調整 GC 頻率（高值=少 GC，低值=多 GC） |
| `SetMemoryLimit(n)` | 設定記憶體軟限制（1.19+） | 防止 OOM，限制記憶體使用 |
| `FreeOSMemory()` | 強制歸還記憶體給 OS | 釋放閒置記憶體 |
| `PrintStack()` | 印出當前 goroutine stack trace | Debug 時追蹤呼叫路徑 |
| `SetMaxStack(n)` | 設定單一 goroutine stack 最大值 | 限制 stack 記憶體使用 |
| `SetMaxThreads(n)` | 設定 OS thread 最大數量 | 限制系統 thread 數量 |

- 取得 build 資訊
    ```go
    info, ok := debug.ReadBuildInfo()
    if ok {
        fmt.Printf("Go version: %s\n", info.GoVersion)
        fmt.Printf("Main module: %s@%s\n", info.Main.Path, info.Main.Version)
        
        for _, dep := range info.Deps {
            fmt.Printf("Dependency: %s@%s\n", dep.Path, dep.Version)
        }
    }
    
    // Go version: go1.25.1
    // Main module: myapp@v1.0.0
    // Dependency: github.com/gin-gonic/gin@v1.9.1
    ```
- 調整 GC 行為
    ```go
    // 記憶體優先 積極回收)
    old := debug.SetGCPercent(50)           // heap 增長 50% 即觸發 GC
    fmt.Printf("Old GC percent: %d\n", old)
    
    // 效能優先 (較少回收)
    debug.SetGCPercent(200)                 // heap 增長 200% 才觸發 GC
    
    // 停用自動 GC (需手動呼叫 runtime.GC())
    debug.SetGCPercent(-1)
    ```
- 設定記憶體限制 (1.19+)
    ```go
    debug.SetMemoryLimit(1 << 30)           // 限制程式最多使用 1 GB 記憶體

    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    limit := int64(float64(m.Sys) * 0.8)    // 限制為系統記憶體的 80%
    debug.SetMemoryLimit(limit)
    ```
- 強制釋放記憶體給 OS
    ```go
    runtime.GC()              // 先觸發 GC
    debug.FreeOSMemory()      // 再歸還給 OS
    ```
- 印出 stack trace
    ```go
    func processRequest() {
        validateInput()
    }
    
    func validateInput() {
        debug.PrintStack()    // 印出當前呼叫堆疊
    }
    
    func main() {
        processRequest()
    }
    
    // goroutine 1 [running]:
    // runtime/debug.Stack()
    //     /usr/local/go/src/runtime/debug/stack.go:24 +0x64
    // runtime/debug.PrintStack()
    //     /usr/local/go/src/runtime/debug/stack.go:16 +0x1c
    // main.validateInput()
    //     /path/to/main.go:8 +0x20
    // main.processRequest()
    //     /path/to/main.go:4 +0x20
    // main.main()
    //     /path/to/main.go:12 +0x20
    ```
- 限制單一 goroutine stack 大小
    ```go
    func main() {
        debug.SetMaxStack(10 << 20)  // 限制為 10 MB
        
        go deepRecursion(0)
        
        time.Sleep(time.Second)
    }
    
    func deepRecursion(n int) {
        if n > 100000 {
            return
        }
        deepRecursion(n + 1)        // 超過限制會 panic
    }
    ```
- 限制 OS thread 數量
    ```go
    func main() {
        debug.SetMaxThreads(100)        // 限制為 100 個 thread
        
        for i := 0; i < 200; i++ {
            go func() {
                runtime.LockOSThread()  // 超過 100 個會 panic
                select {}
            }()
        }
        
        time.Sleep(time.Second)
    }
    ```
