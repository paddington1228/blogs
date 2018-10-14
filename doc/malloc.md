## allocators

### 引言
- 本文旨在分解三个内存分配器ptmalloc 、tcmalloc、jemalloc实现原理，以及分析三个内存分配器的性能
- 性能指标包含：**内存分配速度，内存碎片率，应对race condition能力(内存属于共享资源)，cache locality，多核扩展性**
- 多核扩展性可以理解为**多核处理器上，在内存大小有限，随着内存分配量的增加，分配内存的线程数增加，程序是否还可以正常运转**，直接影响多核扩展性的因素包含：是否有效较低了race condition，是否有效解决了cache bouncing和false sharing问题（详见[cache line](https://github.com/paddington1228/blogs/blob/master/doc/cache-line.md)）

### 概述
- ptmalloc诞生于非多核处理器时代，所以当初的设计很少考虑其多核扩展性，而tcmalloc和jemalloc诞生于多核处理器时代，故而在设计时，便考虑如何解决在多核处理器上所要面临的问题
- 但是并不是内存分配器适用于所有的内存分配模式，有的内存分配器在有些分配模式下表现较好，三者中表现较为优异的是jemalloc，在剖析完内存分配器后，我会列举一个具体的应用实例来对比三个内存分配器的性能

### ptmalloc

#### 数据结构
- **ptmalloc用chunk表示内存分配单元，用bins管理chunk，用arena管理bins以及不由bins管理的top chunk；为了提升效率，ptmalloc将内存释放时，不会立即把内存返还给操作系统，而是用arena管理起来**
- 本博文只简单列举分析ptmalloc必要的数据结构，更详尽的请参考[ptmalloc](https://paper.seebug.org/papers/Archive/refs/heap/glibc%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86ptmalloc%E6%BA%90%E4%BB%A3%E7%A0%81%E5%88%86%E6%9E%90.pdf)

##### chunk
![](https://github.com/paddington1228/blogs/blob/master/images/allocators/ptmalloc/ptmalloc-chunk.png)
- 每个chunk包含其前后chunk的大小用于确定前后chunk的起始位置，以及当前chunk的大小+状态信息（用于chunk的合并），以及真正的用户数据

##### bins
![](https://github.com/paddington1228/blogs/blob/master/images/allocators/ptmalloc/ptmalloc-bins.png)

- bins分为：unsorted bin， small bins，large bins，fast bins以及bin之外的top chunk
- small bins管理小于512B的chunk
- large bins管理大于512B的chunk，其中每个bin中包含大小在一定范围内的bin，链表中的bin大小按升序排列，大小相同的按照最近使用时间排列，ptmalloc使用“smallest-first，best-fit”原则在空闲large bins中查找内存块
- 当chunk被放到bin中时，ptmalloc会检查其前后的chunk是否也处于未被使用状态，如果是，便将可合并的chunk合并并放入unsorted bin中
- fast bins存放chunk大小小于max_fast（默认64B）的chunk，fast bin的意义在于：程序运行时，经常需要分配较小的内存空间，在较小chunk被归还并合并后，有较小内存分配，便需重新切割大的chunk，引入fast bins管理小内存，以提升内存分配速率和使用效率
- unsorted bin 的队列使用 bins 数组的第一个，如果被用户释放的 chunk 大于 max_fast，或者 fast bins 中的空闲 chunk 合并后，这些 chunk 首先会被放到 unsorted bin 队列中，在进行 malloc 操作的时候，如果在 fast bins 中没有找到合适的 chunk，则 ptmalloc 会先在 unsorted bin 中查找合适的空闲 chunk，然后才查找 bins。如果 unsorted bin 不能满足分配要求。malloc便会将 unsorted bin 中的 chunk 加入 bins 中。然后再从 bins 中继续进行查找和分配过程。从这个过程可以看出来，unsorted bin 可以看做是 bins 的一个缓冲区，增加它只是为了加快分配的速度
- top chunk为当从操作系统分配的内存被切割后，是从操作系统分配的内存块被切割后，剩下的部分，**内存分配是从低地址开始的，所以剩余的部分叫top chunk**

##### arena
- arena 分为main arena和none main arena，但是没太看出来两者主要的区别
- 当线程需要分配内存时，首先获取分配区的锁，为了防止多个线程同时访问同一个分配区，在进行分配之前需要取得分配区域的锁。线程先查看线程私有实例中是否已经存在一个分配区，如果存在尝试对该分配区加锁，如果加锁成功，使用该分配区分配内存，否则，该线程搜索分配区循环链表试图获得一个空闲（没有加锁）的分配区。如果所有的分配区都已经加锁，那么 ptmalloc 会开辟一个新的分配区，把该分配区加入到全局分配区循环链表和线程的私有实例中并加锁，然后使用该分配区进行分配操作。开辟出来的新分配区一定为非主分配区，因为主分配区是从父进程那里继承来的
- 分配流程如下：
![](https://github.com/paddington1228/blogs/blob/master/images/allocators/ptmalloc/ptmalloc-alloc-small.png)

##### ptmalloc分析
- **利用效率：**
   1. 为了能够做到chunk被释放后可以合并，chunk中存储了过多的非用户数据，实际内存利用率有损
   2. bin管理的chunk size划分粒度不够精细，超过512B后使用**size区间**管理，使得内存碎片变得**不可控**
   3. arena之间的内存不可转移，从哪个arena分配释放到哪个arena，当线程A持有的arena有较多空闲的chunk，线程B持有的arena有较少的空闲chunk，只能向操作系统申请内存
   4. 后分配的内存先释放，因为 ptmalloc 收缩内存是从 top chunk 开始,如果与 top chunk 相邻的 chunk 不能释放，top chunk 以下的 chunk 都无法释放（**内存孔洞**），极端情况会导致**内存暴涨**
- **分配速率：**
  1. ptmalloc分配一块合适的内存，分配前需要加锁竞争，以及要经过“分配流程图”中所示的各种尝试（临界区较大），分配效率不可观
- **多核处理器扩展性：**
  1. 在多核处理器上，往往使用多于核数的线程，每个线程每次分配内存，都需要加锁arena；另外，若线程数增加，arena数量也会增加，带来的问题是内存碎片增加，有效利用率降低
  2. 多个线程使用同一个arena，多核处理器上会面临cache bouncing和false sharing问题
- **其他问题：**
   1. ptmalloc 不适合用于管理长生命周期的内存，特别是持续不定期分配和释放长生命周期的内存，这将导致 ptmalloc 内存暴增
- 分析了这么多，显得ptmalloc有很多缺点，但毕竟ptmalloc是内存分配器的开山之作，使得之后的人可以在其基础之上开发更有效的内存分配器


### tcmalloc
- 对tcmalloc的分析内心稍微有点虚，因为在实际使用tcmalloc 2.0时，特定场景下，其性能有些问题，其原因待探究，另外也有使用者在gperftools下反馈类似问题：[multi-threaded scene](https://github.com/gperftools/gperftools/issues/446), [spin lock](https://github.com/gperftools/gperftools/issues/663), 但相信谷歌大牛们一定会有解决方案

#### 数据结构
- **tcmalloc为小于256k的内存分配与管理提供三级缓存，ThreadCache->CentralCache->PageHeap，为每个线程设置线程独有的ThreadCache，使得部分内存的分配可以避免race condition，极大的提高了内存分配速率；设置Central Cache，线程将超过阈值的内存释放给Central Cache，内存不够的线程向Central Cache获取；设置PageHeap用来管理大内存的分配**

##### SizeMap
- **内存对齐**：
 - tcmalloc为了提高内存分配效率以及减少内存碎片，对小内存进行了精细的划分，size在(0-16)按照8字节对齐，size在[16,128)之间，按16字节对齐来分配内存，size在[128,256*1024)，按(2^(n+1)-2^n)/8字节对齐来分配内存(n的值为log2(size)取整
- **映射表class_array_[kClassArraySize]**
  - tcmalloc将size进行分类，在class_array_存储size到class的映射关系，由于不同size区间的内存对齐方式不同，所以先用ClassIndex函数计算size在class_array_中的索引，进而找到size对应的class（详细计算可参考tcmalloc源码）
- **映射表class_to_size_[kNumClasses]**
   - 表示class到对齐后size的映射关系，此size和原始传入要分配的size有区别，对齐后的size是经过计算得到的
- **映射表class_to_pages_[kNumClasses]**
  - 表示Central Cache每次从PageHeap获取内存时，对应的size class每次需要从PageHeap获取几页内存
- **映射表num_objects_to_move_[kNumClasses]**
   - 表示ThreadCache每次从Central Cache获取内存时，对应的size class每次需要从Central Cache获取Object个数
- 下图为以8k为一页时，四个映射表的内容：
![](https://github.com/paddington1228/blogs/blob/master/images/allocators/tcmalloc/tcmalloc-sheet.png)

##### ThreadCache
- ThreadCache为线程级别的缓存，tcmalloc将所有ThreadCache使用双向链表连接，意在某个线程ThreadCache不够时，可以从相邻的ThreadCache steal
- 每个ThreadCache保存FreeList  list_[kNumClasses]数组，每个FreeList独立管理一个size_class，每个size_class的Object头部前4或8字节保存下一个Object地址，形成单项链表
- Thread Cache结构
![](https://github.com/paddington1228/blogs/blob/master/images/allocators/tcmalloc/tcmalloc-thread-cache.png)

- **内存分配**：
  1. 通过线程私有数据接口获取属于自己的ThreadCache，如果不存在，则创建一个
  2. 通过映射表class_array_定位到size_class，并在list_中找到对应的FreeList，如果FreeList中有未使用的Object存在，则将链表第一个Object返回
  3. 否则去Central Cache的Central FreeList获取一定数量的Object插入到私有的FreeList中并返回第一个Object，至于从Central FreeList获取多少个Object由**num_objects_to_move_**决定，**查看但tcmalloc 2.0源码可以发现，NumToMove函数做了hard code，Object最多移动个数不能超过32，tcmalloc 2.6版本有修改，定义成了GFLAG，可以自由配置**
- **内存释放**：
  1. 通过线程私有数据接口获取属于自己的ThreadCache
  2. 计算出ptr在系统内存的哪页，即PageID(PageID = (ptr) >> kPageShift)
  3. 然后调用PageHeap的接口GetSizeClassIfCached(PageID p)从映射表pagemap_cache_得到该页被哪个size_class的所使用的(因为CentralCache从PageHeap申请内存时会注册页到size_class的映射关系)
  4. 将ptr放到了ThreadCache的list_对应的链表头部
  5. ThreadCache会在其某个FreeList的长度大于最大长度或者其ThreadCache size大于ThreadCache最大值时，将一定量的内存归还给CentralCache
- **慢启动算法**：
  1. 下面的示图中，只给出了慢启动算法的大体逻辑，详细可以阅读tcmalloc源码
  2. 使用慢启动算法主要应对两类问题：一类是有些线程，在其生命周期内只做很少的内存分配，如果初始便分给它很多内存，定会造成内存浪费；另外一类是有些线程总做内存释放，其不需要分配内存
![](https://github.com/paddington1228/blogs/blob/master/images/allocators/tcmalloc/tcmalloc-slow-start-small.png)

 - **基本分析**：
 1. 分析完ThreadCache分配内存逻辑，如果对tcmalloc的性能比较感兴趣，可能要细致的想一下，tcmalloc增加ThreadCache的目的是减少race condition以及多核处理器上多线程使用共享资源的cache bouncing问题
 2. tcmalloc和ptmalloc的性能对比可以在tcmalloc的官方文档看到[thread caching maclloc](http://goog-perftools.sourceforge.net/doc/tcmalloc.html)
 3.  那影响ThreadCache发挥作用的因素有哪些：
     - 首当其冲的便是线程的数量，为什么这么说，因为所有线程ThreadCache大小的总额是确定的，默认大小是32M，最大值为1G，如果ThreadCache size过小，那ThreadCache便会频繁的将超额的内存释放给CentralCache，这个过程会加自旋锁
     -  那线程多了，把所有ThreadCache的总大小调到最大，这样不就好了，如果看完释放逻辑，还可以看到，另外一个因素就是num_object_to_move，如果这个值设置的不恰当，也会是ThreadCache和CentralCache频繁的交互
      - 另外还有如果程序中存在一些线程总是分配，一些线程总是释放，也会导致两波线程的TreadCache和CentralCache频繁的交互
 
 - **其他分析**：
 1. 在gperftools issue下有几个issue反应tcmalloc会频繁的CentralCache的spin lock，影响程序性能，后来谷歌大神找到的问题根源是tcmalloc 2.0 hard code了NumToMove函数的返回值，同时建议调大所有ThreadCache的总大小
 2. 另外，如果一个程序过多的创建线程，同时存在频繁的跨线程内存分配释放，若ThreadCache较小，NumToMove返回值限制在32，那tcmalloc在频繁分配内存的场景中会退化为频繁访问CentralCache，会面临严重的race condition和cache bouncing

##### CentralCache
-  Central Cache所有的数据都在Static::central_cache_[kNumClasses]中，即采用数组的方式来管理所有的Cache，每个sizeclass的CentralFreeList对应一个数组元素，对不同sizeclass的CentralFreeList的访问是不冲突的
![](https://github.com/paddington1228/blogs/blob/master/images/allocators/tcmalloc/tcmalloc-central-cache.png)

- **TCEntry tc_slots_[kMaxNumTransferEntries]**：
1. tc_slots_用来保存从ThreadCache返还的Object，单次返还个数是num_objects_to_move_[size_class]
2. TCEntry会存储返还Object链表的表头和表尾，以保证Object可以快速在ThreadCache和CentralCache之间快速移动
3. kMaxNumTransferEntries是预设值，代表单个size class对应的CentralFreeList有可能被填满
4. ThreadCache需要内存时优先从CentralFreeList的tc_slots_获取Object，否则从nonempty_管理的Span获取Object

- **Span  nonempty_**：
1. ThreadCache向CentralCache申请内存，当CentralFreeList空间不够用时，便从PageHeap申请一定的Object缓存到nonempty_中，申请是按页申请，页数由class_to_pages_[size_class]来确定
2. ThreadCache归还Object个数小于num_objects_to_move_[size_class]或者tc_slots_满时，放入nonempty_

- **Span empty_**： 用来保存不在有Object的节点

- **Span**：
1. Span用于管理连续的内存页，一个Span可以处于两种不同的状态：allocated和free
2. free状态表示Span在PageHeap里，其内部记录了连续内存页的start和end
3. allocated表示Span在CentralCache的CentralFreeList中，它管理已经被切割的一系列Object
4. 在PageHeap中的Span只是对它管理的连续内存的第一页和最后一页在PageMap pagemap_进行登记
5. 而在CentralFreeList中的Span，它会将它管理的所有的页都在PageMap pagemap_进行注册，ThreadCache归还内存时是按Object来返回的，从Object只能找到它所在的页，最后找到它所归属的Span
![](https://github.com/paddington1228/blogs/blob/master/images/allocators/tcmalloc/tcmalloc_span.png)

- **内存分配**：
1. ThreadCache向CentralCache申请内存，根据申请内存的size class定位到CentralFreeList
2. CentralFreeList首先从tc_cache_中查看是否有未使用的TCEntry，如果有，则直接返回，如果没有，则到nonempty_中申请，如果nonempty_也没有申请到，则从PageHeap中申请固定的页并按照size class做切割放入CentralFreeList的nonempty_中，同时会将获取的页在PageMap pagemap_中注册(接口是RegisterSizeClass)，并且在pagemap_cache_中注册每页的sizeclass(接口是CacheSizeClass())

- **内存释放**：
1. 根据释放内存的size class定位到CentralFreeList，根据释放Object的个数N与object_num_to_move_[size_class]的关系决定当前批Object释放的位置：N -  (N % object_num_to_move_[size_class] > 0)的部分放入tc_slots_，若tc_slots_已满，则放入nonempty_，将N % object_num_to_move_[size_class] 部分放入nonempty_
2. 返回给Span  nonempty_ 的Objects是一个Object一个Object返回的，因为从ThreadCache返回的所有Objects不一定是属于同一个Span的，而Span管理的Object必须是从原来从PageHeap中申请的连续页的内存，所以原来的Objects从哪个Span申请，必须返回到哪个Span
3. 如果Span原来管理的所有的Objects都返回到了Span中(span->refcount == 0)[注意: 最后一个Object不需要插入链表，因为PageHeap关心整块内存，不关心Object]，即没有被central cache中的tc_slots_[]缓存 或Thread
 cache使用，则需要将这个Span管理的内存归还给PageHeap

- **基本分析**：
1. ThreadCache特定条件下向CentralCache释放Object，确保在某个ThreadCache有冗余内存而另外ThreadCache缺少内存时，内存可以有效复用
2. CentralCache的每个CentralFreeList使用tc_slots_，nonempty_管理Object，使用empty_管理空闲Span，当ThreadCache向CentralCache申请内存，但CentralFreeList内存不够用时，是向PageHeap申请内存，未优先使用已经dirty的empty_
3. 线程对CentralFreeList的访问是全程加锁的，如果线程数过多，或者内存分配过于频繁，都会影响性能，**所以在使用tcmalloc时，要注意控制线程的数量以及降低内存分配的频率，可以开发使用对象池之类的数据结构**
4. **每个size class对应的CentralFreeList是多线程共享资源，在多核处理器上，共享资源需要保证cache coherency，频繁的修改CentralFreeList也会损耗程序性能**

##### PageHeap
- PageHeap在TCMalloc中主要作为Central Cache和操作系统之间的内存缓存和大块内存的申请和释放
- PageHeap有两块缓存，一个是用于管理内存小于等于1M的连续页内存，即SpanList free_[kMaxPages]，另一个是那些内存大于1M的连续页内存，即SpanList large_
- SpanList free_[kMaxPages]数组是按页大小递增的，即free_[1]是存放管理1页的Span，free_[2]是存放管理2页的Span，依此类推。SpanList结构下面有两个Span链表，这两个链表作用是不同的，Span
 normal存放的那些还没有释放给系统的Span，而Span returned则存放的是那些已经释放给系统的Span
 ![](https://github.com/paddington1228/blogs/blob/master/images/allocators/tcmalloc/tcmalloc_pageheap.png)

- **内存分配**：
1. CentralFreeList向PageHeap申请n页内存时（接口是PageHeap::New(Length n)），首先查找free_[n].normal，如果没有，则查找free_[n].returned
2. 如果没有找到，在free_[n+i] (i >=1)中查找，如果找到，则将页面切割成n 和 i ，并将i 插入到对应的free_[i]中，插入过程中，还会查找 i 左右相邻的页面是否在SpanList中，如果存在，将其合并，插入到新的SpanList中
3. 如果在free_中没有找到，则在large_中查找，查找过程和1，2所列类似
4. 如果在free_和large_中都没有找到，但是PageHeap中还有大量空闲的页面，说明存在较多的内存碎片，将大量的内存页尽可能的合并之后，在分配
5. 如果依旧没有分配到，则像操作系统分配

- **内存释放**：
1. 当CentralFreeList的某个Span所管理的内存都已经返回给这个Span后(有Span->refcount指示)，CentralFreeList就将相应的Span管理的内存归还给PageHeap(接口: PageHeap::Delete(Span* span))
2. PageHeap会将这个Span和 free_[n].normal或larg_.normal中的Span进行合并(前提是这个Span管理的页的前页或后页在相应的Span链表中)。如果aggressive_decommit_为TRUE(表示每次回到PageHeap的内存都要归还给系统)，则也可以和free_[n].returned或larg_.returned合并，并且都归还给系统(系统会将物理内存收回作为他用，虚拟进程中的虚拟内存还是存在的)
3. 查看是否需要向系统释放内存，如果需要，则以Round Robin的方式将某个SpanList的尾部的Span释放给系统。释放内存是根据配置和PageHeap中累积的Page数量来执行的，具体的算法见函数PageHeap::IncrementalScavenge(Length n)

##### tcmalloc分析
- **利用效率**：
1. tcmalloc使用链表将不同的内存块链接，并使用内存ptr定位其所有页面PageID(PageID = (ptr) >> kPageShift)，有效减少了非用户数据信息的存储
2. tcmalloc将小于256k的内存划分为86个size class，使得内存size划分更精细，减少内存碎片
3. 每个Thread优先使用ThreadCache，ThreadCache中的内存Object会在特定条件下被回收到CentralCache，使得内存可以在不同Thread之间复用，解决了tcmalloc中某个线程arena持有的内存不能被其他线程使用的问题

- **分配速率**：
1. tcmalloc使用ThreadCache，CentralCache，PageHeap三级缓存，最后向操作系统申请内存，精细化的管理可以避免频繁和操作系统打交道
2. ThreadCache加速了内存的分配，因为其是lock-free的，但是如果线程需要频繁分配内存ThreadCache不能满足需求时，需要加锁访问CentralCache，如果面临这样问题的线程较多，所有线程竞争CentralCache的spin lock，会降低内存的分配速率，从而影响程序性能
3. **关注分配速率的时候可以调整：thread cache size和num_object_to_move以及尽可能的缩减程序的线程数量，不要为提高程序的并发度，随意增加线程**

- **多核处理器扩展性**：
1. tcmalloc ThreadCache可以完全避开共享资源在多核处理器上的cache bouncing和false sharing问题
2. 但是随着线程数的增加以及内存分配频率的增加，多线程会频繁的加锁访问共享的CentralCache，致使程序的性能变差

### jemalloc
- **jemalloc是目前为止，在线程数固定场景，是性能比较优异的内存分配器，如Jeson Evans所述，jemalloc集成了很多其他内存分配器已经验证过的基本算法理论，并这些观点和理论结合起来，形成了jemalloc**

#### 基本算法
- **minimize active page set**：
1. 系统内核使用页的方式管理虚拟内存，当实际内存不够用时，便将数据swap到硬盘，所以需要尽可能的将数据排列紧密，以减少swap的频率，phkmalloc已经验证了这个原则，这也是jemalloc关注的重点

- **cache locality**：
1. 虽然RAM的价格在降低以及存储空间大小在增加，但随着CPU的不断更新换代，内存的读取速度远低于CPU运转速度，为了减少访问内存次数，提升CPU工作效率，在CPU和内存之间增加了CPU Cache，jemalloc也将重点放在如何提升cache locality
2. 相对其他内存分配器，使用较少内存的分配器可以不用有较好的cache locality，当程序的工作区间没有命中缓存时，但其工作区间在内存中是**紧密排列**的
3. 在时间趋势上，相近时间被分配的Object，很有可能在相近的时间被使用，所以如果内存分配器可以将这些分配的Object**紧密排列**，也可以提高cache locality
4. 从实践中，内存使用总量可以反应出cache locality的好坏，故而jemalloc将重点放在如何尽量减少内存的使用，以及尽量将分配的内存对象**紧密排列**

- **multiple arena**:
1. 多核处理器上，若两个线程线程运行在不同的核上，读写的对象却在同一个cache line中，那么会出现两个线程竞争同一个cache line的情况，这种情况被称为false sharing，false sharing会严重影响程序的性能
2. 解决false sharing的方法是将对象按cache line对齐，不过这会和将对象尽可能的紧密排列相悖，jemalloc解决false sharing的方法是使用多个arena，每个线程尽可能的在独立arena分配内存
3. arena的数量是核数的4倍，并使用round-robin的随机算法为每个线程分配arena
4. 多核处理器上，为保证cache一致性，对于多线程共享资源，除了面临会损耗性能的加锁问题，还要面临cache bouncing问题，同样会影响程序性能，为每个线程分配独立的arena可以极大减少加锁带来的性能损耗和保持cache一致性带来的cache bouncing

- **minimize lock contention**：
1. 除了让每个线程使用独立的arena分配内存，jemalloc也参考tcmalloc，为每个线程增加thread local cache，进一步降低lock contention带来的性能损耗

- **carefully choose size class**：
1. 如果size class划分的粒度过粗，那会使得分配出来的内存块未被使用的尾部空间变大，内存碎片增加；如果划分的粒度过细，会存在有很多预分配而未被使用的内存

- **lowest address first**:
1. 优先复用较低的地址，这种方式已经被phkmalloc验证过，是jemalloc保证较低内存碎片的关键

#### 数据结构
- **arena**：
1. jemalloc为每个线程指定独立的arena（类似ptmalloc的arena），使用round-robin的方式为每个线程分配arena，每个arena可分配的对象分类如下：
A. Small: [8], [16, 32, 48, ..., 128], [192, 256, 320, ..., 512], [768, 1024, 1280, ..., 3840]
B. Large: [4 KiB, 8 KiB, 12 KiB, ..., 4072 KiB]
C. Huge: [4 MiB, 8 MiB, 12 MiB, ...]
2. arena的数量是核数的4倍，当程序线程总数大于核数4倍时，lock contention便会有所增加
3. **huge allocation**直接使用chunk分配，其分配信息存储在红黑树中

- **run**：
1. 对于small和large对象的分配，可以将chunk可以细分为runs，run的大小可以最小可以为页，run的信息存储在chunk的开头，这样做的好处是：
A. 内存页分配即可使用，无需关心是否需要在当前页存储分配信息
B. 这也使得大小为大于半页&小于半个chunk的large allocation变得简单
2. **small allocation**分为3个子类：tiny, quantum-spaced, and sub-page
3. 每个run专门负责一个small class的分配，每个run的开始部分存储一个bitmap，这样的好处是：
A. 使用位图，可以快速定位未使用的区域
B. 分配信息和用户数据信息是分开存放的，增强了用户数据信息的locality
C. 实现相较于freelist简单
![](https://github.com/paddington1228/blogs/blob/master/images/allocators/jemalloc/jemalloc-arena.png)

- **arena and thread cache**：
![](https://github.com/paddington1228/blogs/blob/master/images/allocators/jemalloc/jemalloc-thread-cache.png)

#### jemalloc分析
- **利用效率**：
1. jemalooc和tcmalloc都很注重size class的划分，尽可能的降低内存碎片
2. jemalloc中的arena使用红黑树记录未满的专供small allocation的run信息，未申请内存的size class优先使用拥有低内存地址的run
3. jemalloc使用两个红黑树分别追踪未使用的run和已经dirty的run，并优先使用拥有低地址的dirty run

- **分配速率**：
1. jemalloc设定的arena个数是核数的4倍，当线程数小于核数4倍时，每个线程独享一个arena，极少受lock contention和cache bouncing影响
2. jemalloc使用facebook内部研发的新型red-black tree，加速了内存的分配效率

- **多核处理器扩展性**：
1. jemalloc为应对多核处理器的lock contention和cache bouncing问题，使用arena，并将数量设置为核数的4倍，当线程数少于核数4倍时，每个线程可独享arena，大于核数4倍时，arena内部将锁的粒度尽可能的降低，从数据结构的设计，到分配算法的使用，jemalloc都能很好的适应多核处理

### 参考文档
1. [jemalloc](https://github.com/paddington1228/blogs/blob/master/papers/jemalloc.pdf)
2. [scalable-jemalloc](https://github.com/paddington1228/blogs/blob/master/papers/scalable-memory-allocation-using-jemalloc.pdf)
3. [phkmalloc](https://github.com/paddington1228/blogs/blob/master/papers/phkmalloc.pdf)
4. [tcmalloc分析笔记](https://blog.csdn.net/zwleagle/article/details/45113303)
5. [tcmalloc分析-知乎](https://zhuanlan.zhihu.com/p/29415507)
6. [how tcmalloc works](https://github.com/paddington1228/blogs/blob/master/papers/how-tcmalloc-works.pdf)
7. [内存优化总结](http://www.cnhalo.net/2016/06/13/memory-optimize/)
8. [glibc内存管理](https://paper.seebug.org/papers/Archive/refs/heap/glibc%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86ptmalloc%E6%BA%90%E4%BB%A3%E7%A0%81%E5%88%86%E6%9E%90.pdf)
