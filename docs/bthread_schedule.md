[调度执行bthread的主要数据结构](#调度执行bthread的主要数据结构)

[一个pthread调度执行私有TaskGroup的任务队列中各个bthread的过程](#一个pthread调度执行私有TaskGroup的任务队列中各个bthread的过程)

[多核环境下M个bthead在N个pthread上调度执行的具体过程](#多核环境下M个bthead在N个pthread上调度执行的具体过程)

## 调度执行bthread的主要数据结构
在一个线上环境系统中，会产生大量的bthread，系统的cpu核数有限，如何让大量的bthread在有限的cpu核心上得到充分调度执行，实现全局的最大并发主要是由TaskGroup对象、TaskControl对象实现的。

1. 每一个TaskGroup对象是系统线程pthread的线程私有对象，它内部包含有任务队列，并控制pthread如何执行任务队列中的众多bthread任务。TaskGroup中主要的成员有：

   - _remote_rq：如果一个pthread 1想让pthread 2执行bthread 1，则pthread 1会将bthread 1的id压入pthread 2的TaskGroup的_remote_rq队列中。
   
   - _rq：pthread 1在执行从自己私有的TaskGroup中取出的bthread 1时，如果bthread 1执行过程中又创建了新的bthread 2，则bthread 1将bthread 2的id压入pthread 1的TaskGroup的_rq队列中。pthread从其他pthread的TaskGroup中steal来的bthread的id也压入自己私有的TaskGroup的_rq中。

   - _main_tid&_main_stack：一个pthread会在TaskGroup::run_main_task()中执行while()循环，不断获取并执行bthread任务，一个pthread的执行流不是永远在bthread中，比如等待任务时，pthread没有执行任何bthread，执行流就是直接在pthread上。可以将pthread在“等待bthread-获取到bthread-进入bthread执行任务函数之前”这个过程也抽象成一个bthread，称作一个pthread的“调度bthread”或者“主bthread”，它的tid和私有栈就是_main_tid和_main_stack。
   
   - _cur_meta：当前正在执行的bthread的TaskMeta对象的地址。
   
2. 

## 一个pthread调度执行私有TaskGroup的任务队列中各个bthread的过程
一个pthread调度执行私有TaskGroup任务队列中的各个bthread，这些bthread是在pthread上串行执行的，彼此间不会有竞争。一个bthread的执行过程可能会有三种状态：

1. bthread的任务处理函数执行完成。一个bthread的任务函数结束后，该bthread需要负责查看TaskGroup的任务队列中是否还有bthread，如果有，则pthread执行流直接进入下一个bthread的任务函数中去执行；如果没有，则执行流返回pthread的调度bthread，等待其他pthread传递新的bthread；

2. bthread在任务函数执行过程中yield挂起，则pthread去执行任务队列中下一个bthread，如果任务队列为空，则执行流返回pthread的调度bthread，等待其他pthread传递新的bthread。挂起的bthread何时恢复运行取决于具体的业务场景，它应该被某个bthread唤醒，与pthread的调度无关。这样的例子有负责向TCP连接写数据的bthread因等待inode输出缓冲可写而被yield挂起、等待Butex互斥锁的bthread被yield挂起等。

3. bthread在任务函数执行过程中可以创建新的bthread，因为新的bthread一般是优先级更高的bthread，所以pthread执行流立即进入新bthread的任务函数，原先的bthread被重新加入到任务队列的尾部，不久后它仍然可以被pthread执行。但由于work-steal机制，它不一定会在原先的pthread执行，可能会被steal到其他pthread上执行。

按照以上的原则，分析下brpc中的实现过程。



## 多核环境下M个bthead在N个pthread上调度执行的具体过程


调度顺序：调度过程 bthread A 调度过程 bthread B

1. bthread可能直接执行完
2. bthread可能yield让出cpu
3. bthread可能新建bthread

