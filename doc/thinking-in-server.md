## thinking in server

### 引言
- 本博客为做服务器近一年来的思考总结，有些观点还有稚嫩之处，不过会随着经验的积累逐步丰富这篇博客 - 总结依照类别分为三个大类：**基础设施**，**业务架构**，**数据监控**

### 基础设施
- 有一个高性能、稳定性强、可扩展性强的系统，才能有效的发展业务
#### 最大并发
- 对于服务器来讲，最大并发也可以理解为过载保护，即为利用给定资源，能提供多少**有效**服务，最大并发是一个服务集群能够有效运转的基石
- 设置最大并发，能够有效掌控以下两点：
1. 如果不设置最大并发，服务在应对突增的流量时有可能会过载，如果服务持续过载，会导致越来越多的请求积压，最终所有的请求都必须等待较长时间才能被处理，从而使整个服务处于瘫痪状态；反之，如果服务到达最大并发后拒绝一部分流量，并且能够通知请求方及时更改请求目标，则可以尽可能的保住流量
2. 设置最大并发后，能对一个服务器集群能够承受的最大QPS做到心中有数：当前的服务器集群能够有效承载多少流量；如果流量增大，是否需要做扩容
- 可以使用[little‘s law](https://en.wikipedia.org/wiki/Little%27s_law)来计算最大并发度，计算公式为：最大并发度=极限QPS*低负载延迟，从公式中可以看到，最大并发会随着负载延迟改变，即当单次响应耗时因业务发展而增加时，最大并发就会降低
- 最大并发需要通过压测得到，比如平时的QPS是10000，服务集群可以提供有效服务，但如果将QPS提高到20000，那服务集群是否还可以有效运转？
- 压测要定期执行，做到常态化的备战，另外brpc也提供了[自适应限流方法](https://github.com/brpc/brpc/blob/master/docs/cn/auto_concurrency_limiter.md)，可以降低压测频率

****

#### 内存管理
 **内存管理可谓是关系到程序性能、稳定性、扩展性的关键因素**, 内存管理涉及的方面较为复杂，依据实践经验，列出如下点：
1. 首先要依据程序内存分配的特征选择一个较好的内存分配器，同时要对内存分配器有一个基本的了解，实践中表现较为优异的是facebook的jemalloc，更详细内存分配器分析请参考[allocators](https://github.com/paddington1228/blogs/blob/master/doc/malloc.md)
2. 内存分配器较为底层，在了解内存分配器的原理后，在用户级别，应该做到：
   - 降低内存分配频率：特别是较大对象的拷贝，一个程序，性能的关键点往往在于高频分配、释放内存的地方，在这样的位置可以使用**对象池**来降低内存分配频率，比如rpc的protobuf对象可以使用对象池管理；像广告服务器这种场景，广告信息会从固定的服务获取，若是不增加优化措施，每次都会从下游服务器获取、分配空间、用完释放，在类似于这种场景，可以采用**lru cache**缓存以降低内存分配、释放频率
   - 减少内存跨线程分配、释放：allocator做内存分配释放的时候，往往会加锁，例如使用tcmalloc，一组线程分配的内存总是由另外一组线程释放，会导致两组线程频繁的加锁竞争CentralCache，程序的性能会因此下降，稳定性、扩展性也会受影响
3. 尽可能的使用C++标准提供的unique_ptr/scopred_ptr等管理对象的生命周期，避免出现内存泄漏
4. 除了以上3点之外，还要熟练使用分析工具分析（google gperftools等）分析内存，做到能够掌握程序哪里使用内存较多，若出现内存泄漏问题，还可以使用工具分析定位泄漏位置
5. 对象池推荐brpc的ObjectPool，其参考了tcmalloc的内存分配思想，同时频繁使用的rpc request、response也可以做成池的形式

****

#### 线程管理
##### 并发引起的损耗
1. 对于一个服务端程序，编写多线程程序可以有效利用多核处理器，但在多核处理器上，不可避免的要面对synchronization(race condition)，context switching，cache coherency(cache bouncing，false sharing)造成的性能损耗
2. 如果每个请求来都**动态**分配内存，然后在不同线程间传递，最终由**不同**的线程释放，在多核处理器上，会遭遇严重的性能损耗
3. 为了提高程序的吞吐量，不合理的增加线程数量也会对程序的性能造成影响，因为线程数增多，第1点所列的问题会加剧外，依赖thread local cache提升性能的**基础组件**也会因线程数的增加（比如tcmalloc）受影响

因此如何选择服务端程序线程模型显得尤为重要，因为其会影响程序的**基本性能**、**扩展性**、**稳定性**，好的线程模型会尝试最小化并发会引起的性能损耗

##### 线程模型
- 对于一个完整的服务端程序，通常线程会被分为两大类：工作线程和IO线程，在线程模型中，一个线程可以被当做：工作线程、IO线程、工作+IO线程使用
- 常见的线程模型包含：reactor、half-sync/half-reactive、proactor、leader/followers、brpc bthread，这些线程模型都兼顾网络数据读取发送以及服务相关的计算（比如广告召回）功能，更详细的介绍[thread-model](https://github.com/paddington1228/blogs/blob/master/doc/thread-model.md)
- 线程模型是一个很复杂的范畴，很难一言两语分析清楚，更详尽的介绍可以阅读英文文档：[practor](https://github.com/paddington1228/blogs/blob/master/papers/proactor.pdf)、 [leader/followers](https://github.com/paddington1228/blogs/blob/master/papers/leader-followers.pdf)

##### 实践经验
**boost asio**:
![](https://github.com/paddington1228/blogs/blob/master/images/thinking-in-server/thread-model-sample-1.png)


- **使用场景**： boost asio使用proactor模型，asio设置两个线程池：worker线程池和callback线程池，worker线程池负责网络数据的接收和发送，callback线程池负责异步回调，如果一个计算密集型同时拥有多个下游的服务程序除了使用boost asio，为了提升计算效率，也会增加一个供计算使用的线程池，当收到一个请求时，基本流程：使用computation thread pool做计算->使用boost workers发送网络请求->使用boost workers接收网络请求->使用boost callback回调 ->继续使用computation thread pool做计算
- **性能分析**：
1. 如果当前服务程序下游较多，那应该确保callback线程**只做回调**，回调后数据的解析工作要以任务的形式添加到computation thread pool中，要不然在高QPS下callback线程过多的被占用，会导致asio资源队列阻塞，服务受影响，甚至有可能出现暂时性的**[雪崩](https://github.com/brpc/brpc/blob/master/docs/cn/avalanche.md)**
2. 因为把线程按功能划分，线程总是以**动态**的方式分配对象，callback线程分配的对象始终由computation thread pool释放，使用mem perf tool分析，这样的线程模型程序在内存分配和释放的损耗较大
3. boost asio以及computation thread pool中都设置有任务队列，多线程要加锁访问任务队列，锁带来的损耗也较大，并且处在多线程环境下**多生产者多消费者**的任务队列也会因cache bouncing使得程序性能有损耗


****

#### 削减竞争
线程加锁访问资源，所有线程串行访问临界区，不仅影响程序并发，而且会因争抢锁损耗性能，因此，程序为了获得良好的性能，要削减竞争(reduce contention); 除了我们能够感知到的对共享资源的竞争，还有比较底层的对CPU cache line的竞争，对cache line的竞争会引起f[alse sharing](https://github.com/paddington1228/blogs/blob/master/papers/cache%20line.pdf)问题


##### 有效方案
- **thread local cache**：为每个线程分配独立的内存空间，其他线程不可以访问，完全没有竞争，thread local cache已经在tcmalloc、jemalloc上使用，验证可以明显提升程序性能
- **minimize critical section**：有的临界区本可以不那么大，比如使用lock锁住两个共享资源A、B，但对于A、B的修改并没有明显的关联，则可以增加一个锁来分别保护共享资源A、B
- **extend resources**：拓展共享资源，假设多线程会并发修改一个资源队列，为了减少竞争，那么可以将资源队列拆分，线程按其线程号散列访问多个资源队列中的某一个
- **atomic**：对于基础的数据结构，可以使用原子操作代替加锁，使用原子操作时，要深入了解[memory order](http://senlinzhan.github.io/2017/12/04/cpp-memory-order/)和[ABA](https://en.wikipedia.org/wiki/ABA_problem)问题
- **lock-free**：对于复杂的数据结构，可以使用lock-free的数据接口代替，比如boost的lock-free queue、lock-free stack，tbb的concurrent hash map；对于不好实现lock-free的数据结构，还可以采用读写分离的方式实现lock-free
- **align by cache line**: 针对频繁读、写的对象，解决false sharing问题的有效方式是按cache line对齐，另外更多相关的知识，可以参考[关于CPU cache line](http://cenalulu.github.io/linux/all-about-cpu-cache/)

****

#### 负载均衡
- 常用的负载均衡算法包含：round robin、随机分流、WRR分流算法、一致性哈希，以及brpc所使用的[lalb](https://github.com/brpc/brpc/blob/master/docs/cn/lalb.md)
- 随机分流和round robin存在的问题是不能依据下游服务器的性能以及网络指标分配流量，而brpc的lalb能够依据下游服务器的延时和吞吐量做分流，能够较好的负载均衡

****

#### 压缩算法
- 目前比较主流的压缩算法中，性能比较好的是lz4压缩算法，网络请求往往需要响应比较多的数据，使用较好的压缩算法能缓解网络传输压力，压缩算法的性能对比可以参考[evaluating databse compression methods](https://www.percona.com/blog/2016/04/13/evaluating-database-compression-methods-update/)

****

### 业务架构

#### 服务降级
- **降级的场景**：对于淘宝搜索广告服务(服务架构: adserver->retrieval)，搜索品牌类词比如“nike”所需要做的计算远比搜索一个冷门词高，而且这种情况在淘宝双十一期间突增，假设平时正常流量高峰时2万QPS，设置最大并发后，能够处理的最大QPS是6万，在接近极限时QPS响应耗时有可能已经变得很高，用户体验会变得很差。那这种场景是选择拒绝部分流量，还是选择服务降级？拒绝流量会有收益上的损失，而服务降级能够同时保证收入和用户体验
- **为什么需要服务降级？**
1. 有些功能，带来的效益不是很大，但是对服务性能有较为明显的影响，那这部分功能在需要降级时，为了保证用户体验和主要功能的稳定，是可以暂时关闭的
2. 对于公有模块，比如搜索、推荐、站外等流量都要访问用户信息、库存信息，在双十一这种流量突增的时刻，有些用户信息、库存是可以做降级，而且对收入的影响较小
3. **总之，降级就是为了在保证收入和用户体验的情况下，将一些不是那么主要但又影响程序性能的功能暂时关闭掉**

****

#### 业务规划
- 服务程序的性能需要不断的优化，而良好的业务规划对改善服务的性能起着很重要的作用
- **实践经验**：
1. **按需获取**：一个成熟的业务框架，其中会有有几个模块负责提供公有服务，比如商品信息，但是不同的模块（搜索服务器、模型系统、检索系统等）所需要的商品信息并非是完整的商品信息集合，访问公有服务模块获取的信息有的不用，既浪费网络贷款，又影响需求方的内存甚至程序性能，故需要做到按需访问
2. **分片访问**：比如搜索模块要向索引系统请求800个广告，单次请求200个广告的耗时是40ms，单次请求800个广告的耗时是100ms，那为了缩短请求耗时，将800个广告分为4个分片并发的请求，请求4台不同的服务器，这样便能有效降低响应耗时，不过也带来了额外的工作，4个分片返回的广告要做去重处理以及如何保证4个分片尽可能返回较少重复的广告
