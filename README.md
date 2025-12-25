# Go Guide

> Golang 開發常用指南與實踐<br>
> 講解 Go 語言開發過程中的核心知識、標準庫使用方式及常見問題解答，提供快速查閱與學習資源。

## 內容結構

### 核心語言與執行環境

- [types](./core-language-and-runtime/types.md) - Slice、Array、Map 與型別
- [channel-select](./core-language-and-runtime/channel-select.md) - Channel 與 Select 並發控制
- [runtime](./core-language-and-runtime/runtime.md) - Goroutine、記憶體、GC 與系統資訊
- [runtime/debug](./core-language-and-runtime/runtime-debug.md) - Build 資訊、GC 調校、Stack dump
- [runtime/pprof](./core-language-and-runtime/runtime-pprof.md) - CPU 與記憶體 profiling
- [runtime/metrics](./core-language-and-runtime/runtime-metrics.md) - 運行時性能指標

## 版本說明

- 文件內容以 **Go 1.20+** 後為主
- 主要針對標準庫與常用庫
- 標註新版本特性（如 1.21+）
