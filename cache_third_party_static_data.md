# 如何优雅的缓存来自第三方的静态数据。

关于如何缓存数据的文章已经是满天飞了，我们这里讨论下如何缓存来自第三方的静态数据。

本文讨论一种简单可行的设计思路，如果您发现有问题或建议，欢迎一起讨论。

为了简单直观，我们设计一个简单的应用场景：

我们的系统里有很多地方需要地区信息（国家或地区/省份/城市），比如用户信息、公司信息、位置分享等等，这些信息通过id记录地区信息。

国家或地区/省份/城市信息来自另外一个服务，所以我们需要去外部服务获取，它们算是静态数据，不会动态更新。

因为系统中很多地方都会获取这些信息，比如用户信息展示、批量查询、聊天展示等等，

这些常用静态数据可能访问量巨大，我们可以将这些信息缓存在本地内存中，这样我们就能尽可能的减少rpc调用，提升性能。

现在我们来讨论如何缓存这些静态数据。

国家/省份/城市的数据量大概在：200， 5000， 13w.

我们可以在应用启动的时候去全量获取数据，如果数据比较多的话，比如城市数据，可能需要多次获取，因为有的服务调用方式比如grpc，
会有数据大小限制。
这个方案显然不太好，启动就去获取数据，会影响应用启动时间，另外如果获取失败了，会导致启动失败，要不然这个数据就获取不到了。
但是可能这个数据不是很重要，不影响系统正常运行，因为这个缓存获取失败就启动失败是不合理的。
另外，对于城市这种量比较大的数据，如果我们全量更新，太浪费内存了。

更好的方案应该是像我们平时采用的缓存方案，按需缓存。

对于国家这种数据量比较小，我们可以全量缓存，我们定义获取国家信息的函数：

```
type countryCacheT struct {
    data map[int64]*Country
}
func retrieveCountry(id int64) (*Country, bool, error){
    if cache not built{
        buildCache()
        return cached data
    }else{
        return cached data
    }
}
```
在这个函数中，第一次请求去第三方服务请求数据，并构建缓存，后续的请求就只从缓存查询。

最简单的做法是给cache加把读写锁，第一个获取到锁的请求去第三方请求数据并构建缓存，后续的请求都访问缓存。

从性能提升的角度来看，除了第一个拿到锁并成功构建缓存的请求，后续所有请求都是读缓存，但是还是要执行加锁解锁操作，要是能优化掉就好了。

自然的，我想到了sync.Once，但是sync.Once不满足我们的要求，为什么呢？

我们去第三方拿数据可能会失败，失败了我们需要再继续尝试去获取并构建缓存，而sync.Once做不到，他不管你发生了什么，就只给你执行一次。

sync.Once很简单，尝试改造下：

```go
type Once struct {
	done uint32
	m    sync.Mutex
}

func (o *Once) Do(f func() error) error {

	if atomic.LoadUint32(&o.done) == 0 {
		return o.doSlow(f)
	}
	return nil
}

func (o *Once) doSlow(f func() error) error {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		if err := f(); err != nil {
			return err
		}
		atomic.StoreUint32(&o.done, 1)
	}
	return nil
}
```

大致思路就是当Do函数失败的时候，后续的Do函数会继续执行我们提供的函数，直到成功为止。

那么我们用改造后的once去获取第三方数据并构建缓存，成功后，后续的操作就不用lock再unlock了，只会执行 `atomic.LoadUint32(&o.done)`的逻辑判断中
，这个操作就比加解锁快太多啦。


接下来讨论下渐进更新缓存。

```
var cache NewLru()
func retrieveCity(id int64) (*City, bool, error){
    if cached {
    return cached data
    }else {
        get data from third party
        update cache
        return data
    }
}
```

对于城市这种数据量比较大的，我们采用渐进式更新缓存：如果缓存有，使用缓存数据，否则去第三方获取，在更新缓存。

上面的函数有一个问题：

如果我们在批量数据中查询城市，比如用户列表，每个用户都有城市信息，可能再一次批量查询中，如果这些数据都没有缓存到，我们可能就会
在短时间内大量请求第三方获取数据。

这里我们可以简单的优化下：

遍历用户列表，

获取城市id列表，

遍历城市id列表，

在缓存中查询没有缓存到的id列表，

然后再根据这个未缓存列表id去第三方一次性查出数据，

再根据查询到的数据来更新缓存。

再次遍历用户列表，根据城市id查询缓存，再设置详细城市信息。

比之前那个减少了大量第三方请求调用，但是太啰嗦了，有木有，如果有很多信息批量查询，手都要写累。

我之前写了个库:**[batch](https://github.com/EchoUtopia/batch)**

可以简化这个过程：

```go
    import github.com/EchoUtopia/batch
    
    ***

	bFn := func(ctx context.Context, keys Keys) (map[Key]interface{}, error) {
        cityIds := make([]int64, 0, len(keys))
        for _, v := range keys {
            cityIds = append(cityIds, int64(key.(batch.Int64Key)))
        }   
        cities, err := GetBatchDataFromThirdParty(ctx, cityIds)
        if err != nil {
            return nil, err 
        }
        out := make(map[Key]interface{}, len(cities))
        for _, v := range cities {
            out[batch.Int64Key(v.CityID)] = v
        }   
        return out, nil
	}
	b := NewBatch(ctx, bFn, WithBatchSpan(100), WithCache(DefaultCache()))
	for user := range users {
		b.Do(Int64Key(user.CityID), func(res interface{}) {
			user.City = res.(*City)
		})
	}
	if err := b.Flush(); err != nil {
		panic(err)
	}
```

简单多了有木有。




