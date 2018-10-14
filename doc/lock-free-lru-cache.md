#### 目标
- 具有LRU特性
- 增、删、改、查时间复杂度O(1)
- 多线程读写安全
- 满足[lock-free](https://en.wikipedia.org/wiki/Non-blocking_algorithm#Wait-freedom)
- 管理所存储对象生命周期，可做到安全删除
- 尽可能减少内存分配和释放（待实现）

#### 方案
- 使用list实现LRU特性：
   - 新的数据对象插入链表头部
   - 最近使用过的对象移动到链表头部
   - 链表溢出时删除链表尾部 
- 使用链表做不到查询的时间复杂度是O(1)，故借助unordered_map存储查询key和指向链表的指针
- 为满足多线程读写安全，需要对cache的操作加锁，但若加锁，多线程操作同一个cache时会有竞争，影响性能且不能满足lock-free，故采用双buffer实现读写分离，去除加锁逻辑
- 当buffer内容有更新，读写buffer切换时，需要考虑多线程读写单块或者多块内存时保证[内存一致性](https://en.wikipedia.org/wiki/Cache_coherence) ，使用brpc的工具类[DoublyBufferedData](https://github.com/brpc/brpc/blob/master/src/butil/containers/doubly_buffered_data.h)管理读写buffer的切换，更多知识可参考brpc原子操作的文章[atomic_instruction](https://github.com/brpc/brpc/blob/master/docs/cn/atomic_instructions.md)
- Cache中存储的数据对象在生命周期内不允许拷贝，使用者使用指针或引用访问，由Cache管理对象生命周期，并需要做到安全删除，使用multi set管理使用Cache的开始时间和结束时间（比如一个请求，进入server和离开server的时间），结合[garbage collector](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science)) 做延迟回收，确保Cache中过期或者溢出的对象安全删除
- Cache在使用过程中，需要不断的插入新数据、移动MRU(most recent use)数据、删除过期以及溢出的数据，故启动一条线程**定期**执行更新操作
- 为保持读写buffer数据同步，在更新后台(old background)buffer且切换前后台buffer后，将对老后台buffer的更新操作同步到新后台buffer(old foreground)
- Cache的对象都分配在堆上，两个buffer中的list都存储指向堆的指针，将无用对象从Cache中清除包含移除两个buffer的list中的指针以及删除堆中的内存，为保证**查即可用**，删除Cache中对象时，在切换buffer后，删除新后台（old foreground）保存的指针时，删除指针指向的堆
- 假设两个buffer分别为A、B，A为前台，B为后台，切换后A为后台，B为前台，对于更新操作要保证按照B-A-A-B的顺序更新，切勿出现按A-A-B的情况，这会造成更新不一致，进而使得两个buffer中的内容不一致
- 因为Cache容量固定，为减少大对象内存分配和释放，可采用对象池管理对象的分配（待实现）
- Cache内存使用boost lock-free queue（MPSC）管理需要新加入、最近访问、过期的数据，可保证多线程读写
- 因为存在多线程读的情况，同一个段时间内（可能很小）多个线程使用指定的key没有找到对应的object，会发起网络请求访问下游服务器，取回数据对象后，插入到lock-free queue中，lock-free queue中会存在重复数据，故在插入新数据时，若判断插入的key有值，则删除待插入的重复数据

#### 拓展
- DoublyBufferedData如何确保读线程能读切换后最新的前台? 请参考brpc对的讲解[DoublyBufferedData](https://github.com/brpc/brpc/blob/master/docs/cn/lalb.md)

#### cache结构图
- 请查看流程图
![](https://github.com/paddington1228/blogs/blob/master/images/lru-cache/lock-free-lru-cache.png)
