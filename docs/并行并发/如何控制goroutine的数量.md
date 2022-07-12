## 问题

在Golang中，Goroutine虽然很好，但是数量太多了，往往会带来很多麻烦，比如耗尽系统资源导致程序崩溃，或者CPU使用率过高导致系统忙不过来。

所以我们可以限制下Goroutine的数量,这样就需要在每一次执行go之前判断goroutine的数量，如果数量超了，就要阻塞go的执行。


## 解决方案

### 解决方案一：构建简单的Go程池

在我们日常大部分场景下，不需要使用协程池。因为Goroutine非常轻量，默认2kb，使用go func()很难成为性能瓶颈。当然一些极端情况下需要追求性能，可以使用协程池实现资源的复用，例如FastHttp使用协程池性能提高许多。

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"time"
)

// Pool Goroutine Pool
type Pool struct {
	queue chan int
	wg *sync.WaitGroup
}

// New 新建一个协程池
func NewPool(size int) *Pool{
	if size <=0{
		size = 1
	}
	return &Pool{
		queue:make(chan int,size),
		wg:&sync.WaitGroup{},
	}
}

// Add 新增一个执行
func (p *Pool)Add(delta int){
	// delta为正数就添加
	for i :=0;i<delta;i++{
		p.queue <-1
	}
	// delta为负数就减少
	for i:=0;i>delta;i--{
		<-p.queue
	}
	p.wg.Add(delta)
}

// Done 执行完成减一
func (p *Pool) Done(){
	<-p.queue
	p.wg.Done()
}

// Wait 等待Goroutine执行完毕
func (p *Pool) Wait(){
	p.wg.Wait()
}

func main(){
	// 这里限制5个并发
	pool := NewPool(5)
	fmt.Println("开始时Go程数量:",runtime.NumGoroutine())
	for i:=0;i<20;i++{
		pool.Add(1)
		go func(i int) {
			time.Sleep(time.Second)
			fmt.Println("the NumGoroutine continue is:",runtime.NumGoroutine())
			pool.Done()
		}(i)
	}
	pool.Wait()
	fmt.Println("运行完成的Go程数量:",runtime.NumGoroutine())
}
```

### 解决方案二：同一，但是不尽兴池化


```go
package main

import (
  "fmt"
  "sync"
  "time"
)

type Glimit struct {
  n int
  c chan struct{}
}

// initialization Glimit struct
func New(n int) *Glimit {
  return &Glimit{
    n: n,
    c: make(chan struct{}, n),
  }
}
// Run f in a new goroutine but with limit.
func (g *Glimit) Run(f func()) {
  g.c <- struct{}{}
  go func() {
    f()
    <-g.c
  }()
}

var wg = sync.WaitGroup{}

func main() {
  number := 10
  g := New(2)
  for i := 0; i < number; i++ {
    wg.Add(1)
    value :=i
    goFunc := func() {
      // 做一些业务逻辑处理
      fmt.Printf("go func: %d\n", value)
      time.Sleep(time.Second)
      wg.Done()
    }
    g.Run(goFunc)
  }
  wg.Wait()
}

```

目前github上已经有开源的项目ants来实现 Goroutine 池。ants已经实现了对大规模 Goroutine 的调度管理、Goroutine 复用，允许使用者在开发并发程序的时候限制 Goroutine 数量，复用资源，达到更高效执行任务的效果。

