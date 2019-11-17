[多线程下内存分配与回收的设计原则](#多线程下内存分配与回收的设计原则)

[ResourcePool的内存布局](#ResourcePool的内存布局)

[ResourcePool的源码解析](#ResourcePool的源码解析)

[多线程下内存分配与回收的具体示例](#多线程下内存分配与回收的具体示例)

## 多线程下内存分配与回收的设计原则
多线程高度竞争环境下的内存分配，需要考虑两个主要问题：

1. 如何尽可能地避免内存分配过程和回收过程的线程间的竞争。brpc的ResourcePool的设计方案是：

   - 为每个线程维护一个私有的内存区和一个空闲对象列表；
   
   - 当一个线程需要为某个新建对象分配内存时，先假设此时线程私有的、以及全局的空闲对象列表都为空，线程会从自己的私有内存区上分配，私有内存区用满后，从全局内存区再次申请一块内存作为新的私有内存区。所以为新建对象分配内存的动作不存在线程间的竞争，不需要用锁。只有从全局内存区申请一块线程私有内存时可能存在竞态，需要加锁；
   
   - 当一个对象不再被使用时，不析构该对象，不释放对象占用的内存，只是将对象的唯一id（不仅是对象的唯一标识id，而且通过此id能在O(1)时间内定位到对象的内存地址，见下文描述）存入线程的私有空闲对象列表。当私有空闲对象列表已满后，将已满的空闲对象列表拷贝一份，将拷贝列表压入全局的空闲列表队列，并清空线程私有的空闲对象列表。所以回收对象内存的动作不存在线程间的竞争，不需要用锁，只有将已满的空闲对象列表压入全局的空闲列表队列时需要加锁。线程为新建对象分配内存时，首先查看私有的空闲对象列表，如果不为空，则取出一个对象id，通过id找到一个之前已回收的对象，重置对象状态，作为新对象使用；如果私有空闲对象列表为空，则再尝试从全局的空闲列表队列中弹出一个空闲列表，将列表内容拷贝到私有空闲对象列表，再从私有空闲对象列表拿出一个已回收对象的id；如果全局的空闲列表队列也为空，则再从线程私有内存区上分配一块内存；
   
   - brpc的ResourcePool的分配、回收对象内存的方式不解决ABA问题，例如，线程A使用对象obj后将该对象回收，一段时间后线程B拿到了obj，重置obj的状态后作为自己的对象使用，此时线程A用obj的id仍然能访问到obj，但obj的所有权已不属于线程A了，线程A若再操作obj会导致程序混乱。所以需要在应用程序中使用版本号等方法防止内存分配回收的ABA问题。

2. 如何避免内存碎片。brpc的ResourcePool为每一个类型的对象（TaskMeta、Socket、Id等）单独建立一个全局的singleton内存区，每个singleton内存区上分配的对象都是等长的，所以分配、回收过程不会有内存碎片。

## ResourcePool的内存布局

一个ResourcePool单例对象表示某一个类型的对象的全局singleton内存区。先解释下ResourcePool中的成员变量和几个宏所表达的意义：

1. RP_MAX_BLOCK_NGROUP表示一个ResourcePool中BlockGroup的数量；RP_GROUP_NBLOCK表示一个BlockGroup中的Block*的数量；

2. ResourcePool是模板类，模板参数是对象类型，主要的成员变量：

   - _local_pool：一个ResourcePool单例上，每个线程有各自私有的LocalPool对象，_local_pool是私有LocalPool对象的指针。LocalPool类的成员有：
   
     - _cur_block：从ResourcePool单例的全局内存区申请到的一块Block的指针。
     
     - _cur_block_index：_cur_block指向的Block在ResourcePool单例的全局内存区中的Block索引号。
     
     - _cur_free：LocalPool对象中未存满被回收对象id的空闲对象列表。
   
   - _nlocal：一个ResourcePool单例上，LocalPool对象的数量。
   
   - _block_groups：一个ResourcePool单例上，BlockGroup指针的数组。一个BlockGroup中含有RP_GROUP_NBLOCK个Block的指针。一个Block中含有若干个对象。
   
   - _ngroup：一个ResourcePool单例上，BlockGroup的数量。
   
   - _free_chunks：一个ResourcePool单例上，已经存满被回收对象id的空闲列表的队列。
   
一个ResourcePool对象的内存布局如下图表示：

<img src="../images/.png" width="100%" height="100%"/>

## ResourcePool的源码解析
为对象分配内存和回收内存的主要代码都在ResourcePool类中。

1. 针对一种类型的对象，获取全局的ResourcePool的单例的接口函数为ResourcePool::singleton()，该函数可被多个线程同时执行，要注意代码中的double check逻辑判断：

   ```c++
    static inline ResourcePool* singleton() {
        // 如果当前_singleton指针不为空，则之前已经有线程为_singleton赋过值，直接返回非空的_singleton值即可。
        ResourcePool* p = _singleton.load(butil::memory_order_consume);
        if (p) {
            return p;
        }
        // 修改_singleton的代码可被多个线程同时执行，必须先加锁。
        pthread_mutex_lock(&_singleton_mutex);
        // double check，再次检查_singleton指针是否为空。
        // 因为可能有两个线程同时进入ResourcePool::singleton()函数，同时检测到_singleton值为空，
        // 接着同时执行到pthread_mutex_lock(&_singleton_mutex)，但只能有一个线程（A）执行_singleton.store()，
        // 另一个线程（B）必须等待。线程A执行pthread_mutex_unlock(&_singleton_mutex)后，线程B恢复执行，必须再次
        // 判断_singleton是否为空，因为_singleton之前已经被线程A赋了值，线程B不能再次给_singleton赋值。
        p = _singleton.load(butil::memory_order_consume);
        if (!p) {
            // 创建一个新的ResourcePool对象，对象指针赋给_singleton。
            p = new ResourcePool();
            _singleton.store(p, butil::memory_order_release);
        } 
        pthread_mutex_unlock(&_singleton_mutex);
        return p;
    }
   ```

2. 为一个对象分配内存的接口函数为

## 多线程下内存分配与回收的具体示例



柔性数组
分配pool时需要按cacheline对齐  考虑局部性原理

double check的几处使用

ResourcePoolBlockItemNum  BLOCK_NITEM   一个block里面容纳Item的数量
FREE_CHUNK_NITEM=BLOCK_NITEM

一个Block  64KB

RP_GROUP_NBLOCK  一个BlockGroup中的Block*的数量

fetch_add  返回原子对象的旧值

_cur_block_index  block在全局中的索引号

