### `runtime/metrics`
> 運行時性能指標

#### 常用函數

| 函數 | 說明 | 使用時機 |
|-|-|-|
| `All()` | 取得所有可用 metric 描述 | 探索可用指標 |
| `Read([]Sample)` | 讀取指定 metric 值 | 監控特定指標 |

#### 常用 Metrics

| Metric | 說明 |
|-|-|
| `/gc/heap/allocs:bytes` | heap 分配總量 |
| `/gc/heap/frees:bytes` | heap 釋放總量 |
| `/gc/heap/goal:bytes` | GC 目標 heap 大小 |
| `/gc/cycles/total:gc-cycles` | GC 執行次數 |
| `/sched/goroutines:goroutines` | goroutine 數量 |
| `/memory/classes/total:bytes` | 記憶體總使用量 |
| `/cpu/classes/gc/total:cpu-seconds` | GC 使用的 CPU 時間 |

- 生產環境 HTTP 監控
    ```go
    func metricsHandler(w http.ResponseWriter, r *http.Request) {
        samples := []metrics.Sample{
            {Name: "/gc/heap/allocs:bytes"},
            {Name: "/sched/goroutines:goroutines"},
            {Name: "/memory/classes/total:bytes"},
            {Name: "/gc/cycles/total:gc-cycles"},
        }
        
        metrics.Read(samples)
        
        w.Header().Set("Content-Type", "text/plain")
        fmt.Fprintf(w, "heap_allocs_bytes %d\n", samples[0].Value.Uint64())
        fmt.Fprintf(w, "goroutines_total %d\n", samples[1].Value.Uint64())
        fmt.Fprintf(w, "memory_bytes %d\n", samples[2].Value.Uint64())
        fmt.Fprintf(w, "gc_cycles_total %d\n", samples[3].Value.Uint64())
    }
    
    func main() {
        http.HandleFunc("/metrics", metricsHandler)
        log.Fatal(http.ListenAndServe(":8080", nil))
    }
    ```
- 監測 Memory leak
    ```go
    func detectMemoryLeak() {
        samples := []metrics.Sample{
            {Name: "/gc/heap/allocs:bytes"},
            {Name: "/gc/heap/frees:bytes"},
        }
        
        var lastHeapUsed uint64
        ticker := time.NewTicker(10 * time.Second)
        
        for range ticker.C {
            metrics.Read(samples)
            
            allocs := samples[0].Value.Uint64()
            frees := samples[1].Value.Uint64()
            heapUsed := allocs - frees
            
            if lastHeapUsed > 0 {
                growth := heapUsed - lastHeapUsed
                if growth > 100 << 20 { // 增長超過 100MB
                    log.Printf("WARNING: Heap grew by %d MB\n", growth/1024/1024)
                }
            }
            
            lastHeapUsed = heapUsed
            log.Printf("Current heap: %d MB\n", heapUsed/1024/1024)
        }
    }
    ```
- 監測 Goroutine leak
    ```go
    func monitorGoroutineLeak(threshold uint64) {
        samples := []metrics.Sample{
            {Name: "/sched/goroutines:goroutines"},
        }
        
        ticker := time.NewTicker(5 * time.Second)
        
        for range ticker.C {
            metrics.Read(samples)
            count := samples[0].Value.Uint64()
            
            if count > threshold {
                log.Printf("ALERT: Goroutine count %d exceeds threshold %d\n", count, threshold)
                runtime.GC()
            } else {
                log.Printf("Goroutines: %d\n", count)
            }
        }
    }
    
    func main() {
        go monitorGoroutineLeak(100)
        select {}
    }
    ```