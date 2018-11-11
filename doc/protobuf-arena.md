### protobuf arena 

#### 引言 
- 本文主要讲解protobuf arena相关知识，讲解arena之前，先讲解必要的内存分配相关的知识，之后讲解arena是什么，为什么要使用arena，使用arena能带来什么

#### new/delete 
**how does new and delete operator works internally ?** 
- **new**： 
1. In first step new operator allocates the storage of size (sizeof type) on the heap by calling operator new(). operator new() takes the size_t as parameter and returns the pointer of newly allocated memory 
2. In second stop new operator initialize that memory by calling constructor of passed type/class 

- **delete**： 
1. In first step delete operator de-initialize that memory by calling destructor of that type/class 
2. In second step delete operator deallocates the memory from the heap by calling operator delete() 

****

#### placement new 
**placement new allows you to construct an object on memory that's already allocated（operator new创建对象的第二步）**
- **operator new vs placement new**：
1. placement new相较于operator new的优势在于使用事先分配好的内存构造对象，可以省去调用operator new时查找合适内存的时间
2. placement new operator can construct an object on a pre-allocated buffer，this is useful when building **a memory pool**， a garbage collector or simply when performance and exception safety are paramount (there's no danger of allocation failure since the memory has already been allocated，and constructing an object on a pre-allocated buffer takes less time
- **deallocation in placement new**：
1. you should not deallocate every object that is using the memory buffer. Instead you should delete[] only the original buffer. you would have to then call the destructors directly of your classes manually

****

#### arena是什么
- 基于上述给出的基本介绍，简单来说，arena就是使用placement new创建对象的内存池

****

#### 为什么使用arena
- 谷歌官方文档有一篇关于arena的介绍，其中标注了使用arena可以提升程序性能的原因，详情请查看: [google arena](https://developers.google.com/protocol-buffers/docs/reference/arenas)

- 在多核处理器上内存的分配和使用要考虑到：
1. 当内存分配和释放的频率增很高时，怎么有效降低内存分配和释放的频率？因为内存分配和释放的频率涉及：与操作系统打交到的频率，内存分配器比如tcmalloc、jemalloc内部核心算法运转的频率，这两者频率越高越影响程序性能
2. 在分配好的内存被使用时，内存块被加载到cpu cache中的有效使用率，即为cpu cache的命中率，内存块的局部性越强，cpu cache的命中率越高（所谓的局部性越强是指：空间上对象分配的越紧凑、时间上分配越近的地址，越容易被使用）
3. 另外当所需要的内存不是常驻内存时，有可能被交换至外存，当需要时被加载到内存，如果所需要的内存排列不紧凑，那有可能需要多次从外存置换（缺页中断），置换的次数越多，越影响程序性能

- 基于以上三点的考虑，arena应运而生，arena是提前分配好一块内存，在其上做对象的构造，当此块内存回收时，在其上构造的对象均被析构，同时使用arena可以提高分配对象的紧凑性，故arena有效的减少了：调用new查找内存的频率，调用delete删除对象的频率，内存分配紧凑时提升cpu cache命中率减少缺页中断次数

****

#### arena怎么使用
- 谷歌文档建议对于服务端程序，一个完成的请求，使用一个arena，弄懂arena基本算法之后，还可以探索更多应用的可能性，具体api使用方式，可以参考谷歌官方文档[google arena](https://developers.google.com/protocol-buffers/docs/reference/arenas)以及其源码
