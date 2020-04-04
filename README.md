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

