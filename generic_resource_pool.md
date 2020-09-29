## 通用资源池的思考

一个通用pool大概长这样：

```go
type Pool struct {
	idle []*resource // 空闲资源池
	all []*resource // 所有资源池
	max int // 资源最大数量控制
	maxIdle int // 最大idle数量控制
	// cond *sync.Cond
}
```


当获取资源时，遵循以下逻辑：
1，如果idle里有，则直接获取
2，如果没有，并且all的数量小于max，则创建一个，并放入all
3，如果没有，并且len(all) >= max, 则等待其他线程或者协程返还资源，然后直接拿去使用。

如何等待其他线程释放资源呢？


标准库database包做法是创建一个map[uint64]chan connRequest

当等待资源时，监听connRequest，当有资源被释放时，检查map里有没有等待的请求，有则把资源传给connRequest

我之前找到一个挺好的通用资源池库puddle,它的做法是：
pool里存放一个sync.Cond，通过cond.Wait()来等待资源释放，
当其他线程释放资源时，调用cond.Signal(),通知等待的某个groutine重新获取资源，当资源池被关闭，则cond.Broadcast()。

但是等待资源时，如果有cancel context，则还需要监听ctx.Done(),puddle的做法是，建立一个channel，然后新开启一个线程调用cond.Wait()阻塞住,
当阻塞解除时，就传值给channel，原先的协程里，监听ctx.Done()和channel，这样就实现了同时监听ctx.Done和cond通知了。


这里有另一个问题：

资源有可能会过期或者失效，这个时候需要我们有个容错机制，当资源失效时，我们需要重新获取资源。

标准库database的做法是，当从idle池里获取每次获取conn时，支持检查conn有没有过期，如果过期则抛弃。

即使没过期，资源也可能失效，所以获取到conn后，执行操作时，如果conn失效，则再重试一次。
优点是一般情况下都有效，只需要一次，缺点：很多操作，那么每个操作都要重试，代码难看。但是可以通过封装代码来统一来做重试操作

还有种做法是获取到资源监测下资源是否有效，优点：可以统一检查，不要到处都写检查的代码。
缺点：每次都检查，可能大部分资源在大多数时候都有效，但是还是需要去检查是否有效，造成新能影响。


