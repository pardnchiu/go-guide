## `builtin`
> 內建函數與型別

### 型別

| 型別 | 說明 |
|-|-|
| `bool` | 布林值 |
| `byte` | `uint8` 別名 |
| `rune` | `int32` 別名，代表 Unicode code point |
| `int`, `int8`, `int16`, `int32`, `int64` | 有號整數 |
| `uint`, `uint8`, `uint16`, `uint32`, `uint64` | 無號整數 |
| `uintptr` | 儲存指標的整數型別 |
| `float32`, `float64` | 浮點數 |
| `complex64`, `complex128` | 複數 |
| `string` | 字串 |
| `error` | 錯誤介面 |
| `any` | `interface{}` 別名（1.18+） |
| `comparable` | 可比較型別約束（1.18+） |


### 常用函數

| 函數 | 說明 |
|-|-|
| `make(t Type, size ...int)` | 建立 slice、map、channel |
| `new(Type)` | 配置記憶體並回傳指標 |
| `len(v)` | 回傳長度 |
| `cap(v)` | 回傳容量 |
| `append(slice, elems...)` | 附加元素至 slice |
| `copy(dst, src)` | 複製 slice 元素 |
| `delete(m, key)` | 刪除 map 中的 key |
| `close(c)` | 關閉 channel |
| `panic(v)` | 觸發 panic |
| `recover()` | 捕獲 panic |
| `clear(t)` | 清空 map 或 slice（1.21+） |
| `min(x, y, ...)` | 回傳最小值（1.21+） |
| `max(x, y, ...)` | 回傳最大值（1.21+） |

### 除錯函數
> `print`、`println` 為內建 debug 函數，輸出至 stderr<br>
> 正式環境請用 `fmt` 或 `log/slog`（1.21+）

| 函數 | 說明 |
|-|-|
| `print(args...)` | 輸出至標準錯誤（僅供除錯） |
| `println(args...)` | 輸出至標準錯誤並換行（僅供除錯） |

### `Slice` & `Array`

#### 特性差異

| 特性 | Array | Slice |
|-|-|-|
| 宣告 | `[n]int` | `[]int` |
| 大小 | 固定 | 動態 |
| 型別 | 值 | 引用 |
| 傳遞 | 完整複製 | 複製 header |
| 容量 | len = cap | len ≤ cap |

#### `Array`

```go
// Array 建立
arr := [3]int{1, 2, 3}      // [1,2,3]
arr2 := arr                 // [1,2,3] (深拷貝)
arr2[0] = 99                // arr=[1,2,3], arr2=[99,2,3] (互不影響)
// arr = append(arr, 4)     // (錯誤) 大小固定不支持添加
```

#### `Slice`

```go

// Slice 建立
s1 := make([]int, 5)        // [0,0,0,0,0]
s2 := make([]int, 0, 10)    // []
s3 := []int{1, 2, 3}        // [1,2,3]

length := len(s3)           // 3
capacity := cap(s3)         // 3

// 新增
s2 = append(s2, 1)          // [1]
s2 = append(s2, 2, 3, 4)    // [1,2,3,4]
s2 = append(s2, s3...)      // [1,2,3,4,1,2,3]

// 修改
s3[0] = 10                  // [10,2,3]

// 讀取
val := s3[0]                // 10

// 切片
sub := s3[1:3]              // [2,3]
sub2 := s3[:2]              // [10,2]
sub3 := s3[1:]              // [2,3]
copy := append([]int(nil), s3...)  // 深拷貝

// 移除 index = i 元素
// s3[:i] = s3[:1] = [10]
// s3[i+1:] = s3[2:] = [3]
i := 1
s3 = append(s3[:i], s3[i+1:]...)   // [10,3]

// 移除 last 元素
s3 = s3[:len(s3)-1]         // [10]

// 清空（1.21+）
clear(s3)                   // s=[0]，len 和 cap 不變
```

### `Map`

