### `runtime/pprof`
> CPU 與記憶體 profiling

#### 常用函數

| 函數 | 說明 | 使用時機 |
|-|-|-|
| `StartCPUProfile(w)` | 開始 CPU profiling | 分析 CPU 使用熱點 |
| `StopCPUProfile()` | 停止 CPU profiling | 配合 StartCPUProfile 使用 |
| `WriteHeapProfile(w)` | 寫入 heap profile | 分析記憶體分配 |
| `Lookup(name)` | 取得指定的 profile | 分析 goroutine、block、mutex |

#### Profile 類型

| Profile | 說明 | 使用時機 |
|-|-|-|
| `cpu` | CPU 使用時間 | 找出 CPU 密集的函數 |
| `heap` | 記憶體分配 | 找出記憶體分配熱點 |
| `goroutine` | 所有 goroutine 狀態 | 檢測 goroutine 洩漏 |
| `block` | goroutine 阻塞事件 | 找出阻塞熱點 |
| `mutex` | mutex 爭用 | 找出鎖競爭問題 |
| `threadcreate` | OS thread 建立 | 分析 thread 使用 |

- 分析 CPU 性能 (HTTP API)
    ```go
    func main() {
        f, _ := os.Create("cpu.prof")
        defer f.Close()
        
        pprof.StartCPUProfile(f)
        defer pprof.StopCPUProfile()
        
        http.HandleFunc("/api/search", searchHandler)
        http.ListenAndServe(":8080", nil)
    }
    
    func searchHandler(w http.ResponseWriter, r *http.Request) {
        query := r.URL.Query().Get("q")
        results := searchDatabase(query)
        json.NewEncoder(w).Encode(results)
    }
    ```
- 檢測 cache Memory leak
    ```go
    var cache = make(map[string]*UserSession)
    
    func main() {
        http.HandleFunc("/login", loginHandler)
        go func() {
            time.Sleep(5 * time.Minute)
            runtime.GC()
            
            f, _ := os.Create("heap.prof")
            defer f.Close()
            pprof.WriteHeapProfile(f)
        }()
        http.ListenAndServe(":8080", nil)
    }
    
    func loginHandler(w http.ResponseWriter, r *http.Request) {
        sessionID := generateSessionID()
        cache[sessionID] = &UserSession{Data: make([]byte, 1024*100)}
        // 模擬未清理過期 session，讓記憶體持續增長
    }
    ```
- 檢測 worker pool Goroutine leak
    ```go
    func main() {
        ch := make(chan Job)
        
        for i := 0; i < 10; i++ {
            go worker(ch)
        }
        
        for i := 0; i < 1000; i++ {
            ch <- Job{ID: i}
        }
        
        time.Sleep(10 * time.Second)
        
        f, _ := os.Create("goroutine.prof")
        defer f.Close()

        pprof.Lookup("goroutine").WriteTo(f, 0)
    }
    
    func worker(jobs <-chan Job) {
        for job := range jobs {
            go processJob(job)  // 讓每次都啟動新的 goroutine，模擬超過 work pool 數量
        }
    }
    ```
- 分析資料庫連接池的阻塞問題
    ```go
    func main() {
        runtime.SetBlockProfileRate(1)
        
        db := &ConnectionPool{maxConn: 5}
        
        for i := 0; i < 100; i++ {
            go func() {
                conn := db.GetConnection()
                time.Sleep(500 * time.Millisecond) // 讓每個請求都超過 500ms 模擬佔用與超量的連接延遲
                db.ReleaseConnection(conn)
            }()
        }
        
        time.Sleep(10 * time.Second)
        
        f, _ := os.Create("block.prof")
        defer f.Close()
        pprof.Lookup("block").WriteTo(f, 0)
    }
    ```
- 分析快取讀寫的鎖競爭
    ```go
    type Cache struct {
        mu   sync.RWMutex
        data map[string]string
    }
    
    func main() {
        runtime.SetMutexProfileFraction(1)
        
        cache := &Cache{data: make(map[string]string)}
        var wg sync.WaitGroup
        
        // 模擬 200 個並發操作（100 寫 + 100 讀）競爭同一個鎖
        for i := 0; i < 100; i++ {
            wg.Add(2)
            go func(id int) {
                defer wg.Done()
                cache.Set(fmt.Sprintf("key%d", id), "value")
            }(i)
            go func(id int) {
                defer wg.Done()
                cache.Get(fmt.Sprintf("key%d", id))
            }(i)
        }
        
        wg.Wait()
        
        f, _ := os.Create("mutex.prof")
        defer f.Close()
        pprof.Lookup("mutex").WriteTo(f, 0)
    }
    ```

#### 分析 profile 檔案

```bash
go tool pprof cpu.prof                      # 互動式分析
go tool pprof -http=:8080 cpu.prof          # Web UI
go tool pprof -top cpu.prof                 # 直接查看 top 10
go tool pprof -http=:8080 -flame cpu.prof   # 生成火焰圖
```
