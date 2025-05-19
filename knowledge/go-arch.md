## Go 语言核心知识笔记

---

## 1. Channel

### 1.1 基本概念

* channel 是 goroutine 之间通信的主要方式，具有类型安全性。
* 类似 CSP（通信顺序进程）模型，用于传递数据。

### 1.2 创建与使用

```go
ch := make(chan int)          // 无缓冲 channel
ch := make(chan int, 3)       // 有缓冲 channel，容量为 3
```

### 1.3 读写阻塞行为

* 无缓冲 channel：写入会阻塞直到另一个 goroutine 读取。
* 有缓冲 channel：缓冲未满时写入不阻塞，满时阻塞；读取时缓冲为空则阻塞。

### 1.4 关闭 channel

* 使用 `close(ch)` 关闭 channel。
* 关闭后不能写入，可以继续读取，读取会返回零值和 false。

### 1.5 注意事项

* 读写 nil channel 会永久阻塞。
* 关闭 nil channel 会 panic。
* 关闭已关闭的 channel 会 panic。
* 向已关闭的 channel 写数据会 panic。

### 1.6 底层结构（简化）

```go
type hchan struct {
    qcount   uint      // 队列中元素个数
    dataqsiz uint      // 队列容量
    buf      unsafe.Pointer // 缓冲区指针
    sendx    uint      // 发送索引
    recvx    uint      // 接收索引
    recvq    waitq     // 等待读队列
    sendq    waitq     // 等待写队列
    lock     mutex
}
```

---

## 2. Slice

### 2.1 底层结构

```go
type slice struct {
    array unsafe.Pointer // 底层数组指针
    len   int
    cap   int
}
```

### 2.2 基本使用

```go
s := []int{1, 2, 3}
s = append(s, 4)
```

### 2.3 注意事项

* append 可能导致数组扩容，新 slice 与原 slice 指向不同底层数组。
* slice 是引用类型，多个 slice 可能共享底层数组。
* 并发读写 slice 不安全。
* 使用 copy 进行深拷贝。

---

## 3. Map

### 3.1 初始化方式

```go
m := make(map[string]int)
m := map[string]int{"a": 1, "b": 2}
```

### 3.2 注意事项

* 并发读写 map 会造成竞态，需加锁或使用 sync.Map。
* delete 实际只是标记删除，回收需等到 map 缩容。

### 3.3 底层结构（简化）

```go
type hmap struct {
    count     int      // 元素数量
    buckets   *bmap    // bucket 数组
    oldbuckets *bmap   // 扩容前的旧 bucket
    flags     uint8
}
```

* buckets：数组，存放 key-value。
* 每个 bucket 最多 8 个元素，超出后用 overflow bucket 链接。

---

## 4. Goroutine & GMP 调度

### 4.1 概念

* Goroutine 是 Go 的轻量级线程，由调度器管理。
* GMP 模型：G（goroutine），M（操作系统线程），P（调度器处理器）

### 4.2 GMP 模型简图

* G 存储代码逻辑、上下文；
* P 是调度核心，维护 G 队列；
* M 是实际执行 G 的内核线程；

### 4.3 调度过程

* 全局队列、P 本地队列、netpoller 管理任务调度。
* 存在 work stealing，从其他 P 抢任务。

---

## 5. GC（垃圾回收）

### 5.1 三色标记清除法

* 白色：未访问，需回收
* 灰色：已访问，子节点未处理
* 黑色：已访问，子节点已处理

### 5.2 STW（Stop the World）

* 在 GC 的起始和终止阶段，需要暂停所有 goroutine。
* Go 采用并发 GC，尽量减少 STW 时间。

### 5.3 内存分配策略

* 小对象使用 per-P 分配缓存（mcache）
* 大对象直接向操作系统申请内存

---

## 6. Context

### 6.1 作用

* 用于控制 goroutine 生命周期，传递取消信号、截止时间、超时等。

### 6.2 使用方式

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
```

### 6.3 常见类型

* `context.Background()`：根 context
* `context.TODO()`：占位用
* `context.WithCancel(parent)`
* `context.WithTimeout(parent, duration)`
* `context.WithDeadline(parent, time)`

---

## 📌 常见面试问题

* 为什么 map 是并发不安全的？如何解决？
* slice 扩容规则？底层是如何变化的？
* channel 关闭后还能读吗？为什么？
* GMP 是如何进行调度的？
* GC 为什么采用三色标记？如何避免误回收？