```go
// 建立
m1 := make(map[string]int)              // map[]
m2 := make(map[string]int, 100)         // map[]
m3 := map[string]int{"a": 1, "b": 2}    // map[a:1 b:2]

// 新增
m1["key"] = 100             // map[key:100]

// 修改
m3["a"] = 200               // map[a:200,b:2]

// 讀取
val, exists := m1["key"]    // val=100, exists=true
val2, exists2 := m1["a"]    // val2=0, exists2=false
val3 := m1["key"]           // val3=100

// 刪除
delete(m1, "key")           // map[]
delete(m1, "a")             // key 不存在 (略過動作)

// loop
for k, v := range m3 {
    // k: key, v: value
}

// 清空（1.21+）
clear(m3)                   // map[] (但記憶體未釋放)
```

### `Channel` & `Select`

#### `Channel`

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

#### `Select`

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

###### `Select` 注意事項

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

- `make` 與 `new` 的差異
    ```go
    s1 := make([]int, 5)    // [0,0,0,0,0]
    s2 := new([]int)        // *[]int

    // 差異
    s1 = append(s1, 1)      // [0,0,0,0,0,1]
    // s2 = append(s2, 1)   // (錯誤) s2 無法直接 append
    *s2 = append(*s2, 1)    // [1]，*s2 是 nil slice，所以會建立新 slice
    ```
- `append` 可能觸發記憶體重新配置
    ```go
    s := make([]int, 0, 2)
    s = append(s, 1, 2)     // [1,2]，len=2, cap=2
    s = append(s, 3)        // 容量不足，觸發重新配置：
                            // 1. 配置新底層陣列（cap 擴展為 4）新容量 = 舊容量 × 2
                            // 2. 複製舊資料 [1,2] 到新陣列
                            // 3. 新增元素 3 → [1,2,3]
                            // 4. slice 指向新陣列
                            // 5. 舊陣列等待 GC 回收（若無其他引用）
    ```
- `copy` 回傳實際複製的元素數
    ```go
    src := []int{1, 2, 3, 4, 5}     // [1,2,3,4,5]
    dst := make([]int, 3)           // [0,0,0]
    n := copy(dst, src)             // n=3
                                    // dst=[1,2,3]
    ```
- `close` 已關閉的 channel 會 `panic`
    ```go
    ch := make(chan int)
    close(ch)
    // close(ch)            // panic
    ```
- `recover` 必須在 `defer` 中直接調用
    ```go
    func example() {
        defer func() {
            if r := recover(); r != nil {
                // recovered
            }
        }()
        panic("error")
    }
    ```

### 效能考量

- 預先分配容量避免多次擴展 (注意事項之二)
    ```go
    // 推薦
    s := make([]int, 0, 1000)
    for i := 0; i < 1000; i++ {
        s = append(s, i)
    }

    // 不推薦
    s := []int{}
    for i := 0; i < 1000; i++ {
        s = append(s, i)
    }

    // 推薦
    m := make(map[string]int, 100)

    // 不推薦
    m := make(map[string]int)
    ```
- `copy` vs `append` (深拷貝比較)
    ```go
    src := []int{1, 2, 3}
    
    // copy
    dst := make([]int, len(src))
    copy(dst, src)              // dst=[1,2,3], len=3, cap=3

    // append
    dst1 := append([]int(nil), src...)  // dst1=[1,2,3], len=3, cap=3
    
    // 容量不足時會觸發擴容
    dst2 := make([]int, 0, 2)   // len=0, cap=2
    dst2 = append(dst2, src...) // 需要 3 個空間但 cap=2 不足
                                // 觸發擴容：新 cap=4 (2×2)
                                // dst2=[1,2,3], len=3, cap=4
    // copy 更明確控制容量
    // append 可能配置超過實際需要的容量
    ```

### 型別斷言與型別切換

```go
var i any = "hello"
// var i interface{} = "hello"
s := i.(string)         // 成功
// n := i.(int)         // panic

if s, ok := i.(string); ok {
    // i 為字串
}

switch v := i.(type) {
case string:
    // i 為字串
case int:
    // i 為數字
default:
    // i 不符合字串與數字
}
```