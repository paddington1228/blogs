#### 引言
- **本博文会逐一讲解NUMA、CPU Cache结构、CPU Cache Line、Cache Coherency、Cache Bouncing、False Sharing**
- 首先能够想到的问题有：
  1. NUMA是什么，和CPU Cache Line有什么关系？
  2. CPU Cache结构和CPU Cache Line是什么关系？
  3. 什么是Cache Coherency？
  4. Cache Bouncing和False Sharing是什么？
  5. 了解了这些之后，我们能够在编程中注意什么？

#### NUMA
图：uma-numa

- 与NUMA对应的是UMA（Uniform Memory Access），它们描述了在多核处理器上CPU和共享内存的排列方式
- 在UMA架构中每个处理器，不论在什么位置，都得通过Bus总线访问内存，访问耗时与数据处于内存位置无太大关系（如图所示）
- 在NUMA架构中，每个处理器都有一块属于自己的本地内存，且可以访问属于其他处理器的远程内存，访问本地中的数据相对较快，访问远程内存相对较慢
- NUMA架构的优势在于作为一个分层的共享内存结构能够通过尽可能多的访问本地内存提高访问数据的速度，从而提高程序性能
- 现在多核处理器中，一般采用两者结合的形式，多个CPU组成CPU Package，每个CPU Package都有自己的内存，同一个CPU Package中不同CPU按照UMA结构排列，不同CPU Package之间按照NUMA结构排列，一个CPU Package可以访问另外一个CPU Package所拥有的内存

#### CPU Cache
##### 结构
- 在单核CPU结构中，为了缓解CPU指令流水中cycle冲突，L1分成了指令（L1P）和数据（L1D）两部分，而L2则是指令和数据共存
- 多核CPU的结构与单核相似，但是多了所有CPU共享的L3三级缓存。在多核CPU的结构中，L1和L2是CPU私有的，L3则是所有CPU核心共享的
- **因为L3是所有CPU核心共享，刨除L3，4 CPU core + L1 + L2可以组成一个UMA-NUMA-Mix架构中的一个CPU Package**
图：multi-cpu


##### CPU Cache意义
- 为什么需要CPU Cache？因为CPU的频率太快了，快到主存跟不上，这样在处理器时钟周期内，CPU常常需要等待主存，浪费资源。所以Cache的出现，是为了缓解CPU和内存之间速度的不匹配问题
- CPU Cache有什么意义？Cache的容量远远小于主存，因此出现Cache Miss在所难免，既然Cache不能包含CPU所需要的所有数据，那么Cache的存在真的有意义吗？当然是有意义的——局部性原理：
  A. 时间局部性：如果某个数据被访问，那么在不久的将来它很可能被再次访问
  B. 空间局部性：如果某个数据被访问，那么与它相邻的数据很快也可能被访问

#### CPU Cache Line
- 为了有效的使用CPU Cache，便将Cache分割成了很多固定大小的单元，通常为64字节或者128字节，这些小的CPU Cache单元叫做CPU Cache Line
- 当CPU将内存中数据读取到CPU Cache中，也是按照CPU Cache Line的大小读取，从而将单个CPU Cache Line填满
- **至此，我们看到CPU Cache Line是CPU Cache的一个小部分，当多线程运行在多核CPU中，对CPU Cache Line的操作就必须要适应UMA or NUMA架构**

#### Cache Coherency
- 假设这样的场景：当多线程程序P运行在多核机器上，P中所有线程共享资源M，当P中线程A将M读入Cache C1，B线程将M读入Cache C2，A线程对M进行修改，当B线程使用M时怎么保证B线程读到的是A修改后的？**这就需要Cache Coherency：用于保证多线程程序运行在多核机器上，CPU Cache中缓存共享数据的一致性**

##### 写回主存和cache操作
- CPU Cache中的数据在修改后要确保能够**写会主存**，写回方式：
  1. 写通：每次CPU修改了cache中的内容，立即更新到内存，也就意味着每次CPU写共享数据，都会导致总线事务，因此这种方式常常会引起总线事务的竞争，高一致性，但是效率非常低
  2. 写回：每次CPU修改了cache中的数据，不会立即更新到内存，而是等到cache line在某一个必须或合适的时机才会更新到内存中
