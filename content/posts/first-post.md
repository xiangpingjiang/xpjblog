---
date: '2025-05-25T17:53:12+08:00'
draft: false
title: 'goroutine 的并发安全'
---


Go 协程 (goroutine) 本身是轻量级的执行线程，其本身并没有内置的并发安全机制。并发安全问题通常发生在多个 goroutine 访问和修改共享内存中的同一数据时。

## 并发不安全的场景示例

### 计数器并发自增

```go
var num int

func increment() {
    num++ // 非原子操作，并发不安全
}

func main() {
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            increment()
        }()
    }
    wg.Wait()
    fmt.Println(num) // 结果通常小于 1000
}
```

num++包含读取、计算、写入三个步骤，多个 goroutine 可能同时读取到旧值并进行写入，导致部分增加操作失效

### 切片并发追加

```go
var s []int

func appendValue(i int) {
    s = append(s, i) // 并发 append，不安全
}

func main() {
    for i := 0; i < 10000; i++ {
        go appendValue(i)
    }
    time.Sleep(time.Second)
    fmt.Println(len(s)) // 结果很可能小于 10000
}
```

多个 goroutine 可能同时进入 append 逻辑，一些 goroutine 的追加操作会被覆盖


## 解决方案

### 互斥锁 (sync.Mutex)


适用场景： 保护临界区，任意时刻只有一个 goroutine 可访问
```go
var num int
var mu sync.Mutex // 声明互斥锁

func increment() {
	mu.Lock()         // 获取锁
	defer mu.Unlock() // 确保函数返回前释放锁
	num++             // 现在这是安全的临界区操作
}

func main() {
	var wg sync.WaitGroup
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			increment()
		}()
	}
	wg.Wait()
	fmt.Println(num) // 现在结果始终为 1000
}
```

### 读写锁 (sync.RWMutex)

适用场景： 读多写少的场景，允许多个读并发

```go
type Counter struct {
	value   int
	rwMutex sync.RWMutex // 使用读写锁保护共享资源
}

// 读取计数器值
func (c *Counter) GetValue() int {
	c.rwMutex.RLock()         // 获取读锁
	defer c.rwMutex.RUnlock() // 函数返回前释放读锁
	return c.value
}

// 增加计数器值
func (c *Counter) Increment() {
	c.rwMutex.Lock()         // 获取写锁
	defer c.rwMutex.Unlock() // 函数返回前释放写锁
	c.value++
}

func main() {
	counter := &Counter{value: 0}

	// 启动多个读协程
	for i := 0; i < 5; i++ {
		go func(id int) {
			for j := 0; j < 3; j++ {
				val := counter.GetValue()
				fmt.Printf("Goroutine %d 读取到值: %d\n", id, val)
				time.Sleep(25 * time.Millisecond)
			}
		}(i)
	}

	// 启动一个写协程
	go func() {
		for i := 0; i < 3; i++ {
			counter.Increment()
			fmt.Printf("写协程将计数器增加到: %d\n", i+1)
			time.Sleep(25 * time.Millisecond) // 模拟写操作耗时
		}
	}()

	time.Sleep(2 * time.Second) // 等待所有协程执行完毕
	fmt.Println("最终计数器值:", counter.GetValue())
}

```

```
写协程将计数器增加到: 1
Goroutine 0 读取到值: 1
Goroutine 3 读取到值: 1
Goroutine 4 读取到值: 1
Goroutine 2 读取到值: 1
Goroutine 1 读取到值: 1
Goroutine 2 读取到值: 1
Goroutine 0 读取到值: 1
Goroutine 3 读取到值: 2
写协程将计数器增加到: 2
Goroutine 4 读取到值: 1
Goroutine 1 读取到值: 2
Goroutine 4 读取到值: 2
Goroutine 3 读取到值: 2
Goroutine 2 读取到值: 3
写协程将计数器增加到: 3
Goroutine 0 读取到值: 3
Goroutine 1 读取到值: 3
最终计数器值: 3
```