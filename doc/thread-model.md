### 简介
- 此博文由简而难的介绍IO网络模型reactor、half sync/half reactor、proactor、leader/followers的工作原理以及优缺点
- 准备知识：[同步异步IO](https://segmentfault.com/a/1190000003063859?utm_source=Weibo&utm_medium=shareLink&utm_campaign=socialShare&from=timeline&isappinstalled=0)

### Reactor
#### 执行时序
![](https://github.com/paddington1228/blogs/blob/master/images/thread-model/reactor-workflow.png)

#### 具体步骤
1. 主程序调用接口注册处理事件的handler
2. 主程序调用reactor的执行函数，reactor的执行函数启动一根线程进入无限循环，循环中由具有event demultiplex功能的select或者epoll监听事件的发生
3. 当有事件发生时，select或者epoll将执行权限交给reactor线程，reactor线程调用对应事件的handler函数处理事件，处理完成后继续调用select监听事件

#### 剖析
- 简易执行图：
![](https://github.com/paddington1228/blogs/blob/master/images/thread-model/reactor-delay.png)

##### 实现
- reactor实现了一个**被动**的事件分离和分发模型，调用select和epoll之后，操作系统可以在多个事件源上等待
- 当有事件发生后，reactor执行线程获得执行权限，不受间断的同步处理事件
- **网络IO场景**：若在网络IO场景，reactor线程获取到发生的event，read or write数据，解析数据，处理数据

##### 优点
- 对于callback执行耗时较短的场景较为高效，如brpc中的TimerThread
- 同时reactor将应用无关的多路分解和分配机制与应用相关的回调函数分离，当应用有新的需要关注的事件，只需要注册事件、实现回调函数

##### 缺点
- reactor中的handler不适合执行耗时任务，否则会因长时间占用执行线程导致后续的事件处理被delay


### Half Sync/Half Reactor
#### 执行时序
![](https://github.com/paddington1228/blogs/blob/master/images/thread-model/half-sync-half-reactor.png)

#### 剖析

##### 实现
- 考虑到单线程reactor模式不适合执行耗时任务，可以将耗时任务交由worker thread pool处理，IO线程承担demultiplex和dispatch任务
- 操作系统监听到事件发生，将执行权限交给reactor中IO线程，IO线程read or write数据，将数据解析和数据处理封装成task，放入thread pool的task queue中，由worker线程执行
- 为什么由IO线程做read or write操作？如果将read or write操作封装到task中由worker线程执行，会出现多线程读写同一个资源的问题，比如多线程读写fd，致使程序执行出现不确定性甚至崩溃

##### 优点
- 添加线程池减轻了reactor执行线程的负担，且可以使多任务并发的执行

##### 缺点
1.  IO线程和thread pool之间的**任务队列**成了临界区，IO线程向其添加任务、thread pool从中获取任务必须同步执行
2. IO线程读取数据分配内存，封装到task中，由thread pool中某一个线程执行，thread pool的线程执行完成后，释放内存，内存跨线程分配和释放，会使依赖thread local的程序例如tcmalloc变慢，进而影响程序性能
3. task由IO线程封装，thread pool中线程执行，涉及线程的上下文切换
4. 如果在多核cpu上，对共享资源**任务队列**为保证cpu cache coherency，需要大量的cpu同步操作，造成cache bouncing问题
5. 单条IO线程负责read or write操作，若read write操作较为耗时，同样会使后续事件处理被delay

### Proactor
#### 执行时序
![](https://github.com/paddington1228/blogs/blob/master/images/thread-model/proactor.png)

#### 剖析
- [boost asio](https://www.boost.org/doc/libs/1_67_0/doc/html/boost_asio/overview/core/async.html)使用了proactor模式

##### 实现
1. 由Initiator初始化async operation processor线程和proactor线程
2. 当操作系统监听到事件发生，将执行权限交给async operation processor，async operation processor将IO的read、write操作交给操作系统**异步执行**，操作系统read、write完成后，通知async operation processor，async operator operator将结果放入result queue中
3. 把read、write操作交给操作系统，可以实现多个事件的read、write操作并发执行，对比reactor需由IO线程执行read、write操作灵活很多
4. proactor线程充当dispatch角色，将放在result queue中的结果分发到worker thread pool中执行
5. result queue连接async operation processor和proactor，为了获得高并发，可以使用多个result queue，多个proactor

#### 优点
- 将reactor中IO线程read、write的操作由async operation processor交给操作系统，由操作系统实现多事件的read、write操作并发
- 将操作系统执行的结果放到多个result queue中，proactor做任务分发，proactor可实现多个dispatcher，类比多个reactor，提高了吞吐量
- **计良好的多线程reactor程序往往能比同一台机器上的多个单线程reactor进程更均匀地使用不同核心（摘自brpc文档）**

#### 缺点
1. reactor面临的问题1-4，proactor均没有很好的解决，如下图（图片来源brpc文档）
2. **由于cache一致性的限制，多线程reactor并不能获得线性于核心数的性能，在特定的场景中，粗糙的多线程reactor实现跑在24核上甚至没有精致的单线程reactor实现跑在1个核上快（摘自brpc文档）**
![](https://github.com/paddington1228/blogs/blob/master/images/thread-model/multi-reactor.png)


### leader/followers
- 良好的多线程IO模型需要考虑的因素较多：内存分配（这里一定很费解），同步（任务队列），多核CPU上线程上下文切换，多核CPU上cache一致性同步
- 为保证IO线程安全，reactor模式和proactor模式对于IO handle集合均由同一个线程操作，没有很好的复用（demultiplex），对于worker thread pool的复用也存在cache bouncing问题
- 如何更好的复用IO handle set和worker thread pool便成为了leader/followers模式的关键

#### 执行时序
![](https://github.com/paddington1228/blogs/blob/master/images/thread-model/leader-followers.png)


#### 剖析
##### 实现
1. 注册监听事件，获得IO handle集合
2. 将worker thread pool中的线程分为两种，选取一个作为leader线程，其他为follower线程，去掉IO线程
3. 当操作系统监听到事件将执行权限交给leader线程，leader线程反注册当前有事件发生的handle，在followers中选取一个线程作为new leader，old leader线程继续处理事件相关
4. 新的leader线程重复步骤3
5. leader线程在处理完当前事件后，重新变为follower
6. 在一个又一个leader线程获取到事件后，多线程可以并发的处理事件相关
7. 当follower线程集合变为空时，操作系统将所发生的事件放入事件队列中，等有新leader线程可用时，交由其处理
8. leader/followers模式为了IO handle集合可复用，需要实现线程安全的IO handle

##### 优点
1. leader/followers模式没有reactor或者proactor模式中的任务队列，去除了资源同步问题以及极大的减少了cache bouncing问题（io handle set还有cache bouncing）
2. leader/followers模式，从监听到某个事件的发生到对事件的处理，均在同一个线程中完成，消除了线程上下文切换，以及消除了内存跨线程分配和释放问题
3. leader/followers支持unbound和bound两种模式
4. 更详细的分析请查阅[leader/followers](https://pdfs.semanticscholar.org/5e03/0f03d4a043552ed8aa827098f33f4530e37e.pdf)（**后续会翻译此paper**）

### 基于bthread的leader/followers
#### 实现
1. brpc中实现了M:N的线程库，即为btread模式，在bthread基础之上实现了leader/followers，详情请查阅[thread_overview](https://pdfs.semanticscholar.org/5e03/0f03d4a043552ed8aa827098f33f4530e37e.pdf) 以及 [brpc io](https://pdfs.semanticscholar.org/5e03/0f03d4a043552ed8aa827098f33f4530e37e.pdf)