`go`语言允许通过`runtime.SetFinalizer`函数对某个变量做一些析构前的操作。比如下面代码

```go
import (
	"fmt"
	"runtime"
	"time"
)

func testGc(aa *int){
	fmt.Println("+++++ testGc")
}


func main(){
	aa := new(int)
	runtime.SetFinalizer(aa, testGc)

	for i := 10; i > 0; i--{ //这个循环是必需
		time.Sleep(time.Second)
		runtime.GC()
	}

	fmt.Println("hello world")
}
```



输出为

```text
+++++ testGc
hello world
```



这个`runtime.SetFinalizer`可以用到业务代码的什么地方呢？ 可以用到减少显示关闭或在释放某些资源上。以[go-cache](https://github.com/patrickmn/go-cache)为例。它有一个结构体`cache`用于管理`key-value`存储，同时为了能够定期清除过期的`key`，为此它单独起了一个协程专门负责定期清理，关键代码如下:

```go
type Item struct {
	Object     interface{}
	Expiration int64
}

type cache struct {
	items             map[string]Item //用于存放key-value
	mu                sync.RWMutex
	janitor           *janitor
}

func (j *janitor) Run(c *cache) {
	ticker := time.NewTicker(j.Interval)
	for {
		select {
		case <-ticker.C:
			c.DeleteExpired()//清理过期key
		case <-j.stop:
			ticker.Stop()
			return
		}
	}
}
```

可以观察到`Run`函数里面是一个死循环，并且是以`*cache`作为参数的。

如果有一个`*cache`变量`c`，一开始在用着它，后面不用了，或者执行`c = nil`。此时其对应的资源并没有得到释放，因为还有一个协程还在跑着`Run`函数，该函数还有一个变量引用着该资源。最简单的解决方案是增加一个关闭函数

```go
func (c *cache) Close(){
    c.janitor.stop <- true
}
```

由于`go`没有析构函数，这个关闭很难保证一定会执行。此外业务代码中也不想出现这些跟资源释放有关的代码。



此时`runtime.SetFinalizer`就派上用场了。但光引入`runtime.SetFinalizer`还不够，因为`Run`函数引用着一个`*cache`变量这个事实并没有改变，也就是说资源还有人用着，触发不了`GC`。为了解决这个问题，`go-cache`引入了另外一个结构体

```go
type Cache struct {
	*cache
	// If this is confusing, see the comment at the bottom of New()
}
```



关键代码如下，可以观察到，`New`返回的是`*Cache`变量，而不是`Run`参数的变量`*cache`。这样在外部将`*Cache`变量置为`nil`后会使得资源没有其他引用了，从而触发`stopJanitor`的调用。

```go
func New()*Cache{
    c := newCache()
	C := &Cache{c} //通过小写c创建大写C
    
    runJanitor(c, ci) //会启动一个协程执行Run()
	runtime.SetFinalizer(C, stopJanitor)//对大写C调用runtime.SetFinalizer
	
    return C
}	


func stopJanitor(c *Cache) {
	c.janitor.stop <- true
}
```

