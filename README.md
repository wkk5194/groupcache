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