- 论是写通还是写回，在多线程环境下为了保证缓存**一致性**，处理器又提供了写失效（write invalidate）和写更新（write update）两个操作：
  1. 写失效：当一个CPU修改了数据，如果其他CPU有该数据，则通知其为无效
  2. 写更新：当一个CPU修改了数据，如果其他CPU有该数据，则通知其跟新数据
- 对比写失效，如果采用写更新，那Cache Cache除了要读取内存，还要与其他Cache交互，增加了实现的复杂度，试想写更新不光要更新UMA-NUMA-Mix中同CPU Package中的CPU Cache，还要更新不同CPU Package中的CPU Cache
- 采用**写失效**，每个Cache的控制器不仅要知道自己管理Cache的操作事件（local read、local write），通过监听也要知道其他Cache的操作事件（remote read、remote write），若发生其他Cache写，则当前Cache中数据失效，且在自己Cache发生变化时，通知其他Cache；四种事件处理：
  1. local read（LR）：读取本地cache中的数据
  2. local write （LW）：写入本地cache
  3. remote read（RR）：读取内存中的数据
  4. remote write （RW）：将数据写回主存
- 当多线程操作不同CPU的Cache Line时，或者当有4种不同操作交叉发生在Cache Line时，为保证多线程读取正确的共享内存数据，Cache Line也要有**不同的状态**，即为：
  1. modify：当前CPU Cache拥有最新数据（最新的cache line），其他CPU拥有失效数据（cache line的状态是invalid），虽然当前CPU中的数据和主存是不一致的，但是以当前CPU的数据为准
  2. exclusive：只有当前CPU中有数据，其他CPU中没有改数据，当前CPU的数据和主存中的数据是一致的
  3. shared：当前CPU和其他CPU中都有共同数据，并且和主存中的数据一致
  4. invalid：当前CPU中的数据失效，数据应该从主存中获取，其他CPU中可能有数据也可能无数据，当前CPU中的数据和主存被认为是不一致

##### cache操作和状态转换
- 初始场景：在最初的时候，所有CPU中都没有数据，某一个CPU发生读操作，此时发生RR，数据从主存中读取到当前CPU的cache，状态为E（独占，只有当前CPU有数据，且和主存一致），此时如果有其他CPU也读取数据，则状态修改为S（共享，多个CPU之间拥有相同数据，并且和主存保持一致），如果其中某一个CPU发生数据修改，那么该CPU中数据状态修改为M（拥有最新数据，和主存不一致，但是以当前CPU中的为准），并通知其他拥有该数据的CPU数据失效，其他CPU中的cache line状态修改为I（失效，和主存中的数据被认为不一致，数据不可用应该重新获取）
- modify
场景：当前CPU中数据的状态是modify，表示当前CPU中拥有最新数据，虽然主存中的数据和当前CPU中的数据不一致，但是以当前CPU中的数据为准
1. LR：此时如果发生local read，即当前CPU读数据，直接从cache中获取数据，拥有最新数据，因此状态不变
2. LW：直接修改本地cache数据，修改后也是当前CPU拥有最新数据，因此状态不变
3. RR：因为本地内存中有最新数据，因此当前CPU不会发生RR和RW，当本地cache控制器监听到总线上有RR发生的时，必然是其他CPU发生了读主存的操作，此时为了保证一致性，当前CPU应该将数据写回主存，而随后的RR将会使得其他CPU和当前CPU拥有共同的数据，因此状态修改为S
4. RW：同RR，当cache控制器监听到总线发生RW，当前CPU会将数据写回主存，因为随后的RW将会导致主存的数据修改，因此状态修改成I
- exclusive
场景：当前CPU中的数据状态是exclusive，表示当前CPU独占数据（其他CPU没有数据），并且和主存的数据一致
1. LR：从本地cache中直接获取数据，状态不变
2. LW：修改本地cache中的数据，状态修改成M（因为其他CPU中并没有该数据，因此不存在共享问题，不需要通知其他CPU修改cache line的状态为I）
3. RR：因为本地cache中有最新数据，因此当前CPU cache操作不会发生RR和RW，当cache控制器监听到总线上发生RR的时候，必然是其他CPU发生了读取主存的操作，而RR操作不会导致数据修改，因此两个CPU中的数据和主存中的数据一致，此时cache line状态修改为S
4. RW：同RR，当cache控制器监听到总线发生RW，发生其他CPU将最新数据写回到主存，此时为了保证缓存一致性，当前CPU的数据状态修改为I
- shared
场景：当前CPU中的数据状态是shared，表示当前CPU和其他CPU共享数据，且数据在多个CPU之间一致、多个CPU之间的数据和主存一致
1. LR：直接从cache中读取数据，状态不变
2. LW：发生本地写，并不会将数据立即写回主存，而是在稍后的一个时间再写回主存，因此为了保证缓存一致性，当前CPU的cache line状态修改为M，并通知其他拥有该数据的CPU该数据失效，其他CPU将cache line状态修改为I
3. RR：状态不变，因为多个CPU中的数据和主存一致
4. RW：当监听到总线发生了RW，意味着其他CPU发生了写主存操作，此时本地cache中的数据既不是最新数据，和主存也不再一致，因此当前CPU的cache line状态修改为I
- invalid
场景：当前CPU中的数据状态是invalid，表示当前CPU中是脏数据，不可用，其他CPU可能有数据、也可能没有数据
1. LR：因为当前CPU的cache line数据不可用，因此会发生RR操作，此时的情形如下
    A. 如果其他CPU中无数据则状态修改为E
    B. 如果其他CPU中有数据且状态为S或E则状态修改为S
    C. 如果其他CPU中有数据且状态为M，那么其他CPU首先发生RW将M状态的数据写回主存并修改状态为S，随后当前CPU读取主存数据，也将状态修改为S
