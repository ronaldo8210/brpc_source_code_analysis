[bthread粒度挂起与唤醒的设计原理](#bthread粒度挂起与唤醒的设计原理)

[brpc中Butex的源码解释](#brpc中Butex的源码解释)

## bthread粒度挂起与唤醒的设计原理
由于bthread任务是在pthread系统线程中执行，在需要bthread间互斥的场景下不能使用pthread级别的锁（如pthread_mutex_lock或者C++的unique_lock等），否则pthread会被挂起，不仅当前的bthread中止执行，pthread私有的TaskGroup的任务队列中其他bthread也无法在该pthread上调度执行。因此需要在应用层实现bthread粒度的互斥机制，一个bthread被挂起时，pthread仍然要保持运行状态，保证TaskGroup任务队列中的其他bthread的正常执行不受影响。

要实现bthread粒度的互斥，方案如下：

1. 在同一个pthread上执行的多个bthread是串行执行的，不需要考虑互斥；

2. 如果位于heap内存上或static静态区上的一个对象A可能会被在不同pthread执行的多个bthread同时访问，则为对象A维护一个锁（一般是一个原子变量）和等待队列，同时访问对象A的多个bthread首先要竞争锁（一般是一个原子变量），假设三个bthread 1、2、3分别在pthread 1、2、3上执行，bthread 1、bthread 2、bthread 3同时访问heap内存上的一个对象A，这时就产生了竞态，假设bthread 1获取到锁，可以去访问对象A，bthread 2、bthread 3先将自身必要的信息（bthread id等）存入等待队列，然后自动yiled，让出cpu，让pthread 2、pthread 3继续去执行各自私有TaskGroup的任务队列中的下一个bthread，这就实现了bthread 2、bthread 3的挂起；

3. 

下面分析下brpc是如何实现bthread粒度的挂起与唤醒的。

## brpc中Butex的源码解释
brpc实现bthread互斥的主要代码在src/bthread/butex.cpp中，
