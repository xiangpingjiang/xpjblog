---
date: '2025-05-25T17:53:12+08:00'
draft: false
title: '分布式锁'
---
## 何为分布式锁？
分布式锁是控制分布式系统同步访问共享资源的一种机制，它允许在分布式环境下的多个节点（服务器、服务或进程）之间进行协调，确保在同一时刻，只有一个节点可以执行特定的操作或访问特定的资源


### 为何需要分布式锁？
在单机服务器上，我们可以使用Java中的synchronized关键字或Lock接口来保证同一时间只有一个线程执行某段代码，这被称为线程锁。然而，当应用扩展到分布式架构，同一个应用部署在多台服务器上时，线程锁就无法在不同服务器的JVM进程之间起作用了

### 应用场景
分布式锁正是为了解决分布式环境下的并发问题而生的，主要应用于以下场景

· **避免数据错误**：例如在“秒杀”活动中，防止超量的商品被成功下单，确保库存数据的准确性。

· **防止系统过载**：在高并发场景下（如淘宝双11），通过锁机制控制访问数据库的流量，保护后端系统不被冲垮。

· **保证任务幂等性**：确保重要的任务（如重复提交、定时任务调度）不会被多个节点重复执行。

### 分布式锁的关键特性
一个可靠的分布式锁通常应具备以下特性：

· **互斥性**：这是最核心的特性，必须保证在任何时刻，锁只能被一个节点持有。  
· **高可用性**：获取锁和释放锁的操作必须高可用，锁服务本身不能成为单点故障。  
· **可重入性**：同一个节点内的同一个线程可以多次获取同一把锁，避免死锁。  
· **锁失效机制**：锁必须有超时时间，防止因为持有锁的节点崩溃而导致锁永远无法释放（死锁）。  
· **持锁人解锁**：加锁和解锁必须是同一个节点，不能出现节点A释放了节点B持有的锁的情况。

## 如何实现分布式锁

### 借助 redis 实现

把这个问题抛给 gpt ， 得到一个流程图如下：

![](/photo/redis_lock.png)


### 参考 go 语言 redsync 的实现

[redsync](https://github.com/go-redsync/redsync) :  使用 Redis 实现 Go 语言的分布式互斥锁。



redsync 使用示例

```go
package main

import (
	goredislib "github.com/redis/go-redis/v9"
	"github.com/go-redsync/redsync/v4"
	"github.com/go-redsync/redsync/v4/redis/goredis/v9"
)

func main() {
	// Create a pool with go-redis (or redigo) which is the pool redsync will
	// use while communicating with Redis. This can also be any pool that
	// implements the `redis.Pool` interface.
	client := goredislib.NewClient(&goredislib.Options{
		Addr: "localhost:6379",
	})
	pool := goredis.NewPool(client) // or, pool := redigo.NewPool(...)

	// Create an instance of redsync to be used to obtain a mutual exclusion
	// lock.
	rs := redsync.New(pool)

	// Obtain a new mutex by using the same name for all instances wanting the
	// same lock.
	mutexname := "my-global-mutex"
	mutex := rs.NewMutex(mutexname)

	// Obtain a lock for our given mutex. After this is successful, no one else
	// can obtain the same lock (the same mutex name) until we unlock it.
	if err := mutex.Lock(); err != nil {
		panic(err)
	}

	// Do your work that requires the lock.

	// Release the lock so other processes or threads can obtain a lock.
	if ok, err := mutex.Unlock(); !ok || err != nil {
		panic("unlock failed")
	}
}
```

获取锁代码：


```go

func (m *Mutex) acquire(ctx context.Context, pool redis.Pool, value string) (bool, error) {
	conn, err := pool.Get(ctx)
	if err != nil {
		return false, err
	}
	defer conn.Close()
	reply, err := conn.SetNX(m.name, value, m.expiry)
	if err != nil {
		return false, err
	}
	return reply, nil
}
```



释放锁代码：


```go

func (m *Mutex) release(ctx context.Context, pool redis.Pool, value string) (bool, error) {
	conn, err := pool.Get(ctx)
	if err != nil {
		return false, err
	}
	defer conn.Close()
    // deleteScript 为 lua 脚本
	status, err := conn.Eval(deleteScript, m.name, value)
	if err != nil {
		return false, err
	}
	if status == int64(-1) {
		return false, ErrLockAlreadyExpired
	}
	return status != int64(0), nil
}

```

deleteScript
```go
var deleteScript = redis.NewScript(1, `
	local val = redis.call("GET", KEYS[1])
	if val == ARGV[1] then
		return redis.call("DEL", KEYS[1])
	elseif val == false then
		return -1
	else
		return 0
	end
`, "e950836ed1e694540c503ef9972b8de518044d3b")
```