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

It can be argued that it can make the code a bit more elegant.

```go
func (c *Counter) Inc() {
	c.Lock()
	defer c.Unlock()
	c.value++
}
```

This _looks_ nice but while programming is a hugely subjective discipline, this is **bad and wrong**. 

Sometimes people forget that embedding types means the methods of that type becomes _part of the public interface_; and you often will not want that. Remember that we should be very careful with our public APIs, the moment we make something public is the moment other code can couple themselves to it. We always want to avoid unnecessary coupling.  

Exposing `Lock` and `Unlock` is at best confusing but at worst potentially very harmful to your software if callers of your type start calling these methods.

![Showing how a user of this API can wrongly change the state of the lock](https://i.imgur.com/SWYNpwm.png)

_This seems like a really bad idea_

## Copying mutexes

Our test passes but our code is still a bit dangerous

If you run `go vet` on your code you should get an error like the following

```
sync/v2/sync_test.go:16: call of assertCounter copies lock value: v1.Counter contains sync.Mutex
sync/v2/sync_test.go:39: assertCounter passes lock by value: v1.Counter contains sync.Mutex
```

A look at the documentation of [`sync.Mutex`](https://golang.org/pkg/sync/#Mutex) tells us why

> A Mutex must not be copied after first use.

When we pass our `Counter` (by value) to `assertCounter` it will try and create a copy of the mutex. 

To solve this we should pass in a pointer to our `Counter` instead, so change the signature of `assertCounter`

```go
func assertCounter(t *testing.T, got *Counter, want int)
```

Our tests will no longer compile because we are trying to pass in a `Counter` rather than a `*Counter`. To solve this I prefer to create a constructor which shows readers of your API that it would be better to not initialise the type yourself.

```go
func NewCounter() *Counter {
	return &Counter{}
}
```

Use this function in your tests when initialising `Counter`.

## Wrapping up

We've covered a few things from the [sync package](https://golang.org/pkg/sync/)

- `Mutex` allows us to add locks to our data
- `Waitgroup` is a means of waiting for goroutines to finish jobs

### When to use locks over channels and goroutines?

[We've previously covered goroutines in the first concurrency chapter](concurrency.md) which let us write safe concurrent code so why would you use locks?   
[The go wiki has a page dedicated to this topic; Mutex Or Channel](https://github.com/golang/go/wiki/MutexOrChannel)

> A common Go newbie mistake is to over-use channels and goroutines just because it's possible, and/or because it's fun. Don't be afraid to use a sync.Mutex if that fits your problem best. Go is pragmatic in letting you use the tools that solve your problem best and not forcing you into one style of code.

Paraphrasing:

- **Use channels when passing ownership of data** 
- **Use mutexes for managing state**

### go vet

Remember to use go vet in your build scripts as it can alert you to some subtle bugs in your code before they hit your poor users.

### Don't use embedding because it's convenient

- Think about the effect embedding has on your public API.
- Do you _really_ want to expose these methods and have people coupling their own code to them?
- With respect to mutexes, this could be potentially disastrous in very unpredictable and weird ways, imagine some nefarious code unlocking a mutex when it shouldn't be; this would cause some very strange bugs that will be hard to track down.
