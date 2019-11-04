# Sync(同步)

**[你可以在这里找到本章节的代码](https://github.com/quii/learn-go-with-tests/tree/master/sync)**

我们想创建一个counter来学习怎么安全的使用多线程

首先创建一个不安全的counter结构,然后在单线程的环境中来验证它是否能够正常工作

然后我们在多线程的环境下执行它,以验证它为什么是不安全的并且再修复它


## 首先写出第一个测试用例

首先counter结构需要实现两个方法,一个是实现内部的值自增,另一个是获取其内部的值

```go
func TestCounter(t *testing.T) {
	t.Run("incrementing the counter 3 times leaves it at 3", func(t *testing.T) {
		counter := Counter{}
		counter.Inc()
		counter.Inc()
		counter.Inc()

		if counter.Value() != 3 {			
			t.Errorf("got %d, want %d", counter.Value(), 3)
		}
	})
}
```

## 尝试运行该测试

```
./sync_test.go:9:14: undefined: Counter
```

## 根据报错信息,完善代码

Let's define `Counter`. 

```go
type Counter struct {
	
}
```

再次运行

```
./sync_test.go:14:10: counter.Inc undefined (type Counter has no field or method Inc)
./sync_test.go:18:13: counter.Value undefined (type Counter has no field or method Value)
```

再次根据报错信息完善代码

```go
func (c *Counter) Inc() {
	
}

func (c *Counter) Value() int {
	return 0
}
```

运行并显示测试失败

```
=== RUN   TestCounter
=== RUN   TestCounter/incrementing_the_counter_3_times_leaves_it_at_3
--- FAIL: TestCounter (0.00s)
    --- FAIL: TestCounter/incrementing_the_counter_3_times_leaves_it_at_3 (0.00s)
    	sync_test.go:27: got 0, want 3
```

## 完善代码使其通过测试

我们需要在counter内部保存一些状态,并且每当我们调用'Inc'方法的时候该状态值都会加一

```go
type Counter struct {
	value int
}

func (c *Counter) Inc() {
	c.value++
}

func (c *Counter) Value() int {
	return c.value
}
```

## 重构

为了方便的写出更多的子测试,可以把重复代码单独编写成一个assertCounter()函数

```go
t.Run("incrementing the counter 3 times leaves it at 3", func(t *testing.T) {
    counter := Counter{}
    counter.Inc()
    counter.Inc()
    counter.Inc()
    
    assertCounter(t, counter, 3)
})

func assertCounter(t *testing.T, got Counter, want int)  {
	t.Helper()
	if got.Value() != want {
		t.Errorf("got %d, want %d", got.Value(), want)
	}
}
```

## 下一步

看起来它的功能足够了,但是我们需要使其在多线程的环境中能够安全运行. 首先编写一个失败的案例

## Write the test first

```go
t.Run("it runs safely concurrently", func(t *testing.T) {
    wantedCount := 1000
    counter := Counter{}

    var wg sync.WaitGroup
    wg.Add(wantedCount)

    for i:=0; i<wantedCount; i++ {
        go func(w *sync.WaitGroup) {
            counter.Inc()
            w.Done()
        }(&wg)
    }
    wg.Wait()

    assertCounter(t, counter, wantedCount)
})
```

执行一个循环,每个循环中开启一个goroutine,并让其执行counter.Inc()

我们使用了[`sync.WaitGroup`](https://golang.org/pkg/sync/#WaitGroup),这是一个能够方便的让多线程同步的工具

> WaitGroup 能够等待一组goroutines执行完成. 在主线程中调用Add()函数来增加需要等待线程的数量. 当每个线程执行完成之后需要调用Done()方法.同事,Wait()方法可以等待所有线程执行完成.

## 尝试运行测试

```
=== RUN   TestCounter/it_runs_safely_in_a_concurrent_envionment
--- FAIL: TestCounter (0.00s)
    --- FAIL: TestCounter/it_runs_safely_in_a_concurrent_envionment (0.00s)
    	sync_test.go:26: got 939, want 1000
FAIL
```

测试可能会以不同的运行结果而失败,但是尽管如此它能够证明该counter结构在多线程环境中是不能安全运行的.

## 修复代码,以使其能够通过测试

最简单的方式就是给`Counter`结构加一个lock(锁), 一个 [`Mutex`]锁(https://golang.org/pkg/sync/#Mutex)

> Mutex 是一种互斥锁. Mutex的值为0表示解锁状态

```go
type Counter struct {
	mu sync.Mutex
	value int
}

func (c *Counter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.value++
}
```

其意义在于,每个想要改变couter的值的线程都必须要先获取到锁.如果当前有线程正在使用counter.value的值,那么其他线程必须等待其释放锁.

再次运行测试,就会通过测试

## 还有其他使用`sync.Mutex`的例子, 就是把锁耦合在结构体中.

比如像这样的

```go
type Counter struct {
	sync.Mutex
	value int
}
```

然后这样调用

```go
func (c *Counter) Inc() {
	c.Lock()
	defer c.Unlock()
	c.value++
}
```
看起来代码变得简介了,但实际上这是完全错误的做法

有时候开发人员会忘记,把一种类型嵌入到另一种类型意味着这种类型的方法成为了被嵌入类型公开接口的一部分.但是这是大部分开发者所不希望达到的.我们应该非常谨慎的对待类型的公开接口,因为公开接口可以让别的代码去调用而可能改变被调用代码内部的数据.

所以像上面这种方式直接暴露 Lock和Unlock是非常危险的行为.


![API的使用者改变锁的状态](https://i.imgur.com/SWYNpwm.png)

_看起来是个糟糕个方式_

## 复制锁

虽然我们的代码能够通过测试,但是它依然是危险的.

如果你运行 `go vet` 命令,你就会看到如下的报错信息.

```
sync/v2/sync_test.go:16: call of assertCounter copies lock value: v1.Counter contains sync.Mutex
sync/v2/sync_test.go:39: assertCounter passes lock by value: v1.Counter contains sync.Mutex
```

看会看锁这部分的文档 [`sync.Mutex`](https://golang.org/pkg/sync/#Mutex) tells us why

>锁一但使用,绝不允许拷贝

当我们传递 `Counter` (值传递) 到 `assertCounter` 实际上会拷贝一份 mutex的副本. 

为了解决这个问题,我们可以使用指针传递`Counter`, 改变函数`assertCounter`

```go
func assertCounter(t *testing.T, got *Counter, want int)
```

现在编译会不通过,因为我们尝试传递一个`Counter`而不是一个`*Counter`.解决这个问题的最好方式是编写一个构造方法.

```go
func NewCounter() *Counter {
	return &Counter{}
}
```

然后使用构造方法NewCounter()来初始化Counter

## 总结

我们涉及到了一些新的东西[sync package](https://golang.org/pkg/sync/)

- `Mutex` 允许我们给特定数据加上锁
- `Waitgroup` 可以等待goroutine完成

### 什么时候在通道和线程上使用锁?

[复习第一章节学到的多线程](concurrency.md) 它已经可以安全的实现多线程了,为什么我们还需要用到锁?   
[go 百科关于这部分的解释](https://github.com/golang/go/wiki/MutexOrChannel)

> go语言新手常犯的一个错误是过度使用channels和goroutines,仅仅是因为它可以做到(某些事)或者是因为它很有趣.不要害怕使用sync.Mutex,如果它能够更好的修复你的bug. go语言是一个实用的工具用来帮你更好的解决实际问题的而并不是强迫你使用一种代码风格.

关键词:

- **传递数据所有权时使用通道** 
- **使用锁管理数据状态**

### go vet

使用go vet可以发现一些细微的bug

### 不要因为方便而使用嵌入类型

- 思考嵌入类型会对公开API有什么影响?
- 是否真的需要公开API?
- 对于互斥锁，这可能会以非常难以预测和怪异的方式造成灾难性的后果，想象一下一些恶意代码在不应该互斥时解锁互斥锁; 这将导致一些非常奇怪的错误，并将很难对其进行跟踪。
