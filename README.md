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

## peers and http

这两个模块就完成了peer的分工和协作

先看peer模块，我们这里就不管nopeers的部分了（也没有什么东西），直接看peers

```go
type ProtoGetter interface {
	Get(ctx context.Context, in *pb.GetRequest, out *pb.GetResponse) error
}
type PeerPicker interface {
	PickPeer(key string) (peer ProtoGetter, ok bool)
}
```

定义了两个个接口，关注一下PeerPicker接口，用于在给定key的时候返回处理这个key的peer，类型是protogetter，也就是第一个interface

那么相对应的pickpeer注册如下（可以看到只能注入一次）

```go
var (
	portPicker func(groupName string) PeerPicker
)
func RegisterPeerPicker(fn func() PeerPicker) {
	if portPicker != nil {
		panic("RegisterPeerPicker called more than once")
	}
	portPicker = func(_ string) PeerPicker { return fn() }
}
```

注册完如何找到key对应peer的函数，在groupcache调用getPeers如下的时候可以查找到peer

```go
func getPeers(groupName string) PeerPicker {
	if portPicker == nil {
		return NoPeers{}
	}
	pk := portPicker(groupName)
	if pk == nil {
		pk = NoPeers{}
	}
	return pk
}
```

然后看相对比较复杂的http模块

```go
type HTTPPool struct {
	Context func(*http.Request) context.Context
	Transport func(context.Context) http.RoundTripper
	// this peer's base URL, e.g. "https://example.net:8000"
	self string
	// opts specifies the options.
	opts HTTPPoolOptions

	mu          sync.Mutex // guards peers and httpGetters
	peers       *consistenthash.Map
	httpGetters map[string]*httpGetter // keyed by e.g. "http://10.0.0.2:8008"
}
type HTTPPoolOptions struct {
	BasePath string
	Replicas int
	HashFn consistenthash.Hash
}
```

有上述的options参与定义httppool，所以用到了consistent hash，也就是存放peers hash对应的数据结构，所以会将peers加入到consistent hash中

```go
type httpGetter struct {
	transport func(context.Context) http.RoundTripper
	baseURL   string
}
// ... other code
func (p *HTTPPool) Set(peers ...string) {
	p.mu.Lock()
	defer p.mu.Unlock()
	p.peers = consistenthash.New(p.opts.Replicas, p.opts.HashFn)
	p.peers.Add(peers...)
	p.httpGetters = make(map[string]*httpGetter, len(peers))
	for _, peer := range peers {
		p.httpGetters[peer] = &httpGetter{transport: p.Transport, baseURL: peer + p.opts.BasePath}
	}
}
```

我们关注到httpGetter是实际处理通信的，源码这里的transport是一个可自定义的通信函数，默认使用 http.DefaultTransport

那么httpGetter是怎样的呢？其实就是通过protobuf发送get请求

```go
func (h *httpGetter) Get(ctx context.Context, in *pb.GetRequest, out *pb.GetResponse) error {
	u := fmt.Sprintf(
		"%v%v/%v",
		h.baseURL,
		url.QueryEscape(in.GetGroup()),
		url.QueryEscape(in.GetKey()),
	)
	req, err := http.NewRequest("GET", u, nil)
	if err != nil {
		return err
	}
	req = req.WithContext(ctx)
	tr := http.DefaultTransport
	if h.transport != nil {
		tr = h.transport(ctx)
	}
	res, err := tr.RoundTrip(req)
	if err != nil {
		return err
	}
	defer res.Body.Close()
	if res.StatusCode != http.StatusOK {
		return fmt.Errorf("server returned: %v", res.Status)
	}
	b := bufferPool.Get().(*bytes.Buffer)
	b.Reset()
	defer bufferPool.Put(b)
	_, err = io.Copy(b, res.Body)
	if err != nil {
		return fmt.Errorf("reading response body: %v", err)
	}
	err = proto.Unmarshal(b.Bytes(), out)
	if err != nil {
		return fmt.Errorf("decoding response body: %v", err)
	}
	return nil
}
```

那peer作为客户端发送get请求，作为服务端还要响应get请求吧，所以又有如下的handle处理

```go
func (p *HTTPPool) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	// Parse request.
	if !strings.HasPrefix(r.URL.Path, p.opts.BasePath) {
		panic("HTTPPool serving unexpected path: " + r.URL.Path)
	}
	parts := strings.SplitN(r.URL.Path[len(p.opts.BasePath):], "/", 2)
	if len(parts) != 2 {
		http.Error(w, "bad request", http.StatusBadRequest)
		return
	}
	groupName := parts[0]
	key := parts[1]

	// Fetch the value for this group/key.
	group := GetGroup(groupName)
	if group == nil {
		http.Error(w, "no such group: "+groupName, http.StatusNotFound)
		return
	}
	var ctx context.Context
	if p.Context != nil {
		ctx = p.Context(r)
	} else {
		ctx = r.Context()
	}

	group.Stats.ServerRequests.Add(1)
	var value []byte
	err := group.Get(ctx, key, AllocatingByteSliceSink(&value))
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	// Write the value to the response body as a proto message.
	body, err := proto.Marshal(&pb.GetResponse{Value: value})
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	w.Header().Set("Content-Type", "application/x-protobuf")
	w.Write(body)
}
```

## groupcache



