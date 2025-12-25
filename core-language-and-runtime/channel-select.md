## `Channel` & `Select`
> 並發控制與通訊機制

### `Channel`

```go
// 建立
ch1 := make(chan int)       // unbuffered channel
ch2 := make(chan int, 10)   // buffered channel (cap=10)
ch3 := make(chan string, 5) // buffered channel (cap=5)

// 發送與接收
ch2 <- 42                   // 發送
val := <-ch2                // 接收
val, ok := <-ch2            // 接收並檢查是否關閉

// 關閉
close(ch2)
// ch2 <- 1                 // (錯誤) 通道已被關閉

// 單向
var sendOnly chan<- int = ch1     // 只能發送
var recvOnly <-chan int = ch1     // 只能接收 (不能關閉)

// 雙向透過函數限制為單向使用
func forSend(out chan<- int) {
    out <- 42                      // 發送
    // val := <-out                // (錯誤) 不能接收
    close(out)                     // 關閉
}

func forRecv(in <-chan int) {
    val := <-in                    // 接收
    // in <- 99                    // (錯誤) 不能發送
    // close(in)                   // (錯誤) 不能關閉
}

ch := make(chan int)
forSend(ch)                        // chan int → chan<- int
forRecv(ch)                        // chan int → <-chan int

// 流程判斷
func pipeline() <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for i := 0; i < 10; i++ {
            out <- i
        }
    }()
    return out
}
ch := pipeline()
v, ok := <-ch
if !ok {
    // channel 已關閉
}

// === Copy 操作 ===

src := []int{1, 2, 3, 4, 5}
dst := make([]int, 3)
n := copy(dst, src)         // n=3, dst=[1,2,3]

// 複製到自己（重疊）
copy(src[1:], src[:4])      // [1,1,2,3,4]

// === Min/Max（1.21+） ===

smallest := min(1, 2, 3)              // 1
largest := max(1.5, 2.5, 3.5)         // 3.5
minStr := min("apple", "banana")      // "apple"
```

### `Select`

- 基本語法
    ```go
    select {
    case v := <-ch1:
        // 從 ch1 接收
    case ch2 <- 100:
        // 發送到 ch2
    case <-time.After(time.Second):
        // timeout
    default:
        // 非阻塞操作
    }
    ```
- 阻塞等待
    ```go
    select {
    case msg := <-ch1:
        // 從 ch1 接收
    case msg := <-ch2:
        // 從 ch2 接收
    }
    // 任一就緒即執行
    ```
- 非阻塞接收
    ```go
    select {
    case msg := <-ch:
        // 從 ch 接收
    default:
        // ch 無資料時立即執行
    }
    ```
- 非阻塞發送
    ```go
    select {
    case ch <- value:
        // 發送成功
    default:
        // ch 已滿（或無接收）時立即執行
    }
    ```
- 超時處理
    ```go
    select {
    case msg := <-ch:
        // 從 ch 接收
    case <-time.After(3 * time.Second):
        // 等待 3 秒後若 ch 無資料則執行
    }
    ```
- 停止訊號（context）
    ```go
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    select {
    case msg := <-ch:
        // 從 ch 接收
    case <-ctx.Done():
        // context 被取消時立即執行
        return
    }
    ```
- 監聽接收與停止
    ```go
    for {
        select {
        case msg, ok := <-ch:
            if !ok {
                return  // channel 已關閉
            }
            // 執行
        case <-quit:
            return
        }
    }
    ```
- 優先級模擬（透過巢狀 select）
    ```go
    select {
    case msg := <-highPriority:
        // 執行高優先級
    default:
        select {
        case msg := <-lowPriority:
            // 執行低優先級
        default:
            // 無訊息
        }
    }
    ```

#### `Select` 注意事項

- `select` 無 `case` 會導致阻塞
    ```go
    select {}  // 永久阻塞，用於保持運行
    ```
- `nil` channel 永遠阻塞
    ```go
    var ch chan int  // nil channel
    select {
        case v := <-ch:  
            // 永遠不會執行
        case ch <- 1:    
            // 永遠不會執行
    }

    // 實際應用
    // 根據配置決定是否監聽某個 channel
    var debugCh chan string
    var errorCh chan error = make(chan error)

    func debug(ch chan string) {
        for msg := range ch {
            slog.Debug(msg)
        }
    }

    // 開發模式啟用 debug 日誌
    if isDebugMode {
        debugCh = make(chan string)  // 建立 channel，啟用監聽
        go debug(debugCh)
        // 透過 debugCh <- "user login" 打印
    }
    // 生產模式 debugCh 保持 nil

    for {
        select {
        case msg := <-debugCh:
            // isDebugMode=false 時 debugCh 為 nil 則永遠阻塞
        case err := <-errorCh:
            // 監聽錯誤
        }
    }
    ```
- 已關閉 channel 接收立即返回零值
    ```go
    ch := make(chan int)
    close(ch)

    select {
        case v := <-ch:  
        // channel 已關閉則 v=0，立即執行
    }
    
    select {
        case v, ok := <-ch:
            if !ok {
                // channel 已關閉
            }
    }
    ```
- 多個 case 就緒時隨機選擇
    ```go
    ch1 := make(chan int, 1)
    ch2 := make(chan int, 1)
    ch1 <- 1
    ch2 <- 2
    select {
        case <-ch1:
            // 可能執行
        case <-ch2:
            // 可能執行
    }
    // 隨機執行
    ```
- 對已關閉 channel 發送會 panic
    ```go
    ch := make(chan int)
    close(ch)

    select {
        case ch <- 1:  // panic
    }
    ```
- `time.After` 在 `for`+`select` 中會記憶體洩漏
    ```go
    // 錯誤寫法
    for {
        select {
            case <-ch:
            case <-time.After(time.Second):  // 每次都會建立新 timer
        }
    }

    // 正確寫法，迴圈外宣告 timer
    timer := time.NewTimer(time.Second)
    defer timer.Stop()

    for {
        select {
        case <-ch:
            timer.Reset(time.Second)
        case <-timer.C:
        }
    }
    ```

### 注意事項

- `close` 已關閉的 channel 會 `panic`
    ```go
    ch := make(chan int)
    close(ch)
    // close(ch)            // panic
    ```
