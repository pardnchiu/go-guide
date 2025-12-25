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
