## 通用资源池的思考

在我们的项目中，会使用到各种资源池，比如数据库连接池、redis连接池、http请求连接池等等，每一个连接我们都可以看作一个资源，

我自己总结了它们的一些特性：

- 如果不进行复用，频繁的创建销毁资源，会消耗大量的时间，影响总体性能。
- 适当数量的资源能够提升性能
- 连接可能随时失效

所以针对这些特性，我们需要创建一定数量的资源，并且尽量复用这些资源，不要频繁创建销毁，资源池就是根据这些原则被设计出来的。

golang的数据库基础库database自带一个资源池，对上层保持透明，让开发者减少管理数据库连接的负担。

我们平时使用的grpc、redis等等都有实现连接池。

当我们开发新的需要长连接远程服务的客户端时，我们都应该实现连接池，但是每个库都重新实现并测试一遍资源池，费时又费力。

所以设计一个通用的资源池还是很有用处的。


一个通用pool大概长这样：

```go
type Pool struct {
	idle []*Resource // 空闲资源池
	all []*Resource // 所有资源池
	max int // 资源最大数量控制
	maxIdle int // 最大idle数量控制
	constructor Constructor // 通用创建资源方法
	destructor  Destructor // 通用销毁资源方法
	// cond *sync.Cond
}

type Resource struct {
	value        interface{}
	pool         *Pool
	createdAt time.Time
	lastUsedAt time.Time
	status       byte // idle/inUsing等等
}

type Constructor func(ctx context.Context) (res interface{}, err error)

type Destructor func(res interface{})
```

资源创建和销毁的方法由开发者项目自己实现，以供pool调用。

resource的createdAt用来记录资源创建时间，lastUsedAt记录上次被使用的时间。

当资源被放在idle池里面时间长了，可能会失效，我们可以通过检查createdAt和lastUsedAt判断如果超过一定值直接失效。


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

标准库database的做法是，当从idle池里获取每次获取conn时，如果conn是从idle池里拿的，并且生命周期超过一定值，就抛出错误ErrBadConn
当创建资源失败时也返回ErrBadConn

上层函数拿到conn执行操作时，如果得到错误ErrBadConn，则再重试一次。

还有很多思路，比如定期让idle池里的资源ping，实现心跳效果，来保活资源。

或者拿到资源时ping一下，只是这种做法会导致大量没必要的ping，因为正常情况下可能大多数idle资源是正常的。


