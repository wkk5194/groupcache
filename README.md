# groupcache

## Note

GroupCache源码阅读的一些笔记 写在这里

## lru

总体而言实现方式为`双向链表`+`map`

主要逻辑如下：

- 添加一个新key/或者更新已有key，将key对应的element执行PushFront/MoveToFront操作，也就是放到链表最前，然后判断是否超过了最大容量，超过就删除链表最末element
- 查询一个key，同样执行MoveToFront操作
- 此外当key-element被删除的时候，源码显示可以执行OnEvicted操作，但是似乎在lru cache注册阶段没有写对这个回调函数的注入（???）

缺点的话：主要就是依赖的container/list是线程不安全的，不支持并行，效率有点低，外部需要维护一下同步的问题

## consistenthash

一致性哈希 实现方式就是根据理论来

主要逻辑如下：

- 为了hash ring上节点能尽可能平均分布，因此允许虚拟节点，为此传入replicas表示总节点数=实节点*replicas
- hash函数允许自定义，默认的hash函数为crc32，crc是一种追求速度不追求低碰撞率的hash算法，具体可以看[wiki](https://en.wikipedia.org/wiki/Cyclic_redundancy_check)
- 对虚拟节点的处理通过加前置index再hash

```go
    for i := 0; i < m.replicas; i++ {
			hash := int(m.hash([]byte(strconv.Itoa(i) + key)))
			m.keys = append(m.keys, hash)
			m.hashMap[hash] = key
		}
```

- 放进去的keys为了接下来在get的时候找到最近，需要进行排序，源码直接根据hash的字母序排序
- get某个key的时候就是寻找key对应的hash在hash ring上距离最近的节点，源码用的是sort.Search，一个sort依赖模块的二分查找方法

源码没有写对key的remove操作，难道要clear掉重新add吗

## singleflight

这个模块实现的功能可以说是groupcache的一大亮点，而且代码量极小

当get请求的key找不到的时候，并发的相同的get请求可能会击穿缓存

通过singleflight对并发相同key的get请求进行拦截，那么真正只会去get一个（无论是去peer还是怎样），其他就可以直接返回

结合代码看一下：
```go
type call struct {
	wg  sync.WaitGroup
	val interface{}
	err error
}
type Group struct {
	mu sync.Mutex       
	m  map[string]*call 
}
func (g *Group) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[key]; ok {
		g.mu.Unlock()
		c.wg.Wait()
		return c.val, c.err
	}
	c := new(call)
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	c.val, c.err = fn()
	c.wg.Done()

	g.mu.Lock()
	delete(g.m, key)
	g.mu.Unlock()

	return c.val, c.err
}
```
- 对Group的操作通过mutex锁保护起来
- 每一个key都有一个call，内部有wg锁来阻塞
- 当命中key的时候，wg wait，也就是被阻塞住直到执行阻塞的第一次get返回了值
- 如果没有命中key，那就是第一次get这个key，所以新建一个call，然后wg add 1，然后执行fn函数去取得值，最后wg done释放阻塞
- 记得用完要把这个key删掉哦 不然下次进来直接命中key而且无阻塞返回val了，导致错误

代码短小精悍，对并发的控制却是精妙绝伦！

## groupcachepb

pb就是protobuf的简写，所以这就是一个提供通信的模块，利用protobuf做序列化和反序列化

里面有groupcache.pb.go和groupcache.proto两个文件，其实就是编写了proto文件然后利用protobuf生成go文件

```go
message GetRequest {
  required string group = 1;
  required string key = 2; // not actually required/guaranteed to be UTF-8
}

message GetResponse {
  optional bytes value = 1;
  optional double minute_qps = 2;
}

service GroupCache {
  rpc Get(GetRequest) returns (GetResponse) {
  };
}
```

具体去看proto文件，定义了request和response的一些字段，还定义了rpc通信接口

## byteview and sinks

byteview模块封装了一个string与byte[] 的统一接口，也就是说用byteview提供的接口，可以屏蔽掉string与byte[] 的不同，使用时可以不用考虑是string还是byte[]

然后sinks模块在其基础上实现了几个sinks struct，相当于做了数据的储存，可以set；可以view；setproto方法是用来从protobuf的message中把数据sink下来

这里涉及到的代码重复度比较高，几个struct逻辑都是基本相同的，在此不赘述

## peers

## http

## groupcache