2. LW：因为当前CPU的cache line数据无效，因此发生LW会直接操作本地cache，此时的情形如下
    A. 如果其他CPU中无数据，则将本地cache line的状态修改为M
    B. 如果其他CPU中有数据且状态为S或E，则修改本地cache，通知其他CPU将数据修改为I，当前CPU中的cache line状态修改为M
    C. 如果其他CPU中有数据且状态为M，则其他CPU首先将数据写回主存，并将状态修改为I，当前CPU中的cache line转台修改为M
3. RR：监听到总线发生RR操作，表示有其他CPU读取内存，和本地cache无关，状态不变
4. RW：监听到总线发生RW操作，表示有其他CPU写主存，和本地cache无关，状态不变

#### Cache Bouncing
- 试想，一个多线程程序P在多核CPU上共享内存资源M，多个线程均可以读取和修改M，那每个线程依附的CPU中的Cache的Cache Line为保证数据的一致性，要不停的做状态转换和数据读取，即为：Cache Bouncing
- 同时数据的读取更多的是Remote Read和Remote Write操作，**当CPU能够从cache中拿到有效数据的时候，消耗几个CPU cycle，如果发生Cache Miss，则会消耗几十上百个CPU Cycle**
- **Cache Bouncing问题会严重影响程序性能，由于实现Cache一致性往往有硬件锁，Cache Bouncing是一种隐式的的全局竞争**
- 了解到Cache Bouncing（使用共享内存）会影响程序性能后，应该更多使用thread local变量来避免Cache Bouncing
- 多生产者多消费者队列以及多线程共享资源存在比较验证的Cache Bouncing问题，brpc TimerThread的设计相对sofa的timer规避了Cache Bouncing问题，推荐学习借鉴消除Cache Bouncing的思路

#### False Sharing
- cache bouncing使访问频繁修改的变量的开销陡增，甚至还会使访问同一个cacheline中不常修改的变量也变慢，这个现象是[false sharing](https://en.wikipedia.org/wiki/False_sharing),  关于false sharing的详细分析，亦可参考[CoolShell CPU Cache Line](https://coolshell.cn/articles/10249.html)

#### 总结
- 为了有效利用多核处理器，我们采用多线程编程，为了降低CPU和内存访问速度不同带来的性能影响，我们增加了CPU Cache，但如果不细致的了解CPU Cache，大量使用共享内存，那么为了保证Cache一致性需要大量耗时操作，进而引发Cache Bouncing和False Sharing问题，在了解CPU Cache是如何工作之后，当我们编写程序时，需要尽量采取合理的数据结构和有效利用thread local cache来提高程序性能

#### 参考资料
- [cpu cache](https://en.wikipedia.org/wiki/CPU_cache)
- [cache coherency](https://en.wikipedia.org/wiki/Cache_coherence)
- [false sharing](https://en.wikipedia.org/wiki/False_sharing)PK 