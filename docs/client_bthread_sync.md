### 概述

在一次RPC过程中，由于设置超时定时器和开启Backup Request机制，不同的bthread可能会同时操作该次RPC独有的Controller结构，会存在下列几种竞态情况：
* 第一次Request发出后，在backup_request_ms内未收到响应，触发Backup Request定时器，定时任务执行的同时可能收到了第一次Request的Response，处理定时任务的bthread和处理Response的bthread需要做互斥
* 第一次Request和Backup Request可能同时收到Response，分别处理两个Response的bthread间需要做互斥

brpc中

### 内存布局

一次RPC过程中，Id、Controller、Butex的内存布局如下图所示：

![img](../images/client_bthread_sync_1.png)

Id结构主要字段意义：
first_ver：如果Butex结构的value值为first_ver，则表示当前没有bthread在访问Controller结构
locked_ver：如果Butex结构的value值被设为locked_ver，则表示当前已有一个bthread在操作Controller
mutex：类似futex的线程锁，由于试图操作同一Controller的若干bthread可能在不同的系统线程pthread上被执行，所以同时访问Controller时需要先做pthread间的互斥
data：指向Controller的指针


主要代码在src/bthread/id.cpp中，解释下几个主要的函数的作用：

### 执行时序示例

假设有三个bthread A、B、C（位于三个不同TaskGroup的可执行任务队列中，三个TaskGroup分别是三个pthread的线程私有对象）同时访问Controller，一个可能的执行时序如下：
T1时刻：A、B、C三个bthread同时执行到

### 具体实例

